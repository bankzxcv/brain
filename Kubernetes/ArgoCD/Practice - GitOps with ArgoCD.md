---
title: "K8s Practice: GitOps with ArgoCD"
date: 2026-04-07
tags:
  - kubernetes
  - practice
  - argocd
  - gitops
parent: "[[Kubernetes Study]]"
---

# Practice - GitOps with ArgoCD

Progressive hands-on exercises for mastering ArgoCD-based GitOps workflows on minikube.

## Prerequisites

- minikube running with at least 4GB memory (`minikube start --memory=4096`)
- kubectl configured to use minikube context
- Helm 3 installed
- `argocd` CLI installed (`brew install argocd`)
- A Git repository for test manifests (see setup below)

## Setting Up a Test Git Repo

You need a Git repo containing Kubernetes manifests for ArgoCD to watch. You can use either a local repo (served via Gitea on minikube) or a GitHub repo.

**Option A: GitHub repo (recommended)**
```bash
# Create a new repo on GitHub called "argocd-practice"
# Clone it locally
git clone https://github.com/<your-username>/argocd-practice.git
cd argocd-practice

# Create initial structure
mkdir -p apps/nginx apps/httpecho apps/guestbook
```

**Option B: Local Gitea on minikube**
```bash
helm repo add gitea-charts https://dl.gitea.com/charts/
helm install gitea gitea-charts/gitea \
  --set service.http.type=NodePort \
  --set gitea.admin.username=gitea_admin \
  --set gitea.admin.password=password123 \
  --set persistence.enabled=false
```

**Populate the test repo with base manifests:**
```bash
cd argocd-practice

# apps/nginx/deployment.yaml
cat <<'EOF' > apps/nginx/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
EOF

# apps/nginx/service.yaml
cat <<'EOF' > apps/nginx/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
EOF

git add . && git commit -m "Initial manifests" && git push
```

---

## P1: Install ArgoCD and Access the UI

**Task:**
1. Install ArgoCD on minikube in the `argocd` namespace
2. Wait for all ArgoCD pods to be ready
3. Retrieve the initial admin password
4. Port-forward the ArgoCD server and login via both CLI and the web UI

**Expected Result:**
- All ArgoCD pods running in the `argocd` namespace
- Can access the ArgoCD UI at `https://localhost:8080`
- Successfully logged in as `admin`

**Verification:**
```bash
kubectl get pods -n argocd
argocd account get-user-info
```

> [!hint]- Hint
> Install via the official manifest:
> ```bash
> kubectl create namespace argocd
> kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
> ```
> The initial admin password is stored in a Secret called `argocd-initial-admin-secret`.

> [!success]- Solution
> ```bash
> # Install ArgoCD
> kubectl create namespace argocd
> kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
>
> # Wait for all pods to be ready
> kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
>
> # Verify pods
> kubectl get pods -n argocd
> # argocd-application-controller-0     Running
> # argocd-applicationset-controller    Running
> # argocd-dex-server                   Running
> # argocd-notifications-controller     Running
> # argocd-redis                        Running
> # argocd-repo-server                  Running
> # argocd-server                       Running
>
> # Get initial admin password
> kubectl -n argocd get secret argocd-initial-admin-secret \
>   -o jsonpath='{.data.password}' | base64 -d
> # Save this password
>
> # Port-forward the ArgoCD server (run in background or separate terminal)
> kubectl port-forward svc/argocd-server -n argocd 8080:443 &
>
> # Login via CLI
> argocd login localhost:8080 --username admin --password <password> --insecure
>
> # Verify login
> argocd account get-user-info
>
> # (Optional) Change the admin password
> argocd account update-password \
>   --current-password <initial-password> \
>   --new-password MyNewPassword123
>
> # Access UI: open https://localhost:8080 in browser
> # Login with admin / <password>
> ```

---

## P2: Create an Application via CLI

**Task:** Create an ArgoCD Application using the `argocd` CLI that:
1. Points to your test Git repo's `apps/nginx` directory
2. Deploys to the `default` namespace on the local cluster
3. Sync it manually

**Expected Result:**
- Application `nginx` visible in ArgoCD
- nginx Deployment and Service created in the `default` namespace
- Application status shows `Synced` and `Healthy`

**Verification:**
```bash
argocd app get nginx
kubectl get deploy nginx
kubectl get svc nginx
```

