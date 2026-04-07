---
title: "K8s Practice: Configuration and Storage"
date: 2026-04-07
tags:
  - kubernetes
  - practice
  - configuration
  - storage
parent: "[[Kubernetes Study]]"
---

# K8s Practice: Configuration and Storage

13 progressive exercises covering Namespaces, ConfigMaps, Secrets, Volumes, PersistentVolumes, StorageClasses, ResourceQuotas, and LimitRanges. Each exercise builds on concepts from the previous ones.

## Prerequisites

- minikube installed and running (`minikube start`)
- kubectl installed and configured
- openssl installed (for P5 TLS certificate generation)
- Basic familiarity with YAML syntax and kubectl commands

```bash
# Verify your environment
minikube status
kubectl cluster-info
kubectl get nodes
```

---

## P1: Namespaces and Pod Deployment

**Task:** Create two Namespaces called `dev` and `staging`. Deploy an nginx pod in each namespace. List pods across all namespaces to confirm.

**Expected Result:**
```
NAMESPACE   NAME        READY   STATUS    RESTARTS   AGE
dev         nginx-dev   1/1     Running   0          30s
staging     nginx-stg   1/1     Running   0          30s
```

**Verification:**
```bash
kubectl get pods --all-namespaces -l purpose=ns-demo
```

> [!hint]- Hint
> - `kubectl create namespace <name>` creates a namespace
> - Use `-n <namespace>` flag or `metadata.namespace` in YAML to target a namespace
> - You can use `kubectl run` for quick pod creation or write full YAML manifests

> [!success]- Solution
> **Create namespaces:**
> ```bash
> kubectl create namespace dev
> kubectl create namespace staging
> ```
>
> **nginx-dev.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: nginx-dev
>   namespace: dev
>   labels:
>     purpose: ns-demo
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>       ports:
>         - containerPort: 80
> ```
>
> **nginx-stg.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: nginx-stg
>   namespace: staging
>   labels:
>     purpose: ns-demo
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>       ports:
>         - containerPort: 80
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f nginx-dev.yaml
> kubectl apply -f nginx-stg.yaml
> kubectl get pods -n dev
> kubectl get pods -n staging
> kubectl get pods --all-namespaces -l purpose=ns-demo
> ```

---

## P2: ConfigMap from Literal Values as Environment Variables

**Task:** Create a ConfigMap named `app-config` in the `dev` namespace with the following key-value pairs: `APP_ENV=development`, `LOG_LEVEL=debug`, `MAX_RETRIES=3`. Deploy a busybox pod that reads these values as environment variables and prints them.

**Expected Result:**
```
APP_ENV=development
LOG_LEVEL=debug
MAX_RETRIES=3
```

**Verification:**
```bash
kubectl logs configmap-env-demo -n dev
```

> [!hint]- Hint
> - `kubectl create configmap <name> --from-literal=key=value` creates a ConfigMap
> - Use `envFrom` with `configMapRef` to inject all keys as env vars
> - Alternatively use `env[].valueFrom.configMapKeyRef` for individual keys
> - busybox command: `["sh", "-c", "env | grep -E 'APP_ENV|LOG_LEVEL|MAX_RETRIES'"]`

