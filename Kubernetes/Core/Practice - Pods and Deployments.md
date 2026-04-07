---
title: "K8s Practice: Pods and Deployments"
date: 2026-04-07
tags:
  - kubernetes
  - practice
  - pods
  - deployments
parent: "[[Kubernetes Study]]"
---

# Practice - Pods and Deployments

15 progressive exercises building from basic Pods to production-ready Deployments. All exercises run on minikube.

## Prerequisites

- minikube installed and running: `minikube start`
- kubectl configured: `kubectl cluster-info`
- Create a working namespace: `kubectl create namespace practice`
- Set it as default: `kubectl config set-context --current --namespace=practice`

---

## P1: Simple Pod

**Task:** Write a Pod YAML manifest that runs a single nginx container. Apply it and verify it is running.

**Expected result:**
- `kubectl get pods` shows the pod in `Running` state

**Verification:**

```bash
kubectl get pods
kubectl describe pod nginx-pod
```

> [!hint]- Hint
> - API version for Pods is `v1`
> - Kind is `Pod`
> - The container spec needs `name` and `image`
> - Use image `nginx:1.25-alpine`

> [!success]- Solution
> **nginx-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: nginx-pod
>   namespace: practice
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.25-alpine
>       ports:
>         - containerPort: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f nginx-pod.yaml
> kubectl get pods
> # NAME        READY   STATUS    RESTARTS   AGE
> # nginx-pod   1/1     Running   0          10s
> 
> kubectl describe pod nginx-pod
> 
> # Cleanup
> kubectl delete pod nginx-pod
> ```

---

## P2: Labels and Annotations

**Task:** Create a Pod with labels (`app: web`, `env: dev`, `team: platform`) and annotations (`description: "Practice pod"`, `owner: "student"`). Query pods using label selectors.

**Expected result:**
- `kubectl get pods -l app=web` returns the pod
- `kubectl get pods -l env=dev,team=platform` returns the pod
- `kubectl get pods -l env=prod` returns nothing

**Verification:**

```bash
kubectl get pods --show-labels
kubectl get pods -l app=web
kubectl get pods -l env=dev,team=platform
```

> [!hint]- Hint
> - Labels go under `metadata.labels`
> - Annotations go under `metadata.annotations`
> - Use `-l` flag with `kubectl get pods` to filter by labels
> - Combine selectors with commas: `-l key1=val1,key2=val2`

> [!success]- Solution
> **labeled-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: labeled-pod
>   namespace: practice
>   labels:
>     app: web
>     env: dev
>     team: platform
>   annotations:
>     description: "Practice pod"
>     owner: "student"
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.25-alpine
>       ports:
>         - containerPort: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f labeled-pod.yaml
> 
> # Show all labels
> kubectl get pods --show-labels
> 
> # Filter by single label
> kubectl get pods -l app=web
> 
> # Filter by multiple labels (AND logic)
> kubectl get pods -l env=dev,team=platform
> 
> # This should return nothing
> kubectl get pods -l env=prod
> 
> # View annotations
> kubectl describe pod labeled-pod | grep -A 5 Annotations
> 
> # Cleanup
> kubectl delete pod labeled-pod
> ```

---

## P3: Multi-Container Pod (Sidecar Pattern)

**Task:** Create a Pod with two containers:
1. **nginx** - serves files from a shared volume mounted at `/usr/share/nginx/html`
2. **log-writer** (busybox) - writes a timestamp to a file in the shared volume every 5 seconds

**Expected result:**
- `curl`ing the nginx container returns content written by the busybox sidecar

**Verification:**

```bash
kubectl exec sidecar-pod -c nginx -- cat /usr/share/nginx/html/index.html
kubectl port-forward sidecar-pod 8080:80 &
curl http://localhost:8080
```

> [!hint]- Hint
> - Use an `emptyDir` volume shared between both containers
> - Mount it at `/usr/share/nginx/html` in nginx and `/html` in busybox
> - busybox command: `while true; do date >> /html/index.html; sleep 5; done`
> - Both containers must reference the same `volumeMounts.name`