> [!hint]- Hint
> ```bash
> argocd app create <name> \
>   --repo <git-url> \
>   --path <path-in-repo> \
>   --dest-server https://kubernetes.default.svc \
>   --dest-namespace default
> ```
> After creating, sync with `argocd app sync <name>`.

> [!success]- Solution
> ```bash
> # Create the application
> argocd app create nginx \
>   --repo https://github.com/<your-username>/argocd-practice.git \
>   --path apps/nginx \
>   --dest-server https://kubernetes.default.svc \
>   --dest-namespace default
>
> # Check status -- should be OutOfSync initially
> argocd app get nginx
>
> # Sync the application
> argocd app sync nginx
>
> # Verify the sync
> argocd app get nginx
> # Status:       Synced
> # Health:       Healthy
>
> # Verify resources in cluster
> kubectl get deploy nginx
> kubectl get svc nginx
> kubectl get pods -l app=nginx
>
> # View the app in the UI at https://localhost:8080
> ```

---

## P3: Create an Application via YAML (Application CRD)

**Task:** Create an ArgoCD Application using a YAML manifest (Application CRD) instead of the CLI:
1. Write an Application YAML for the `httpecho` app
2. The app should deploy `hashicorp/http-echo` with the text "Hello ArgoCD"
3. Apply the Application CRD with `kubectl apply`

**Expected Result:**
- Application `httpecho` appears in ArgoCD
- After sync, the http-echo pod is running

**Verification:**
```bash
argocd app list
kubectl get application httpecho -n argocd
```