> [!success]- Solution
> **Create the ConfigMap:**
> ```bash
> kubectl create configmap app-config \
>   --from-literal=APP_ENV=development \
>   --from-literal=LOG_LEVEL=debug \
>   --from-literal=MAX_RETRIES=3 \
>   -n dev
> ```
>
> **Verify the ConfigMap:**
> ```bash
> kubectl describe configmap app-config -n dev
> ```
>
> **configmap-env-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: configmap-env-demo
>   namespace: dev
> spec:
>   containers:
>     - name: demo
>       image: busybox:1.36
>       command: ["sh", "-c", "echo APP_ENV=$APP_ENV; echo LOG_LEVEL=$LOG_LEVEL; echo MAX_RETRIES=$MAX_RETRIES; sleep 3600"]
>       envFrom:
>         - configMapRef:
>             name: app-config
>   restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f configmap-env-pod.yaml
> kubectl logs configmap-env-demo -n dev
> ```

---

## P3: ConfigMap from File Mounted as Volume

**Task:** Create a file called `nginx.conf` with a custom nginx configuration that changes the default port to 8080. Create a ConfigMap from this file and mount it into an nginx pod at `/etc/nginx/conf.d/default.conf`. Verify nginx is listening on port 8080.

**Expected Result:**
```bash
# Port-forward and curl should return nginx welcome page
kubectl port-forward nginx-custom-conf 8080:8080 -n dev
curl http://localhost:8080
# Should return: Welcome to nginx!
```

**Verification:**
```bash
kubectl exec nginx-custom-conf -n dev -- nginx -T | grep "listen"
# Should show: listen 8080;
```

> [!hint]- Hint
> - Create the config file locally first, then use `kubectl create configmap --from-file=<file>`
> - Mount using `volumes` + `volumeMounts` in the pod spec
> - Use `subPath` to mount a single file without replacing the entire directory
> - nginx server block needs `listen 8080;`

> [!success]- Solution
> **Create the nginx config file locally:**
> ```bash
> cat <<'EOF' > /tmp/default.conf
> server {
>     listen 8080;
>     server_name localhost;
>
>     location / {
>         root   /usr/share/nginx/html;
>         index  index.html index.htm;
>     }
> }
> EOF
> ```
>
> **Create ConfigMap from the file:**
> ```bash
> kubectl create configmap nginx-conf \
>   --from-file=default.conf=/tmp/default.conf \
>   -n dev
> ```
>
> **nginx-custom-conf.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: nginx-custom-conf
>   namespace: dev
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>       ports:
>         - containerPort: 8080
>       volumeMounts:
>         - name: nginx-config
>           mountPath: /etc/nginx/conf.d/default.conf
>           subPath: default.conf
>   volumes:
>     - name: nginx-config
>       configMap:
>         name: nginx-conf
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f nginx-custom-conf.yaml
> kubectl wait --for=condition=Ready pod/nginx-custom-conf -n dev --timeout=60s
> kubectl exec nginx-custom-conf -n dev -- nginx -T | grep "listen"
> kubectl port-forward nginx-custom-conf 8080:8080 -n dev &
> curl http://localhost:8080
> ```

---

## P4: Secret from Literal Values as Environment Variables

**Task:** Create a Secret named `db-credentials` in the `dev` namespace with `DB_USER=admin` and `DB_PASSWORD=s3cretP@ss`. Deploy a busybox pod that reads these secrets as environment variables and prints them. Also verify that the secret values are base64-encoded in the cluster.

**Expected Result:**
```
DB_USER=admin
DB_PASSWORD=s3cretP@ss
```

**Verification:**
```bash
kubectl get secret db-credentials -n dev -o jsonpath='{.data.DB_USER}' | base64 -d
# Should output: admin
kubectl logs secret-env-demo -n dev
```

> [!hint]- Hint
> - `kubectl create secret generic <name> --from-literal=key=value`
> - Secrets are base64-encoded (NOT encrypted) by default
> - Use `secretRef` instead of `configMapRef` in `envFrom`
> - You can also use `env[].valueFrom.secretKeyRef` for individual keys

> [!success]- Solution
> **Create the Secret:**
> ```bash
> kubectl create secret generic db-credentials \
>   --from-literal=DB_USER=admin \
>   --from-literal=DB_PASSWORD='s3cretP@ss' \
>   -n dev
> ```
>
> **Verify base64 encoding:**
> ```bash
> kubectl get secret db-credentials -n dev -o yaml
> # data section shows base64-encoded values
> kubectl get secret db-credentials -n dev -o jsonpath='{.data.DB_USER}' | base64 -d
> echo  # newline
> kubectl get secret db-credentials -n dev -o jsonpath='{.data.DB_PASSWORD}' | base64 -d
> echo
> ```
>
> **secret-env-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: secret-env-demo
>   namespace: dev
> spec:
>   containers:
>     - name: demo
>       image: busybox:1.36
>       command: ["sh", "-c", "echo DB_USER=$DB_USER; echo DB_PASSWORD=$DB_PASSWORD; sleep 3600"]
>       envFrom:
>         - secretRef:
>             name: db-credentials
>   restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f secret-env-pod.yaml
> kubectl logs secret-env-demo -n dev
> ```

---

## P5: TLS Secret Mounted in Nginx for HTTPS

**Task:** Generate a self-signed TLS certificate and key. Create a TLS Secret from those files. Deploy an nginx pod configured for HTTPS using the mounted certificate. Verify HTTPS works via port-forward.

**Expected Result:**
```bash
curl -k https://localhost:8443
# Should return nginx welcome page over HTTPS
```

**Verification:**
```bash
kubectl get secret tls-secret -n dev -o jsonpath='{.type}'
# Should output: kubernetes.io/tls
kubectl exec nginx-tls -n dev -- ls /etc/nginx/ssl/
# Should show: tls.crt  tls.key
```

> [!hint]- Hint
> - Generate cert: `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=localhost"`
> - `kubectl create secret tls <name> --cert=tls.crt --key=tls.key`
> - You need a custom nginx config that uses `ssl_certificate` and `ssl_certificate_key`
> - Mount the TLS secret as a volume at `/etc/nginx/ssl/`
> - Mount the nginx SSL config via a ConfigMap