> [!success]- Solution
> **sidecar-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: sidecar-pod
>   namespace: practice
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.25-alpine
>       ports:
>         - containerPort: 80
>       volumeMounts:
>         - name: shared-data
>           mountPath: /usr/share/nginx/html
> 
>     - name: log-writer
>       image: busybox:1.36
>       command:
>         - sh
>         - -c
>         - "while true; do date >> /html/index.html; sleep 5; done"
>       volumeMounts:
>         - name: shared-data
>           mountPath: /html
> 
>   volumes:
>     - name: shared-data
>       emptyDir: {}
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f sidecar-pod.yaml
> 
> # Wait for both containers to be ready
> kubectl get pods sidecar-pod
> # READY should show 2/2
> 
> # Check the content being written
> kubectl exec sidecar-pod -c nginx -- cat /usr/share/nginx/html/index.html
> 
> # Port forward and curl
> kubectl port-forward sidecar-pod 8080:80 &
> curl http://localhost:8080
> # Should show timestamps written by the busybox sidecar
> 
> kill %1  # Stop port-forward
> 
> # Cleanup
> kubectl delete pod sidecar-pod
> ```

---

## P4: Resource Requests and Limits (OOMKilled)

**Task:** Create a Pod with resource requests and limits. Then create a second Pod with a very low memory limit (5Mi) running a memory-hungry process to observe OOMKilled.

**Expected result:**
- First pod runs fine with reasonable limits
- Second pod gets OOMKilled due to the low memory limit

**Verification:**

```bash
kubectl get pods
kubectl describe pod memory-hog
# Look for OOMKilled in Last State
```

> [!hint]- Hint
> - Resources go under `spec.containers[].resources`
> - `requests` = minimum guaranteed, `limits` = maximum allowed
> - To trigger OOM: use a `stress` tool or busybox allocating memory
> - `busybox` command to eat memory: `dd if=/dev/zero of=/dev/null bs=10M`
> - Or use image `polinux/stress` with `--vm 1 --vm-bytes 50M`

> [!success]- Solution
> **resource-pod.yaml (healthy pod):**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: resource-pod
>   namespace: practice
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.25-alpine
>       resources:
>         requests:
>           cpu: "100m"
>           memory: "64Mi"
>         limits:
>           cpu: "200m"
>           memory: "128Mi"
> ```
> 
> **memory-hog.yaml (will OOMKill):**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: memory-hog
>   namespace: practice
> spec:
>   containers:
>     - name: stress
>       image: polinux/stress
>       command: ["stress"]
>       args: ["--vm", "1", "--vm-bytes", "50M", "--vm-hang", "1"]
>       resources:
>         requests:
>           memory: "5Mi"
>         limits:
>           memory: "5Mi"
> ```
> 
> **Commands:**
> ```bash
> # Healthy pod
> kubectl apply -f resource-pod.yaml
> kubectl get pods resource-pod
> # Should be Running
> 
> # Memory hog (will OOMKill)
> kubectl apply -f memory-hog.yaml
> 
> # Watch it crash
> kubectl get pods memory-hog -w
> # STATUS will show OOMKilled, then CrashLoopBackOff
> 
> # Check details
> kubectl describe pod memory-hog | grep -A 3 "Last State"
> # Reason: OOMKilled
> 
> # Cleanup
> kubectl delete pod resource-pod memory-hog
> ```

---

## P5: Liveness Probe

**Task:** Deploy an app that becomes unhealthy after 30 seconds. Add an HTTP liveness probe so Kubernetes automatically restarts the container when it becomes unhealthy.

**Expected result:**
- Pod starts and is healthy initially
- After ~30s, the liveness probe fails
- Kubernetes restarts the container (RESTARTS count increases)

**Verification:**

```bash
kubectl get pods -w
# Watch the RESTARTS column increment
```

> [!hint]- Hint
> - Use image `registry.k8s.io/e2e-test-images/agnhost:2.40` with args `liveness-http`
> - This image returns 200 for the first 10 seconds, then 500
> - Or use `busybox` with a health file that gets deleted after 30s
> - Liveness probe: `spec.containers[].livenessProbe`
> - Set `initialDelaySeconds`, `periodSeconds`, `failureThreshold`

> [!success]- Solution
> **liveness-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: liveness-pod
>   namespace: practice
> spec:
>   containers:
>     - name: app
>       image: busybox:1.36
>       command:
>         - sh
>         - -c
>         - |
>           touch /tmp/healthy
>           echo "App started"
>           sleep 30
>           rm -f /tmp/healthy
>           echo "App is now unhealthy"
>           sleep 600
>       livenessProbe:
>         exec:
>           command:
>             - cat
>             - /tmp/healthy
>         initialDelaySeconds: 5
>         periodSeconds: 5
>         failureThreshold: 3
> ```
> 
> **With HTTP probe (alternative):**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: liveness-http-pod
>   namespace: practice
> spec:
>   containers:
>     - name: app
>       image: registry.k8s.io/e2e-test-images/agnhost:2.40
>       args: ["liveness-http"]
>       livenessProbe:
>         httpGet:
>           path: /healthz
>           port: 8080
>         initialDelaySeconds: 3
>         periodSeconds: 3
>         failureThreshold: 3
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f liveness-pod.yaml
> 
> # Watch the pod - RESTARTS will increment after ~40s
> kubectl get pods liveness-pod -w
> 
> # Check events
> kubectl describe pod liveness-pod | grep -A 10 Events
> # Should see "Liveness probe failed" and "Killing" events
> 
> # Cleanup
> kubectl delete pod liveness-pod
> ```

---

## P6: Readiness Probe

**Task:** Deploy a Pod with a readiness probe. The app has a slow startup (takes 15 seconds before it is ready to serve traffic). The readiness probe should prevent traffic from being sent to the Pod until it is ready.

**Expected result:**
- Pod starts but shows `0/1 READY` for the first 15 seconds
- After the readiness probe passes, it becomes `1/1 READY`

**Verification:**

```bash
kubectl get pods -w
# Watch READY column change from 0/1 to 1/1
```

> [!hint]- Hint
> - Readiness probe: `spec.containers[].readinessProbe`
> - Use the same syntax as liveness but the purpose is different
> - A failing readiness probe removes the pod from Service endpoints (no traffic)
> - A failing liveness probe restarts the container
> - Simulate slow startup: `sleep 15 && touch /tmp/ready && sleep 600`

> [!success]- Solution
> **readiness-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: readiness-pod
>   namespace: practice
>   labels:
>     app: slow-start
> spec:
>   containers:
>     - name: app
>       image: busybox:1.36
>       command:
>         - sh
>         - -c
>         - |
>           echo "Starting slow initialization..."
>           sleep 15
>           echo "Ready!"
>           touch /tmp/ready
>           sleep 600
>       readinessProbe:
>         exec:
>           command:
>             - cat
>             - /tmp/ready
>         initialDelaySeconds: 5
>         periodSeconds: 3
>         failureThreshold: 1
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f readiness-pod.yaml
> 
> # Watch the pod - READY will be 0/1 initially
> kubectl get pods readiness-pod -w
> # NAME             READY   STATUS    RESTARTS   AGE
> # readiness-pod    0/1     Running   0          5s    <-- not ready
> # readiness-pod    1/1     Running   0          20s   <-- ready!
> 
> # Note: the pod is NOT restarted (unlike liveness)
> # It just isn't included in Service endpoints until ready
> 
> # Check endpoint readiness
> kubectl describe pod readiness-pod | grep -i "ready"
> 
> # Cleanup
> kubectl delete pod readiness-pod
> ```

