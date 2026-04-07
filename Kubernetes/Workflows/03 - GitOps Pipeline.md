---
title: "K8s Workflow: GitOps Pipeline"
date: 2026-04-07
tags:
  - kubernetes
  - workflow
  - project
  - argocd
  - gitops
  - kustomize
  - sealed-secrets
parent: "[[Kubernetes Study]]"
---

# Workflow 03: GitOps Pipeline

Set up a complete GitOps workflow with ArgoCD on minikube. All cluster state is driven by Git -- changes are made by committing to the config repo, and ArgoCD syncs the cluster automatically.

## Architecture Overview

```
  Developer                    Git Repo                        minikube
  ┌───────┐               ┌──────────────┐              ┌──────────────────┐
  │       │  git push      │ gitops-config│   watches     │  ArgoCD Server   │
  │       │──────────────► │              │◄──────────────│                  │
  │       │                │  apps/       │               │  ┌────────────┐  │
  │       │                │  base/       │   auto-sync   │  │ App of Apps│  │
  │       │                │  overlays/   │──────────────►│  │  root-app  │  │
  └───────┘                └──────────────┘               │  └─────┬──────┘  │
                                                          │        │         │
                                                          │   ┌────▼─────┐   │
                                                          │   │ frontend │   │
                                                          │   │ backend  │   │
                                                          │   │ database │   │
                                                          │   └──────────┘   │
                                                          └──────────────────┘

  Sync Wave Order:
  ─1 ──► 0 ──► 1 ──► 2 ──► 3
  NS    Secrets  DB   Backend  Frontend
```

## Repository Structure

```
gitops-config/
├── apps/                          # ArgoCD Application CRDs
│   ├── root-app.yaml              # App of Apps (parent)
│   ├── frontend.yaml              # Frontend Application
│   ├── backend.yaml               # Backend Application
│   └── database.yaml              # Database Application
├── base/                          # Base K8s manifests (Kustomize)
│   ├── frontend/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── backend/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── configmap.yaml
│   ├── database/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── secret.yaml
│   └── namespace.yaml
├── overlays/                      # Environment-specific patches
│   ├── dev/
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   ├── backend-patch.yaml
│   │   └── frontend-patch.yaml
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   ├── backend-patch.yaml
│   │   └── frontend-patch.yaml
│   └── production/
│       ├── kustomization.yaml
│       ├── backend-patch.yaml
│       └── frontend-patch.yaml
└── sealed-secrets/
    └── db-sealed-secret.yaml
```

## Prerequisites

- minikube running with Ingress addon enabled
- kubectl configured
- Images from [[01 - Microservice Deployment]] loaded in minikube

```bash
minikube start --cpus=4 --memory=4096
minikube addons enable ingress
```

---

## Step 1: Set Up the GitOps Config Repository

For this exercise, we use a local Git repository. In production, this would be a GitHub/GitLab repo.

```bash
mkdir -p gitops-config && cd gitops-config
git init
mkdir -p apps base/{frontend,backend,database} overlays/{dev,staging,production} sealed-secrets
```

### Base Manifests

> [!success]- Full Solution: `base/namespace.yaml`
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: myapp
>   annotations:
>     argocd.argoproj.io/sync-wave: "-1"
> ```

> [!success]- Full Solution: `base/database/deployment.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: postgres
>   labels:
>     app: postgres
>   annotations:
>     argocd.argoproj.io/sync-wave: "1"
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
>               value: appdb
>             - name: POSTGRES_USER
>               value: appuser
>             - name: POSTGRES_PASSWORD
>               valueFrom:
>                 secretKeyRef:
>                   name: db-secret
>                   key: POSTGRES_PASSWORD
>           resources:
>             requests:
>               memory: "128Mi"
>               cpu: "250m"
>             limits:
>               memory: "256Mi"
>               cpu: "500m"
>           volumeMounts:
>             - name: postgres-data
>               mountPath: /var/lib/postgresql/data
>       volumes:
>         - name: postgres-data
>           emptyDir: {}
> ```

> [!success]- Full Solution: `base/database/service.yaml`
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: postgres-svc
>   annotations:
>     argocd.argoproj.io/sync-wave: "1"
> spec:
>   selector:
>     app: postgres
>   ports:
>     - port: 5432
>       targetPort: 5432
>   type: ClusterIP
> ```