> [!success]- Solution
> **Generate self-signed certificate:**
> ```bash
> openssl req -x509 -nodes -days 365 \
>   -newkey rsa:2048 \
>   -keyout /tmp/tls.key \
>   -out /tmp/tls.crt \
>   -subj "/CN=localhost"
> ```
>
> **Create the TLS Secret:**
> ```bash
> kubectl create secret tls tls-secret \
>   --cert=/tmp/tls.crt \
>   --key=/tmp/tls.key \
>   -n dev
> ```
>
> **Create nginx SSL config:**
> ```bash
> cat <<'EOF' > /tmp/ssl.conf
> server {
>     listen 443 ssl;
>     server_name localhost;
>
>     ssl_certificate /etc/nginx/ssl/tls.crt;
>     ssl_certificate_key /etc/nginx/ssl/tls.key;
>
>     location / {
>         root   /usr/share/nginx/html;
>         index  index.html index.htm;
>     }
> }
> EOF
> kubectl create configmap nginx-ssl-conf \
>   --from-file=ssl.conf=/tmp/ssl.conf \
>   -n dev
> ```
>
> **nginx-tls.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: nginx-tls
>   namespace: dev
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>       ports:
>         - containerPort: 443
>       volumeMounts:
>         - name: tls-certs
>           mountPath: /etc/nginx/ssl
>           readOnly: true
>         - name: ssl-config
>           mountPath: /etc/nginx/conf.d/ssl.conf
>           subPath: ssl.conf
>   volumes:
>     - name: tls-certs
>       secret:
>         secretName: tls-secret
>     - name: ssl-config
>       configMap:
>         name: nginx-ssl-conf
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f nginx-tls.yaml
> kubectl wait --for=condition=Ready pod/nginx-tls -n dev --timeout=60s
> kubectl port-forward nginx-tls 8443:443 -n dev &
> curl -k https://localhost:8443
> ```

---

## P6: ConfigMap Update Behavior - Volume vs Environment Variables

**Task:** Using the `app-config` ConfigMap and pods from P2 and P3, update the ConfigMap values. Observe that volume-mounted ConfigMaps eventually reflect the change, but environment variables do NOT update without pod restart.

**Expected Result:**
- Volume-mounted file updates within ~60 seconds (no pod restart needed)
- Environment variables remain unchanged until pod is deleted and recreated

**Verification:**
```bash
# After updating ConfigMap, check volume mount (updates automatically):
kubectl exec nginx-custom-conf -n dev -- cat /etc/nginx/conf.d/default.conf

# Check env vars (does NOT update):
kubectl exec configmap-env-demo -n dev -- sh -c 'echo $LOG_LEVEL'
# Still shows: debug (old value)
```

> [!hint]- Hint
> - `kubectl edit configmap <name>` or use `kubectl create configmap --dry-run=client -o yaml | kubectl apply -f -`
> - Volume-mounted ConfigMaps are updated by the kubelet sync loop (default ~60s)
> - Environment variables are injected at pod startup and never updated
> - Note: `subPath` mounts do NOT receive automatic updates either

> [!success]- Solution
> **Update the ConfigMap (app-config):**
> ```bash
> kubectl create configmap app-config \
>   --from-literal=APP_ENV=production \
>   --from-literal=LOG_LEVEL=info \
>   --from-literal=MAX_RETRIES=5 \
>   -n dev \
>   --dry-run=client -o yaml | kubectl apply -f -
> ```
>
> **Check env var pod (will NOT update):**
> ```bash
> kubectl exec configmap-env-demo -n dev -- sh -c 'echo LOG_LEVEL=$LOG_LEVEL'
> # Output: LOG_LEVEL=debug  <-- still the old value!
> ```
>
> **Now update the nginx-conf ConfigMap to test volume mount:**
> ```bash
> cat <<'EOF' > /tmp/default.conf
> server {
>     listen 9090;
>     server_name localhost;
>
>     location / {
>         root   /usr/share/nginx/html;
>         index  index.html index.htm;
>     }
> }
> EOF
> kubectl create configmap nginx-conf \
>   --from-file=default.conf=/tmp/default.conf \
>   -n dev \
>   --dry-run=client -o yaml | kubectl apply -f -
> ```
>
> **Important caveat:** The pod in P3 uses `subPath`, which does NOT receive automatic updates. To see auto-update behavior, create a pod that mounts the entire ConfigMap as a directory:
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: configmap-vol-demo
>   namespace: dev
> spec:
>   containers:
>     - name: demo
>       image: busybox:1.36
>       command: ["sh", "-c", "while true; do cat /config/default.conf; echo '---'; sleep 10; done"]
>       volumeMounts:
>         - name: config-vol
>           mountPath: /config
>   volumes:
>     - name: config-vol
>       configMap:
>         name: nginx-conf
>   restartPolicy: Never
> ```
>
> ```bash
> kubectl apply -f configmap-vol-demo.yaml
> # Wait ~60 seconds, then check logs:
> kubectl logs configmap-vol-demo -n dev --tail=20
> # Should show the updated config (listen 9090)
> ```
>
> **Key takeaways:**
> - Environment variables: never update (requires pod restart)
> - Volume mount (full directory): updates within ~60s
> - Volume mount with subPath: never updates (requires pod restart)