---

## P7: Deployment with Replicas

**Task:** Create a Deployment with 3 replicas of nginx. Verify the Deployment creates a ReplicaSet, and the ReplicaSet manages 3 Pods.

**Expected result:**
- 1 Deployment, 1 ReplicaSet, 3 Pods - all running
- Deleting a Pod causes the ReplicaSet to create a replacement

**Verification:**

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

> [!hint]- Hint
> - API version: `apps/v1`
> - Kind: `Deployment`
> - `spec.replicas: 3`
> - The `spec.selector.matchLabels` must match `spec.template.metadata.labels`
> - This is a common mistake - if they don't match, the Deployment won't work

> [!success]- Solution
> **nginx-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx-deployment
>   namespace: practice
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: nginx
>   template:
>     metadata:
>       labels:
>         app: nginx
>     spec:
>       containers:
>         - name: nginx
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f nginx-deployment.yaml
> 
> # Check all three levels
> kubectl get deployments
> # NAME               READY   UP-TO-DATE   AVAILABLE   AGE
> # nginx-deployment   3/3     3            3           10s
> 
> kubectl get replicasets
> # NAME                         DESIRED   CURRENT   READY   AGE
> # nginx-deployment-xxxxx       3         3         3       10s
> 
> kubectl get pods
> # Should show 3 pods with names like nginx-deployment-xxxxx-yyyyy
> 
> # Delete a pod - ReplicaSet will replace it
> kubectl delete pod $(kubectl get pods -l app=nginx -o jsonpath='{.items[0].metadata.name}')
> kubectl get pods
> # Still 3 pods! The deleted one is replaced.
> 
> # Cleanup
> kubectl delete deployment nginx-deployment
> ```

---

## P8: Rolling Update

**Task:** Update the Deployment image from `nginx:1.25-alpine` to `nginx:1.26-alpine`. Watch the rolling update in progress.

**Expected result:**
- Pods are updated one by one (old pods terminated, new pods created)
- `kubectl rollout status` shows the rollout completing
- All pods end up running the new image

**Verification:**

```bash
kubectl rollout status deployment/nginx-deployment
kubectl get pods -o jsonpath='{.items[*].spec.containers[0].image}'
```

> [!hint]- Hint
> - Apply the Deployment from P7 first if not already running
> - Update with: `kubectl set image deployment/nginx-deployment nginx=nginx:1.26-alpine`
> - Or edit the YAML and `kubectl apply -f`
> - Watch rollout: `kubectl rollout status deployment/nginx-deployment`
> - Check history: `kubectl rollout history deployment/nginx-deployment`

> [!success]- Solution
> ```bash
> # Make sure the deployment from P7 is running
> kubectl apply -f nginx-deployment.yaml
> 
> # Update the image
> kubectl set image deployment/nginx-deployment nginx=nginx:1.26-alpine
> 
> # Watch the rollout
> kubectl rollout status deployment/nginx-deployment
> # Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
> # ...
> # deployment "nginx-deployment" successfully rolled out
> 
> # Verify all pods have new image
> kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
> # All should show nginx:1.26-alpine
> 
> # Check rollout history
> kubectl rollout history deployment/nginx-deployment
> # REVISION  CHANGE-CAUSE
> # 1         <none>
> # 2         <none>
> 
> # Check the ReplicaSets - you should see old (0 replicas) and new (3 replicas)
> kubectl get replicasets
> ```

---

## P9: Rollback a Deployment

**Task:** Deploy a "bad" image version (e.g., `nginx:nonexistent`), observe the failed rollout, then rollback to the previous working revision.

**Expected result:**
- The bad update creates pods that fail with `ImagePullBackOff`
- Rollback restores the previous working version
- All pods are healthy again after rollback

**Verification:**

```bash
kubectl rollout undo deployment/nginx-deployment
kubectl get pods
```

> [!hint]- Hint
> - Set a bad image: `kubectl set image deployment/nginx-deployment nginx=nginx:nonexistent`
> - The old ReplicaSet keeps the old pods running (default strategy keeps some available)
> - Rollback: `kubectl rollout undo deployment/nginx-deployment`
> - To rollback to a specific revision: `kubectl rollout undo deployment/nginx-deployment --to-revision=1`
> - Check history: `kubectl rollout history deployment/nginx-deployment`

> [!success]- Solution
> ```bash
> # Deploy a bad image
> kubectl set image deployment/nginx-deployment nginx=nginx:nonexistent
> 
> # Watch - new pods will fail
> kubectl get pods -w
> # Some pods will show ImagePullBackOff or ErrImagePull
> 
> # Check rollout status (will hang/fail)
> kubectl rollout status deployment/nginx-deployment --timeout=30s
> # error: deployment "nginx-deployment" exceeded its progress deadline
> 
> # Rollback to previous version
> kubectl rollout undo deployment/nginx-deployment
> 
> # Verify recovery
> kubectl rollout status deployment/nginx-deployment
> # deployment "nginx-deployment" successfully rolled out
> 
> kubectl get pods
> # All pods should be Running with the previous good image
> 
> # Check which revision we're on
> kubectl rollout history deployment/nginx-deployment
> 
> # Cleanup
> kubectl delete deployment nginx-deployment
> ```

---

## P10: Scaling a Deployment

**Task:** Create a Deployment with 2 replicas. Scale it up to 5, verify, then scale down to 1.

**Expected result:**
- After scale up: 5 pods running
- After scale down: 1 pod running
- Scaling is immediate - no rolling update needed

**Verification:**

```bash
kubectl get pods
kubectl get deployment nginx-deployment
```

> [!hint]- Hint
> - `kubectl scale deployment/nginx-deployment --replicas=5`
> - Or edit the YAML and `kubectl apply`
> - Or `kubectl patch deployment nginx-deployment -p '{"spec":{"replicas":5}}'`
> - Watch pods appear/disappear: `kubectl get pods -w`

> [!success]- Solution
> **nginx-deployment.yaml (start with 2 replicas):**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx-deployment
>   namespace: practice
> spec:
>   replicas: 2
>   selector:
>     matchLabels:
>       app: nginx
>   template:
>     metadata:
>       labels:
>         app: nginx
>     spec:
>       containers:
>         - name: nginx
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f nginx-deployment.yaml
> kubectl get pods
> # 2 pods running
> 
> # Scale up to 5
> kubectl scale deployment/nginx-deployment --replicas=5
> kubectl get pods
> # 5 pods running
> 
> # Scale down to 1
> kubectl scale deployment/nginx-deployment --replicas=1
> kubectl get pods
> # 1 pod running
> 
> # Verify via deployment
> kubectl get deployment nginx-deployment
> # READY 1/1
> 
> # Cleanup
> kubectl delete deployment nginx-deployment
> ```

