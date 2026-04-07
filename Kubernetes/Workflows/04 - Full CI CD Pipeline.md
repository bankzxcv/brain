---
title: "K8s Workflow: Full CI/CD Pipeline"
date: 2026-04-07
tags:
  - kubernetes
  - workflow
  - project
  - github-actions
  - ci-cd
  - argocd
  - helm
  - gitops
parent: "[[Kubernetes Study]]"
---

# Workflow 04: Full CI/CD Pipeline

End-to-end CI/CD pipeline: GitHub Actions builds and pushes Docker images, updates the GitOps repo, and ArgoCD deploys to Kubernetes. This workflow ties together Docker, Helm, ArgoCD, and GitHub Actions into a production-style pipeline.

## Architecture Overview

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                        CI/CD Pipeline Flow                          │
  └──────────────────────────────────────────────────────────────────────┘

  app-repo (source code)              gitops-repo (K8s config)
  ┌─────────────────────┐             ┌─────────────────────┐
  │  main.go             │             │  helm/               │
  │  Dockerfile          │             │    values-dev.yaml   │
  │  go.mod              │             │    values-staging.yaml│
  │  .github/workflows/  │             │    values-prod.yaml  │
  │    ci.yaml           │             │  apps/               │
  │    cd.yaml           │             │    argocd-app.yaml   │
  └──────┬──────────────┘             └──────┬──────────────┘
         │                                     │
         │ 1. Push code                        │ 4. Update image tag
         ▼                                     ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │                     GitHub Actions                                │
  │                                                                   │
  │  CI Workflow:                    CD Workflow:                      │
  │  ┌──────┐ ┌──────┐ ┌───────┐   ┌──────────────────────┐         │
  │  │ Test │►│ Lint │►│ Build │   │ Update gitops-repo   │         │
  │  └──────┘ └──────┘ └───┬───┘   │ with new image tag   │         │
  │                         │       └──────────┬───────────┘         │
  │                    2. Push                  │                      │
  │                    image                    │ 5. Commit + push     │
  │                         │                   │                      │
  │                         ▼                   ▼                      │
  │                    ghcr.io          gitops-repo (updated)          │
  └──────────────────────────────────────────────────────────────────┘
                                                │
                                                │ 6. ArgoCD detects change
                                                ▼
                                         ┌──────────────┐
                                         │   ArgoCD     │
                                         │              │
                                         │  auto-sync   │
                                         └──────┬───────┘
                                                │
                                                │ 7. Deploy
                                                ▼
                                         ┌──────────────┐
                                         │  Kubernetes  │
                                         │  (minikube)  │
                                         └──────────────┘
```

## Repository Structure

### app-repo (Source Code)

```
app-repo/
├── main.go
├── main_test.go
├── go.mod
├── go.sum
├── Dockerfile
└── .github/
    └── workflows/
        ├── ci.yaml          # Test, lint, build, push image
        └── cd.yaml          # Update gitops repo with new tag