---

## P7: PersistentVolume and PersistentVolumeClaim

**Task:** Create a PersistentVolume (1Gi, hostPath `/mnt/data`, accessMode `ReadWriteOnce`) and a PersistentVolumeClaim that requests 500Mi. Deploy a pod that mounts the PVC and writes a file. Verify the file exists on the host.

**Expected Result:**
```
kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM           
my-pv     1Gi        RWO            Retain           Bound    dev/my-pvc

kubectl get pvc -n dev
NAME     STATUS   VOLUME   CAPACITY   ACCESS MODES
my-pvc   Bound    my-pv    1Gi        RWO
```

**Verification:**
```bash
kubectl exec pv-demo -n dev -- cat /data/hello.txt
# Should output: Hello from PV!
minikube ssh -- cat /mnt/data/hello.txt
# Should output: Hello from PV!
```

> [!hint]- Hint
> - PV is cluster-scoped (no namespace), PVC is namespace-scoped
> - PV `hostPath` maps to a directory on the node
> - PVC binds to a PV that satisfies its requirements (capacity, access mode, storageClassName)
> - Set `storageClassName: manual` on both PV and PVC to ensure they bind to each other
> - With minikube, `minikube ssh` lets you check the node filesystem

> [!success]- Solution
> **pv.yaml:**
> ```yaml
> apiVersion: v1
> kind: PersistentVolume
> metadata:
>   name: my-pv
> spec:
>   capacity:
>     storage: 1Gi
>   accessModes:
>     - ReadWriteOnce
>   persistentVolumeReclaimPolicy: Retain
>   storageClassName: manual
>   hostPath:
>     path: /mnt/data
> ```
>
> **pvc.yaml:**
> ```yaml
> apiVersion: v1
> kind: PersistentVolumeClaim
> metadata:
>   name: my-pvc
>   namespace: dev
> spec:
>   accessModes:
>     - ReadWriteOnce
>   resources:
>     requests:
>       storage: 500Mi
>   storageClassName: manual
> ```
>
> **pv-demo-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: pv-demo
>   namespace: dev
> spec:
>   containers:
>     - name: demo
>       image: busybox:1.36
>       command: ["sh", "-c", "echo 'Hello from PV!' > /data/hello.txt; sleep 3600"]
>       volumeMounts:
>         - name: my-storage
>           mountPath: /data
>   volumes:
>     - name: my-storage
>       persistentVolumeClaim:
>         claimName: my-pvc
>   restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f pv.yaml
> kubectl apply -f pvc.yaml
> kubectl apply -f pv-demo-pod.yaml
> kubectl get pv
> kubectl get pvc -n dev
> kubectl exec pv-demo -n dev -- cat /data/hello.txt
> minikube ssh -- cat /mnt/data/hello.txt
> ```

---

## P8: PostgreSQL with Persistent Storage - Data Survival Test

**Task:** Deploy PostgreSQL using a PVC for data storage. Insert a row into a table. Delete the pod and recreate it. Verify the data persists across pod deletion.

**Expected Result:**
```
# After deleting and recreating the pod:
kubectl exec -it postgres-db -n dev -- psql -U postgres -d testdb -c "SELECT * FROM messages;"
 id |      content       |         created_at
----+--------------------+----------------------------
  1 | Hello, Kubernetes! | 2026-04-07 10:00:00.000000