> [!success]- Full Solution: `base/database/secret.yaml`
> ```yaml
> apiVersion: v1
> kind: Secret
> metadata:
>   name: db-secret
>   annotations:
>     argocd.argoproj.io/sync-wave: "0"
> type: Opaque
> stringData:
>   POSTGRES_PASSWORD: apppassword
>   DB_PASSWORD: apppassword
> ```

> [!success]- Full Solution: `base/database/kustomization.yaml`
> ```yaml
> apiVersion: kustomize.config.k8s.io/v1beta1
> kind: Kustomization
> 
> resources:
>   - deployment.yaml
>   - service.yaml
>   - secret.yaml
> ```

> [!success]- Full Solution: `base/backend/configmap.yaml`
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: backend-config
>   annotations:
>     argocd.argoproj.io/sync-wave: "0"
> data:
>   DB_HOST: "postgres-svc"
>   DB_PORT: "5432"
>   DB_NAME: "appdb"
>   DB_USER: "appuser"
>   PORT: "8080"
>   LOG_LEVEL: "info"
> ```

> [!success]- Full Solution: `base/backend/deployment.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: backend
>   labels:
>     app: backend
>   annotations:
>     argocd.argoproj.io/sync-wave: "2"
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
>           readinessProbe:
>             httpGet:
>               path: /api/ready
>               port: 8080
>             initialDelaySeconds: 5
>             periodSeconds: 5
>           resources:
>             requests:
>               memory: "64Mi"
>               cpu: "100m"
>             limits:
>               memory: "128Mi"
>               cpu: "250m"
> ```

> [!success]- Full Solution: `base/backend/service.yaml`
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: backend-svc
>   annotations:
>     argocd.argoproj.io/sync-wave: "2"
> spec:
>   selector:
>     app: backend
>   ports:
>     - port: 8080
>       targetPort: 8080
>   type: ClusterIP
> ```

> [!success]- Full Solution: `base/backend/kustomization.yaml`
> ```yaml
> apiVersion: kustomize.config.k8s.io/v1beta1
> kind: Kustomization
> 
> resources:
>   - deployment.yaml
>   - service.yaml
>   - configmap.yaml
> ```

> [!success]- Full Solution: `base/frontend/deployment.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: frontend
>   labels:
>     app: frontend
>   annotations:
>     argocd.argoproj.io/sync-wave: "3"
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
> ```

> [!success]- Full Solution: `base/frontend/service.yaml`
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: frontend-svc
>   annotations:
>     argocd.argoproj.io/sync-wave: "3"
> spec:
>   selector:
>     app: frontend
>   ports:
>     - port: 80
>       targetPort: 80
>   type: ClusterIP
> ```

> [!success]- Full Solution: `base/frontend/kustomization.yaml`
> ```yaml
> apiVersion: kustomize.config.k8s.io/v1beta1
> kind: Kustomization
> 
> resources:
>   - deployment.yaml
>   - service.yaml
> ```

---

## Step 2: Environment Overlays

> [!info] Kustomize Overlays
> Overlays let you customize base manifests per environment without copying them. Use `patchesStrategicMerge` for partial updates to existing resources, `namespace` to redirect all resources, and `images` to override container images.

> [!success]- Full Solution: `overlays/dev/kustomization.yaml`
> ```yaml
> apiVersion: kustomize.config.k8s.io/v1beta1
> kind: Kustomization
> 
> namespace: myapp-dev
> 
> resources:
>   - namespace.yaml
>   - ../../base/database
>   - ../../base/backend
>   - ../../base/frontend
> 
> patches:
>   - path: backend-patch.yaml
>   - path: frontend-patch.yaml
> 
> images:
>   - name: myapp-backend
>     newTag: v1
>   - name: myapp-frontend
>     newTag: v1
> 
> commonLabels:
>   environment: dev
> ```

> [!success]- Full Solution: `overlays/dev/namespace.yaml`
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: myapp-dev
>   annotations:
>     argocd.argoproj.io/sync-wave: "-1"
> ```

> [!success]- Full Solution: `overlays/dev/backend-patch.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: backend
> spec:
>   replicas: 1
>   template:
>     spec:
>       containers:
>         - name: backend
>           resources:
>             requests:
>               memory: "64Mi"
>               cpu: "100m"
>             limits:
>               memory: "128Mi"
>               cpu: "250m"
> ```

> [!success]- Full Solution: `overlays/dev/frontend-patch.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: frontend
> spec:
>   replicas: 1
> ```