> [!hint]- Hint
> First, add the httpecho manifests to your Git repo. Then create an Application CRD:
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: httpecho
>   namespace: argocd
> spec:
>   project: default
>   source:
>     repoURL: <your-repo-url>
>     path: apps/httpecho
>     targetRevision: HEAD
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: default
> ```

> [!success]- Solution
> First, add the httpecho manifests to your Git repo:
> ```bash
> cd argocd-practice
>
> cat <<'EOF' > apps/httpecho/deployment.yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: httpecho
>   labels:
>     app: httpecho
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: httpecho
>   template:
>     metadata:
>       labels:
>         app: httpecho
>     spec:
>       containers:
>         - name: httpecho
>           image: hashicorp/http-echo:latest
>           args:
>             - "-text=Hello ArgoCD"
>             - "-listen=:5678"
>           ports:
>             - containerPort: 5678
> EOF
>
> cat <<'EOF' > apps/httpecho/service.yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: httpecho
> spec:
>   selector:
>     app: httpecho
>   ports:
>     - port: 5678
>       targetPort: 5678
>   type: ClusterIP
> EOF
>
> git add . && git commit -m "Add httpecho app" && git push
> ```
>
> Create the Application CRD:
> ```bash
> cat <<'EOF' > httpecho-application.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: httpecho
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/<your-username>/argocd-practice.git
>     path: apps/httpecho
>     targetRevision: HEAD
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: default
>   syncPolicy:
>     syncOptions:
>       - CreateNamespace=true
> EOF
>
> # Apply the Application CRD
> kubectl apply -f httpecho-application.yaml
>
> # Check in ArgoCD
> argocd app list
> argocd app get httpecho
>
> # Sync the application
> argocd app sync httpecho
>
> # Verify
> kubectl get deploy httpecho
> kubectl get pods -l app=httpecho
> kubectl port-forward svc/httpecho 5678:5678 &
> curl localhost:5678
> # Hello ArgoCD
> ```

---

## P4: Configure Automated Sync with Prune and Self-Heal

**Task:** Update the `httpecho` Application to enable:
1. Automated sync -- ArgoCD automatically syncs when Git changes are detected
2. Prune -- automatically delete resources removed from Git
3. Self-heal -- automatically revert manual changes made to the cluster

Test self-heal by manually scaling the deployment and watch ArgoCD revert it.

**Expected Result:**
- Changes in Git are automatically synced without manual intervention
- Manual `kubectl scale` is automatically reverted by ArgoCD

**Verification:**
```bash
argocd app get httpecho
kubectl scale deploy httpecho --replicas=5
# Wait ~30 seconds, check replicas revert to 1
kubectl get deploy httpecho
```

> [!hint]- Hint
> Add `syncPolicy` to the Application spec:
> ```yaml
> syncPolicy:
>   automated:
>     prune: true
>     selfHeal: true
> ```
> You can also enable it via CLI:
> ```bash
> argocd app set httpecho --sync-policy automated --auto-prune --self-heal
> ```

> [!success]- Solution
> **Option A: Update via CLI**
> ```bash
> argocd app set httpecho --sync-policy automated --auto-prune --self-heal
> ```
>
> **Option B: Update via YAML**
> ```bash
> cat <<'EOF' > httpecho-application.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: httpecho
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/<your-username>/argocd-practice.git
>     path: apps/httpecho
>     targetRevision: HEAD
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: default
>   syncPolicy:
>     automated:
>       prune: true
>       selfHeal: true
>     syncOptions:
>       - CreateNamespace=true
> EOF
>
> kubectl apply -f httpecho-application.yaml
> ```
>
> **Test self-heal:**
> ```bash
> # Manually scale the deployment
> kubectl scale deploy httpecho --replicas=5
>
> # Watch -- ArgoCD will detect the drift and revert
> kubectl get deploy httpecho -w
> # Within ~30 seconds, replicas should revert to 1
>
> # Check ArgoCD events
> argocd app get httpecho
> # Should show recent sync events
> ```
>
> **Test automated sync:**
> ```bash
> # Make a change in Git
> cd argocd-practice
> sed -i '' 's/replicas: 1/replicas: 2/' apps/httpecho/deployment.yaml
> git add . && git commit -m "Scale httpecho to 2 replicas" && git push
>
> # Watch ArgoCD automatically sync
> argocd app get httpecho -w
> kubectl get deploy httpecho -w
> # Replicas should update to 2 automatically
> ```

---

## P5: Observe Git-Driven Changes

**Task:**
1. Update the httpecho image tag in Git (change the text to "Hello GitOps v2")
2. Push the change
3. Observe ArgoCD detect the change and sync automatically (if automated sync is enabled) or sync manually

**Expected Result:**
- ArgoCD detects the Git change
- Application syncs to the new state
- The http-echo service returns the new text

**Verification:**
```bash
argocd app get httpecho
kubectl port-forward svc/httpecho 5678:5678 &
curl localhost:5678
# Should return: Hello GitOps v2
```

> [!hint]- Hint
> Simply update the deployment manifest in Git and push. If automated sync is on, ArgoCD polls Git every 3 minutes by default (or instantly with webhooks). You can force a refresh:
> ```bash
> argocd app get httpecho --refresh
> ```

> [!success]- Solution
> ```bash
> # Update the manifest in Git
> cd argocd-practice
>
> cat <<'EOF' > apps/httpecho/deployment.yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: httpecho
>   labels:
>     app: httpecho
> spec:
>   replicas: 2
>   selector:
>     matchLabels:
>       app: httpecho
>   template:
>     metadata:
>       labels:
>         app: httpecho
>     spec:
>       containers:
>         - name: httpecho
>           image: hashicorp/http-echo:latest
>           args:
>             - "-text=Hello GitOps v2"
>             - "-listen=:5678"
>           ports:
>             - containerPort: 5678
> EOF
>
> git add . && git commit -m "Update httpecho to v2" && git push
>
> # Force a refresh if you don't want to wait for polling
> argocd app get httpecho --refresh
>
> # If automated sync is enabled, watch it sync:
> argocd app wait httpecho --sync --timeout 120
>
> # If manual sync:
> # argocd app sync httpecho
>
> # Verify the change
> kubectl port-forward svc/httpecho 5678:5678 &
> curl localhost:5678
> # Hello GitOps v2
>
> # Check sync history
> argocd app history httpecho
> ```

---

## P6: Rollback an Application

**Task:**
1. View the revision history of the `httpecho` application
2. Roll back to the previous revision using the CLI
3. Verify the application reverted to the old state
4. Note: understand what happens with automated sync and rollback

**Expected Result:**
- Application rolls back to the previous revision
- If automated sync is enabled, ArgoCD will re-sync to HEAD (you may need to disable auto-sync first)

**Verification:**
```bash
argocd app history httpecho
argocd app get httpecho
```

> [!hint]- Hint
> - `argocd app history <name>` shows revision history
> - `argocd app rollback <name> <revision>` rolls back
> - Important: if auto-sync is enabled, ArgoCD will immediately re-sync to the latest Git state. Disable auto-sync first if you want the rollback to persist:
>   ```bash
>   argocd app set httpecho --sync-policy none
>   ```

> [!success]- Solution
> ```bash
> # View history
> argocd app history httpecho
> # ID  DATE                   REVISION
> # 1   2026-04-07 10:00:00    abc1234 (Initial manifests)
> # 2   2026-04-07 10:15:00    def5678 (Update httpecho to v2)
>
> # Disable auto-sync first (otherwise rollback will be immediately overwritten)
> argocd app set httpecho --sync-policy none
>
> # Rollback to revision 1
> argocd app rollback httpecho 1
>
> # Verify the rollback
> argocd app get httpecho
> kubectl port-forward svc/httpecho 5678:5678 &
> curl localhost:5678
> # Should return: Hello ArgoCD (the original text)
>
> # Check the app is marked as OutOfSync (since Git still has v2)
> argocd app get httpecho
> # Sync Status: OutOfSync
>
> # Re-enable auto-sync when done -- it will sync back to Git HEAD (v2)
> argocd app set httpecho --sync-policy automated --auto-prune --self-heal
>
> # Wait for auto-sync to restore Git state
> argocd app wait httpecho --sync --timeout 60
> curl localhost:5678
> # Hello GitOps v2 (back to Git HEAD)
> ```

---

## P7: Helm-Based Application in ArgoCD

**Task:** Create an ArgoCD Application that deploys a Helm chart:
1. Create an Application that uses the Bitnami Redis chart as its source
2. Override values using `valueFiles` or inline `values` in the Application spec
3. Sync and verify Redis is running

**Expected Result:**
- ArgoCD Application shows source type as Helm
- Redis pods are running with the configured values

**Verification:**
```bash
argocd app get redis-app
kubectl get pods -l app.kubernetes.io/name=redis
```

> [!hint]- Hint
> For Helm sources, specify the chart repo and chart name:
> ```yaml
> source:
>   repoURL: https://charts.bitnami.com/bitnami
>   chart: redis
>   targetRevision: "19.*"
>   helm:
>     values: |
>       architecture: standalone
> ```

> [!success]- Solution
> ```bash
> cat <<'EOF' > redis-application.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: redis-app
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://charts.bitnami.com/bitnami
>     chart: redis
>     targetRevision: "19.*"
>     helm:
>       releaseName: redis
>       values: |
>         architecture: standalone
>         auth:
>           enabled: false
>         master:
>           persistence:
>             enabled: false
>           resources:
>             limits:
>               cpu: 200m
>               memory: 256Mi
>             requests:
>               cpu: 100m
>               memory: 128Mi
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: default
>   syncPolicy:
>     automated:
>       selfHeal: true
>     syncOptions:
>       - CreateNamespace=true
> EOF
>
> kubectl apply -f redis-application.yaml
>
> # Sync
> argocd app sync redis-app
>
> # Verify
> argocd app get redis-app
> kubectl get pods -l app.kubernetes.io/name=redis
>
> # Test Redis connection
> kubectl exec -it redis-master-0 -- redis-cli ping
> # PONG
> ```
>
> **Alternative: Helm chart from Git repo**
>
> If you have a Helm chart in your Git repo:
> ```yaml
> spec:
>   source:
>     repoURL: https://github.com/<user>/argocd-practice.git
>     path: charts/myapp
>     targetRevision: HEAD
>     helm:
>       valueFiles:
>         - values-dev.yaml
> ```

---

## P8: Sync Waves

**Task:** Create a set of manifests that use sync waves to control deployment order:
1. Namespace (wave -1) -- created first
2. ConfigMap (wave 0) -- created second
3. Deployment (wave 1) -- created third, after config is ready
4. Service (wave 2) -- created last

Deploy these as a single ArgoCD Application and verify the order.

**Expected Result:**
- Resources are synced in order: Namespace -> ConfigMap -> Deployment -> Service
- ArgoCD shows sync waves in the UI

**Verification:**
```bash
argocd app get wave-demo
kubectl get all -n wave-demo
```

> [!hint]- Hint
> Add the `argocd.argoproj.io/sync-wave` annotation to each resource:
> ```yaml
> metadata:
>   annotations:
>     argocd.argoproj.io/sync-wave: "-1"
> ```
> Resources in the same wave sync in parallel. Lower wave numbers sync first. ArgoCD waits for resources in a wave to be healthy before proceeding to the next wave.

> [!success]- Solution
> Add manifests to the Git repo:
> ```bash
> cd argocd-practice
> mkdir -p apps/wave-demo
> ```
>
> Create `apps/wave-demo/namespace.yaml`:
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: wave-demo
>   annotations:
>     argocd.argoproj.io/sync-wave: "-1"
> ```
>
> Create `apps/wave-demo/configmap.yaml`:
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: wave-demo-config
>   namespace: wave-demo
>   annotations:
>     argocd.argoproj.io/sync-wave: "0"
> data:
>   APP_ENV: "staging"
>   APP_VERSION: "1.0.0"
>   LOG_LEVEL: "info"
> ```
>
> Create `apps/wave-demo/deployment.yaml`:
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: wave-demo
>   namespace: wave-demo
>   annotations:
>     argocd.argoproj.io/sync-wave: "1"
> spec:
>   replicas: 2
>   selector:
>     matchLabels:
>       app: wave-demo
>   template:
>     metadata:
>       labels:
>         app: wave-demo
>     spec:
>       containers:
>         - name: nginx
>           image: nginx:1.25
>           ports:
>             - containerPort: 80
>           envFrom:
>             - configMapRef:
>                 name: wave-demo-config
> ```
>
> Create `apps/wave-demo/service.yaml`:
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: wave-demo
>   namespace: wave-demo
>   annotations:
>     argocd.argoproj.io/sync-wave: "2"
> spec:
>   selector:
>     app: wave-demo
>   ports:
>     - port: 80
>       targetPort: 80
>   type: ClusterIP
> ```
>
> Push and create the Application:
> ```bash
> git add . && git commit -m "Add wave-demo app" && git push
>
> cat <<'EOF' > wave-demo-application.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: wave-demo
>   namespace: argocd
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/<your-username>/argocd-practice.git
>     path: apps/wave-demo
>     targetRevision: HEAD
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: wave-demo
>   syncPolicy:
>     syncOptions:
>       - CreateNamespace=true
> EOF
>
> kubectl apply -f wave-demo-application.yaml
> argocd app sync wave-demo
>
> # Watch the sync -- resources deploy in wave order
> argocd app get wave-demo --show-operation
>
> # Verify all resources
> kubectl get all -n wave-demo
> kubectl get configmap -n wave-demo
> ```

---

## P9: PreSync Hook - Database Migration Job

**Task:** Add a PreSync hook to the `wave-demo` application that runs a database migration Job before the main application deploys:
1. Create a Job manifest with the `argocd.argoproj.io/hook: PreSync` annotation
2. Add a `hook-delete-policy` to clean up after success
3. Sync and verify the Job runs before the Deployment updates

**Expected Result:**
- Migration Job runs and completes before the app Deployment syncs
- Job is cleaned up after successful completion

**Verification:**
```bash
argocd app sync wave-demo
kubectl get jobs -n wave-demo
kubectl logs job/wave-demo-migrate -n wave-demo
```

> [!hint]- Hint
> ArgoCD hooks:
> - `PreSync` -- runs before the sync
> - `Sync` -- runs during the sync (alongside manifests)
> - `PostSync` -- runs after all resources are synced and healthy
> - `SyncFail` -- runs if the sync fails
>
> Add these annotations:
> ```yaml
> argocd.argoproj.io/hook: PreSync
> argocd.argoproj.io/hook-delete-policy: HookSucceeded
> ```

> [!success]- Solution
> Create `apps/wave-demo/migration-job.yaml`:
> ```yaml
> apiVersion: batch/v1
> kind: Job
> metadata:
>   name: wave-demo-migrate
>   namespace: wave-demo
>   annotations:
>     argocd.argoproj.io/hook: PreSync
>     argocd.argoproj.io/hook-delete-policy: HookSucceeded
>     argocd.argoproj.io/sync-wave: "-1"
> spec:
>   template:
>     metadata:
>       name: wave-demo-migrate
>     spec:
>       restartPolicy: Never
>       containers:
>         - name: migrate
>           image: busybox:1.36
>           command:
>             - /bin/sh
>             - -c
>             - |
>               echo "=== Database Migration ==="
>               echo "Connecting to database..."
>               sleep 2
>               echo "Running migration 001_create_tables..."
>               sleep 1
>               echo "Running migration 002_seed_data..."
>               sleep 1
>               echo "All migrations completed successfully."
>   backoffLimit: 2
> ```
>
> Push and sync:
> ```bash
> cd argocd-practice
> git add . && git commit -m "Add PreSync migration hook" && git push
>
> # Sync and watch the hook execute
> argocd app sync wave-demo
>
> # Watch the Job run
> kubectl get jobs -n wave-demo -w
>
> # View migration logs
> kubectl logs job/wave-demo-migrate -n wave-demo
> # === Database Migration ===
> # Connecting to database...
> # Running migration 001_create_tables...
> # Running migration 002_seed_data...
> # All migrations completed successfully.
>
> # After success, the Job is cleaned up (HookSucceeded policy)
> kubectl get jobs -n wave-demo
> # No migration job (cleaned up)
>
> # Verify the main app deployed after migration
> kubectl get deploy -n wave-demo
> ```

---

## P10: App of Apps Pattern

**Task:** Create a "root" Application that manages 3 child Applications:
1. Create an `apps/root/` directory in your Git repo
2. Place Application YAML files for nginx, httpecho, and wave-demo in that directory
3. Create a root Application that points to `apps/root/`
4. When the root app syncs, all 3 child applications are created and synced

**Expected Result:**
- One root Application in ArgoCD
- The root Application manages 3 child Application resources
- All 3 child apps are synced and healthy

**Verification:**
```bash
argocd app list
# Should show: root-app, nginx, httpecho, wave-demo
```

> [!hint]- Hint
> The App of Apps pattern stores Application CRD YAMLs in Git. The root Application points to the directory containing those YAMLs. When ArgoCD syncs the root app, it creates/updates the child Application resources, which ArgoCD then syncs individually.
>
> Key: each child Application YAML in the directory is a standard Application CRD.

> [!success]- Solution
> ```bash
> cd argocd-practice
> mkdir -p apps/root
> ```
>
> Create `apps/root/nginx-app.yaml`:
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: nginx
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/<your-username>/argocd-practice.git
>     path: apps/nginx
>     targetRevision: HEAD
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: default
>   syncPolicy:
>     automated:
>       selfHeal: true
>       prune: true
> ```
>
> Create `apps/root/httpecho-app.yaml`:
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: httpecho
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/<your-username>/argocd-practice.git
>     path: apps/httpecho
>     targetRevision: HEAD
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: default
>   syncPolicy:
>     automated:
>       selfHeal: true
>       prune: true
> ```
>
> Create `apps/root/wave-demo-app.yaml`:
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: wave-demo
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/<your-username>/argocd-practice.git
>     path: apps/wave-demo
>     targetRevision: HEAD
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: wave-demo
>   syncPolicy:
>     automated:
>       selfHeal: true
>       prune: true
>     syncOptions:
>       - CreateNamespace=true
> ```
>
> Push and create the root Application:
> ```bash
> git add . && git commit -m "Add app-of-apps root" && git push
>
> # Delete existing individual apps first
> argocd app delete nginx --cascade=false -y 2>/dev/null
> argocd app delete httpecho --cascade=false -y 2>/dev/null
> argocd app delete wave-demo --cascade=false -y 2>/dev/null
>
> # Create the root application
> cat <<'EOF' > root-application.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: root-app
>   namespace: argocd
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/<your-username>/argocd-practice.git
>     path: apps/root
>     targetRevision: HEAD
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: argocd
>   syncPolicy:
>     automated:
>       selfHeal: true
>       prune: true
> EOF
>
> kubectl apply -f root-application.yaml
>
> # Sync the root app
> argocd app sync root-app
>
> # List all apps -- should see root + 3 children
> argocd app list
> # NAME        SYNC STATUS  HEALTH STATUS
> # root-app    Synced       Healthy
> # nginx       Synced       Healthy
> # httpecho    Synced       Healthy
> # wave-demo   Synced       Healthy
> ```