```

**Verification:**
```bash
# Data should survive pod deletion
kubectl delete pod postgres-db -n dev
# Recreate pod (apply same YAML)
kubectl apply -f postgres-pod.yaml
kubectl wait --for=condition=Ready pod/postgres-db -n dev --timeout=120s
kubectl exec -it postgres-db -n dev -- psql -U postgres -d testdb -c "SELECT * FROM messages;"
```

> [!hint]- Hint
> - PostgreSQL stores data in `/var/lib/postgresql/data`
> - Set the `POSTGRES_PASSWORD` env var (required by the postgres image)
> - Use `PGDATA` env var to set `/var/lib/postgresql/data/pgdata` (subdirectory avoids mount conflicts)
> - Create a PV + PVC with at least 1Gi for PostgreSQL
> - After first pod is running, use `kubectl exec` to create the database and table

> [!success]- Solution
> **postgres-pv.yaml:**
> ```yaml
> apiVersion: v1
> kind: PersistentVolume
> metadata:
>   name: postgres-pv
> spec:
>   capacity:
>     storage: 2Gi
>   accessModes:
>     - ReadWriteOnce
>   persistentVolumeReclaimPolicy: Retain
>   storageClassName: manual
>   hostPath:
>     path: /mnt/postgres-data
> ---
> apiVersion: v1
> kind: PersistentVolumeClaim
> metadata:
>   name: postgres-pvc
>   namespace: dev
> spec:
>   accessModes:
>     - ReadWriteOnce
>   resources:
>     requests:
>       storage: 2Gi
>   storageClassName: manual
> ```
>
> **postgres-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: postgres-db
>   namespace: dev
> spec:
>   containers:
>     - name: postgres
>       image: postgres:16
>       ports:
>         - containerPort: 5432
>       env:
>         - name: POSTGRES_PASSWORD
>           value: mysecretpassword
>         - name: PGDATA
>           value: /var/lib/postgresql/data/pgdata
>       volumeMounts:
>         - name: postgres-storage
>           mountPath: /var/lib/postgresql/data
>   volumes:
>     - name: postgres-storage
>       persistentVolumeClaim:
>         claimName: postgres-pvc
>   restartPolicy: Always
> ```
>
> **Apply and set up data:**
> ```bash
> kubectl apply -f postgres-pv.yaml
> kubectl apply -f postgres-pod.yaml
> kubectl wait --for=condition=Ready pod/postgres-db -n dev --timeout=120s
>
> # Create database and insert data
> kubectl exec -it postgres-db -n dev -- psql -U postgres -c "CREATE DATABASE testdb;"
> kubectl exec -it postgres-db -n dev -- psql -U postgres -d testdb -c "
>   CREATE TABLE messages (
>     id SERIAL PRIMARY KEY,
>     content TEXT NOT NULL,
>     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
>   );
>   INSERT INTO messages (content) VALUES ('Hello, Kubernetes!');
> "
>
> # Verify data exists
> kubectl exec -it postgres-db -n dev -- psql -U postgres -d testdb -c "SELECT * FROM messages;"
> ```
>
> **Delete and recreate:**
> ```bash
> kubectl delete pod postgres-db -n dev
> kubectl apply -f postgres-pod.yaml
> kubectl wait --for=condition=Ready pod/postgres-db -n dev --timeout=120s
>
> # Verify data survived!
> kubectl exec -it postgres-db -n dev -- psql -U postgres -d testdb -c "SELECT * FROM messages;"
> ```

---

## P9: StorageClass with Dynamic Provisioning

**Task:** Use minikube's default StorageClass (`standard`) to dynamically provision a PersistentVolume. Create a PVC without pre-creating a PV. Verify that a PV is automatically created and bound.

**Expected Result:**
```
kubectl get sc
NAME                 PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE
standard (default)   k8s.io/minikube-hostpath   Delete          Immediate

kubectl get pvc -n dev
NAME          STATUS   VOLUME                                     CAPACITY
dynamic-pvc   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   1Gi

kubectl get pv
# A new PV should appear, auto-created by the provisioner
```

**Verification:**
```bash
kubectl get pvc dynamic-pvc -n dev -o jsonpath='{.status.phase}'
# Should output: Bound
kubectl exec dynamic-demo -n dev -- cat /data/dynamic.txt
# Should output: Dynamically provisioned!
```

> [!hint]- Hint
> - Check available StorageClasses: `kubectl get sc`
> - When you omit `storageClassName` or use the default, minikube auto-provisions
> - You do NOT need to create a PV manually - the provisioner handles it
> - minikube's `standard` StorageClass uses `k8s.io/minikube-hostpath` provisioner