> [!success]- Full Solution: `overlays/staging/kustomization.yaml`
> ```yaml
> apiVersion: kustomize.config.k8s.io/v1beta1
> kind: Kustomization
> 
> namespace: myapp-staging
> 
> resources:
>   - ../../base/database
>   - ../../base/backend
>   - ../../base/frontend
> 
> patches:
>   - path: backend-patch.yaml
>   - path: frontend-patch.yaml
> 
> images:
>   - name: myapp-backend
>     newTag: v1
>   - name: myapp-frontend
>     newTag: v1
> 
> commonLabels:
>   environment: staging
> ```

> [!success]- Full Solution: `overlays/staging/backend-patch.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: backend
> spec:
>   replicas: 2
>   template:
>     spec:
>       containers:
>         - name: backend
>           resources:
>             requests:
>               memory: "128Mi"
>               cpu: "200m"
>             limits:
>               memory: "256Mi"
>               cpu: "500m"
> ```

> [!success]- Full Solution: `overlays/staging/frontend-patch.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: frontend
> spec:
>   replicas: 2
> ```

> [!success]- Full Solution: `overlays/production/kustomization.yaml`
> ```yaml
> apiVersion: kustomize.config.k8s.io/v1beta1
> kind: Kustomization
> 
> namespace: myapp-production
> 
> resources:
>   - ../../base/database
>   - ../../base/backend
>   - ../../base/frontend
> 
> patches:
>   - path: backend-patch.yaml
>   - path: frontend-patch.yaml
> 
> images:
>   - name: myapp-backend
>     newTag: v1
>   - name: myapp-frontend
>     newTag: v1
> 
> commonLabels:
>   environment: production
> ```

> [!success]- Full Solution: `overlays/production/backend-patch.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: backend
> spec:
>   replicas: 3
>   template:
>     spec:
>       containers:
>         - name: backend
>           resources:
>             requests:
>               memory: "256Mi"
>               cpu: "500m"
>             limits:
>               memory: "512Mi"
>               cpu: "1000m"
> ```

> [!success]- Full Solution: `overlays/production/frontend-patch.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: frontend
> spec:
>   replicas: 3
> ```

Commit the base config:

```bash
cd gitops-config
git add -A
git commit -m "Initial gitops config structure"
```

---

## Step 3: Install ArgoCD on Minikube

```bash
# Create namespace and install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo  # newline

# Access ArgoCD UI via port-forward
kubectl port-forward svc/argocd-server -n argocd 8443:443 &
```

> [!tip] ArgoCD CLI
> Install the ArgoCD CLI for command-line management:
> ```bash
> # macOS
> brew install argocd
> 
> # Login (use admin + the password from above)
> argocd login localhost:8443 --insecure --username admin --password <password>
> ```

Access the ArgoCD dashboard at `https://localhost:8443` (accept the self-signed certificate).

---

## Step 4: Create ArgoCD Application CRDs

### Individual Service Applications

> [!success]- Full Solution: `apps/database.yaml`
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: myapp-database
>   namespace: argocd
>   annotations:
>     argocd.argoproj.io/sync-wave: "1"
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     # For local repos, use the file:// URL or configure a Git repo
>     repoURL: file:///path/to/gitops-config
>     targetRevision: HEAD
>     path: overlays/dev
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

> [!warning] Local Git Repos and ArgoCD
> ArgoCD expects a reachable Git URL. For local testing, you can either:
> 1. Use a local Git server (e.g., `gitea` running in minikube)
> 2. Push to a GitHub/GitLab repo
> 3. Use ArgoCD's `--directory-recurse` with a ConfigMap-based approach
>
> The simplest approach for this exercise is to push to a private GitHub repo or run a local Git server.

For a simpler setup using a single Application pointing at the dev overlay:

> [!success]- Full Solution: `apps/myapp-dev.yaml` (Single Application approach)
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
>     repoURL: https://github.com/YOUR_USER/gitops-config.git
>     targetRevision: main
>     path: overlays/dev
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: myapp-dev
>   syncPolicy:
>     automated:
>       prune: true       # Delete resources removed from Git
>       selfHeal: true     # Revert manual changes in cluster
>     syncOptions:
>       - CreateNamespace=true
>     retry:
>       limit: 3
>       backoff:
>         duration: 5s
>         factor: 2
>         maxDuration: 3m
> ```

---

## Step 5: App of Apps Pattern

> [!abstract] Key Concept: App of Apps
> Instead of applying each Application CRD individually, you create a "root" Application that manages all other Applications. When you add a new service, you simply add a new Application YAML to the `apps/` directory and commit -- ArgoCD picks it up automatically.

> [!success]- Full Solution: `apps/root-app.yaml`
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
>     repoURL: https://github.com/YOUR_USER/gitops-config.git
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

> [!success]- Full Solution: `apps/frontend.yaml`
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: myapp-frontend
>   namespace: argocd
>   annotations:
>     argocd.argoproj.io/sync-wave: "3"
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/YOUR_USER/gitops-config.git
>     targetRevision: main
>     path: base/frontend
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: myapp-dev
>   syncPolicy:
>     automated:
>       prune: true
>       selfHeal: true
>     syncOptions:
>       - CreateNamespace=true
> ```