```

### gitops-repo (Kubernetes Configuration)

```
gitops-repo/
├── helm/
│   └── myapp/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-staging.yaml
│       ├── values-prod.yaml
│       └── templates/
│           ├── _helpers.tpl
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── ingress.yaml
│           ├── configmap.yaml
│           └── secret.yaml
├── apps/
│   ├── dev.yaml             # ArgoCD Application for dev
│   ├── staging.yaml         # ArgoCD Application for staging
│   └── production.yaml      # ArgoCD Application for production
└── root-app.yaml            # App of Apps
```

## Prerequisites

- Two GitHub repositories (or one monorepo with two directories)
- GitHub Container Registry (ghcr.io) access
- ArgoCD running on minikube (from [[03 - GitOps Pipeline]])
- Helm 3 installed

---

## Step 1: Sample Go Application (app-repo)

> [!success]- Full Solution: `app-repo/main.go`
> ```go
> package main
> 
> import (
> 	"database/sql"
> 	"encoding/json"
> 	"fmt"
> 	"log"
> 	"net/http"
> 	"os"
> 
> 	_ "github.com/lib/pq"
> )
> 
> var version = "dev"  // Set at build time via -ldflags
> 
> type HealthResponse struct {
> 	Status  string `json:"status"`
> 	Version string `json:"version"`
> 	Host    string `json:"host"`
> }
> 
> type Item struct {
> 	ID   int    `json:"id"`
> 	Name string `json:"name"`
> }
> 
> var db *sql.DB
> 
> func main() {
> 	dbHost := getEnv("DB_HOST", "localhost")
> 	dbPort := getEnv("DB_PORT", "5432")
> 	dbUser := getEnv("DB_USER", "appuser")
> 	dbPassword := getEnv("DB_PASSWORD", "apppassword")
> 	dbName := getEnv("DB_NAME", "appdb")
> 	port := getEnv("PORT", "8080")
> 
> 	connStr := fmt.Sprintf(
> 		"host=%s port=%s user=%s password=%s dbname=%s sslmode=disable",
> 		dbHost, dbPort, dbUser, dbPassword, dbName,
> 	)
> 
> 	var err error
> 	db, err = sql.Open("postgres", connStr)
> 	if err != nil {
> 		log.Printf("Warning: DB connection failed: %v", err)
> 	} else {
> 		db.Exec(`CREATE TABLE IF NOT EXISTS items (
> 			id SERIAL PRIMARY KEY,
> 			name VARCHAR(255) NOT NULL
> 		)`)
> 	}
> 
> 	http.HandleFunc("/api/health", healthHandler)
> 	http.HandleFunc("/api/ready", readyHandler)
> 	http.HandleFunc("/api/items", itemsHandler)
> 	http.HandleFunc("/api/version", versionHandler)
> 
> 	log.Printf("Server v%s starting on port %s", version, port)
> 	log.Fatal(http.ListenAndServe(":"+port, nil))
> }
> 
> func healthHandler(w http.ResponseWriter, r *http.Request) {
> 	hostname, _ := os.Hostname()
> 	w.Header().Set("Content-Type", "application/json")
> 	json.NewEncoder(w).Encode(HealthResponse{
> 		Status:  "ok",
> 		Version: version,
> 		Host:    hostname,
> 	})
> }
> 
> func readyHandler(w http.ResponseWriter, r *http.Request) {
> 	if db == nil || db.Ping() != nil {
> 		w.WriteHeader(http.StatusServiceUnavailable)
> 		json.NewEncoder(w).Encode(map[string]string{"status": "not ready"})
> 		return
> 	}
> 	json.NewEncoder(w).Encode(map[string]string{"status": "ready"})
> }
> 
> func versionHandler(w http.ResponseWriter, r *http.Request) {
> 	hostname, _ := os.Hostname()
> 	json.NewEncoder(w).Encode(map[string]string{
> 		"version":  version,
> 		"hostname": hostname,
> 	})
> }
> 
> func itemsHandler(w http.ResponseWriter, r *http.Request) {
> 	w.Header().Set("Content-Type", "application/json")
> 	switch r.Method {
> 	case http.MethodGet:
> 		rows, err := db.Query("SELECT id, name FROM items")
> 		if err != nil {
> 			http.Error(w, err.Error(), 500)
> 			return
> 		}
> 		defer rows.Close()
> 		items := []Item{}
> 		for rows.Next() {
> 			var item Item
> 			rows.Scan(&item.ID, &item.Name)
> 			items = append(items, item)
> 		}
> 		json.NewEncoder(w).Encode(items)
> 	case http.MethodPost:
> 		var item Item
> 		json.NewDecoder(r.Body).Decode(&item)
> 		db.QueryRow("INSERT INTO items (name) VALUES ($1) RETURNING id", item.Name).Scan(&item.ID)
> 		w.WriteHeader(http.StatusCreated)
> 		json.NewEncoder(w).Encode(item)
> 	}
> }
> 
> func getEnv(key, fallback string) string {
> 	if v, ok := os.LookupEnv(key); ok {
> 		return v
> 	}
> 	return fallback
> }
> ```

> [!success]- Full Solution: `app-repo/main_test.go`
> ```go
> package main
> 
> import (
> 	"encoding/json"
> 	"net/http"
> 	"net/http/httptest"
> 	"testing"
> )
> 
> func TestHealthHandler(t *testing.T) {
> 	req := httptest.NewRequest(http.MethodGet, "/api/health", nil)
> 	w := httptest.NewRecorder()
> 
> 	healthHandler(w, req)
> 
> 	if w.Code != http.StatusOK {
> 		t.Errorf("expected status 200, got %d", w.Code)
> 	}
> 
> 	var resp HealthResponse
> 	json.NewDecoder(w.Body).Decode(&resp)
> 
> 	if resp.Status != "ok" {
> 		t.Errorf("expected status 'ok', got '%s'", resp.Status)
> 	}
> }
> 
> func TestVersionHandler(t *testing.T) {
> 	req := httptest.NewRequest(http.MethodGet, "/api/version", nil)
> 	w := httptest.NewRecorder()
> 
> 	versionHandler(w, req)
> 
> 	if w.Code != http.StatusOK {
> 		t.Errorf("expected status 200, got %d", w.Code)
> 	}
> 
> 	var resp map[string]string
> 	json.NewDecoder(w.Body).Decode(&resp)
> 
> 	if resp["version"] != version {
> 		t.Errorf("expected version '%s', got '%s'", version, resp["version"])
> 	}
> }
> ```

> [!success]- Full Solution: `app-repo/go.mod`
> ```
> module github.com/YOUR_USER/app-repo
> 
> go 1.21
> 
> require github.com/lib/pq v1.10.9
> ```

> [!success]- Full Solution: `app-repo/Dockerfile`
> ```dockerfile
> # ---- Build Stage ----
> FROM golang:1.21-alpine AS builder
> 
> ARG VERSION=dev
> 
> WORKDIR /app
> COPY go.mod go.sum ./
> RUN go mod download
> 
> COPY . .
> RUN CGO_ENABLED=0 GOOS=linux go build \
>     -ldflags "-X main.version=${VERSION}" \
>     -o /server main.go
> 
> # ---- Runtime Stage ----
> FROM alpine:3.19
> 
> RUN apk --no-cache add ca-certificates
> RUN adduser -D -u 1000 appuser
> 
> COPY --from=builder /server /server
> 
> USER appuser
> EXPOSE 8080
> 
> ENTRYPOINT ["/server"]
> ```

---

## Step 2: GitHub Actions CI Workflow

> [!success]- Full Solution: `app-repo/.github/workflows/ci.yaml`
> ```yaml
> name: CI Pipeline
> 
> on:
>   push:
>     branches: [main]
>   pull_request:
>     branches: [main]
> 
> env:
>   REGISTRY: ghcr.io
>   IMAGE_NAME: ${{ github.repository }}
> 
> jobs:
>   test:
>     name: Test
>     runs-on: ubuntu-latest
>     steps:
>       - uses: actions/checkout@v4
> 
>       - uses: actions/setup-go@v5
>         with:
>           go-version: '1.21'
> 
>       - name: Run tests
>         run: go test -v -race -coverprofile=coverage.out ./...
> 
>       - name: Upload coverage
>         uses: actions/upload-artifact@v4
>         with:
>           name: coverage
>           path: coverage.out
> 
>   lint:
>     name: Lint
>     runs-on: ubuntu-latest
>     steps:
>       - uses: actions/checkout@v4
> 
>       - uses: actions/setup-go@v5
>         with:
>           go-version: '1.21'
> 
>       - name: golangci-lint
>         uses: golangci/golangci-lint-action@v4
>         with:
>           version: latest
> 
>   build-and-push:
>     name: Build and Push Image
>     runs-on: ubuntu-latest
>     needs: [test, lint]
>     # Only build on push to main, not on PRs
>     if: github.event_name == 'push' && github.ref == 'refs/heads/main'
>     permissions:
>       contents: read
>       packages: write
> 
>     outputs:
>       image_tag: ${{ steps.meta.outputs.version }}
>       image_digest: ${{ steps.build.outputs.digest }}
> 
>     steps:
>       - uses: actions/checkout@v4
> 
>       - name: Log in to Container Registry
>         uses: docker/login-action@v3
>         with:
>           registry: ${{ env.REGISTRY }}
>           username: ${{ github.actor }}
>           password: ${{ secrets.GITHUB_TOKEN }}
> 
>       - name: Extract metadata
>         id: meta
>         uses: docker/metadata-action@v5
>         with:
>           images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
>           tags: |
>             type=sha,prefix=
>             type=ref,event=branch
>             type=semver,pattern={{version}}
> 
>       - name: Build and push
>         id: build
>         uses: docker/build-push-action@v5
>         with:
>           context: .
>           push: true
>           tags: ${{ steps.meta.outputs.tags }}
>           labels: ${{ steps.meta.outputs.labels }}
>           build-args: |
>             VERSION=${{ github.sha }}
> 
>   # Trigger CD workflow after successful build
>   trigger-cd:
>     name: Trigger CD
>     runs-on: ubuntu-latest
>     needs: [build-and-push]
>     steps:
>       - name: Trigger CD workflow
>         uses: peter-evans/repository-dispatch@v3
>         with:
>           token: ${{ secrets.GITOPS_PAT }}
>           repository: YOUR_USER/gitops-repo
>           event-type: deploy
>           client-payload: |
>             {
>               "image_tag": "${{ needs.build-and-push.outputs.image_tag }}",
>               "image": "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}",
>               "sha": "${{ github.sha }}",
>               "ref": "${{ github.ref }}",
>               "actor": "${{ github.actor }}"
>             }
> ```

> [!warning] Required Secrets
> - `GITHUB_TOKEN`: Automatically available, used for ghcr.io push
> - `GITOPS_PAT`: A Personal Access Token with `repo` scope for the gitops-repo. Create this in GitHub Settings > Developer Settings > Personal Access Tokens and add it as a secret in the app-repo.

---

## Step 3: GitHub Actions CD Workflow (in gitops-repo)

> [!success]- Full Solution: `gitops-repo/.github/workflows/cd.yaml`
> ```yaml
> name: CD Pipeline
> 
> on:
>   repository_dispatch:
>     types: [deploy]
> 
> jobs:
>   update-dev:
>     name: Update Dev Environment
>     runs-on: ubuntu-latest
>     steps:
>       - uses: actions/checkout@v4
>         with:
>           token: ${{ secrets.GITOPS_PAT }}
> 
>       - name: Update image tag in dev values
>         run: |
>           IMAGE_TAG="${{ github.event.client_payload.image_tag }}"
>           IMAGE="${{ github.event.client_payload.image }}"
>           SHA="${{ github.event.client_payload.sha }}"
> 
>           echo "Updating dev to image: ${IMAGE}:${IMAGE_TAG}"
> 
>           # Update the image tag in values-dev.yaml using yq
>           yq eval ".backend.image.repository = \"${IMAGE}\"" -i helm/myapp/values-dev.yaml
>           yq eval ".backend.image.tag = \"${IMAGE_TAG}\"" -i helm/myapp/values-dev.yaml
> 
>       - name: Commit and push
>         run: |
>           git config user.name "github-actions[bot]"
>           git config user.email "github-actions[bot]@users.noreply.github.com"
> 
>           git add helm/myapp/values-dev.yaml
>           git commit -m "deploy(dev): update backend to ${{ github.event.client_payload.image_tag }}
> 
>           Source commit: ${{ github.event.client_payload.sha }}
>           Triggered by: ${{ github.event.client_payload.actor }}"
> 
>           git push
> 
>   update-staging:
>     name: Update Staging Environment
>     runs-on: ubuntu-latest
>     needs: [update-dev]
>     environment: staging   # Requires manual approval in GitHub
>     steps:
>       - uses: actions/checkout@v4
>         with:
>           token: ${{ secrets.GITOPS_PAT }}
>           ref: main  # Pull latest (dev update already merged)
> 
>       - name: Update image tag in staging values
>         run: |
>           IMAGE_TAG="${{ github.event.client_payload.image_tag }}"
>           IMAGE="${{ github.event.client_payload.image }}"
> 
>           yq eval ".backend.image.repository = \"${IMAGE}\"" -i helm/myapp/values-staging.yaml
>           yq eval ".backend.image.tag = \"${IMAGE_TAG}\"" -i helm/myapp/values-staging.yaml
> 
>       - name: Commit and push
>         run: |
>           git config user.name "github-actions[bot]"
>           git config user.email "github-actions[bot]@users.noreply.github.com"
>           git pull origin main
>           git add helm/myapp/values-staging.yaml
>           git commit -m "deploy(staging): update backend to ${{ github.event.client_payload.image_tag }}"
>           git push
> 
>   update-production:
>     name: Update Production Environment
>     runs-on: ubuntu-latest
>     needs: [update-staging]
>     environment: production   # Requires manual approval in GitHub
>     steps:
>       - uses: actions/checkout@v4
>         with:
>           token: ${{ secrets.GITOPS_PAT }}
>           ref: main
> 
>       - name: Update image tag in production values
>         run: |
>           IMAGE_TAG="${{ github.event.client_payload.image_tag }}"
>           IMAGE="${{ github.event.client_payload.image }}"
> 
>           yq eval ".backend.image.repository = \"${IMAGE}\"" -i helm/myapp/values-prod.yaml
>           yq eval ".backend.image.tag = \"${IMAGE_TAG}\"" -i helm/myapp/values-prod.yaml
> 
>       - name: Commit and push
>         run: |
>           git config user.name "github-actions[bot]"
>           git config user.email "github-actions[bot]@users.noreply.github.com"
>           git pull origin main
>           git add helm/myapp/values-prod.yaml
>           git commit -m "deploy(production): update backend to ${{ github.event.client_payload.image_tag }}"
>           git push
> ```

> [!info] GitHub Environments
> The `environment: staging` and `environment: production` fields reference GitHub Environments configured in the repo settings. You can add required reviewers, wait timers, and branch restrictions to control deployments.

---

## Step 4: Helm Chart in GitOps Repo

> [!success]- Full Solution: `gitops-repo/helm/myapp/Chart.yaml`
> ```yaml
> apiVersion: v2
> name: myapp
> description: Microservice application deployed via GitOps
> type: application
> version: 0.1.0
> appVersion: "1.0.0"
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/values.yaml`
> ```yaml
> backend:
>   replicaCount: 1
>   image:
>     repository: ghcr.io/YOUR_USER/app-repo
>     tag: "latest"
>     pullPolicy: IfNotPresent
>   service:
>     type: ClusterIP
>     port: 8080
>   resources:
>     requests:
>       memory: "64Mi"
>       cpu: "100m"
>     limits:
>       memory: "128Mi"
>       cpu: "250m"
>   env:
>     DB_HOST: "postgres-svc"
>     DB_PORT: "5432"
>     DB_NAME: "appdb"
>     DB_USER: "appuser"
>     LOG_LEVEL: "info"
> 
> frontend:
>   replicaCount: 1
>   image:
>     repository: myapp-frontend
>     tag: "v1"
>     pullPolicy: Never
>   service:
>     type: ClusterIP
>     port: 80
> 
> ingress:
>   enabled: true
>   className: nginx
>   host: app.local
> 
> database:
>   enabled: true
>   image: postgres:16-alpine
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/values-dev.yaml`
> ```yaml
> backend:
>   replicaCount: 1
>   image:
>     repository: ghcr.io/YOUR_USER/app-repo
>     tag: "latest"   # Updated by CD pipeline
>   env:
>     LOG_LEVEL: "debug"
> 
> frontend:
>   replicaCount: 1
> 
> ingress:
>   host: dev.app.local
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/values-staging.yaml`
> ```yaml
> backend:
>   replicaCount: 2
>   image:
>     repository: ghcr.io/YOUR_USER/app-repo
>     tag: "latest"   # Updated by CD pipeline
>   env:
>     LOG_LEVEL: "info"
>   resources:
>     requests:
>       memory: "128Mi"
>       cpu: "200m"
>     limits:
>       memory: "256Mi"
>       cpu: "500m"
> 
> frontend:
>   replicaCount: 2
> 
> ingress:
>   host: staging.app.local
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/values-prod.yaml`
> ```yaml
> backend:
>   replicaCount: 3
>   image:
>     repository: ghcr.io/YOUR_USER/app-repo
>     tag: "latest"   # Updated by CD pipeline
>   env:
>     LOG_LEVEL: "warn"
>   resources:
>     requests:
>       memory: "256Mi"
>       cpu: "500m"
>     limits:
>       memory: "512Mi"
>       cpu: "1000m"
> 
> frontend:
>   replicaCount: 3
> 
> ingress:
>   host: app.example.com
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/templates/_helpers.tpl`
> ```
> {{- define "myapp.name" -}}
> {{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
> {{- end }}
> 
> {{- define "myapp.fullname" -}}
> {{- if .Values.fullnameOverride }}
> {{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
> {{- else }}
> {{- $name := default .Chart.Name .Values.nameOverride }}
> {{- if contains $name .Release.Name }}
> {{- .Release.Name | trunc 63 | trimSuffix "-" }}
> {{- else }}
> {{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
> {{- end }}
> {{- end }}
> {{- end }}
> 
> {{- define "myapp.labels" -}}
> helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
> app.kubernetes.io/managed-by: {{ .Release.Service }}
> app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
> {{- end }}
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/templates/deployment.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: {{ include "myapp.fullname" . }}-backend
>   labels:
>     app: backend
>     {{- include "myapp.labels" . | nindent 4 }}
> spec:
>   replicas: {{ .Values.backend.replicaCount }}
>   selector:
>     matchLabels:
>       app: backend
>   template:
>     metadata:
>       labels:
>         app: backend
>     spec:
>       containers:
>         - name: backend
>           image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
>           imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
>           ports:
>             - containerPort: {{ .Values.backend.service.port }}
>           env:
>             {{- range $key, $value := .Values.backend.env }}
>             - name: {{ $key }}
>               value: {{ $value | quote }}
>             {{- end }}
>             - name: DB_PASSWORD
>               valueFrom:
>                 secretKeyRef:
>                   name: db-secret
>                   key: DB_PASSWORD
>           livenessProbe:
>             httpGet:
>               path: /api/health
>               port: {{ .Values.backend.service.port }}
>             initialDelaySeconds: 5
>             periodSeconds: 10
>           readinessProbe:
>             httpGet:
>               path: /api/ready
>               port: {{ .Values.backend.service.port }}
>             initialDelaySeconds: 5
>             periodSeconds: 5
>           resources:
>             {{- toYaml .Values.backend.resources | nindent 12 }}
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: {{ include "myapp.fullname" . }}-frontend
>   labels:
>     app: frontend
>     {{- include "myapp.labels" . | nindent 4 }}
> spec:
>   replicas: {{ .Values.frontend.replicaCount }}
>   selector:
>     matchLabels:
>       app: frontend
>   template:
>     metadata:
>       labels:
>         app: frontend
>     spec:
>       containers:
>         - name: frontend
>           image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
>           imagePullPolicy: {{ .Values.frontend.image.pullPolicy }}
>           ports:
>             - containerPort: {{ .Values.frontend.service.port }}
>           livenessProbe:
>             httpGet:
>               path: /healthz
>               port: {{ .Values.frontend.service.port }}
>             initialDelaySeconds: 3
>             periodSeconds: 10
>           readinessProbe:
>             httpGet:
>               path: /healthz
>               port: {{ .Values.frontend.service.port }}
>             initialDelaySeconds: 3
>             periodSeconds: 5
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/templates/service.yaml`
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: {{ include "myapp.fullname" . }}-backend
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
> spec:
>   type: {{ .Values.backend.service.type }}
>   selector:
>     app: backend
>   ports:
>     - port: {{ .Values.backend.service.port }}
>       targetPort: {{ .Values.backend.service.port }}
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: {{ include "myapp.fullname" . }}-frontend
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
> spec:
>   type: {{ .Values.frontend.service.type }}
>   selector:
>     app: frontend
>   ports:
>     - port: {{ .Values.frontend.service.port }}
>       targetPort: {{ .Values.frontend.service.port }}
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/templates/ingress.yaml`
> ```yaml
> {{- if .Values.ingress.enabled }}
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: {{ include "myapp.fullname" . }}-ingress
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
>   annotations:
>     nginx.ingress.kubernetes.io/rewrite-target: /$2
> spec:
>   ingressClassName: {{ .Values.ingress.className }}
>   rules:
>     - host: {{ .Values.ingress.host }}
>       http:
>         paths:
>           - path: /api(/|$)(.*)
>             pathType: ImplementationSpecific
>             backend:
>               service:
>                 name: {{ include "myapp.fullname" . }}-backend
>                 port:
>                   number: {{ .Values.backend.service.port }}
>           - path: /()(.*)
>             pathType: ImplementationSpecific
>             backend:
>               service:
>                 name: {{ include "myapp.fullname" . }}-frontend
>                 port:
>                   number: {{ .Values.frontend.service.port }}
> {{- end }}
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/templates/configmap.yaml`
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: {{ include "myapp.fullname" . }}-config
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
> data:
>   {{- range $key, $value := .Values.backend.env }}
>   {{ $key }}: {{ $value | quote }}
>   {{- end }}
> ```

> [!success]- Full Solution: `gitops-repo/helm/myapp/templates/secret.yaml`
> ```yaml
> apiVersion: v1
> kind: Secret
> metadata:
>   name: db-secret
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
> type: Opaque
> stringData:
>   DB_PASSWORD: "apppassword"
>   POSTGRES_PASSWORD: "apppassword"
> ```

---

## Step 5: ArgoCD Application CRDs

> [!success]- Full Solution: `gitops-repo/apps/dev.yaml`
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: myapp-dev
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/YOUR_USER/gitops-repo.git
>     targetRevision: main
>     path: helm/myapp
>     helm:
>       valueFiles:
>         - values.yaml
>         - values-dev.yaml
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: myapp-dev
>   syncPolicy:
>     automated:
>       prune: true
>       selfHeal: true
>     syncOptions:
>       - CreateNamespace=true
>     retry:
>       limit: 3
>       backoff:
>         duration: 5s
>         factor: 2
>         maxDuration: 3m
> ```

> [!success]- Full Solution: `gitops-repo/apps/staging.yaml`
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: myapp-staging
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/YOUR_USER/gitops-repo.git
>     targetRevision: main
>     path: helm/myapp
>     helm:
>       valueFiles:
>         - values.yaml
>         - values-staging.yaml
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: myapp-staging
>   syncPolicy:
>     automated:
>       prune: true
>       selfHeal: true
>     syncOptions:
>       - CreateNamespace=true
> ```

> [!success]- Full Solution: `gitops-repo/apps/production.yaml`
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: myapp-production
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/YOUR_USER/gitops-repo.git
>     targetRevision: main
>     path: helm/myapp
>     helm:
>       valueFiles:
>         - values.yaml
>         - values-prod.yaml
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: myapp-production
>   syncPolicy:
>     # Production: manual sync only (no automated)
>     syncOptions:
>       - CreateNamespace=true
> ```

> [!success]- Full Solution: `gitops-repo/root-app.yaml`
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: root-app
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/YOUR_USER/gitops-repo.git
>     targetRevision: main
>     path: apps
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: argocd
>   syncPolicy:
>     automated:
>       prune: true
>       selfHeal: true
> ```

> [!note] Production Safety
> Notice that the production Application does **not** have `automated` sync policy. This means you must manually sync production deployments via the ArgoCD UI or CLI after the values file is updated. This is a safety gate.

---

## Step 6: PR-Based Workflow for Preview Environments

> [!success]- Full Solution: `app-repo/.github/workflows/pr-preview.yaml`
> ```yaml
> name: PR Preview Environment
> 
> on:
>   pull_request:
>     types: [opened, synchronize, reopened, closed]
> 
> env:
>   REGISTRY: ghcr.io
>   IMAGE_NAME: ${{ github.repository }}
> 
> jobs:
>   build-preview:
>     name: Build Preview Image
>     runs-on: ubuntu-latest
>     if: github.event.action != 'closed'
>     permissions:
>       contents: read
>       packages: write
>     outputs:
>       image_tag: pr-${{ github.event.number }}
>     steps:
>       - uses: actions/checkout@v4
> 
>       - uses: docker/login-action@v3
>         with:
>           registry: ${{ env.REGISTRY }}
>           username: ${{ github.actor }}
>           password: ${{ secrets.GITHUB_TOKEN }}
> 
>       - name: Build and push preview image
>         uses: docker/build-push-action@v5
>         with:
>           context: .
>           push: true
>           tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:pr-${{ github.event.number }}
>           build-args: |
>             VERSION=pr-${{ github.event.number }}-${{ github.sha }}
> 
>   deploy-preview:
>     name: Deploy Preview
>     runs-on: ubuntu-latest
>     needs: [build-preview]
>     steps:
>       - name: Create preview environment in gitops-repo
>         uses: peter-evans/repository-dispatch@v3
>         with:
>           token: ${{ secrets.GITOPS_PAT }}
>           repository: YOUR_USER/gitops-repo
>           event-type: preview-deploy
>           client-payload: |
>             {
>               "pr_number": "${{ github.event.number }}",
>               "image_tag": "pr-${{ github.event.number }}",
>               "image": "${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}",
>               "action": "deploy"
>             }
> 
>       - name: Comment PR with preview URL
>         uses: peter-evans/create-or-update-comment@v4
>         with:
>           issue-number: ${{ github.event.number }}
>           body: |
>             ## Preview Environment
>             Preview deployed! Access at: `http://pr-${{ github.event.number }}.app.local`
>             ArgoCD: `https://localhost:8443/applications/myapp-pr-${{ github.event.number }}`
> 
>   cleanup-preview:
>     name: Cleanup Preview
>     runs-on: ubuntu-latest
>     if: github.event.action == 'closed'
>     steps:
>       - name: Remove preview environment
>         uses: peter-evans/repository-dispatch@v3
>         with:
>           token: ${{ secrets.GITOPS_PAT }}
>           repository: YOUR_USER/gitops-repo
>           event-type: preview-deploy
>           client-payload: |
>             {
>               "pr_number": "${{ github.event.number }}",
>               "action": "destroy"
>             }
> ```

> [!success]- Full Solution: `gitops-repo/.github/workflows/preview.yaml`
> ```yaml
> name: Manage Preview Environments
> 
> on:
>   repository_dispatch:
>     types: [preview-deploy]
> 
> jobs:
>   manage-preview:
>     runs-on: ubuntu-latest
>     steps:
>       - uses: actions/checkout@v4
>         with:
>           token: ${{ secrets.GITOPS_PAT }}
> 
>       - name: Deploy preview
>         if: github.event.client_payload.action == 'deploy'
>         run: |
>           PR_NUM="${{ github.event.client_payload.pr_number }}"
>           IMAGE="${{ github.event.client_payload.image }}"
>           TAG="${{ github.event.client_payload.image_tag }}"
> 
>           # Create preview values file
>           cat > helm/myapp/values-pr-${PR_NUM}.yaml << EOF
>           backend:
>             replicaCount: 1
>             image:
>               repository: "${IMAGE}"
>               tag: "${TAG}"
>             env:
>               LOG_LEVEL: "debug"
>           frontend:
>             replicaCount: 1
>           ingress:
>             host: pr-${PR_NUM}.app.local
>           EOF
> 
>           # Create ArgoCD Application for preview
>           cat > apps/preview-pr-${PR_NUM}.yaml << 'APPEOF'
>           apiVersion: argoproj.io/v1alpha1
>           kind: Application
>           metadata:
>             name: myapp-pr-${PR_NUM}
>             namespace: argocd
>             labels:
>               preview: "true"
>               pr-number: "${PR_NUM}"
>             finalizers:
>               - resources-finalizer.argocd.argoproj.io
>           spec:
>             project: default
>             source:
>               repoURL: https://github.com/YOUR_USER/gitops-repo.git
>               targetRevision: main
>               path: helm/myapp
>               helm:
>                 valueFiles:
>                   - values.yaml
>                   - values-pr-${PR_NUM}.yaml
>             destination:
>               server: https://kubernetes.default.svc
>               namespace: myapp-pr-${PR_NUM}
>             syncPolicy:
>               automated:
>                 prune: true
>                 selfHeal: true
>               syncOptions:
>                 - CreateNamespace=true
>           APPEOF
> 
>           # Replace variable references
>           sed -i "s/\${PR_NUM}/${PR_NUM}/g" apps/preview-pr-${PR_NUM}.yaml
> 
>           git config user.name "github-actions[bot]"
>           git config user.email "github-actions[bot]@users.noreply.github.com"
>           git add .
>           git commit -m "preview: deploy PR #${PR_NUM}"
>           git push
> 
>       - name: Destroy preview
>         if: github.event.client_payload.action == 'destroy'
>         run: |
>           PR_NUM="${{ github.event.client_payload.pr_number }}"
> 
>           rm -f apps/preview-pr-${PR_NUM}.yaml
>           rm -f helm/myapp/values-pr-${PR_NUM}.yaml
> 
>           git config user.name "github-actions[bot]"
>           git config user.email "github-actions[bot]@users.noreply.github.com"
>           git add .
>           git commit -m "preview: destroy PR #${PR_NUM}"
>           git push
> ```

---

## Step 7: Rollback Mechanism

Rollback in a GitOps pipeline is straightforward -- revert the commit that changed the image tag:

```bash
# Find the commit that deployed the bad version
cd gitops-repo
git log --oneline helm/myapp/values-dev.yaml

# Revert it
git revert <commit-sha>
git push origin main

# ArgoCD auto-syncs the reverted values -> previous image deployed
```

Alternatively, rollback via ArgoCD CLI:

```bash
# View sync history
argocd app history myapp-dev

# Rollback to a specific revision (cluster-side only, creates drift)
argocd app rollback myapp-dev <revision-number>
```

> [!warning] Prefer Git Revert
> Always prefer `git revert` over `argocd rollback`. ArgoCD rollback creates drift between Git and the cluster. Git revert keeps the single source of truth intact.

---

## Step 8: Notifications

> [!success]- Full Solution: GitHub Status Check (add to `gitops-repo/.github/workflows/cd.yaml`)
> ```yaml
>   notify:
>     name: Notify Deployment
>     runs-on: ubuntu-latest
>     needs: [update-dev]
>     if: always()
>     steps:
>       - name: Set deployment status
>         uses: actions/github-script@v7
>         with:
>           github-token: ${{ secrets.GITOPS_PAT }}
>           script: |
>             const { owner, repo } = {
>               owner: 'YOUR_USER',
>               repo: 'app-repo'
>             };
>             const sha = '${{ github.event.client_payload.sha }}';
>             const status = '${{ needs.update-dev.result }}' === 'success' ? 'success' : 'failure';
> 
>             await github.rest.repos.createCommitStatus({
>               owner,
>               repo,
>               sha,
>               state: status,
>               target_url: `https://localhost:8443/applications/myapp-dev`,
>               description: status === 'success'
>                 ? 'Deployed to dev environment'
>                 : 'Deployment to dev failed',
>               context: 'cd/deploy-dev'
>             });
> 
>       - name: Send Slack notification
>         if: always()
>         uses: slackapi/slack-github-action@v1.25.0
>         with:
>           payload: |
>             {
>               "text": "Deployment ${{ needs.update-dev.result }}: Backend ${{ github.event.client_payload.image_tag }} -> dev",
>               "blocks": [
>                 {
>                   "type": "section",
>                   "text": {
>                     "type": "mrkdwn",
>                     "text": "*Deployment ${{ needs.update-dev.result }}*\nImage: `${{ github.event.client_payload.image_tag }}`\nEnvironment: dev\nTriggered by: ${{ github.event.client_payload.actor }}"
>                   }
>                 }
>               ]
>             }
>         env:
>           SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
>           SLACK_WEBHOOK_TYPE: INCOMING_WEBHOOK
> ```

---

## End-to-End Test on Minikube

While the full pipeline requires GitHub infrastructure, you can simulate the flow locally:

```bash
# 1. Build new image version
docker build -t myapp-backend:v2 --build-arg VERSION=v2 ./app-repo/

# 2. Load into minikube
minikube image load myapp-backend:v2

# 3. Simulate what the CD workflow does: update image tag
cd gitops-repo
yq eval '.backend.image.tag = "v2"' -i helm/myapp/values-dev.yaml
yq eval '.backend.image.repository = "myapp-backend"' -i helm/myapp/values-dev.yaml
yq eval '.backend.image.pullPolicy = "Never"' -i helm/myapp/values-dev.yaml

git add . && git commit -m "deploy(dev): update backend to v2"
git push origin main

# 4. ArgoCD detects change and syncs (or manual sync)
argocd app sync myapp-dev

# 5. Verify
kubectl get pods -n myapp-dev
curl http://dev.app.local/api/version
```

---

## Pipeline Summary

```
Code Push ─► CI (test/lint/build) ─► Push Image to ghcr.io
                                           │
                                           ▼
                                    CD updates gitops-repo
                                           │
                              ┌────────────┼────────────┐
                              ▼            ▼            ▼
                          Dev (auto)   Staging      Production
                                      (approve)    (approve + manual sync)
                              │            │            │
                              ▼            ▼            ▼
                          ArgoCD syncs each environment independently
```

> [!abstract] Key Takeaways
> - **Separation of concerns**: App repo (source + CI) is separate from GitOps repo (config + CD)
> - **Image tags, not branches**: Environments are differentiated by image tags in values files, not Git branches
> - **Progressive delivery**: dev (auto) -> staging (approval) -> production (approval + manual sync)
> - **Preview environments**: PRs get ephemeral environments that are cleaned up on merge/close
> - **Rollback via Git**: `git revert` is the canonical way to roll back in GitOps
> - **Notifications**: GitHub status checks and Slack webhooks close the feedback loop