> [!success]- Solution
> **Check available StorageClasses:**
> ```bash
> kubectl get sc
> ```
>
> **dynamic-pvc.yaml:**
> ```yaml
> apiVersion: v1
> kind: PersistentVolumeClaim
> metadata:
>   name: dynamic-pvc
>   namespace: dev
> spec:
>   accessModes:
>     - ReadWriteOnce
>   resources:
>     requests:
>       storage: 1Gi
>   storageClassName: standard
> ```
>
> **dynamic-demo-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: dynamic-demo
>   namespace: dev
> spec:
>   containers:
>     - name: demo
>       image: busybox:1.36
>       command: ["sh", "-c", "echo 'Dynamically provisioned!' > /data/dynamic.txt; sleep 3600"]
>       volumeMounts:
>         - name: dynamic-storage
>           mountPath: /data
>   volumes:
>     - name: dynamic-storage
>       persistentVolumeClaim:
>         claimName: dynamic-pvc
>   restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f dynamic-pvc.yaml
> kubectl apply -f dynamic-demo-pod.yaml
>
> # Notice: no PV was created manually!
> kubectl get pvc -n dev
> kubectl get pv
> # A PV was automatically created and bound to the PVC
>
> kubectl wait --for=condition=Ready pod/dynamic-demo -n dev --timeout=60s
> kubectl exec dynamic-demo -n dev -- cat /data/dynamic.txt
> ```

---

## P10: emptyDir Volume - Sharing Data Between Containers

**Task:** Create a pod with two containers (`producer` and `consumer`) that share an `emptyDir` volume. The producer writes a timestamp to a file every 5 seconds. The consumer reads and prints the file contents continuously.

**Expected Result:**
```bash
kubectl logs emptydir-demo -c consumer -n dev --tail=5
# Should show timestamps written by the producer:
# 2026-04-07 10:00:05 - Message from producer
# 2026-04-07 10:00:10 - Message from producer
# 2026-04-07 10:00:15 - Message from producer
```

**Verification:**
```bash
kubectl logs emptydir-demo -c producer -n dev --tail=3
kubectl logs emptydir-demo -c consumer -n dev --tail=3
# Both should reference the same shared data
```

> [!hint]- Hint
> - `emptyDir` is created when a pod is assigned to a node and exists for the pod's lifetime
> - Both containers must mount the same volume name at their respective mount paths
> - Producer: `while true; do date >> /shared/log.txt; sleep 5; done`
> - Consumer: `tail -f /shared/log.txt`
> - The volume is deleted when the pod is removed

> [!success]- Solution
> **emptydir-demo.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: emptydir-demo
>   namespace: dev
> spec:
>   containers:
>     - name: producer
>       image: busybox:1.36
>       command:
>         - sh
>         - -c
>         - |
>           while true; do
>             echo "$(date '+%Y-%m-%d %H:%M:%S') - Message from producer" >> /shared/log.txt
>             echo "Wrote message at $(date)"
>             sleep 5
>           done
>       volumeMounts:
>         - name: shared-data
>           mountPath: /shared
>     - name: consumer
>       image: busybox:1.36
>       command:
>         - sh
>         - -c
>         - |
>           echo "Waiting for log file..."
>           while [ ! -f /shared/log.txt ]; do sleep 1; done
>           echo "Found log file, tailing..."
>           tail -f /shared/log.txt
>       volumeMounts:
>         - name: shared-data
>           mountPath: /shared
>   volumes:
>     - name: shared-data
>       emptyDir: {}
>   restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f emptydir-demo.yaml
> kubectl wait --for=condition=Ready pod/emptydir-demo -n dev --timeout=60s
>
> # Wait a few seconds for some messages to accumulate
> sleep 15
>
> kubectl logs emptydir-demo -c producer -n dev --tail=5
> kubectl logs emptydir-demo -c consumer -n dev --tail=5
> ```

---

## P11: Projected Volumes - ConfigMap + Secret + Downward API

**Task:** Create a pod that uses a projected volume to combine data from a ConfigMap, a Secret, and the Downward API into a single mount point at `/projected`. Verify all three sources appear as files in the same directory.

**Expected Result:**
```bash
kubectl exec projected-demo -n dev -- ls /projected/
# Should show:
# app-env           (from ConfigMap)
# db-password       (from Secret)
# pod-name          (from Downward API)
# pod-namespace     (from Downward API)

kubectl exec projected-demo -n dev -- cat /projected/app-env
# development

kubectl exec projected-demo -n dev -- cat /projected/db-password
# s3cretP@ss

kubectl exec projected-demo -n dev -- cat /projected/pod-name
# projected-demo
```