---

## P11: ApplicationSet with Git Generator

**Task:** Create an ApplicationSet that automatically generates Applications from your Git repository's directory structure:
1. Use the Git directory generator to scan `apps/` for subdirectories
2. Each subdirectory automatically becomes an ArgoCD Application
3. Add a new app directory to Git and verify ArgoCD creates it automatically

**Expected Result:**
- ApplicationSet creates one Application per subdirectory in `apps/`
- Adding a new directory creates a new Application automatically

**Verification:**
```bash
kubectl get applicationsets -n argocd
argocd app list
```

> [!hint]- Hint
> ApplicationSet with Git generator:
> ```yaml
> apiVersion: argoproj.io/v1alpha1
> kind: ApplicationSet
> metadata:
>   name: my-appset
>   namespace: argocd
> spec:
>   generators:
>     - git:
>         repoURL: <repo-url>
>         revision: HEAD
>         directories:
>           - path: apps/*
>   template:
>     metadata:
>       name: '{{path.basename}}'
>     spec:
>       source:
>         path: '{{path}}'
> ```
> The `{{path.basename}}` and `{{path}}` are template variables from the generator.

> [!success]- Solution
> First, clean up existing apps:
> ```bash
> argocd app delete root-app -y 2>/dev/null
> kubectl delete application nginx httpecho wave-demo -n argocd 2>/dev/null
> ```
>
> Create the ApplicationSet:
> ```bash
> cat <<'EOF' > appset.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: ApplicationSet
> metadata:
>   name: apps-appset
>   namespace: argocd
> spec:
>   generators:
>     - git:
>         repoURL: https://github.com/<your-username>/argocd-practice.git
>         revision: HEAD
>         directories:
>           - path: apps/*
>           - path: apps/root
>             exclude: true
>   template:
>     metadata:
>       name: '{{path.basename}}'
>       namespace: argocd
>     spec:
>       project: default
>       source:
>         repoURL: https://github.com/<your-username>/argocd-practice.git
>         path: '{{path}}'
>         targetRevision: HEAD
>       destination:
>         server: https://kubernetes.default.svc
>         namespace: '{{path.basename}}'
>       syncPolicy:
>         automated:
>           selfHeal: true
>           prune: true
>         syncOptions:
>           - CreateNamespace=true
> EOF
>
> kubectl apply -f appset.yaml
> ```
>
> Verify existing apps were generated:
> ```bash
> kubectl get applicationsets -n argocd
> argocd app list
> # Should show: nginx, httpecho, wave-demo (auto-generated from dirs)
> ```
>
> Test by adding a new app:
> ```bash
> cd argocd-practice
> mkdir -p apps/whoami
>
> cat <<'EOF' > apps/whoami/deployment.yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: whoami
>   labels:
>     app: whoami
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: whoami
>   template:
>     metadata:
>       labels:
>         app: whoami
>     spec:
>       containers:
>         - name: whoami
>           image: traefik/whoami:latest
>           ports:
>             - containerPort: 80
> EOF
>
> cat <<'EOF' > apps/whoami/service.yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: whoami
> spec:
>   selector:
>     app: whoami
>   ports:
>     - port: 80
>       targetPort: 80
>   type: ClusterIP
> EOF
>
> git add . && git commit -m "Add whoami app" && git push
>
> # Wait for ArgoCD to detect the new directory
> argocd app list
> # Should now include: whoami
>
> # Verify the new app
> argocd app get whoami
> kubectl get pods -n whoami
> ```