---

## P11: Rolling Update Strategy (Zero-Downtime)

**Task:** Configure a Deployment with a rolling update strategy that ensures zero downtime: `maxSurge=1`, `maxUnavailable=0`. This means Kubernetes will create a new pod BEFORE terminating an old one.

**Expected result:**
- During rollout, pod count is always >= replicas (never below 3)
- New pods are created first, old pods terminated after

**Verification:**

```bash
kubectl describe deployment nginx-deployment | grep -A 5 Strategy
kubectl get pods -w  # During rollout, count should never drop below 3
```

> [!hint]- Hint
> - Strategy goes under `spec.strategy`
> - `type: RollingUpdate` with `rollingUpdate.maxSurge: 1` and `rollingUpdate.maxUnavailable: 0`
> - `maxSurge: 1` = at most 1 extra pod during update
> - `maxUnavailable: 0` = never reduce available pods below desired count

> [!success]- Solution
> **zero-downtime-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx-deployment
>   namespace: practice
> spec:
>   replicas: 3
>   strategy:
>     type: RollingUpdate
>     rollingUpdate:
>       maxSurge: 1
>       maxUnavailable: 0
>   selector:
>     matchLabels:
>       app: nginx
>   template:
>     metadata:
>       labels:
>         app: nginx
>     spec:
>       containers:
>         - name: nginx
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f zero-downtime-deployment.yaml
> kubectl get pods
> # 3 pods running
> 
> # Trigger a rolling update and watch
> kubectl set image deployment/nginx-deployment nginx=nginx:1.26-alpine
> 
> # In another terminal, watch pod count (should never go below 3)
> kubectl get pods -w
> # You'll see 4 pods briefly (3 old + 1 new), then back to 3
> 
> # Verify strategy
> kubectl describe deployment nginx-deployment | grep -A 5 "RollingUpdate"
> # Max unavailable: 0
> # Max surge: 1
> 
> # Cleanup
> kubectl delete deployment nginx-deployment
> ```

---

## P12: Init Containers

**Task:** Create a Pod with an init container that waits for a Service to become available. The main container only starts after the init container completes.

**Expected result:**
- The init container keeps the Pod in `Init:0/1` status until the service exists
- After creating the service, the init container completes and the main container starts

**Verification:**

```bash
kubectl get pods -w
# Watch it go from Init:0/1 to Running
```

> [!hint]- Hint
> - Init containers go under `spec.initContainers` (same syntax as `containers`)
> - Use `busybox` with `nslookup` to wait for a service DNS name
> - Command: `until nslookup mydb-service.practice.svc.cluster.local; do sleep 2; done`
> - Create the service separately to unblock the init container

> [!success]- Solution
> **init-container-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: init-demo
>   namespace: practice
> spec:
>   initContainers:
>     - name: wait-for-db
>       image: busybox:1.36
>       command:
>         - sh
>         - -c
>         - |
>           echo "Waiting for mydb-service..."
>           until nslookup mydb-service.practice.svc.cluster.local; do
>             echo "Service not found, retrying in 2s..."
>             sleep 2
>           done
>           echo "Service found! Init complete."
> 
>   containers:
>     - name: app
>       image: nginx:1.25-alpine
>       ports:
>         - containerPort: 80
> ```
> 
> **mydb-service.yaml:**
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: mydb-service
>   namespace: practice
> spec:
>   selector:
>     app: mydb
>   ports:
>     - port: 5432
>       targetPort: 5432
> ```
> 
> **Commands:**
> ```bash
> # Apply the pod first - it will wait
> kubectl apply -f init-container-pod.yaml
> 
> # Watch - init container is waiting
> kubectl get pods -w
> # NAME        READY   STATUS     RESTARTS   AGE
> # init-demo   0/1     Init:0/1   0          5s
> 
> # Check init container logs
> kubectl logs init-demo -c wait-for-db
> 
> # Now create the service to unblock the init container
> kubectl apply -f mydb-service.yaml
> 
> # Watch the pod transition
> kubectl get pods -w
> # init-demo   0/1     Init:0/1   0          30s
> # init-demo   0/1     PodInitializing   0   35s
> # init-demo   1/1     Running           0   36s
> 
> # Cleanup
> kubectl delete pod init-demo
> kubectl delete service mydb-service
> ```

---

## P13: Pod Disruption Budget

**Task:** Create a Deployment with 3 replicas and a PodDisruptionBudget (PDB) requiring at least 2 pods to be available at all times. Attempt to drain a node and observe the PDB in action.

**Expected result:**
- PDB prevents Kubernetes from evicting more than 1 pod at a time
- `kubectl drain` respects the PDB

**Verification:**

```bash
kubectl get pdb
kubectl describe pdb nginx-pdb
```

> [!hint]- Hint
> - API version: `policy/v1`
> - Kind: `PodDisruptionBudget`
> - `spec.minAvailable: 2` or `spec.maxUnavailable: 1` (equivalent for 3 replicas)
> - The PDB's `spec.selector.matchLabels` must match the Deployment's pod labels
> - With minikube (single node), drain will be blocked until the PDB can be satisfied

> [!success]- Solution
> **nginx-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx-deployment
>   namespace: practice
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: nginx
>   template:
>     metadata:
>       labels:
>         app: nginx
>     spec:
>       containers:
>         - name: nginx
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 80
> ```
> 
> **nginx-pdb.yaml:**
> ```yaml
> apiVersion: policy/v1
> kind: PodDisruptionBudget
> metadata:
>   name: nginx-pdb
>   namespace: practice
> spec:
>   minAvailable: 2
>   selector:
>     matchLabels:
>       app: nginx
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f nginx-deployment.yaml
> kubectl apply -f nginx-pdb.yaml
> 
> # Check PDB
> kubectl get pdb
> # NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
> # nginx-pdb   2               N/A               1                     10s
> 
> kubectl describe pdb nginx-pdb
> 
> # Try to drain the minikube node (will be constrained by PDB)
> # Note: on single-node minikube, this will block because pods can't reschedule
> kubectl drain minikube --ignore-daemonsets --delete-emptydir-data --timeout=30s
> # Will show: "evicting pod practice/nginx-deployment-xxxx"
> # Eventually times out on single-node since pods can't move elsewhere
> 
> # Uncordon the node to restore scheduling
> kubectl uncordon minikube
> 
> # Cleanup
> kubectl delete pdb nginx-pdb
> kubectl delete deployment nginx-deployment
> ```