**Verification:**
```bash
kubectl exec projected-demo -n dev -- ls -la /projected/
kubectl exec projected-demo -n dev -- sh -c 'for f in /projected/*; do echo "=== $f ==="; cat "$f"; echo; done'
```

> [!hint]- Hint
> - Use `projected` volume type with `sources` array
> - Sources can be: `configMap`, `secret`, `downwardAPI`, `serviceAccountToken`
> - Downward API exposes pod metadata via `fieldRef` (e.g., `metadata.name`, `metadata.namespace`)
> - You can reuse the `app-config` ConfigMap and `db-credentials` Secret from earlier exercises

> [!success]- Solution
> **Ensure ConfigMap and Secret exist (from P2 and P4):**
> ```bash
> # Recreate if needed
> kubectl create configmap app-config \
>   --from-literal=APP_ENV=development \
>   --from-literal=LOG_LEVEL=debug \
>   -n dev --dry-run=client -o yaml | kubectl apply -f -
>
> kubectl create secret generic db-credentials \
>   --from-literal=DB_USER=admin \
>   --from-literal=DB_PASSWORD='s3cretP@ss' \
>   -n dev --dry-run=client -o yaml | kubectl apply -f -
> ```
>
> **projected-demo.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: projected-demo
>   namespace: dev
> spec:
>   containers:
>     - name: demo
>       image: busybox:1.36
>       command: ["sh", "-c", "ls -la /projected/; echo '---'; for f in /projected/*; do echo \"=== $f ===\"; cat \"$f\"; echo; done; sleep 3600"]
>       volumeMounts:
>         - name: all-in-one
>           mountPath: /projected
>   volumes:
>     - name: all-in-one
>       projected:
>         sources:
>           - configMap:
>               name: app-config
>               items:
>                 - key: APP_ENV
>                   path: app-env
>           - secret:
>               name: db-credentials
>               items:
>                 - key: DB_PASSWORD
>                   path: db-password
>           - downwardAPI:
>               items:
>                 - path: pod-name
>                   fieldRef:
>                     fieldPath: metadata.name
>                 - path: pod-namespace
>                   fieldRef:
>                     fieldPath: metadata.namespace
>   restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f projected-demo.yaml
> kubectl wait --for=condition=Ready pod/projected-demo -n dev --timeout=60s
> kubectl logs projected-demo -n dev
> kubectl exec projected-demo -n dev -- ls /projected/
> kubectl exec projected-demo -n dev -- cat /projected/pod-name
> ```

---

## P12: ResourceQuota - Limit Namespace Resources

**Task:** Create a ResourceQuota in the `staging` namespace that limits: max 2 pods, total CPU requests to 500m, total memory requests to 512Mi. Deploy pods until you exceed the quota and observe the error.

**Expected Result:**
```bash
# First two pods succeed:
kubectl get pods -n staging
NAME              READY   STATUS    RESTARTS   AGE
quota-pod-1       1/1     Running   0          30s
quota-pod-2       1/1     Running   0          20s

# Third pod fails:
Error from server (Forbidden): pods "quota-pod-3" is forbidden:
exceeded quota: staging-quota, requested: pods=1, used: pods=2, limited: pods=2
```

**Verification:**
```bash
kubectl describe resourcequota staging-quota -n staging
# Should show used vs hard limits
```

> [!hint]- Hint
> - `ResourceQuota` is namespace-scoped
> - When a ResourceQuota exists for compute resources, pods MUST specify resource requests/limits
> - Use `spec.hard` to define limits: `pods`, `requests.cpu`, `requests.memory`, `limits.cpu`, `limits.memory`
> - Try creating a third pod to see the quota rejection

> [!success]- Solution
> **resource-quota.yaml:**
> ```yaml
> apiVersion: v1
> kind: ResourceQuota
> metadata:
>   name: staging-quota
>   namespace: staging
> spec:
>   hard:
>     pods: "2"
>     requests.cpu: "500m"
>     requests.memory: "512Mi"
>     limits.cpu: "1"
>     limits.memory: "1Gi"
> ```
>
> **quota-pod.yaml (template for each pod):**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: quota-pod-1
>   namespace: staging
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>       resources:
>         requests:
>           cpu: "200m"
>           memory: "128Mi"
>         limits:
>           cpu: "400m"
>           memory: "256Mi"
>   restartPolicy: Never
> ```
>
> **Apply and test:**
> ```bash
> kubectl apply -f resource-quota.yaml
> kubectl describe resourcequota staging-quota -n staging
>
> # Create first pod - should succeed
> kubectl apply -f quota-pod.yaml
>
> # Create second pod (change name to quota-pod-2)
> sed 's/quota-pod-1/quota-pod-2/' quota-pod.yaml | kubectl apply -f -
>
> # Check quota usage
> kubectl describe resourcequota staging-quota -n staging
>
> # Create third pod - should FAIL (exceeds pod count)
> sed 's/quota-pod-1/quota-pod-3/' quota-pod.yaml | kubectl apply -f -
> # Error: exceeded quota
>
> # Also try exceeding CPU quota (even with only 2 pods):
> cat <<'EOF' | kubectl apply -f -
> apiVersion: v1
> kind: Pod
> metadata:
>   name: quota-pod-cpu-hog
>   namespace: staging
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>       resources:
>         requests:
>           cpu: "800m"
>           memory: "128Mi"
>         limits:
>           cpu: "1"
>           memory: "256Mi"
>   restartPolicy: Never
> EOF
> # Error: exceeded quota for requests.cpu
> ```

