---
title: Kubernetes Study
date: 2026-04-07
tags:
  - kubernetes
  - docker
  - devops
  - helm
  - argocd
  - github-actions
  - study
aliases:
  - K8s Study
  - Kubernetes Course
status: in-progress
---

# Kubernetes Study

> [!tip] Why Kubernetes?
> Kubernetes (K8s) is the industry standard for container orchestration. It automates deployment, scaling, and management of containerized applications. Essential for any backend/DevOps engineer.

> [!abstract] Course Structure
> **Reference** - This note covers all core concepts with real-world context.
> **Practice** - Progressive drills across 8 topic areas. See [[#Practice Drills]].
> **Workflows** - 5 real-world projects combining all tools. See [[#Workflow Projects]].
> **Local Lab** - All exercises run on **minikube** on your local machine.

> [!info] Local Lab Setup
> ```bash
> # Install minikube
> brew install minikube
>
> # Start cluster (Docker driver recommended)
> minikube start --driver=docker --cpus=4 --memory=8192
>
> # Enable common addons
> minikube addons enable ingress
> minikube addons enable metrics-server
> minikube addons enable dashboard
>
> # Verify
> kubectl cluster-info
> kubectl get nodes
>
> # Access dashboard
> minikube dashboard
>
> # Docker environment (use minikube's Docker daemon)
> eval $(minikube docker-env)
> ```

---

## Part 1: Docker Foundations

### Why Docker First?

Kubernetes runs containers. You must understand Docker before K8s makes sense. Docker = build and package. K8s = run and orchestrate.

### Dockerfile

```dockerfile
# Multi-stage build (production pattern)
# Stage 1: Build
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /server .

# Stage 2: Runtime (minimal image)
FROM alpine:3.19
RUN apk --no-cache add ca-certificates
COPY --from=builder /server /server
EXPOSE 8080
USER nobody:nobody
ENTRYPOINT ["/server"]
```

> [!tip] Dockerfile Best Practices
> - **Order matters** - put least-changing layers first (deps before code)
> - **Multi-stage builds** - build in fat image, run in slim image
> - **Don't run as root** - use `USER nobody` or create a specific user
> - **Use specific tags** - `golang:1.22-alpine` not `golang:latest`
> - **`.dockerignore`** - exclude `.git`, `node_modules`, test files
> - **One process per container** - don't run multiple services in one container

### Key Docker Commands

```bash
# Build
docker build -t myapp:v1 .
docker build -t myapp:v1 -f Dockerfile.prod .

# Run
docker run -d -p 8080:8080 --name myapp myapp:v1
docker run -it --rm alpine sh              # interactive, auto-remove

# Image management
docker images
docker tag myapp:v1 registry.io/myapp:v1
docker push registry.io/myapp:v1
docker rmi myapp:v1

# Container management
docker ps                   # running containers
docker ps -a                # all containers
docker logs myapp           # view logs
docker logs -f myapp        # follow logs
docker exec -it myapp sh   # shell into container
docker stop myapp
docker rm myapp

# Cleanup
docker system prune -a      # remove all unused
```

### Docker Compose (Local Development)

```yaml
# docker-compose.yml
version: "3.8"
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=db
      - DB_PORT=5432
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./:/app  # hot reload for development

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d myapp"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

```bash
docker compose up -d          # start all services
docker compose down           # stop and remove
docker compose logs -f app    # follow app logs
docker compose ps             # status
docker compose exec app sh    # shell into app
```

**Practice:** [[Kubernetes/Docker/Practice - Docker Fundamentals]]

---

## Part 2: Kubernetes Core Concepts

### Architecture Overview

```
                    ┌─────────────────────────────────────────┐
                    │              Control Plane               │
                    │  ┌───────────┐  ┌──────────────────┐   │
                    │  │ API Server│  │   etcd (state DB) │   │
                    │  └─────┬─────┘  └──────────────────┘   │
                    │        │                                 │
                    │  ┌─────┴─────┐  ┌──────────────────┐   │
                    │  │ Scheduler │  │Controller Manager │   │
                    │  └───────────┘  └──────────────────┘   │
                    └─────────────────────────────────────────┘
                                     │
                    ┌────────────────┼────────────────────┐
                    │                │                     │
               ┌────┴────┐    ┌────┴─────┐    ┌─────┴────┐
               │  Node 1  │    │  Node 2   │    │  Node 3   │
               │ ┌──────┐ │    │ ┌──────┐  │    │ ┌──────┐  │
               │ │kubelet│ │    │ │kubelet│  │    │ │kubelet│  │
               │ └──────┘ │    │ └──────┘  │    │ └──────┘  │
               │ ┌──────┐ │    │ ┌──────┐  │    │ ┌──────┐  │
               │ │kube-  │ │    │ │kube-  │  │    │ │kube-  │  │
               │ │proxy  │ │    │ │proxy  │  │    │ │proxy  │  │
               │ └──────┘ │    │ └──────┘  │    │ └──────┘  │
               │ ┌──┐┌──┐ │    │ ┌──┐┌──┐  │    │ ┌──┐┌──┐  │
               │ │P1││P2│ │    │ │P3││P4│  │    │ │P5││P6│  │
               │ └──┘└──┘ │    │ └──┘└──┘  │    │ └──┘└──┘  │
               └──────────┘    └──────────┘    └──────────┘
```

| Component | Role |
|---|---|
| **API Server** | Front door to K8s. All kubectl commands go here |
| **etcd** | Key-value store holding ALL cluster state |
| **Scheduler** | Assigns pods to nodes based on resources/constraints |
| **Controller Manager** | Runs control loops (Deployment, ReplicaSet, etc.) |
| **kubelet** | Agent on each node, manages pods on that node |
| **kube-proxy** | Network proxy on each node, handles Service routing |

### Pods

The smallest deployable unit. Usually 1 container per pod.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
    env: dev
spec:
  containers:
    - name: myapp
      image: myapp:v1
      ports:
        - containerPort: 8080
      resources:
        requests:
          cpu: "100m"        # 0.1 CPU core
          memory: "128Mi"    # 128 MB
        limits:
          cpu: "500m"
          memory: "256Mi"
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 3
        periodSeconds: 5
      env:
        - name: APP_ENV
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
  restartPolicy: Always
```

> [!warning] Never deploy bare Pods in production
> Always use a Deployment (or StatefulSet/DaemonSet). Bare pods don't get rescheduled if a node dies.

### Deployments & ReplicaSets

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # 1 extra pod during update
      maxUnavailable: 0     # never go below desired count
  template:
    metadata:
      labels:
        app: myapp
        version: v1
    spec:
      containers:
        - name: myapp
          image: myapp:v1
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
```

```bash
# Deployment operations
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl describe deployment myapp
kubectl rollout status deployment/myapp
kubectl rollout history deployment/myapp
kubectl rollout undo deployment/myapp              # rollback to previous
kubectl rollout undo deployment/myapp --to-revision=2
kubectl scale deployment myapp --replicas=5
kubectl set image deployment/myapp myapp=myapp:v2  # update image
```

> [!info] Deployment -> ReplicaSet -> Pod
> A Deployment manages ReplicaSets. Each rollout creates a new ReplicaSet. The old ReplicaSet is kept (scaled to 0) for rollback. You rarely interact with ReplicaSets directly.

### Services

Stable networking endpoint for a set of Pods (which have ephemeral IPs).

```yaml
# ClusterIP (internal only - default)
apiVersion: v1
kind: Service
metadata:
  name: myapp-svc
spec:
  type: ClusterIP
  selector:
    app: myapp            # matches pod labels
  ports:
    - port: 80            # service port
      targetPort: 8080    # container port
      protocol: TCP

---
# NodePort (external via node IP:port)
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080     # 30000-32767 range

---
# LoadBalancer (cloud provider creates external LB)
apiVersion: v1
kind: Service
metadata:
  name: myapp-lb
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080

---
# Headless Service (for StatefulSets, direct pod DNS)
apiVersion: v1
kind: Service
metadata:
  name: myapp-headless
spec:
  clusterIP: None
  selector:
    app: myapp
  ports:
    - port: 8080
```

> [!tip] Service Types Cheat Sheet
> | Type | Accessible From | Use Case |
> |---|---|---|
> | **ClusterIP** | Inside cluster only | Backend services, databases |
> | **NodePort** | External via `<NodeIP>:<NodePort>` | Dev/testing, minikube |
> | **LoadBalancer** | External via cloud LB | Production public services |
> | **ExternalName** | DNS alias (CNAME) | External service reference |
> | **Headless** | Direct pod DNS | StatefulSets, service discovery |

### DNS in Kubernetes

```
# Service DNS patterns:
<service-name>.<namespace>.svc.cluster.local
myapp-svc.default.svc.cluster.local

# Short forms (within same namespace):
myapp-svc

# Cross-namespace:
myapp-svc.other-namespace

# Headless service (per-pod DNS for StatefulSets):
<pod-name>.<service-name>.<namespace>.svc.cluster.local
mydb-0.mydb-headless.default.svc.cluster.local
```

### Ingress

Layer 7 (HTTP) routing - host/path-based routing to Services.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
    - host: admin.myapp.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: admin-svc
                port:
                  number: 80
  tls:
    - hosts:
        - myapp.local
      secretName: myapp-tls
```

```bash
# minikube: enable ingress controller
minikube addons enable ingress

# Add to /etc/hosts for local testing
echo "$(minikube ip) myapp.local admin.myapp.local" | sudo tee -a /etc/hosts
```

**Practice:** [[Kubernetes/Core/Practice - Pods and Deployments]] | [[Kubernetes/Core/Practice - Services and Networking]]

---

## Part 3: Configuration & Storage

### Namespaces

```bash
kubectl create namespace staging
kubectl create namespace production

# Set default namespace for context
kubectl config set-context --current --namespace=staging

# Operations in specific namespace
kubectl get pods -n production
kubectl get all --all-namespaces  # or -A
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
```

### ConfigMaps

```yaml
# From YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  config.yaml: |
    database:
      host: db-svc
      port: 5432
    cache:
      host: redis-svc
      port: 6379
```

```bash
# From literal values
kubectl create configmap app-config --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=info

# From file
kubectl create configmap nginx-conf --from-file=nginx.conf

# From directory
kubectl create configmap configs --from-file=./config-dir/
```

```yaml
# Using ConfigMap in Pod
spec:
  containers:
    - name: myapp
      # As environment variables
      envFrom:
        - configMapRef:
            name: app-config
      # Or specific keys
      env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
      # As mounted file
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
          readOnly: true
  volumes:
    - name: config-vol
      configMap:
        name: app-config
```

### Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=          # base64 encoded "admin"
  password: cEBzc3dvcmQxMjM=  # base64 encoded "p@ssword123"
---
# stringData for plain text (encoded automatically)
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  username: admin
  password: "p@ssword123"
```

```bash
# Create from literal
kubectl create secret generic db-secret --from-literal=username=admin --from-literal=password='p@ssword123'

# Create TLS secret
kubectl create secret tls myapp-tls --cert=tls.crt --key=tls.key

# Create Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=registry.io \
  --docker-username=user \
  --docker-password=pass
```

> [!warning] Secrets are NOT encrypted by default
> Secrets are only base64-encoded (not encrypted!) in etcd. For real security:
> - Enable encryption at rest in etcd
> - Use External Secrets Operator + Vault/AWS Secrets Manager
> - Use Sealed Secrets for GitOps
> - RBAC to restrict Secret access

### Persistent Volumes

```yaml
# PersistentVolume (cluster-level resource)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce     # RWO: single node read-write
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  hostPath:
    path: /data/pv-001

---
# PersistentVolumeClaim (namespace-level, requests storage)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage

---
# Using PVC in Pod
spec:
  containers:
    - name: db
      image: postgres:16
      volumeMounts:
        - name: db-storage
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: db-storage
      persistentVolumeClaim:
        claimName: db-pvc
```

> [!tip] Access Modes
> | Mode | Abbreviation | Description |
> |---|---|---|
> | ReadWriteOnce | RWO | Single node read-write |
> | ReadOnlyMany | ROX | Multiple nodes read-only |
> | ReadWriteMany | RWX | Multiple nodes read-write |

**Practice:** [[Kubernetes/Core/Practice - Configuration and Storage]]

---

## Part 4: Workloads & Scheduling

### Jobs & CronJobs

```yaml
# One-time Job
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  backoffLimit: 3          # retry 3 times on failure
  activeDeadlineSeconds: 300
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:v1
          command: ["./migrate", "up"]
      restartPolicy: Never

---
# CronJob (scheduled)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-backup
spec:
  schedule: "0 2 * * *"    # 2 AM daily
  concurrencyPolicy: Forbid # don't overlap
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: backup-tool:v1
              command: ["./backup.sh"]
          restartPolicy: OnFailure
```

### DaemonSet

Runs one pod per node (log collectors, monitoring agents, network plugins).

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      containers:
        - name: fluentd
          image: fluentd:v1.16
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

### StatefulSet

For stateful apps (databases, message queues) needing stable network identity and persistent storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless   # required headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:            # auto-creates PVCs per pod
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

> [!info] StatefulSet Guarantees
> - **Stable network identity**: `postgres-0`, `postgres-1`, `postgres-2`
> - **Ordered deployment**: pods created 0 -> 1 -> 2, deleted 2 -> 1 -> 0
> - **Stable storage**: each pod gets its own PVC that persists across rescheduling
> - **DNS**: `postgres-0.postgres-headless.default.svc.cluster.local`

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
# Requires metrics-server
minikube addons enable metrics-server

kubectl autoscale deployment myapp --cpu-percent=70 --min=2 --max=10
kubectl get hpa
kubectl top pods
kubectl top nodes
```

### Resource Management

```yaml
spec:
  containers:
    - name: myapp
      resources:
        requests:         # scheduler uses this to place pod
          cpu: "250m"     # 0.25 core (guaranteed minimum)
          memory: "256Mi" # 256 MB
        limits:           # kubelet enforces this ceiling
          cpu: "500m"     # 0.5 core (throttled beyond this)
          memory: "512Mi" # killed (OOMKilled) beyond this
```

> [!warning] CPU vs Memory Limits
> - **CPU**: pod is **throttled** when exceeding limit (slowed down, not killed)
> - **Memory**: pod is **OOMKilled** when exceeding limit (killed and restarted)
> - Always set memory limits. CPU limits are debated - some teams skip them.

### RBAC (Role-Based Access Control)

```yaml
# Role (namespace-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "watch", "list"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
  namespace: default

---
# ClusterRole (cluster-wide)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
```

**Practice:** [[Kubernetes/Core/Practice - Workloads and Scheduling]]

---

## Part 5: Helm Charts

Helm is the package manager for Kubernetes. Charts = reusable, templated K8s manifests.

### Helm Basics

```bash
# Install Helm
brew install helm

# Add repos
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Search
helm search repo postgres
helm search hub wordpress     # Artifact Hub

# Install a chart
helm install my-postgres bitnami/postgresql \
  --set auth.postgresPassword=mypass \
  --namespace databases --create-namespace

# List releases
helm list -A

# Upgrade
helm upgrade my-postgres bitnami/postgresql \
  --set auth.postgresPassword=newpass

# Rollback
helm rollback my-postgres 1    # revision number

# Uninstall
helm uninstall my-postgres -n databases

# Dry run / template
helm install my-release ./mychart --dry-run --debug
helm template my-release ./mychart
```

### Chart Structure

```
mychart/
  Chart.yaml          # metadata (name, version, dependencies)
  values.yaml         # default configuration values
  values-prod.yaml    # environment-specific overrides
  templates/
    _helpers.tpl      # template helpers (named templates)
    deployment.yaml
    service.yaml
    ingress.yaml
    configmap.yaml
    secret.yaml
    hpa.yaml
    NOTES.txt         # post-install instructions
  charts/             # dependency charts
  .helmignore
```

### Chart.yaml

```yaml
apiVersion: v2
name: myapp
description: My application Helm chart
type: application
version: 0.1.0          # chart version
appVersion: "1.0.0"     # app version

dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled
  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### values.yaml

```yaml
# values.yaml - default configuration
replicaCount: 2

image:
  repository: myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false
  className: nginx
  hosts:
    - host: myapp.local
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilization: 70

postgresql:
  enabled: true
  auth:
    postgresPassword: ""    # override at install time

env:
  APP_ENV: production
  LOG_LEVEL: info
```

### Templating

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8080
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
```

### Helpers (_helpers.tpl)

```yaml
# templates/_helpers.tpl
{{- define "mychart.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{- define "mychart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name (include "mychart.name" .) | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}

{{- define "mychart.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{- define "mychart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mychart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

> [!tip] Helm Template Functions
> | Function | Example | Description |
> |---|---|---|
> | `default` | `default "nginx" .Values.image` | Fallback value |
> | `quote` | `value: {{ .Values.env \| quote }}` | Wrap in quotes |
> | `toYaml` | `{{ toYaml .Values.x \| nindent 4 }}` | Convert to YAML |
> | `include` | `{{ include "name" . }}` | Call named template |
> | `required` | `{{ required "msg" .Values.x }}` | Fail if empty |
> | `tpl` | `{{ tpl .Values.x . }}` | Render string as template |
> | `lookup` | `{{ lookup "v1" "Secret" "ns" "name" }}` | Query cluster |

**Practice:** [[Kubernetes/Helm/Practice - Helm Charts]]

---

## Part 6: ArgoCD (GitOps)

ArgoCD implements GitOps - Git is the single source of truth for your cluster state. Changes to Git automatically sync to Kubernetes.

### Core Concepts

```
     Developer                Git Repository              ArgoCD                  K8s Cluster
        │                         │                        │                         │
        ├── push code ──────────>│                        │                         │
        │                         ├── ArgoCD watches ────>│                         │
        │                         │                        ├── detect diff ────────>│
        │                         │                        │   (desired vs live)     │
        │                         │                        ├── sync (apply) ───────>│
        │                         │                        │                         │
        │                         │                        ├── health check ───────>│
        │                         │                        │<── healthy ────────────│
```

### Install ArgoCD on Minikube

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=120s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Install CLI
brew install argocd

# Login
argocd login localhost:8080 --username admin --password <password> --insecure
```

### Application CRD

```yaml
# application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

  source:
    repoURL: https://github.com/yourname/k8s-manifests.git
    targetRevision: main
    path: apps/myapp/overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true           # delete resources removed from Git
      selfHeal: true        # revert manual changes
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Helm-based Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourname/helm-charts.git
    targetRevision: main
    path: charts/myapp
    helm:
      valueFiles:
        - values-production.yaml
      parameters:
        - name: image.tag
          value: "v1.2.3"
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### App of Apps Pattern

One "root" Application that manages all other Applications.

```yaml
# root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/yourname/gitops-config.git
    targetRevision: main
    path: apps           # directory containing Application YAMLs
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```
# Repository structure for App of Apps
gitops-config/
  apps/
    myapp.yaml           # Application CR for myapp
    api-gateway.yaml     # Application CR for API gateway
    monitoring.yaml      # Application CR for monitoring stack
    database.yaml        # Application CR for database
```

### ApplicationSet (Dynamic Application Generation)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-apps
  namespace: argocd
spec:
  generators:
    # Generate one Application per directory in the repo
    - git:
        repoURL: https://github.com/yourname/gitops-config.git
        revision: main
        directories:
          - path: "envs/*/apps/*"
  template:
    metadata:
      name: "{{path.basename}}-{{path[1]}}"
    spec:
      project: default
      source:
        repoURL: https://github.com/yourname/gitops-config.git
        targetRevision: main
        path: "{{path}}"
      destination:
        server: https://kubernetes.default.svc
        namespace: "{{path.basename}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

### Sync Waves & Hooks

Control the order of resource deployment.

```yaml
# Deploy namespace first (wave -1), then app (wave 0), then ingress (wave 1)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"   # lower = earlier

---
# Pre-sync hook (runs before sync)
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp:v1
          command: ["./migrate", "up"]
      restartPolicy: Never
```

```bash
# ArgoCD CLI
argocd app list
argocd app get myapp
argocd app sync myapp
argocd app diff myapp
argocd app history myapp
argocd app rollback myapp <revision>
argocd app delete myapp
```

**Practice:** [[Kubernetes/ArgoCD/Practice - GitOps with ArgoCD]]

---

## Part 7: GitHub Actions (CI/CD)

### Workflow Basics

```yaml
# .github/workflows/ci.yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

permissions:
  contents: read
  packages: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"

      - name: Run tests
        run: go test -race -coverprofile=coverage.out ./...

      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage.out

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: golangci/golangci-lint-action@v6
        with:
          version: latest

  build:
    needs: [test, lint]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Deploy to Kubernetes

```yaml
# .github/workflows/deploy.yaml
name: Deploy

on:
  workflow_run:
    workflows: ["CI"]
    branches: [main]
    types: [completed]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4

      - name: Configure kubeconfig
        run: |
          echo "${{ secrets.KUBE_CONFIG }}" | base64 -d > $HOME/.kube/config

      - name: Deploy with Helm
        run: |
          helm upgrade --install myapp ./charts/myapp \
            --namespace production \
            --create-namespace \
            --set image.tag=${{ github.sha }} \
            --set image.repository=ghcr.io/${{ github.repository }} \
            --wait --timeout 300s

      - name: Verify deployment
        run: |
          kubectl rollout status deployment/myapp -n production --timeout=120s
```

### GitOps Deploy (Update Image Tag in Config Repo)

```yaml
# .github/workflows/gitops-deploy.yaml
name: GitOps Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build and push image
        # ... (same as above)

      - name: Update GitOps repo
        uses: actions/checkout@v4
        with:
          repository: yourname/gitops-config
          token: ${{ secrets.GITOPS_PAT }}
          path: gitops-config

      - name: Update image tag
        run: |
          cd gitops-config
          # Update the image tag in values file
          yq e '.image.tag = "${{ github.sha }}"' -i apps/myapp/values-production.yaml
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add .
          git commit -m "chore: update myapp image to ${{ github.sha }}"
          git push
        # ArgoCD picks up the change automatically!
```

### Matrix Builds

```yaml
jobs:
  test:
    strategy:
      matrix:
        go-version: ["1.21", "1.22"]
        os: [ubuntu-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: ${{ matrix.go-version }}
      - run: go test ./...
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-build.yaml
name: Reusable Build

on:
  workflow_call:
    inputs:
      image-name:
        required: true
        type: string
      dockerfile:
        required: false
        type: string
        default: Dockerfile
    secrets:
      registry-password:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build and push
        run: |
          echo "${{ secrets.registry-password }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker build -t ghcr.io/${{ inputs.image-name }}:${{ github.sha }} -f ${{ inputs.dockerfile }} .
          docker push ghcr.io/${{ inputs.image-name }}:${{ github.sha }}
```

```yaml
# Calling workflow
jobs:
  build-app:
    uses: ./.github/workflows/reusable-build.yaml
    with:
      image-name: myorg/myapp
    secrets:
      registry-password: ${{ secrets.GITHUB_TOKEN }}
```

> [!tip] GitHub Actions Best Practices
> - **Cache dependencies** - use `actions/cache` or built-in caching in setup actions
> - **Use environments** - `environment: production` for approval gates
> - **Pin action versions** - use SHA, not `@main` (supply chain security)
> - **Minimize secrets** - use OIDC for cloud providers when possible
> - **Fail fast** - `strategy.fail-fast: true` on matrix builds
> - **Reusable workflows** - DRY principle for common CI/CD patterns

**Practice:** [[Kubernetes/GitHub Actions/Practice - CI CD Pipelines]]

---

## Essential kubectl Commands

```bash
# Get resources
kubectl get pods                    # list pods
kubectl get pods -o wide            # more details (node, IP)
kubectl get pods -o yaml            # full YAML
kubectl get all                     # pods, services, deployments...
kubectl get all -A                  # all namespaces

# Describe (detailed info + events)
kubectl describe pod myapp-xxx
kubectl describe node minikube

# Logs
kubectl logs myapp-xxx              # current logs
kubectl logs myapp-xxx -f           # follow
kubectl logs myapp-xxx -c sidecar   # specific container
kubectl logs myapp-xxx --previous   # previous crashed container

# Exec into pod
kubectl exec -it myapp-xxx -- sh
kubectl exec -it myapp-xxx -c sidecar -- sh

# Port forward
kubectl port-forward svc/myapp 8080:80
kubectl port-forward pod/myapp-xxx 8080:8080

# Apply / Delete
kubectl apply -f manifest.yaml
kubectl apply -f ./manifests/       # apply all in directory
kubectl delete -f manifest.yaml
kubectl delete pod myapp-xxx

# Debug
kubectl get events --sort-by=.lastTimestamp
kubectl top pods                    # resource usage
kubectl top nodes
kubectl run debug --rm -it --image=busybox -- sh  # temporary debug pod

# Dry run (generate YAML)
kubectl create deployment myapp --image=myapp:v1 --dry-run=client -o yaml > deployment.yaml
kubectl create service clusterip myapp --tcp=80:8080 --dry-run=client -o yaml > service.yaml

# Context management
kubectl config get-contexts
kubectl config use-context minikube
kubectl config set-context --current --namespace=production
```

---

## Practice Drills

> [!example] Progressive Practice
> Each topic has 12-15 hands-on exercises. Each builds on the previous - you'll create actual resources on minikube. Write every YAML from scratch!

| # | Topic | Link | Problems |
|---|---|---|---|
| 01 | Docker Fundamentals | [[Kubernetes/Docker/Practice - Docker Fundamentals]] | 14 |
| 02 | Pods & Deployments | [[Kubernetes/Core/Practice - Pods and Deployments]] | 15 |
| 03 | Services & Networking | [[Kubernetes/Core/Practice - Services and Networking]] | 13 |
| 04 | Configuration & Storage | [[Kubernetes/Core/Practice - Configuration and Storage]] | 13 |
| 05 | Workloads & Scheduling | [[Kubernetes/Core/Practice - Workloads and Scheduling]] | 13 |
| 06 | Helm Charts | [[Kubernetes/Helm/Practice - Helm Charts]] | 14 |
| 07 | GitOps with ArgoCD | [[Kubernetes/ArgoCD/Practice - GitOps with ArgoCD]] | 12 |
| 08 | CI/CD with GitHub Actions | [[Kubernetes/GitHub Actions/Practice - CI CD Pipelines]] | 13 |

---

## Workflow Projects

> [!success] After finishing all practice drills, implement these real-world scenarios. Each project chains together Docker + K8s + Helm + ArgoCD + GitHub Actions.

| # | Project | Link | Concepts Used |
|---|---|---|---|
| 01 | Deploy Microservices | [[Kubernetes/Workflows/01 - Microservice Deployment]] | Docker, Deployment, Service, Ingress, ConfigMap, Secrets |
| 02 | Helm Multi-Tier App | [[Kubernetes/Workflows/02 - Helm Multi-Tier App]] | Helm chart, dependencies, values, templates, environments |
| 03 | GitOps Pipeline | [[Kubernetes/Workflows/03 - GitOps Pipeline]] | ArgoCD, App of Apps, sync waves, hooks, Sealed Secrets |
| 04 | Full CI/CD Pipeline | [[Kubernetes/Workflows/04 - Full CI CD Pipeline]] | GitHub Actions + Docker + Helm + ArgoCD end-to-end |
| 05 | Advanced Deployments | [[Kubernetes/Workflows/05 - Advanced Deployment Strategies]] | Blue/Green, Canary, rollback, HPA, monitoring |

---

## Resources

- [Kubernetes Official Docs](https://kubernetes.io/docs/)
- [Kubernetes the Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [Helm Docs](https://helm.sh/docs/)
- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [GitHub Actions Docs](https://docs.github.com/en/actions)
- [Docker Docs](https://docs.docker.com/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [CNCF Landscape](https://landscape.cncf.io/)