> [!success]- Full Solution: `apps/backend.yaml`
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: myapp-backend
>   namespace: argocd
>   annotations:
>     argocd.argoproj.io/sync-wave: "2"
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/YOUR_USER/gitops-config.git
>     targetRevision: main
>     path: base/backend
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: myapp-dev
>   syncPolicy:
>     automated:
>       prune: true
>       selfHeal: true
>     syncOptions:
>       - CreateNamespace=true
> ```

Bootstrap the root app:

```bash
kubectl apply -f apps/root-app.yaml
```

---

## Step 6: Sync Waves

> [!abstract] Key Concept: Sync Waves
> Sync waves control the order in which ArgoCD applies resources. Lower wave numbers sync first. Resources in the same wave sync in parallel.
>
> | Wave | Resource | Why |
> |------|----------|-----|
> | -1 | Namespace | Must exist before anything else |
> | 0 | Secrets, ConfigMaps | Config must exist before apps reference them |
> | 1 | Database | Must be running before backend connects |
> | 2 | Backend | Must be running before frontend routes to it |
> | 3 | Frontend | Last to deploy, depends on backend |

Sync waves are set via the `argocd.argoproj.io/sync-wave` annotation, already included in the manifests above. ArgoCD processes waves in order and waits for each wave's resources to become healthy before proceeding.

---

## Step 7: PreSync Hook for Database Migrations

> [!success]- Full Solution: Add to `base/database/migration-job.yaml`
> ```yaml
> apiVersion: batch/v1
> kind: Job
> metadata:
>   name: db-migration
>   annotations:
>     argocd.argoproj.io/hook: PreSync
>     argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
>     argocd.argoproj.io/sync-wave: "1"
> spec:
>   backoffLimit: 3
>   template:
>     spec:
>       restartPolicy: Never
>       initContainers:
>         - name: wait-for-db
>           image: busybox:1.36
>           command:
>             - sh
>             - -c
>             - |
>               until nc -z postgres-svc 5432; do
>                 echo "Waiting for database..."
>                 sleep 2
>               done
>       containers:
>         - name: migrate
>           image: myapp-backend:v1
>           imagePullPolicy: Never
>           command: ["/backend"]
>           args: ["--migrate-only"]
>           env:
>             - name: DB_HOST
>               value: "postgres-svc"
>             - name: DB_PORT
>               value: "5432"
>             - name: DB_NAME
>               value: "appdb"
>             - name: DB_USER
>               value: "appuser"
>             - name: DB_PASSWORD
>               valueFrom:
>                 secretKeyRef:
>                   name: db-secret
>                   key: DB_PASSWORD
> ```

Update `base/database/kustomization.yaml` to include it:

```yaml
resources:
  - deployment.yaml
  - service.yaml
  - secret.yaml
  - migration-job.yaml
```

> [!info] ArgoCD Hooks vs Helm Hooks
> ArgoCD hooks use different annotations than Helm hooks:
> - `argocd.argoproj.io/hook: PreSync` (before main sync)
> - `argocd.argoproj.io/hook: Sync` (during sync)
> - `argocd.argoproj.io/hook: PostSync` (after sync)
> - `argocd.argoproj.io/hook: SyncFail` (on failure)

---

## Step 8: Demonstrate GitOps in Action

### Change Image Tag in Git, Watch ArgoCD Auto-Sync

```bash
# Edit the dev overlay to use a new image tag
cd gitops-config

# Update the image tag in overlays/dev/kustomization.yaml
# Change newTag from "v1" to "v2"
sed -i '' 's/newTag: v1/newTag: v2/' overlays/dev/kustomization.yaml

# Commit and push
git add overlays/dev/kustomization.yaml
git commit -m "Deploy backend v2 to dev"
git push origin main
```

Watch ArgoCD detect and sync the change:

```bash
# Via CLI
argocd app get myapp-dev --refresh

# Watch sync status
argocd app wait myapp-dev