---

## P13: LimitRange - Default Resource Requests and Limits

**Task:** Create a LimitRange in the `staging` namespace that sets default CPU request to 100m, default CPU limit to 200m, default memory request to 64Mi, and default memory limit to 128Mi. Deploy a pod WITHOUT specifying any resources. Verify that the defaults are automatically applied.

**Expected Result:**
```bash
kubectl describe pod limitrange-demo -n staging
# Should show in Containers section:
#   Limits:
#     cpu:     200m
#     memory:  128Mi
#   Requests:
#     cpu:     100m
#     memory:  64Mi
```

**Verification:**
```bash
kubectl get pod limitrange-demo -n staging -o jsonpath='{.spec.containers[0].resources}'
# Should output: {"limits":{"cpu":"200m","memory":"128Mi"},"requests":{"cpu":"100m","memory":"64Mi"}}
```

> [!hint]- Hint
> - `LimitRange` uses `spec.limits` with `type: Container`
> - `default` sets default limits, `defaultRequest` sets default requests
> - You can also set `min` and `max` to enforce boundaries
> - The defaults are injected by the LimitRange admission controller when a pod is created without resources
> - Note: if you still have the ResourceQuota from P12, the pod's defaulted resources must fit within the remaining quota

> [!success]- Solution
> **Clean up P12 pods first (to free quota room):**
> ```bash
> kubectl delete pod quota-pod-1 quota-pod-2 -n staging --ignore-not-found
> ```
>
> **limit-range.yaml:**
> ```yaml
> apiVersion: v1
> kind: LimitRange
> metadata:
>   name: default-limits
>   namespace: staging
> spec:
>   limits:
>     - type: Container
>       default:
>         cpu: "200m"
>         memory: "128Mi"
>       defaultRequest:
>         cpu: "100m"
>         memory: "64Mi"
>       min:
>         cpu: "50m"
>         memory: "32Mi"
>       max:
>         cpu: "1"
>         memory: "512Mi"
> ```
>
> **limitrange-demo.yaml (no resources specified!):**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: limitrange-demo
>   namespace: staging
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>       # NOTE: No resources section! LimitRange will inject defaults.
>   restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f limit-range.yaml
> kubectl describe limitrange default-limits -n staging
>
> kubectl apply -f limitrange-demo.yaml
> kubectl wait --for=condition=Ready pod/limitrange-demo -n staging --timeout=60s
>
> # Verify defaults were injected
> kubectl describe pod limitrange-demo -n staging | grep -A 6 "Limits:"
> kubectl get pod limitrange-demo -n staging -o jsonpath='{.spec.containers[0].resources}' | python3 -m json.tool
>
> # Should show:
> # {
> #     "limits": {
> #         "cpu": "200m",
> #         "memory": "128Mi"
> #     },
> #     "requests": {
> #         "cpu": "100m",
> #         "memory": "64Mi"
> #     }
> # }
> ```
>
> **Test the min/max enforcement:**
> ```bash
> # Try deploying a pod that exceeds the max limit:
> cat <<'EOF' | kubectl apply -f -
> apiVersion: v1
> kind: Pod
> metadata:
>   name: limitrange-exceed
>   namespace: staging
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>       resources:
>         requests:
>           cpu: "2"
>           memory: "1Gi"
>   restartPolicy: Never
> EOF
> # Error: cpu max limit is 1, but requested 2
> ```

---

## Cleanup

```bash
# Remove all resources created in these exercises
kubectl delete namespace dev staging
kubectl delete pv my-pv postgres-pv --ignore-not-found
```
