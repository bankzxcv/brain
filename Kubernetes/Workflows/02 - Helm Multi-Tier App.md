---
title: "K8s Workflow: Helm Multi-Tier App"
date: 2026-04-07
tags:
  - kubernetes
  - workflow
  - project
  - helm
  - charts
  - templating
parent: "[[Kubernetes Study]]"
---

# Workflow 02: Helm Multi-Tier App

Package the microservice application from [[01 - Microservice Deployment]] as a Helm chart with environment-specific values, subchart dependencies, hooks, and lifecycle management.

## Architecture Overview

```
myapp-chart/
├── Chart.yaml                  # Chart metadata + dependencies
├── Chart.lock                  # Locked dependency versions
├── values.yaml                 # Default values
├── values-dev.yaml             # Dev overrides (1 replica, debug)
├── values-prod.yaml            # Prod overrides (3 replicas, info)
├── charts/                     # Subchart dependencies (auto-populated)
│   └── postgresql/             # Bitnami PostgreSQL subchart
├── templates/
│   ├── _helpers.tpl            # Template helpers
│   ├── NOTES.txt               # Post-install instructions
│   ├── namespace.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── backend-deployment.yaml
│   ├── backend-service.yaml
│   ├── frontend-deployment.yaml
│   ├── frontend-service.yaml
│   ├── ingress.yaml
│   ├── hooks/
│   │   ├── db-migration-job.yaml    # PreInstall/PreUpgrade hook
│   │   └── smoke-test-job.yaml      # Test hook
│   └── tests/
│       └── connection-test.yaml     # helm test pod
└── .helmignore
```

## Prerequisites

- Completed [[01 - Microservice Deployment]] (images loaded in minikube)
- Helm 3 installed

```bash
# Verify Helm
helm version

# Add Bitnami repo for PostgreSQL subchart
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

---

## Step 1: Create Chart Structure

```bash
# Scaffold the chart
helm create myapp-chart

