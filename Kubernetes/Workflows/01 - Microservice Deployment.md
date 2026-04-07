---
title: "K8s Workflow: Microservice Deployment"
date: 2026-04-07
tags:
  - kubernetes
  - workflow
  - project
  - docker
  - microservices
  - ingress
  - deployments
parent: "[[Kubernetes Study]]"
---

# Workflow 01: Microservice Deployment

Deploy a multi-tier microservice application to Kubernetes from scratch using minikube.

## Architecture Overview

```
                    ┌─────────────────────────────────────┐
                    │           minikube cluster           │
                    │                                      │
  Browser ──────►  │  ┌──────────────────────────────┐    │
  http://app.local │  │     Ingress Controller        │    │
                    │  │   (minikube addon)            │    │
                    │  └──────┬───────────┬───────────┘    │
                    │         │           │                 │
                    │    / path      /api path              │
                    │         │           │                 │
                    │         ▼           ▼                 │
                    │  ┌──────────┐ ┌──────────┐          │
                    │  │ Frontend │ │ Backend  │          │
                    │  │ (nginx)  │ │ (Go API) │          │
                    │  │ :80      │ │ :8080    │          │
                    │  │ 1 replica│ │ 3 replica│          │
                    │  └──────────┘ └────┬─────┘          │
                    │                     │                 │
                    │                     ▼                 │
                    │              ┌──────────┐            │
                    │              │PostgreSQL│            │
                    │              │ :5432    │            │
                    │              │ 1 replica│            │
                    │              └──────────┘            │
                    └─────────────────────────────────────┘
```

## Prerequisites