---

## P14: Node Affinity

**Task:** Label a node with `disktype=ssd` and create a Deployment that only schedules pods on nodes with that label using node affinity.

**Expected result:**
- Pods are scheduled on the labeled node
- Removing the label does NOT evict existing pods (preferred vs required behavior)

**Verification:**

```bash
kubectl get pods -o wide
kubectl describe pod <pod-name> | grep Node:
```

> [!hint]- Hint
> - Label a node: `kubectl label nodes minikube disktype=ssd`
> - Node affinity goes under `spec.template.spec.affinity.nodeAffinity`
> - `requiredDuringSchedulingIgnoredDuringExecution` = must match (hard requirement)
> - `preferredDuringSchedulingIgnoredDuringExecution` = prefer but not required (soft)
> - Use `nodeSelectorTerms` with `matchExpressions`

> [!success]- Solution
> ```bash
> # Label the node
> kubectl label nodes minikube disktype=ssd
> ```
> 
> **affinity-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: ssd-deployment
>   namespace: practice
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: ssd-app
>   template:
>     metadata:
>       labels:
>         app: ssd-app
>     spec:
>       affinity:
>         nodeAffinity:
>           requiredDuringSchedulingIgnoredDuringExecution:
>             nodeSelectorTerms:
>               - matchExpressions:
>                   - key: disktype
>                     operator: In
>                     values:
>                       - ssd
>       containers:
>         - name: nginx
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f affinity-deployment.yaml
> 
> # Verify pods are scheduled on the labeled node
> kubectl get pods -o wide
> # NODE column should show "minikube"
> 
> # Remove the label - existing pods are NOT evicted (IgnoredDuringExecution)
> kubectl label nodes minikube disktype-
> 
> kubectl get pods
> # Pods still running!
> 
> # But new pods won't schedule - scale up to test
> kubectl scale deployment ssd-deployment --replicas=5
> kubectl get pods
> # New pods will be Pending (no matching nodes)
> 
> # Re-add label to fix
> kubectl label nodes minikube disktype=ssd
> kubectl get pods
> # Pending pods now schedule
> 
> # Cleanup
> kubectl delete deployment ssd-deployment
> kubectl label nodes minikube disktype-
> ```

---

## P15: Comprehensive Deployment (Everything Combined)

**Task:** Create a production-ready Deployment that combines everything learned:
- 3 replicas with rolling update strategy (maxSurge=1, maxUnavailable=0)
- Resource requests and limits
- Liveness and readiness probes
- Init container (waits for a config service)
- Pod anti-affinity (spread pods across nodes)
- Pod disruption budget

This is the "final exam" - write it entirely from memory if possible.

**Expected result:**
- All components work together: init container completes, probes pass, pods spread across nodes
- Rolling updates maintain zero downtime
- PDB protects against excessive disruption

**Verification:**

```bash
kubectl get all -l app=production-app
kubectl get pdb
kubectl describe deployment production-app
```

> [!hint]- Hint
> - Start with the Deployment skeleton, then add each feature one by one
> - Anti-affinity uses `spec.template.spec.affinity.podAntiAffinity`
> - Use `preferredDuringSchedulingIgnoredDuringExecution` for anti-affinity on minikube (single node)
> - If you use `required`, pods will be Pending on a single-node cluster since they can't spread
> - Liveness/readiness can both use exec probes on the same file if using busybox

> [!success]- Solution
> **Create the config service first (so init container can pass):**
> ```yaml
> # config-service.yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: config-service
>   namespace: practice
> spec:
>   selector:
>     app: config
>   ports:
>     - port: 80
> ```
> 
> **production-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: production-app
>   namespace: practice
> spec:
>   replicas: 3
>   strategy:
>     type: RollingUpdate
>     rollingUpdate:
>       maxSurge: 1
>       maxUnavailable: 0
>   selector:
>     matchLabels:
>       app: production-app
>   template:
>     metadata:
>       labels:
>         app: production-app
>     spec:
>       affinity:
>         podAntiAffinity:
>           preferredDuringSchedulingIgnoredDuringExecution:
>             - weight: 100
>               podAffinityTerm:
>                 labelSelector:
>                   matchExpressions:
>                     - key: app
>                       operator: In
>                       values:
>                         - production-app
>                 topologyKey: kubernetes.io/hostname
> 
>       initContainers:
>         - name: wait-for-config
>           image: busybox:1.36
>           command:
>             - sh
>             - -c
>             - |
>               until nslookup config-service.practice.svc.cluster.local; do
>                 echo "Waiting for config-service..."
>                 sleep 2
>               done
>               echo "Config service is available!"
> 
>       containers:
>         - name: app
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 80
>           resources:
>             requests:
>               cpu: "100m"
>               memory: "64Mi"
>             limits:
>               cpu: "200m"
>               memory: "128Mi"
>           livenessProbe:
>             httpGet:
>               path: /
>               port: 80
>             initialDelaySeconds: 10
>             periodSeconds: 10
>             failureThreshold: 3
>           readinessProbe:
>             httpGet:
>               path: /
>               port: 80
>             initialDelaySeconds: 5
>             periodSeconds: 5
>             failureThreshold: 3
> ```
> 
> **production-pdb.yaml:**
> ```yaml
> apiVersion: policy/v1
> kind: PodDisruptionBudget
> metadata:
>   name: production-app-pdb
>   namespace: practice
> spec:
>   minAvailable: 2
>   selector:
>     matchLabels:
>       app: production-app
> ```
> 
> **Commands:**
> ```bash
> # Create config service first (so init containers pass)
> kubectl apply -f config-service.yaml
> 
> # Deploy the application
> kubectl apply -f production-deployment.yaml
> kubectl apply -f production-pdb.yaml
> 
> # Watch everything come up
> kubectl get pods -w
> # Init containers complete, then main containers start with probes
> 
> # Verify all components
> kubectl get all -l app=production-app
> kubectl get pdb production-app-pdb
> kubectl describe deployment production-app
> 
> # Verify probes are configured
> kubectl describe pod -l app=production-app | grep -A 5 "Liveness\|Readiness"
> 
> # Test rolling update (zero downtime)
> kubectl set image deployment/production-app app=nginx:1.26-alpine
> kubectl rollout status deployment/production-app
> 
> # Verify anti-affinity (on multi-node clusters, pods would be on different nodes)
> kubectl get pods -o wide
> 
> # Cleanup
> kubectl delete deployment production-app
> kubectl delete pdb production-app-pdb
> kubectl delete service config-service
> ```