# Remove default templates we don't need
rm -rf myapp-chart/templates/deployment.yaml
rm -rf myapp-chart/templates/service.yaml
rm -rf myapp-chart/templates/serviceaccount.yaml
rm -rf myapp-chart/templates/hpa.yaml
```

> [!info] What `helm create` Gives You
> `helm create` scaffolds a chart with sensible defaults: a `Chart.yaml`, `values.yaml`, helper templates in `_helpers.tpl`, and sample Deployment/Service templates. We remove the samples and write our own.

---

## Step 2: Chart.yaml and Dependencies

> [!success]- Full Solution: `myapp-chart/Chart.yaml`
> ```yaml
> apiVersion: v2
> name: myapp
> description: A multi-tier microservice application with Frontend, Backend, and PostgreSQL
> type: application
> version: 0.1.0
> appVersion: "1.0.0"
> 
> keywords:
>   - microservice
>   - go
>   - nginx
>   - postgresql
> 
> maintainers:
>   - name: DevOps Team
>     email: devops@example.com
> 
> dependencies:
>   - name: postgresql
>     version: "15.x.x"
>     repository: "https://charts.bitnami.com/bitnami"
>     condition: postgresql.enabled
> ```

After editing `Chart.yaml`, download the dependency:

```bash
cd myapp-chart
helm dependency update
```

---

## Step 3: Template Helpers

> [!success]- Full Solution: `myapp-chart/templates/_helpers.tpl`
> ```
> {{/*
> Expand the name of the chart.
> */}}
> {{- define "myapp.name" -}}
> {{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
> {{- end }}
> 
> {{/*
> Create a default fully qualified app name.
> */}}
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
> {{/*
> Create chart name and version for chart label.
> */}}
> {{- define "myapp.chart" -}}
> {{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
> {{- end }}
> 
> {{/*
> Common labels
> */}}
> {{- define "myapp.labels" -}}
> helm.sh/chart: {{ include "myapp.chart" . }}
> app.kubernetes.io/managed-by: {{ .Release.Service }}
> app.kubernetes.io/part-of: {{ include "myapp.name" . }}
> app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
> {{- end }}
> 
> {{/*
> Backend labels
> */}}
> {{- define "myapp.backend.labels" -}}
> {{ include "myapp.labels" . }}
> app.kubernetes.io/component: backend
> {{- end }}
> 
> {{/*
> Backend selector labels
> */}}
> {{- define "myapp.backend.selectorLabels" -}}
> app.kubernetes.io/name: {{ include "myapp.name" . }}-backend
> app.kubernetes.io/instance: {{ .Release.Name }}
> {{- end }}
> 
> {{/*
> Frontend labels
> */}}
> {{- define "myapp.frontend.labels" -}}
> {{ include "myapp.labels" . }}
> app.kubernetes.io/component: frontend
> {{- end }}
> 
> {{/*
> Frontend selector labels
> */}}
> {{- define "myapp.frontend.selectorLabels" -}}
> app.kubernetes.io/name: {{ include "myapp.name" . }}-frontend
> app.kubernetes.io/instance: {{ .Release.Name }}
> {{- end }}
> 
> {{/*
> Database host - either subchart service or external
> */}}
> {{- define "myapp.databaseHost" -}}
> {{- if .Values.postgresql.enabled }}
> {{- printf "%s-postgresql" .Release.Name }}
> {{- else }}
> {{- .Values.externalDatabase.host }}
> {{- end }}
> {{- end }}
> ```

---

## Step 4: Values File (Parameterize Everything)

> [!success]- Full Solution: `myapp-chart/values.yaml`
> ```yaml
> # -- Global overrides
> nameOverride: ""
> fullnameOverride: ""
> 
> # ===== Backend Configuration =====
> backend:
>   # -- Number of backend replicas
>   replicaCount: 1
> 
>   image:
>     repository: myapp-backend
>     tag: "v1"
>     pullPolicy: Never    # Using minikube image load
> 
>   service:
>     type: ClusterIP
>     port: 8080
> 
>   # -- Application log level
>   logLevel: "info"
> 
>   resources:
>     requests:
>       memory: "64Mi"
>       cpu: "100m"
>     limits:
>       memory: "128Mi"
>       cpu: "250m"
> 
>   livenessProbe:
>     path: /api/health
>     initialDelaySeconds: 5
>     periodSeconds: 10
> 
>   readinessProbe:
>     path: /api/ready
>     initialDelaySeconds: 5
>     periodSeconds: 5
> 
> # ===== Frontend Configuration =====
> frontend:
>   replicaCount: 1
> 
>   image:
>     repository: myapp-frontend
>     tag: "v1"
>     pullPolicy: Never
> 
>   service:
>     type: ClusterIP
>     port: 80
> 
>   resources:
>     requests:
>       memory: "32Mi"
>       cpu: "50m"
>     limits:
>       memory: "64Mi"
>       cpu: "100m"
> 
> # ===== Ingress Configuration =====
> ingress:
>   enabled: true
>   className: nginx
>   host: app.local
>   annotations:
>     nginx.ingress.kubernetes.io/rewrite-target: /$2
> 
> # ===== Database Configuration =====
> # Bitnami PostgreSQL subchart values
> postgresql:
>   enabled: true
>   auth:
>     username: appuser
>     password: apppassword
>     database: appdb
>   primary:
>     persistence:
>       enabled: false    # No PV needed for minikube demo
>     resources:
>       requests:
>         memory: "128Mi"
>         cpu: "250m"
>       limits:
>         memory: "256Mi"
>         cpu: "500m"
> 
> # For using an external database instead of the subchart
> externalDatabase:
>   host: ""
>   port: 5432
>   user: appuser
>   password: ""
>   database: appdb
> 
> # ===== DB Migration =====
> migration:
>   enabled: true
>   image:
>     repository: myapp-backend
>     tag: "v1"
>     pullPolicy: Never
> ```

---

## Step 5: Templates

### ConfigMap

> [!success]- Full Solution: `myapp-chart/templates/configmap.yaml`
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: {{ include "myapp.fullname" . }}-config
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
> data:
>   DB_HOST: {{ include "myapp.databaseHost" . | quote }}
>   DB_PORT: {{ .Values.postgresql.enabled | ternary "5432" (.Values.externalDatabase.port | toString) | quote }}
>   DB_NAME: {{ .Values.postgresql.enabled | ternary .Values.postgresql.auth.database .Values.externalDatabase.database | quote }}
>   DB_USER: {{ .Values.postgresql.enabled | ternary .Values.postgresql.auth.username .Values.externalDatabase.user | quote }}
>   PORT: {{ .Values.backend.service.port | quote }}
>   LOG_LEVEL: {{ .Values.backend.logLevel | quote }}
> ```

### Secret

> [!success]- Full Solution: `myapp-chart/templates/secret.yaml`
> ```yaml
> apiVersion: v1
> kind: Secret
> metadata:
>   name: {{ include "myapp.fullname" . }}-secret
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
> type: Opaque
> stringData:
>   {{- if .Values.postgresql.enabled }}
>   DB_PASSWORD: {{ .Values.postgresql.auth.password | quote }}
>   {{- else }}
>   DB_PASSWORD: {{ .Values.externalDatabase.password | quote }}
>   {{- end }}
> ```

### Backend Deployment

> [!success]- Full Solution: `myapp-chart/templates/backend-deployment.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: {{ include "myapp.fullname" . }}-backend
>   labels:
>     {{- include "myapp.backend.labels" . | nindent 4 }}
> spec:
>   replicas: {{ .Values.backend.replicaCount }}
>   selector:
>     matchLabels:
>       {{- include "myapp.backend.selectorLabels" . | nindent 6 }}
>   template:
>     metadata:
>       labels:
>         {{- include "myapp.backend.selectorLabels" . | nindent 8 }}
>       annotations:
>         checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
>         checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
>     spec:
>       containers:
>         - name: backend
>           image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
>           imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
>           ports:
>             - containerPort: {{ .Values.backend.service.port }}
>               protocol: TCP
>           envFrom:
>             - configMapRef:
>                 name: {{ include "myapp.fullname" . }}-config
>           env:
>             - name: DB_PASSWORD
>               valueFrom:
>                 secretKeyRef:
>                   name: {{ include "myapp.fullname" . }}-secret
>                   key: DB_PASSWORD
>           livenessProbe:
>             httpGet:
>               path: {{ .Values.backend.livenessProbe.path }}
>               port: {{ .Values.backend.service.port }}
>             initialDelaySeconds: {{ .Values.backend.livenessProbe.initialDelaySeconds }}
>             periodSeconds: {{ .Values.backend.livenessProbe.periodSeconds }}
>           readinessProbe:
>             httpGet:
>               path: {{ .Values.backend.readinessProbe.path }}
>               port: {{ .Values.backend.service.port }}
>             initialDelaySeconds: {{ .Values.backend.readinessProbe.initialDelaySeconds }}
>             periodSeconds: {{ .Values.backend.readinessProbe.periodSeconds }}
>           resources:
>             {{- toYaml .Values.backend.resources | nindent 12 }}
> ```

### Backend Service

> [!success]- Full Solution: `myapp-chart/templates/backend-service.yaml`
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: {{ include "myapp.fullname" . }}-backend
>   labels:
>     {{- include "myapp.backend.labels" . | nindent 4 }}
> spec:
>   type: {{ .Values.backend.service.type }}
>   selector:
>     {{- include "myapp.backend.selectorLabels" . | nindent 4 }}
>   ports:
>     - port: {{ .Values.backend.service.port }}
>       targetPort: {{ .Values.backend.service.port }}
>       protocol: TCP
> ```

### Frontend Deployment

> [!success]- Full Solution: `myapp-chart/templates/frontend-deployment.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: {{ include "myapp.fullname" . }}-frontend
>   labels:
>     {{- include "myapp.frontend.labels" . | nindent 4 }}
> spec:
>   replicas: {{ .Values.frontend.replicaCount }}
>   selector:
>     matchLabels:
>       {{- include "myapp.frontend.selectorLabels" . | nindent 6 }}
>   template:
>     metadata:
>       labels:
>         {{- include "myapp.frontend.selectorLabels" . | nindent 8 }}
>     spec:
>       containers:
>         - name: frontend
>           image: "{{ .Values.frontend.image.repository }}:{{ .Values.frontend.image.tag }}"
>           imagePullPolicy: {{ .Values.frontend.image.pullPolicy }}
>           ports:
>             - containerPort: {{ .Values.frontend.service.port }}
>               protocol: TCP
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
>           resources:
>             {{- toYaml .Values.frontend.resources | nindent 12 }}
> ```

### Frontend Service

> [!success]- Full Solution: `myapp-chart/templates/frontend-service.yaml`
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: {{ include "myapp.fullname" . }}-frontend
>   labels:
>     {{- include "myapp.frontend.labels" . | nindent 4 }}
> spec:
>   type: {{ .Values.frontend.service.type }}
>   selector:
>     {{- include "myapp.frontend.selectorLabels" . | nindent 4 }}
>   ports:
>     - port: {{ .Values.frontend.service.port }}
>       targetPort: {{ .Values.frontend.service.port }}
>       protocol: TCP
> ```

### Ingress

> [!success]- Full Solution: `myapp-chart/templates/ingress.yaml`
> ```yaml
> {{- if .Values.ingress.enabled }}
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: {{ include "myapp.fullname" . }}-ingress
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
>   {{- with .Values.ingress.annotations }}
>   annotations:
>     {{- toYaml . | nindent 4 }}
>   {{- end }}
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

### NOTES.txt

> [!success]- Full Solution: `myapp-chart/templates/NOTES.txt`
> ```
> ============================================
>   {{ include "myapp.fullname" . }} deployed!
> ============================================
> 
> Application URL:
> {{- if .Values.ingress.enabled }}
>   http://{{ .Values.ingress.host }}
> {{- else }}
>   kubectl port-forward svc/{{ include "myapp.fullname" . }}-frontend {{ .Values.frontend.service.port }}:{{ .Values.frontend.service.port }}
> {{- end }}
> 
> Backend API:
> {{- if .Values.ingress.enabled }}
>   http://{{ .Values.ingress.host }}/api/health
> {{- else }}
>   kubectl port-forward svc/{{ include "myapp.fullname" . }}-backend {{ .Values.backend.service.port }}:{{ .Values.backend.service.port }}
> {{- end }}
> 
> Verify everything is running:
>   kubectl get pods -l "app.kubernetes.io/instance={{ .Release.Name }}"
> 
> Run the test suite:
>   helm test {{ .Release.Name }}
> ```

---

## Step 6: Environment-Specific Values

### Dev Values

> [!success]- Full Solution: `myapp-chart/values-dev.yaml`
> ```yaml
> # Development overrides
> backend:
>   replicaCount: 1
>   logLevel: "debug"
>   resources:
>     requests:
>       memory: "64Mi"
>       cpu: "100m"
>     limits:
>       memory: "128Mi"
>       cpu: "250m"
> 
> frontend:
>   replicaCount: 1
> 
> ingress:
>   host: dev.app.local
> 
> postgresql:
>   primary:
>     persistence:
>       enabled: false
> ```

### Prod Values

> [!success]- Full Solution: `myapp-chart/values-prod.yaml`
> ```yaml
> # Production overrides
> backend:
>   replicaCount: 3
>   logLevel: "info"
>   resources:
>     requests:
>       memory: "128Mi"
>       cpu: "250m"
>     limits:
>       memory: "256Mi"
>       cpu: "500m"
> 
> frontend:
>   replicaCount: 2
>   resources:
>     requests:
>       memory: "64Mi"
>       cpu: "100m"
>     limits:
>       memory: "128Mi"
>       cpu: "200m"
> 
> ingress:
>   host: app.example.com
> 
> postgresql:
>   primary:
>     persistence:
>       enabled: true
>       size: 10Gi
>     resources:
>       requests:
>         memory: "256Mi"
>         cpu: "500m"
>       limits:
>         memory: "512Mi"
>         cpu: "1000m"
> ```

---

## Step 7: Helm Hooks

> [!abstract] Key Concept: Helm Hooks
> Hooks let you run actions at specific points in the release lifecycle:
> - `pre-install` / `pre-upgrade`: Run before resources are installed/upgraded (e.g., DB migrations)
> - `post-install` / `post-upgrade`: Run after resources are installed/upgraded
> - `test`: Run on `helm test` (e.g., smoke tests, integration tests)
>
> Hook execution order is controlled by `helm.sh/hook-weight` (lower runs first). Hooks are deleted based on `helm.sh/hook-delete-policy`.

### DB Migration Hook (PreInstall/PreUpgrade)

> [!success]- Full Solution: `myapp-chart/templates/hooks/db-migration-job.yaml`
> ```yaml
> {{- if .Values.migration.enabled }}
> apiVersion: batch/v1
> kind: Job
> metadata:
>   name: {{ include "myapp.fullname" . }}-db-migrate
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
>   annotations:
>     "helm.sh/hook": pre-install,pre-upgrade
>     "helm.sh/hook-weight": "0"
>     "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
> spec:
>   backoffLimit: 3
>   template:
>     metadata:
>       labels:
>         app: db-migration
>     spec:
>       restartPolicy: Never
>       initContainers:
>         # Wait for PostgreSQL to be ready before running migration
>         - name: wait-for-db
>           image: busybox:1.36
>           command:
>             - sh
>             - -c
>             - |
>               until nc -z {{ include "myapp.databaseHost" . }} 5432; do
>                 echo "Waiting for database..."
>                 sleep 2
>               done
>               echo "Database is ready"
>       containers:
>         - name: migrate
>           image: "{{ .Values.migration.image.repository }}:{{ .Values.migration.image.tag }}"
>           imagePullPolicy: {{ .Values.migration.image.pullPolicy }}
>           command:
>             - /backend
>           args:
>             - --migrate-only
>           env:
>             - name: DB_HOST
>               value: {{ include "myapp.databaseHost" . | quote }}
>             - name: DB_PORT
>               value: "5432"
>             - name: DB_NAME
>               value: {{ .Values.postgresql.auth.database | quote }}
>             - name: DB_USER
>               value: {{ .Values.postgresql.auth.username | quote }}
>             - name: DB_PASSWORD
>               valueFrom:
>                 secretKeyRef:
>                   name: {{ include "myapp.fullname" . }}-secret
>                   key: DB_PASSWORD
> {{- end }}
> ```

### Smoke Test Hook

> [!success]- Full Solution: `myapp-chart/templates/hooks/smoke-test-job.yaml`
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: {{ include "myapp.fullname" . }}-smoke-test
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
>   annotations:
>     "helm.sh/hook": test
>     "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
> spec:
>   restartPolicy: Never
>   containers:
>     - name: smoke-test
>       image: curlimages/curl:8.5.0
>       command:
>         - sh
>         - -c
>         - |
>           echo "=== Testing Backend Health ==="
>           curl -sf http://{{ include "myapp.fullname" . }}-backend:{{ .Values.backend.service.port }}/api/health || exit 1
>           echo ""
> 
>           echo "=== Testing Backend Readiness ==="
>           curl -sf http://{{ include "myapp.fullname" . }}-backend:{{ .Values.backend.service.port }}/api/ready || exit 1
>           echo ""
> 
>           echo "=== Testing Frontend ==="
>           curl -sf http://{{ include "myapp.fullname" . }}-frontend:{{ .Values.frontend.service.port }}/ || exit 1
>           echo ""
> 
>           echo "=== All smoke tests passed ==="
> ```

---

## Step 8: Install, Upgrade, and Rollback

### Validate Templates First

```bash
# Lint the chart
helm lint myapp-chart/

# Render templates without installing (dry-run)
helm template myapp-release myapp-chart/ -f myapp-chart/values-dev.yaml

# Dry-run install to check against the cluster
helm install myapp-release myapp-chart/ \
  -f myapp-chart/values-dev.yaml \
  --dry-run --debug
```

### Install with Dev Values

```bash
# Install
helm install myapp-dev myapp-chart/ -f myapp-chart/values-dev.yaml

# Check release
helm list

# Watch pods come up
kubectl get pods -w -l "app.kubernetes.io/instance=myapp-dev"

# Run smoke tests
helm test myapp-dev
```

### Install with Prod Values (Separate Release)

```bash
# Install prod in a different namespace
helm install myapp-prod myapp-chart/ \
  -f myapp-chart/values-prod.yaml \
  --create-namespace \
  --namespace production
```

### Upgrade Demo

```bash
# Change image tag to v2 (simulating a new version)
helm upgrade myapp-dev myapp-chart/ \
  -f myapp-chart/values-dev.yaml \
  --set backend.image.tag=v2

# Check rollout
kubectl rollout status deployment/myapp-dev-backend

# View release history
helm history myapp-dev
```

### Rollback Demo

```bash
# Rollback to the previous revision
helm rollback myapp-dev 1

# Verify rollback
helm history myapp-dev
kubectl get pods -l "app.kubernetes.io/instance=myapp-dev"
```

> [!tip] Helm Revision History
> Every `helm install` and `helm upgrade` creates a new revision. `helm history <release>` shows all revisions. `helm rollback <release> <revision>` reverts to a specific revision.

---

## Package and Distribute

```bash
# Package the chart into a .tgz archive
helm package myapp-chart/

# This creates myapp-0.1.0.tgz

# Install from package
helm install myapp-from-pkg ./myapp-0.1.0.tgz -f myapp-chart/values-dev.yaml
```

---

## Cleanup

```bash
helm uninstall myapp-dev
helm uninstall myapp-prod -n production
```

---

## Troubleshooting

| Problem | Command | What to Look For |
|---|---|---|
| Template errors | `helm template myapp myapp-chart/ --debug` | YAML syntax, undefined values |
| Hook stuck | `kubectl get jobs` | Failed jobs, check pod logs |
| Subchart not found | `helm dependency update myapp-chart/` | Missing `Chart.lock` |
| Values not applied | `helm get values myapp-dev` | Compare with expected values |
| Upgrade conflicts | `helm history myapp-dev` | Failed or pending revisions |

> [!abstract] Key Takeaways
> - Helm charts turn imperative `kubectl apply` into declarative, versioned releases
> - `values.yaml` + environment overrides keep config DRY across dev/staging/prod
> - Subchart dependencies (like PostgreSQL) are managed via `Chart.yaml` and pulled automatically
> - Hooks run Jobs/Pods at lifecycle points (pre-install, post-upgrade, test)
> - `helm history` + `helm rollback` provide built-in version management
> - Template checksums on ConfigMap/Secret annotations force pod restarts on config changes