# Or watch in the ArgoCD UI at https://localhost:8443
```

### Demonstrate Rollback via ArgoCD

```bash
# View sync history
argocd app history myapp-dev

# Rollback to previous revision
argocd app rollback myapp-dev 1

# OR: Rollback via Git (the GitOps way)
git revert HEAD
git push origin main
# ArgoCD auto-syncs the revert
```

> [!warning] ArgoCD Rollback vs Git Revert
> `argocd app rollback` reverts the cluster state but creates drift from Git. The **proper GitOps approach** is to `git revert` the commit -- this keeps Git as the single source of truth and ArgoCD syncs the revert automatically.

---

## Step 9: Sealed Secrets for Secret Management in Git

> [!abstract] Key Concept: Sealed Secrets
> Plain Kubernetes Secrets in Git are just base64-encoded (not encrypted). Sealed Secrets encrypts them with a cluster-side key so they can safely be stored in Git. Only the controller in the cluster can decrypt them.

### Install Sealed Secrets

```bash
# Install the controller in the cluster
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.25.0/controller.yaml

# Install kubeseal CLI (macOS)
brew install kubeseal
```

### Create a Sealed Secret

```bash
# Create a regular secret manifest (do NOT commit this)
cat <<EOF > /tmp/db-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: myapp-dev
type: Opaque
stringData:
  POSTGRES_PASSWORD: "my-super-secret-password"
  DB_PASSWORD: "my-super-secret-password"
EOF

# Seal it (encrypt with the cluster's public key)
kubeseal --format yaml < /tmp/db-secret.yaml > sealed-secrets/db-sealed-secret.yaml

# Clean up the plain secret
rm /tmp/db-secret.yaml
```

> [!success]- Full Solution: Example `sealed-secrets/db-sealed-secret.yaml`
> ```yaml
> # This is an EXAMPLE - actual encrypted values will differ per cluster
> apiVersion: bitnami.com/v1alpha1
> kind: SealedSecret
> metadata:
>   name: db-secret
>   namespace: myapp-dev
> spec:
>   encryptedData:
>     POSTGRES_PASSWORD: AgBy3i4OJSWK+PiTySYZZA9rO...  # encrypted
>     DB_PASSWORD: AgBy3i4OJSWK+PiTySYZZA9rO...        # encrypted
>   template:
>     metadata:
>       name: db-secret
>       namespace: myapp-dev
>     type: Opaque
> ```

The Sealed Secrets controller watches for `SealedSecret` resources and automatically creates the corresponding `Secret` in the cluster. The encrypted YAML is safe to commit to Git.

```bash
git add sealed-secrets/
git commit -m "Add sealed secret for database credentials"
git push origin main
```

---

## Verification Checklist

```bash
# Check ArgoCD applications
argocd app list

# Check sync status
argocd app get myapp-dev

# Verify all pods are running
kubectl get pods -n myapp-dev

# Check ArgoCD UI for visual sync status
# https://localhost:8443
```

---

## Cleanup

```bash
# Delete all ArgoCD applications
argocd app delete root-app --cascade

# Or delete the root app (cascade deletes child apps)
kubectl delete application root-app -n argocd

# Uninstall ArgoCD
kubectl delete namespace argocd

# Delete app namespaces
kubectl delete namespace myapp-dev myapp-staging myapp-production
```

---

## Troubleshooting

| Problem | Command | What to Look For |
|---|---|---|
| App stuck `OutOfSync` | `argocd app diff myapp-dev` | Compare desired vs live state |
| Sync failed | `argocd app get myapp-dev` | Check sync status and error messages |
| Hook failed | `kubectl get jobs -n myapp-dev` | Check job logs for migration errors |
| Sealed Secret not decrypted | `kubectl logs -n kube-system -l name=sealed-secrets-controller` | Controller errors |
| Repo not reachable | `argocd repo list` | Check repo URL and credentials |

> [!abstract] Key Takeaways
> - **GitOps**: Git is the single source of truth for cluster state -- all changes flow through Git
> - **App of Apps**: A root ArgoCD Application manages all child Applications, enabling self-service
> - **Sync Waves**: Control deployment order (namespace -> config -> DB -> backend -> frontend)
> - **Kustomize Overlays**: Same base manifests, different per-environment patches
> - **Sealed Secrets**: Encrypt secrets so they can live safely in Git
> - **Self-Heal**: ArgoCD reverts manual cluster changes back to the Git-defined state
> - **Rollback**: Prefer `git revert` over `argocd rollback` to maintain Git as the source of truth