---

## P12: End-to-End GitOps Workflow

**Task:** Set up a complete GitOps workflow from scratch:
1. Create a new app with Deployment, Service, ConfigMap, and HPA
2. Use sync waves to control deployment order
3. Add a PreSync migration hook
4. Configure automated sync with self-heal
5. Add proper health checks (readiness/liveness probes)
6. Push to Git and verify the full flow works end-to-end
7. Simulate a change (update image tag), push, and observe the automated sync

**Expected Result:**
- Complete application deployed via GitOps
- PreSync hook runs before deployment
- Sync waves enforce correct ordering
- Health checks pass and ArgoCD shows Healthy status
- Changes in Git are automatically synced

**Verification:**
```bash
argocd app get production-app
kubectl get all -n production
argocd app history production-app
```

> [!hint]- Hint
> This is a synthesis exercise. Combine everything from P1-P11:
> - Sync wave -2: Namespace
> - PreSync hook: Migration Job
> - Sync wave -1: ConfigMap + Secret
> - Sync wave 0: Deployment (with probes)
> - Sync wave 1: Service + HPA
> Plan the directory structure first, then write all manifests.

> [!success]- Solution
> ```bash
> cd argocd-practice
> mkdir -p apps/production
> ```
>
> Create `apps/production/namespace.yaml`:
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: production
>   annotations:
>     argocd.argoproj.io/sync-wave: "-2"
> ```
>
> Create `apps/production/migration.yaml`:
> ```yaml
> apiVersion: batch/v1
> kind: Job
> metadata:
>   name: production-migrate
>   namespace: production
>   annotations:
>     argocd.argoproj.io/hook: PreSync
>     argocd.argoproj.io/hook-delete-policy: HookSucceeded
> spec:
>   template:
>     spec:
>       restartPolicy: Never
>       containers:
>         - name: migrate
>           image: busybox:1.36
>           command:
>             - /bin/sh
>             - -c
>             - |
>               echo "Running database migrations..."
>               sleep 3
>               echo "Migrations complete."
>   backoffLimit: 2
> ```
>
> Create `apps/production/configmap.yaml`:
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: production-config
>   namespace: production
>   annotations:
>     argocd.argoproj.io/sync-wave: "-1"
> data:
>   APP_ENV: "production"
>   LOG_LEVEL: "warn"
>   LOG_FORMAT: "json"
>   APP_PORT: "80"
> ```
>
> Create `apps/production/deployment.yaml`:
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: production-app
>   namespace: production
>   annotations:
>     argocd.argoproj.io/sync-wave: "0"
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: production-app
>   template:
>     metadata:
>       labels:
>         app: production-app
>     spec:
>       containers:
>         - name: nginx
>           image: nginx:1.25
>           ports:
>             - containerPort: 80
>           envFrom:
>             - configMapRef:
>                 name: production-config
>           readinessProbe:
>             httpGet:
>               path: /
>               port: 80
>             initialDelaySeconds: 5
>             periodSeconds: 10
>           livenessProbe:
>             httpGet:
>               path: /
>               port: 80
>             initialDelaySeconds: 10
>             periodSeconds: 30
>           resources:
>             requests:
>               cpu: 100m
>               memory: 128Mi
>             limits:
>               cpu: 250m
>               memory: 256Mi
> ```
>
> Create `apps/production/service.yaml`:
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: production-app
>   namespace: production
>   annotations:
>     argocd.argoproj.io/sync-wave: "1"
> spec:
>   selector:
>     app: production-app
>   ports:
>     - port: 80
>       targetPort: 80
>   type: ClusterIP
> ```
>
> Create `apps/production/hpa.yaml`:
> ```yaml
> apiVersion: autoscaling/v2
> kind: HorizontalPodAutoscaler
> metadata:
>   name: production-app
>   namespace: production
>   annotations:
>     argocd.argoproj.io/sync-wave: "1"
> spec:
>   scaleTargetRef:
>     apiVersion: apps/v1
>     kind: Deployment
>     name: production-app
>   minReplicas: 3
>   maxReplicas: 10
>   metrics:
>     - type: Resource
>       resource:
>         name: cpu
>         target:
>           type: Utilization
>           averageUtilization: 70
> ```
>
> Push and create the Application:
> ```bash
> git add . && git commit -m "Add production app with full GitOps setup" && git push
>
> cat <<'EOF' > production-application.yaml
> apiVersion: argoproj.io/v1alpha1
> kind: Application
> metadata:
>   name: production-app
>   namespace: argocd
>   finalizers:
>     - resources-finalizer.argocd.argoproj.io
> spec:
>   project: default
>   source:
>     repoURL: https://github.com/<your-username>/argocd-practice.git
>     path: apps/production
>     targetRevision: HEAD
>   destination:
>     server: https://kubernetes.default.svc
>     namespace: production
>   syncPolicy:
>     automated:
>       selfHeal: true
>       prune: true
>     syncOptions:
>       - CreateNamespace=true
>     retry:
>       limit: 3
>       backoff:
>         duration: 5s
>         factor: 2
>         maxDuration: 3m
> EOF
>
> kubectl apply -f production-application.yaml
>
> # Watch the full sync flow
> argocd app sync production-app
> argocd app get production-app --show-operation
>
> # Verify everything is deployed
> kubectl get all -n production
> kubectl get configmap -n production
> kubectl get hpa -n production
>
> # Simulate a change -- update image tag
> cd argocd-practice
> sed -i '' 's/nginx:1.25/nginx:1.26/' apps/production/deployment.yaml
> git add . && git commit -m "Upgrade nginx to 1.26" && git push
>
> # Watch ArgoCD auto-sync
> argocd app get production-app --refresh
> argocd app wait production-app --sync --timeout 120
>
> # Verify new image
> kubectl get deploy production-app -n production -o jsonpath='{.spec.template.spec.containers[0].image}'
> # nginx:1.26
>
> # Check history
> argocd app history production-app
> ```