- minikube installed and running
- kubectl configured
- Docker CLI available (uses minikube's Docker daemon)

```bash
minikube start --cpus=4 --memory=4096
minikube addons enable ingress
```

---

## Step 1: Write Dockerfiles (Multi-Stage Builds)

> [!info] Multi-Stage Builds
> Multi-stage builds produce minimal final images by separating the build environment from the runtime environment. This reduces image size and attack surface.

### Backend API (Go)

Create the Go application first. This is a simple REST API that connects to PostgreSQL.

> [!success]- Full Solution: Backend Go Application (`backend/main.go`)
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
> type HealthResponse struct {
> 	Status   string `json:"status"`
> 	Database string `json:"database"`
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
> 		log.Printf("Warning: Could not open DB connection: %v", err)
> 	}
> 
> 	// Initialize table
> 	if db != nil {
> 		_, err = db.Exec(`CREATE TABLE IF NOT EXISTS items (
> 			id SERIAL PRIMARY KEY,
> 			name VARCHAR(255) NOT NULL
> 		)`)
> 		if err != nil {
> 			log.Printf("Warning: Could not create table: %v", err)
> 		}
> 	}
> 
> 	http.HandleFunc("/api/health", healthHandler)
> 	http.HandleFunc("/api/ready", readyHandler)
> 	http.HandleFunc("/api/items", itemsHandler)
> 	http.HandleFunc("/api/hostname", hostnameHandler)
> 
> 	log.Printf("Backend starting on port %s", port)
> 	log.Fatal(http.ListenAndServe(":"+port, nil))
> }
> 
> func healthHandler(w http.ResponseWriter, r *http.Request) {
> 	w.Header().Set("Content-Type", "application/json")
> 	json.NewEncoder(w).Encode(HealthResponse{Status: "ok", Database: "unknown"})
> }
> 
> func readyHandler(w http.ResponseWriter, r *http.Request) {
> 	w.Header().Set("Content-Type", "application/json")
> 	if db == nil {
> 		w.WriteHeader(http.StatusServiceUnavailable)
> 		json.NewEncoder(w).Encode(HealthResponse{Status: "not ready", Database: "not connected"})
> 		return
> 	}
> 	err := db.Ping()
> 	if err != nil {
> 		w.WriteHeader(http.StatusServiceUnavailable)
> 		json.NewEncoder(w).Encode(HealthResponse{Status: "not ready", Database: err.Error()})
> 		return
> 	}
> 	json.NewEncoder(w).Encode(HealthResponse{Status: "ready", Database: "connected"})
> }
> 
> func itemsHandler(w http.ResponseWriter, r *http.Request) {
> 	w.Header().Set("Content-Type", "application/json")
> 	switch r.Method {
> 	case http.MethodGet:
> 		rows, err := db.Query("SELECT id, name FROM items")
> 		if err != nil {
> 			http.Error(w, err.Error(), http.StatusInternalServerError)
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
> 		err := db.QueryRow("INSERT INTO items (name) VALUES ($1) RETURNING id", item.Name).Scan(&item.ID)
> 		if err != nil {
> 			http.Error(w, err.Error(), http.StatusInternalServerError)
> 			return
> 		}
> 		w.WriteHeader(http.StatusCreated)
> 		json.NewEncoder(w).Encode(item)
> 	}
> }
> 
> func hostnameHandler(w http.ResponseWriter, r *http.Request) {
> 	hostname, _ := os.Hostname()
> 	w.Header().Set("Content-Type", "application/json")
> 	json.NewEncoder(w).Encode(map[string]string{"hostname": hostname})
> }
> 
> func getEnv(key, fallback string) string {
> 	if value, ok := os.LookupEnv(key); ok {
> 		return value
> 	}
> 	return fallback
> }
> ```

> [!success]- Full Solution: Backend Go Module (`backend/go.mod`)
> ```
> module github.com/example/backend
> 
> go 1.21
> 
> require github.com/lib/pq v1.10.9
> ```

> [!success]- Full Solution: Backend Dockerfile (`backend/Dockerfile`)
> ```dockerfile
> # ---- Build Stage ----
> FROM golang:1.21-alpine AS builder
> 
> WORKDIR /app
> 
> # Copy go module files first for layer caching
> COPY go.mod go.sum ./
> RUN go mod download
> 
> # Copy source and build
> COPY . .
> RUN CGO_ENABLED=0 GOOS=linux go build -o /backend main.go
> 
> # ---- Runtime Stage ----
> FROM alpine:3.19
> 
> RUN apk --no-cache add ca-certificates
> 
> COPY --from=builder /backend /backend
> 
> EXPOSE 8080
> 
> ENTRYPOINT ["/backend"]
> ```

### Frontend (Nginx + Static Files)

> [!success]- Full Solution: Frontend HTML (`frontend/public/index.html`)
> ```html
> <!DOCTYPE html>
> <html lang="en">
> <head>
>     <meta charset="UTF-8">
>     <title>K8s Microservice Demo</title>
>     <style>
>         body { font-family: sans-serif; max-width: 800px; margin: 40px auto; padding: 0 20px; }
>         button { padding: 8px 16px; margin: 4px; cursor: pointer; }
>         pre { background: #f4f4f4; padding: 12px; border-radius: 4px; overflow-x: auto; }
>         .status { padding: 8px; margin: 8px 0; border-radius: 4px; }
>         .ok { background: #d4edda; }
>         .error { background: #f8d7da; }
>     </style>
> </head>
> <body>
>     <h1>K8s Microservice Demo</h1>
>     <h2>Backend Health</h2>
>     <button onclick="checkHealth()">Check Health</button>
>     <button onclick="checkHostname()">Check Hostname (Load Balancing)</button>
>     <div id="health-result"></div>
> 
>     <h2>Items API</h2>
>     <input type="text" id="item-name" placeholder="Item name">
>     <button onclick="addItem()">Add Item</button>
>     <button onclick="getItems()">List Items</button>
>     <pre id="items-result">Click "List Items" to load</pre>
> 
>     <script>
>         async function checkHealth() {
>             try {
>                 const res = await fetch('/api/health');
>                 const data = await res.json();
>                 document.getElementById('health-result').innerHTML =
>                     `<div class="status ok"><pre>${JSON.stringify(data, null, 2)}</pre></div>`;
>             } catch (e) {
>                 document.getElementById('health-result').innerHTML =
>                     `<div class="status error">Error: ${e.message}</div>`;
>             }
>         }
> 
>         async function checkHostname() {
>             try {
>                 const res = await fetch('/api/hostname');
>                 const data = await res.json();
>                 document.getElementById('health-result').innerHTML =
>                     `<div class="status ok">Served by pod: <strong>${data.hostname}</strong></div>`;
>             } catch (e) {
>                 document.getElementById('health-result').innerHTML =
>                     `<div class="status error">Error: ${e.message}</div>`;
>             }
>         }
> 
>         async function addItem() {
>             const name = document.getElementById('item-name').value;
>             if (!name) return alert('Enter an item name');
>             const res = await fetch('/api/items', {
>                 method: 'POST',
>                 headers: { 'Content-Type': 'application/json' },
>                 body: JSON.stringify({ name })
>             });
>             const data = await res.json();
>             document.getElementById('items-result').textContent = JSON.stringify(data, null, 2);
>             document.getElementById('item-name').value = '';
>         }
> 
>         async function getItems() {
>             const res = await fetch('/api/items');
>             const data = await res.json();
>             document.getElementById('items-result').textContent = JSON.stringify(data, null, 2);
>         }
>     </script>
> </body>
> </html>
> ```

> [!success]- Full Solution: Nginx Config (`frontend/nginx.conf`)
> ```nginx
> server {
>     listen 80;
>     server_name _;
> 
>     root /usr/share/nginx/html;
>     index index.html;
> 
>     # Serve static files
>     location / {
>         try_files $uri $uri/ /index.html;
>     }
> 
>     # Health check endpoint
>     location /healthz {
>         return 200 'ok';
>         add_header Content-Type text/plain;
>     }
> }
> ```

> [!success]- Full Solution: Frontend Dockerfile (`frontend/Dockerfile`)
> ```dockerfile
> # ---- Build Stage ----
> FROM node:20-alpine AS builder
> 
> WORKDIR /app
> COPY public/ ./public/
> # In a real app, you would run npm build here
> # For this demo, static files are ready as-is
> 
> # ---- Runtime Stage ----
> FROM nginx:1.25-alpine
> 
> # Remove default nginx config
> RUN rm /etc/nginx/conf.d/default.conf
> 
> # Copy custom config
> COPY nginx.conf /etc/nginx/conf.d/default.conf
> 
> # Copy static files
> COPY --from=builder /app/public /usr/share/nginx/html
> 
> EXPOSE 80
> 
> CMD ["nginx", "-g", "daemon off;"]
> ```

---

## Step 2: Build Images and Load into Minikube

> [!tip] Why `minikube image load`?
> Instead of pushing to a remote registry, `minikube image load` copies local Docker images directly into minikube's container runtime. Set `imagePullPolicy: Never` in your manifests.

```bash
# Build images using your local Docker daemon
docker build -t myapp-backend:v1 ./backend/
docker build -t myapp-frontend:v1 ./frontend/

# Load into minikube
minikube image load myapp-backend:v1
minikube image load myapp-frontend:v1

# Verify images are available
minikube image ls | grep myapp
```

---

## Step 3: Create Kubernetes Manifests

### Namespace

> [!success]- Full Solution: `k8s/namespace.yaml`
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: myapp
>   labels:
>     app.kubernetes.io/part-of: myapp
> ```

### ConfigMap

> [!success]- Full Solution: `k8s/configmap.yaml`
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: backend-config
>   namespace: myapp
> data:
>   DB_HOST: "postgres-svc"
>   DB_PORT: "5432"
>   DB_NAME: "appdb"
>   DB_USER: "appuser"
>   PORT: "8080"
> ```

### Secret

> [!warning] Secrets in Practice
> Kubernetes Secrets are base64-encoded, **not encrypted**. In production, use Sealed Secrets, Vault, or external secret managers. For this exercise, plain Secrets are fine.

> [!success]- Full Solution: `k8s/secret.yaml`
> ```yaml
> apiVersion: v1
> kind: Secret
> metadata:
>   name: db-secret
>   namespace: myapp
> type: Opaque
> stringData:
>   DB_PASSWORD: "apppassword"
>   POSTGRES_PASSWORD: "apppassword"
> ```

### PostgreSQL Deployment and Service

> [!success]- Full Solution: `k8s/postgres.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: postgres
>   namespace: myapp
>   labels:
>     app: postgres
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: postgres
>   template:
>     metadata:
>       labels:
>         app: postgres
>     spec:
>       containers:
>         - name: postgres
>           image: postgres:16-alpine
>           ports:
>             - containerPort: 5432
>           env:
>             - name: POSTGRES_DB
>               valueFrom:
>                 configMapKeyRef:
>                   name: backend-config
>                   key: DB_NAME
>             - name: POSTGRES_USER
>               valueFrom:
>                 configMapKeyRef:
>                   name: backend-config
>                   key: DB_USER
>             - name: POSTGRES_PASSWORD
>               valueFrom:
>                 secretKeyRef:
>                   name: db-secret
>                   key: POSTGRES_PASSWORD
>           volumeMounts:
>             - name: postgres-storage
>               mountPath: /var/lib/postgresql/data
>           resources:
>             requests:
>               memory: "128Mi"
>               cpu: "250m"
>             limits:
>               memory: "256Mi"
>               cpu: "500m"
>       volumes:
>         - name: postgres-storage
>           emptyDir: {}
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: postgres-svc
>   namespace: myapp
> spec:
>   selector:
>     app: postgres
>   ports:
>     - port: 5432
>       targetPort: 5432
>   type: ClusterIP
> ```

### Backend Deployment and Service

> [!success]- Full Solution: `k8s/backend.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: backend
>   namespace: myapp
>   labels:
>     app: backend
> spec:
>   replicas: 1
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
>           image: myapp-backend:v1
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           envFrom:
>             - configMapRef:
>                 name: backend-config
>           env:
>             - name: DB_PASSWORD
>               valueFrom:
>                 secretKeyRef:
>                   name: db-secret
>                   key: DB_PASSWORD
>           livenessProbe:
>             httpGet:
>               path: /api/health
>               port: 8080
>             initialDelaySeconds: 5
>             periodSeconds: 10
>             failureThreshold: 3
>           readinessProbe:
>             httpGet:
>               path: /api/ready
>               port: 8080
>             initialDelaySeconds: 5
>             periodSeconds: 5
>             failureThreshold: 3
>           resources:
>             requests:
>               memory: "64Mi"
>               cpu: "100m"
>             limits:
>               memory: "128Mi"
>               cpu: "250m"
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: backend-svc
>   namespace: myapp
> spec:
>   selector:
>     app: backend
>   ports:
>     - port: 8080
>       targetPort: 8080
>   type: ClusterIP
> ```

### Frontend Deployment and Service

> [!success]- Full Solution: `k8s/frontend.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: frontend
>   namespace: myapp
>   labels:
>     app: frontend
> spec:
>   replicas: 1
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
>           image: myapp-frontend:v1
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 80
>           livenessProbe:
>             httpGet:
>               path: /healthz
>               port: 80
>             initialDelaySeconds: 3
>             periodSeconds: 10
>           readinessProbe:
>             httpGet:
>               path: /healthz
>               port: 80
>             initialDelaySeconds: 3
>             periodSeconds: 5
>           resources:
>             requests:
>               memory: "32Mi"
>               cpu: "50m"
>             limits:
>               memory: "64Mi"
>               cpu: "100m"
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: frontend-svc
>   namespace: myapp
> spec:
>   selector:
>     app: frontend
>   ports:
>     - port: 80
>       targetPort: 80
>   type: ClusterIP
> ```

---

## Step 4: Health Checks (Liveness and Readiness Probes)

> [!abstract] Key Concept: Liveness vs Readiness
> - **Liveness Probe**: "Is the container alive?" If it fails, Kubernetes **restarts** the container. Use for detecting deadlocks or hung processes.
> - **Readiness Probe**: "Can the container serve traffic?" If it fails, Kubernetes **removes the pod from Service endpoints**. The container keeps running. Use for startup dependencies (e.g., waiting for DB).
>
> The probes are already included in the Deployment manifests above:
> - Backend liveness: `GET /api/health` (always returns 200 if process is running)
> - Backend readiness: `GET /api/ready` (returns 200 only when DB is reachable)
> - Frontend liveness/readiness: `GET /healthz` (nginx returns 200)

Probe parameters explained:

| Parameter | Meaning |
|---|---|
| `initialDelaySeconds` | Wait this long after container start before first probe |
| `periodSeconds` | How often to run the probe |
| `failureThreshold` | How many consecutive failures before action is taken |
| `timeoutSeconds` | How long to wait for a probe response (default: 1s) |

---

## Step 5: Ingress Routing

> [!success]- Full Solution: `k8s/ingress.yaml`
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: myapp-ingress
>   namespace: myapp
>   annotations:
>     nginx.ingress.kubernetes.io/rewrite-target: /$2
> spec:
>   ingressClassName: nginx
>   rules:
>     - host: app.local
>       http:
>         paths:
>           - path: /api(/|$)(.*)
>             pathType: ImplementationSpecific
>             backend:
>               service:
>                 name: backend-svc
>                 port:
>                   number: 8080
>           - path: /()(.*)
>             pathType: ImplementationSpecific
>             backend:
>               service:
>                 name: frontend-svc
>                 port:
>                   number: 80
> ```

Configure local DNS:

```bash
# Get minikube IP
minikube ip

# Add to /etc/hosts (replace <MINIKUBE_IP>)
echo "$(minikube ip) app.local" | sudo tee -a /etc/hosts
```

> [!tip] Alternative: minikube tunnel
> If Ingress doesn't work with `minikube ip`, try `minikube tunnel` in a separate terminal. This creates a route to cluster services on your host machine.

---

## Step 6: Deploy and Test End-to-End

### Deploy Everything

```bash
# Apply in order: namespace -> config -> secrets -> database -> backend -> frontend -> ingress
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/postgres.yaml

# Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n myapp --timeout=120s

kubectl apply -f k8s/backend.yaml
kubectl apply -f k8s/frontend.yaml
kubectl apply -f k8s/ingress.yaml
```

### Verify Deployment

```bash
# Check all resources
kubectl get all -n myapp

# Check pods are running and ready
kubectl get pods -n myapp -o wide

# Check services
kubectl get svc -n myapp

# Check ingress
kubectl get ingress -n myapp

# View backend logs
kubectl logs -l app=backend -n myapp

# Test Service DNS from within the cluster
kubectl run -n myapp debug --rm -it --image=busybox -- nslookup postgres-svc
```

### Test the Application

```bash
# Test backend health directly via port-forward
kubectl port-forward svc/backend-svc 8080:8080 -n myapp &
curl http://localhost:8080/api/health
curl http://localhost:8080/api/ready

# Test via Ingress
curl http://app.local/
curl http://app.local/api/health
curl http://app.local/api/items

# Add an item
curl -X POST http://app.local/api/items \
  -H "Content-Type: application/json" \
  -d '{"name": "test item"}'

# List items
curl http://app.local/api/items
```

---

## Step 7: Scale and Verify Load Balancing

```bash
# Scale backend to 3 replicas
kubectl scale deployment backend -n myapp --replicas=3

# Wait for all replicas to be ready
kubectl rollout status deployment/backend -n myapp

# Verify 3 pods running
kubectl get pods -n myapp -l app=backend

# Test load balancing - each request may hit a different pod
for i in $(seq 1 10); do
  curl -s http://app.local/api/hostname | jq -r .hostname
done
```

> [!note] Expected Output
> You should see different hostnames returned across the 10 requests, proving that the Service is load-balancing across all 3 backend replicas. The distribution may not be perfectly even due to the round-robin algorithm and connection reuse.

---

## Cleanup

```bash
kubectl delete namespace myapp
```

---

## Troubleshooting

| Problem | Command | What to Look For |
|---|---|---|
| Pod stuck in `CrashLoopBackOff` | `kubectl logs <pod> -n myapp --previous` | Application errors, missing env vars |
| Pod stuck in `Pending` | `kubectl describe pod <pod> -n myapp` | Insufficient resources, no nodes |
| Readiness probe failing | `kubectl describe pod <pod> -n myapp` | Events section shows probe failures |
| Ingress not working | `kubectl describe ingress myapp-ingress -n myapp` | Check annotations, service names |
| Can't connect to DB | `kubectl exec -it <backend-pod> -n myapp -- env` | Verify env vars are set correctly |

> [!abstract] Key Takeaways
> - Multi-stage Docker builds keep images small and secure
> - ConfigMaps and Secrets separate configuration from code
> - Liveness probes restart broken containers; readiness probes manage traffic routing
> - Service DNS (`<svc-name>.<namespace>.svc.cluster.local`) enables service discovery
> - Ingress routes external traffic to internal Services by path
> - Scaling is a single command; load balancing is automatic via the Service
