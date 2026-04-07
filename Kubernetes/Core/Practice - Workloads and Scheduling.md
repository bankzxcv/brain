---
title: "K8s Practice: Workloads and Scheduling"
date: 2026-04-07
tags:
  - kubernetes
  - practice
  - workloads
  - scheduling
parent: "[[Kubernetes Study]]"
---

# K8s Practice: Workloads and Scheduling

13 progressive exercises covering Jobs, CronJobs, DaemonSets, StatefulSets, HPA, Taints/Tolerations, Pod Priority, RBAC, and Pod Security Standards. Each exercise builds on concepts from the previous ones.

## Prerequisites

- minikube installed and running (`minikube start`)
- kubectl installed and configured
- metrics-server enabled in minikube (`minikube addons enable metrics-server`)
- Basic familiarity with YAML syntax, kubectl commands, and Kubernetes core concepts (Pods, Services, Deployments)
- Completed [[Practice - Configuration and Storage]] (recommended, not required)

```bash
# Verify your environment
minikube status
kubectl cluster-info
kubectl get nodes
minikube addons enable metrics-server
kubectl top nodes  # may take a minute after enabling
```

---

## P1: Job - Batch Computation

**Task:** Create a Job named `fibonacci-job` that runs a container which computes and prints the first 20 Fibonacci numbers, then exits. Verify the Job completes successfully and check the output.

**Expected Result:**
```bash
kubectl get jobs
NAME            COMPLETIONS   DURATION   AGE
fibonacci-job   1/1           5s         30s

kubectl logs job/fibonacci-job
# Should print Fibonacci sequence:
# 0 1 1 2 3 5 8 13 21 34 55 89 144 233 377 610 987 1597 2584 4181
```

**Verification:**
```bash
kubectl get job fibonacci-job -o jsonpath='{.status.succeeded}'
# Should output: 1
kubectl get pods -l job-name=fibonacci-job
# Pod should show Completed status
```

> [!hint]- Hint
> - Jobs create pods that run to completion (unlike Deployments which restart)
> - Use `spec.template.spec.restartPolicy: Never` (Jobs do not allow `Always`)
> - You can compute Fibonacci in a shell one-liner with `awk` or `python`
> - Use `backoffLimit` to control how many times a failed Job retries

> [!success]- Solution
> **fibonacci-job.yaml:**
> ```yaml
> apiVersion: batch/v1
> kind: Job
> metadata:
>   name: fibonacci-job
> spec:
>   backoffLimit: 3
>   template:
>     spec:
>       containers:
>         - name: fibonacci
>           image: python:3.12-slim
>           command:
>             - python
>             - -c
>             - |
>               a, b = 0, 1
>               result = []
>               for _ in range(20):
>                   result.append(str(a))
>                   a, b = b, a + b
>               print(' '.join(result))
>       restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f fibonacci-job.yaml
> kubectl wait --for=condition=Complete job/fibonacci-job --timeout=120s
> kubectl get jobs
> kubectl logs job/fibonacci-job
> kubectl get pods -l job-name=fibonacci-job
> ```

---

## P2: Job with Parallelism and Completions

**Task:** Create a Job named `parallel-job` with `completions: 5` and `parallelism: 2`. Each pod should print its hostname and sleep for 5 seconds. Observe that at most 2 pods run at a time, and the Job completes when all 5 finish.

**Expected Result:**
```bash
kubectl get jobs parallel-job --watch
NAME           COMPLETIONS   DURATION   AGE
parallel-job   0/5           1s         1s
parallel-job   1/5           7s         7s
parallel-job   2/5           7s         7s    # <-- 2 complete at roughly same time
parallel-job   3/5           14s        14s
parallel-job   4/5           14s        14s   # <-- next 2 complete together
parallel-job   5/5           21s        21s   # <-- last one
```

**Verification:**
```bash
kubectl get pods -l job-name=parallel-job
# Should show 5 pods, all Completed
kubectl logs -l job-name=parallel-job --prefix
# Should show 5 different hostnames
```

> [!hint]- Hint
> - `completions` = total number of successful pod completions needed
> - `parallelism` = max number of pods running concurrently
> - With completions=5 and parallelism=2: runs 2 at a time until all 5 are done
> - Use `--watch` flag to observe the Job in real-time
> - Each pod gets a unique hostname like `parallel-job-xxxxx`

> [!success]- Solution
> **parallel-job.yaml:**
> ```yaml
> apiVersion: batch/v1
> kind: Job
> metadata:
>   name: parallel-job
> spec:
>   completions: 5
>   parallelism: 2
>   template:
>     spec:
>       containers:
>         - name: worker
>           image: busybox:1.36
>           command:
>             - sh
>             - -c
>             - |
>               echo "Pod $(hostname) started at $(date)"
>               sleep 5
>               echo "Pod $(hostname) finished at $(date)"
>       restartPolicy: Never
> ```
>
> **Apply and observe:**
> ```bash
> kubectl apply -f parallel-job.yaml
>
> # Watch pods in real-time (in a separate terminal):
> kubectl get pods -l job-name=parallel-job --watch
>
> # Watch job completions:
> kubectl get jobs parallel-job --watch
>
> # After completion, check all logs:
> kubectl wait --for=condition=Complete job/parallel-job --timeout=120s
> kubectl logs -l job-name=parallel-job --prefix
> kubectl get pods -l job-name=parallel-job
> ```

---

## P3: CronJob - Scheduled Execution

**Task:** Create a CronJob named `timestamp-cron` that runs every minute, printing the current timestamp and a random quote. Wait at least 3 minutes and verify that multiple Jobs have been created on schedule.

**Expected Result:**
```bash
kubectl get cronjobs
NAME             SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
timestamp-cron   */1 * * * *   False     0        30s             3m

kubectl get jobs
NAME                          COMPLETIONS   DURATION   AGE
timestamp-cron-1712484060     1/1           3s         3m
timestamp-cron-1712484120     1/1           3s         2m
timestamp-cron-1712484180     1/1           3s         1m
```

**Verification:**
```bash
kubectl get jobs -l job-name  # should show multiple jobs
kubectl logs job/$(kubectl get jobs -o jsonpath='{.items[0].metadata.name}')
# Should show a timestamp
```

> [!hint]- Hint
> - CronJob schedule uses standard cron format: `*/1 * * * *` means every minute
> - `successfulJobsHistoryLimit` and `failedJobsHistoryLimit` control how many old Jobs to keep
> - `concurrencyPolicy` controls what happens if a new schedule fires while the previous is still running
> - CronJobs create Jobs, which create Pods

> [!success]- Solution
> **timestamp-cron.yaml:**
> ```yaml
> apiVersion: batch/v1
> kind: CronJob
> metadata:
>   name: timestamp-cron
> spec:
>   schedule: "*/1 * * * *"
>   successfulJobsHistoryLimit: 3
>   failedJobsHistoryLimit: 1
>   concurrencyPolicy: Forbid
>   jobTemplate:
>     spec:
>       template:
>         spec:
>           containers:
>             - name: timestamp
>               image: busybox:1.36
>               command:
>                 - sh
>                 - -c
>                 - |
>                   echo "=== Cron Run ==="
>                   echo "Timestamp: $(date '+%Y-%m-%d %H:%M:%S')"
>                   echo "Hostname: $(hostname)"
>                   echo "================"
>           restartPolicy: OnFailure
> ```
>
> **Apply and monitor:**
> ```bash
> kubectl apply -f timestamp-cron.yaml
> kubectl get cronjobs
>
> # Wait and watch Jobs being created:
> kubectl get jobs --watch
>
> # After a few minutes:
> kubectl get jobs
> kubectl get cronjobs
>
> # Check logs from the most recent run:
> kubectl logs job/$(kubectl get jobs --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
>
> # Manual trigger (create a Job from the CronJob immediately):
> kubectl create job timestamp-manual --from=cronjob/timestamp-cron
> kubectl logs job/timestamp-manual
> ```

---

## P4: DaemonSet - Log Collector on Every Node

**Task:** Create a DaemonSet named `log-collector` that deploys a fluentd-like log collector pod on every node in the cluster. The pod should mount the host's `/var/log` directory and tail the syslog.

**Expected Result:**
```bash
kubectl get daemonsets
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
log-collector   1         1         1       1             1           <none>          30s

kubectl get pods -l app=log-collector -o wide
# Should show one pod per node (minikube has 1 node)
```

**Verification:**
```bash
kubectl get ds log-collector -o jsonpath='{.status.numberReady}'
# Should equal the number of nodes
kubectl logs -l app=log-collector --tail=5
# Should show log lines from the node
```

> [!hint]- Hint
> - DaemonSets ensure exactly one pod runs on each (or selected) node
> - Unlike Deployments, you do not specify `replicas` - it auto-scales with nodes
> - Mount host's `/var/log` using `hostPath` volume
> - On minikube, log files may be at `/var/log/syslog` or `/var/log/messages` or `/var/log/containers/`
> - Use `tolerations` if you want the DaemonSet to also run on control-plane nodes

> [!success]- Solution
> **log-collector-ds.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: DaemonSet
> metadata:
>   name: log-collector
>   labels:
>     app: log-collector
> spec:
>   selector:
>     matchLabels:
>       app: log-collector
>   template:
>     metadata:
>       labels:
>         app: log-collector
>     spec:
>       tolerations:
>         - key: node-role.kubernetes.io/control-plane
>           operator: Exists
>           effect: NoSchedule
>       containers:
>         - name: log-collector
>           image: busybox:1.36
>           command:
>             - sh
>             - -c
>             - |
>               echo "Log collector started on $(hostname)"
>               echo "Watching /var/log for log files..."
>               # Tail container logs on the node
>               if [ -d /var/log/containers ]; then
>                 tail -f /var/log/containers/*.log 2>/dev/null || \
>                 echo "No container logs found, sleeping..."
>               fi
>               sleep 3600
>           volumeMounts:
>             - name: varlog
>               mountPath: /var/log
>               readOnly: true
>           resources:
>             requests:
>               cpu: 50m
>               memory: 64Mi
>             limits:
>               cpu: 100m
>               memory: 128Mi
>       volumes:
>         - name: varlog
>           hostPath:
>             path: /var/log
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f log-collector-ds.yaml
> kubectl rollout status daemonset/log-collector --timeout=60s
>
> kubectl get daemonsets
> kubectl get pods -l app=log-collector -o wide
> kubectl logs -l app=log-collector --tail=10
>
> # Verify it runs on every node:
> echo "Nodes: $(kubectl get nodes --no-headers | wc -l)"
> echo "DaemonSet pods: $(kubectl get pods -l app=log-collector --no-headers | wc -l)"
> ```

---

## P5: StatefulSet with Headless Service and VolumeClaimTemplates

**Task:** Create a headless Service named `redis-headless` and a StatefulSet named `redis` with 3 replicas. Each replica should have its own PersistentVolumeClaim (via `volumeClaimTemplates`). Use the redis image.

**Expected Result:**
```bash
kubectl get statefulsets
NAME    READY   AGE
redis   3/3     60s

kubectl get pods -l app=redis
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          60s
redis-1   1/1     Running   0          45s
redis-2   1/1     Running   0          30s

kubectl get pvc
NAME                STATUS   VOLUME   CAPACITY   ACCESS MODES
redis-data-redis-0  Bound    ...      1Gi        RWO
redis-data-redis-1  Bound    ...      1Gi        RWO
redis-data-redis-2  Bound    ...      1Gi        RWO
```

**Verification:**
```bash
kubectl get svc redis-headless
# Should show ClusterIP: None (headless)
kubectl get pvc -l app=redis
# Should show 3 PVCs, one per pod
```

> [!hint]- Hint
> - A headless Service has `clusterIP: None` - it enables DNS records for each pod
> - StatefulSet requires `serviceName` field matching the headless Service name
> - `volumeClaimTemplates` creates a unique PVC for each replica
> - Pod names are `<statefulset>-0`, `<statefulset>-1`, etc. (deterministic, ordered)
> - DNS names: `<pod-name>.<service-name>.<namespace>.svc.cluster.local`

> [!success]- Solution
> **redis-statefulset.yaml:**
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: redis-headless
>   labels:
>     app: redis
> spec:
>   clusterIP: None
>   selector:
>     app: redis
>   ports:
>     - port: 6379
>       targetPort: 6379
>       name: redis
> ---
> apiVersion: apps/v1
> kind: StatefulSet
> metadata:
>   name: redis
> spec:
>   serviceName: redis-headless
>   replicas: 3
>   selector:
>     matchLabels:
>       app: redis
>   template:
>     metadata:
>       labels:
>         app: redis
>     spec:
>       containers:
>         - name: redis
>           image: redis:7
>           ports:
>             - containerPort: 6379
>           volumeMounts:
>             - name: redis-data
>               mountPath: /data
>           resources:
>             requests:
>               cpu: 100m
>               memory: 128Mi
>             limits:
>               cpu: 250m
>               memory: 256Mi
>   volumeClaimTemplates:
>     - metadata:
>         name: redis-data
>       spec:
>         accessModes:
>           - ReadWriteOnce
>         resources:
>           requests:
>             storage: 1Gi
>         storageClassName: standard
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f redis-statefulset.yaml
> kubectl rollout status statefulset/redis --timeout=120s
>
> kubectl get statefulsets
> kubectl get pods -l app=redis
> kubectl get pvc -l app=redis
> kubectl get svc redis-headless
>
> # Verify each pod has its own storage:
> kubectl exec redis-0 -- redis-cli SET mykey "from-redis-0"
> kubectl exec redis-0 -- redis-cli GET mykey
> # Each pod has independent data
> ```

---

## P6: StatefulSet Ordering and Stable DNS

**Task:** Using the StatefulSet from P5, verify that pods are created in order (0, 1, 2) and deleted in reverse order (2, 1, 0). Test the stable DNS names by resolving pod hostnames from within the cluster.

**Expected Result:**
```bash
# DNS resolution from inside a pod:
kubectl exec redis-0 -- sh -c "hostname"
# redis-0

# Lookup DNS for each pod:
# redis-0.redis-headless.default.svc.cluster.local
# redis-1.redis-headless.default.svc.cluster.local
# redis-2.redis-headless.default.svc.cluster.local
```

**Verification:**
```bash
# Scale down and watch reverse order:
kubectl scale statefulset redis --replicas=0
# redis-2 deleted first, then redis-1, then redis-0

# Scale back up and watch forward order:
kubectl scale statefulset redis --replicas=3
# redis-0 created first, then redis-1, then redis-2
```

> [!hint]- Hint
> - StatefulSet creates pods sequentially: pod N+1 is not created until pod N is Ready
> - On deletion, pods are removed in reverse order
> - DNS format: `<pod-name>.<headless-svc>.<namespace>.svc.cluster.local`
> - Use a temporary busybox pod with `nslookup` to test DNS
> - `kubectl get pods --watch` shows the ordering in real-time

> [!success]- Solution
> **Observe ordering behavior:**
> ```bash
> # Scale down and watch (in separate terminal):
> kubectl get pods -l app=redis --watch &
>
> kubectl scale statefulset redis --replicas=0
> # Watch output: redis-2 terminates first, then redis-1, then redis-0
>
> # Wait for all to terminate
> kubectl wait --for=delete pod/redis-0 --timeout=120s 2>/dev/null
>
> # Scale back up:
> kubectl scale statefulset redis --replicas=3
> # Watch output: redis-0 created first, then redis-1 (after redis-0 is Ready), then redis-2
>
> kubectl rollout status statefulset/redis --timeout=120s
> ```
>
> **Test stable DNS names:**
> ```bash
> # Launch a temporary DNS test pod:
> kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- sh -c '
>   echo "=== Hostname lookups ==="
>   for i in 0 1 2; do
>     echo "--- redis-$i ---"
>     nslookup redis-$i.redis-headless.default.svc.cluster.local 2>/dev/null
>     echo
>   done
>   echo "=== Headless service lookup ==="
>   nslookup redis-headless.default.svc.cluster.local 2>/dev/null
> '
> ```
>
> **Verify pod identity is stable across restarts:**
> ```bash
> # Write data to redis-0
> kubectl exec redis-0 -- redis-cli SET identity "I am redis-0"
>
> # Delete redis-0 (StatefulSet will recreate it with the same name and PVC)
> kubectl delete pod redis-0
> kubectl wait --for=condition=Ready pod/redis-0 --timeout=60s
>
> # Data persists because the same PVC is reattached
> kubectl exec redis-0 -- redis-cli GET identity
> # Output: I am redis-0
> ```

---

## P7: Horizontal Pod Autoscaler (HPA)

**Task:** Create a Deployment running a CPU-intensive PHP application. Create an HPA targeting 50% CPU utilization, with min 1 and max 5 replicas. Generate load and watch the HPA scale up pods. Remove load and watch it scale back down.

**Expected Result:**
```bash
kubectl get hpa
NAME       REFERENCE           TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-hpa    Deployment/php-app  73%/50%   1         5         3          2m

kubectl get pods -l app=php-app
# Should show multiple pods during load, scaling back to 1 after load stops
```

**Verification:**
```bash
kubectl get hpa php-hpa --watch
# Watch TARGETS column change from <unknown> -> high% -> scaling -> stabilize
```

> [!hint]- Hint
> - metrics-server must be running: `minikube addons enable metrics-server`
> - The Deployment must have `resources.requests.cpu` set (HPA needs this as reference)
> - Use `kubectl autoscale` or write HPA YAML
> - Generate load with: `kubectl run load-gen --image=busybox -- /bin/sh -c "while true; do wget -q -O- http://php-app; done"`
> - HPA checks metrics every 15 seconds by default, scale-down stabilization is 5 minutes

> [!success]- Solution
> **php-app-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: php-app
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: php-app
>   template:
>     metadata:
>       labels:
>         app: php-app
>     spec:
>       containers:
>         - name: php
>           image: registry.k8s.io/hpa-example
>           ports:
>             - containerPort: 80
>           resources:
>             requests:
>               cpu: 200m
>             limits:
>               cpu: 500m
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: php-app
> spec:
>   selector:
>     app: php-app
>   ports:
>     - port: 80
>       targetPort: 80
> ```
>
> **Create the HPA:**
> ```bash
> kubectl apply -f php-app-deployment.yaml
> kubectl wait --for=condition=Available deployment/php-app --timeout=120s
>
> # Create HPA via command:
> kubectl autoscale deployment php-app \
>   --cpu-percent=50 \
>   --min=1 \
>   --max=5
> ```
>
> **Or via YAML (php-hpa.yaml):**
> ```yaml
> apiVersion: autoscaling/v2
> kind: HorizontalPodAutoscaler
> metadata:
>   name: php-hpa
> spec:
>   scaleTargetRef:
>     apiVersion: apps/v1
>     kind: Deployment
>     name: php-app
>   minReplicas: 1
>   maxReplicas: 5
>   metrics:
>     - type: Resource
>       resource:
>         name: cpu
>         target:
>           type: Utilization
>           averageUtilization: 50
> ```
>
> **Generate load (run in a separate terminal):**
> ```bash
> kubectl run load-generator --image=busybox:1.36 --restart=Never -- \
>   /bin/sh -c "while true; do wget -q -O- http://php-app > /dev/null; done"
> ```
>
> **Watch the HPA in action:**
> ```bash
> # Watch HPA (in another terminal):
> kubectl get hpa php-hpa --watch
>
> # Watch pods scaling:
> kubectl get pods -l app=php-app --watch
>
> # After seeing scale-up (may take 1-2 minutes), stop the load:
> kubectl delete pod load-generator
>
> # Watch scale-down (takes ~5 minutes due to stabilization window):
> kubectl get hpa php-hpa --watch
> ```

---

## P8: Taints, Tolerations, and Node Affinity

**Task:** Taint the minikube node with `environment=production:NoSchedule`. Verify that a regular pod cannot be scheduled. Then create a pod with the matching toleration and verify it can be scheduled. Finally, remove the taint.

**Expected Result:**
```bash
# Regular pod stays Pending:
kubectl get pods no-toleration-pod
NAME                 READY   STATUS    RESTARTS   AGE
no-toleration-pod    0/1     Pending   0          30s

# Pod with toleration runs:
kubectl get pods toleration-pod
NAME              READY   STATUS    RESTARTS   AGE
toleration-pod    1/1     Running   0          15s
```

**Verification:**
```bash
kubectl describe pod no-toleration-pod | grep -A 5 "Events:"
# Should show: 0/1 nodes are available: 1 node(s) had untolerated taint
kubectl describe pod toleration-pod | grep "Node:"
# Should show the node name
```

> [!hint]- Hint
> - `kubectl taint nodes <node> key=value:effect` adds a taint
> - Effects: `NoSchedule`, `PreferNoSchedule`, `NoExecute`
> - Tolerations in pod spec must match the taint's key, value, and effect
> - Use `operator: Equal` for exact match or `operator: Exists` for key-only match
> - Remove taint with trailing minus: `kubectl taint nodes <node> key=value:effect-`
> - With minikube (single node), ALL pods will be affected by the taint

> [!success]- Solution
> **First, suspend existing workloads that might interfere:**
> ```bash
> # Get node name
> NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
> echo "Node: $NODE"
>
> # Apply taint
> kubectl taint nodes $NODE environment=production:NoSchedule
> kubectl describe node $NODE | grep Taints
> ```
>
> **no-toleration-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: no-toleration-pod
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>   restartPolicy: Never
> ```
>
> **toleration-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: toleration-pod
> spec:
>   tolerations:
>     - key: "environment"
>       operator: "Equal"
>       value: "production"
>       effect: "NoSchedule"
>   containers:
>     - name: nginx
>       image: nginx:1.27
>   restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f no-toleration-pod.yaml
> kubectl apply -f toleration-pod.yaml
>
> # Check status:
> kubectl get pods no-toleration-pod toleration-pod
>
> # The regular pod should be Pending:
> kubectl describe pod no-toleration-pod | grep -A 3 "Events:"
> # Warning  FailedScheduling: 0/1 nodes are available: 1 node(s) had untolerated taint
>
> # The toleration pod should be Running:
> kubectl describe pod toleration-pod | grep "Node:"
>
> # Clean up: remove the taint
> kubectl taint nodes $NODE environment=production:NoSchedule-
> kubectl describe node $NODE | grep Taints
>
> # Now the pending pod should get scheduled:
> kubectl get pods no-toleration-pod --watch
> # Status changes from Pending to Running
>
> # Clean up pods
> kubectl delete pod no-toleration-pod toleration-pod
> ```

---

## P9: Pod Priority and Preemption

**Task:** Create two PriorityClasses: `low-priority` (value 100) and `high-priority` (value 1000). Deploy a low-priority pod that consumes most of the node's resources. Then deploy a high-priority pod and observe preemption - the low-priority pod should be evicted to make room.

**Expected Result:**
```bash
kubectl get priorityclasses
NAME             VALUE      GLOBAL-DEFAULT   AGE
high-priority    1000       false            30s
low-priority     100        false            30s

# After deploying high-priority pod:
kubectl get pods
NAME              READY   STATUS    RESTARTS   AGE
high-priority-pod 1/1     Running   0          15s
# low-priority-pod may be evicted/Pending
```

**Verification:**
```bash
kubectl get events --sort-by=.lastTimestamp | grep -i preempt
# Should show preemption events
kubectl describe pod high-priority-pod | grep "Priority"
# Should show: Priority: 1000
```

> [!hint]- Hint
> - PriorityClass is cluster-scoped, `value` is an integer (higher = more important)
> - Set `preemptionPolicy: PreemptLowerPriority` (default)
> - Preemption only happens when the high-priority pod cannot be scheduled due to resource shortage
> - The low-priority pod needs to consume significant resources to trigger preemption
> - On minikube, node resources are limited, making preemption easier to trigger
> - Set `globalDefault: false` to avoid affecting other pods

> [!success]- Solution
> **priority-classes.yaml:**
> ```yaml
> apiVersion: scheduling.k8s.io/v1
> kind: PriorityClass
> metadata:
>   name: low-priority
> value: 100
> globalDefault: false
> preemptionPolicy: PreemptLowerPriority
> description: "Low priority workloads"
> ---
> apiVersion: scheduling.k8s.io/v1
> kind: PriorityClass
> metadata:
>   name: high-priority
> value: 1000
> globalDefault: false
> preemptionPolicy: PreemptLowerPriority
> description: "High priority workloads"
> ```
>
> **low-priority-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: low-priority-pod
> spec:
>   priorityClassName: low-priority
>   containers:
>     - name: resource-hog
>       image: busybox:1.36
>       command: ["sh", "-c", "echo 'Low priority running'; sleep 3600"]
>       resources:
>         requests:
>           cpu: "1"
>           memory: "1Gi"
>         limits:
>           cpu: "1"
>           memory: "1Gi"
>   restartPolicy: Never
> ```
>
> **high-priority-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: high-priority-pod
> spec:
>   priorityClassName: high-priority
>   containers:
>     - name: important-app
>       image: nginx:1.27
>       resources:
>         requests:
>           cpu: "1"
>           memory: "1Gi"
>         limits:
>           cpu: "1"
>           memory: "1Gi"
>   restartPolicy: Never
> ```
>
> **Apply and observe preemption:**
> ```bash
> kubectl apply -f priority-classes.yaml
> kubectl get priorityclasses
>
> # Deploy the low-priority pod first (consumes resources):
> kubectl apply -f low-priority-pod.yaml
> kubectl wait --for=condition=Ready pod/low-priority-pod --timeout=60s
> kubectl get pods
>
> # Deploy the high-priority pod (may trigger preemption):
> kubectl apply -f high-priority-pod.yaml
>
> # Watch what happens:
> kubectl get pods --watch
> kubectl get events --sort-by=.lastTimestamp --watch
>
> # Check priority values:
> kubectl describe pod high-priority-pod | grep -E "Priority|Node"
> kubectl describe pod low-priority-pod | grep -E "Priority|Status"
>
> # Note: Preemption depends on actual resource availability on your minikube.
> # If your node has enough resources for both, you may need to increase
> # resource requests or add more low-priority pods to trigger preemption.
> ```

---

## P10: RBAC - ServiceAccount, Role, and RoleBinding

**Task:** Create a ServiceAccount named `pod-reader-sa` in the `default` namespace. Create a Role named `pod-reader` that allows `get`, `list`, and `watch` on pods. Bind the Role to the ServiceAccount. Verify permissions using `kubectl auth can-i`.

**Expected Result:**
```bash
kubectl auth can-i get pods --as=system:serviceaccount:default:pod-reader-sa
# yes

kubectl auth can-i create pods --as=system:serviceaccount:default:pod-reader-sa
# no

kubectl auth can-i delete pods --as=system:serviceaccount:default:pod-reader-sa
# no

kubectl auth can-i get deployments --as=system:serviceaccount:default:pod-reader-sa
# no
```

**Verification:**
```bash
kubectl auth can-i --list --as=system:serviceaccount:default:pod-reader-sa | grep pods
# Should show: pods [] [get list watch]
```

> [!hint]- Hint
> - ServiceAccount is namespace-scoped, used for pod identity
> - Role defines a set of permissions (verbs on resources) within a namespace
> - RoleBinding grants a Role to a user, group, or ServiceAccount
> - `kubectl auth can-i` tests permissions impersonation
> - The `--as` flag format for ServiceAccounts: `system:serviceaccount:<namespace>:<name>`

> [!success]- Solution
> **rbac-pod-reader.yaml:**
> ```yaml
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: pod-reader-sa
>   namespace: default
> ---
> apiVersion: rbac.authorization.k8s.io/v1
> kind: Role
> metadata:
>   name: pod-reader
>   namespace: default
> rules:
>   - apiGroups: [""]
>     resources: ["pods"]
>     verbs: ["get", "list", "watch"]
> ---
> apiVersion: rbac.authorization.k8s.io/v1
> kind: RoleBinding
> metadata:
>   name: pod-reader-binding
>   namespace: default
> roleRef:
>   apiGroup: rbac.authorization.k8s.io
>   kind: Role
>   name: pod-reader
> subjects:
>   - kind: ServiceAccount
>     name: pod-reader-sa
>     namespace: default
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f rbac-pod-reader.yaml
>
> # Verify the resources were created:
> kubectl get serviceaccount pod-reader-sa
> kubectl get role pod-reader
> kubectl get rolebinding pod-reader-binding
>
> # Test permissions:
> kubectl auth can-i get pods --as=system:serviceaccount:default:pod-reader-sa
> # yes
>
> kubectl auth can-i list pods --as=system:serviceaccount:default:pod-reader-sa
> # yes
>
> kubectl auth can-i create pods --as=system:serviceaccount:default:pod-reader-sa
> # no
>
> kubectl auth can-i delete pods --as=system:serviceaccount:default:pod-reader-sa
> # no
>
> kubectl auth can-i get deployments --as=system:serviceaccount:default:pod-reader-sa
> # no
>
> kubectl auth can-i get pods -n kube-system --as=system:serviceaccount:default:pod-reader-sa
> # no (Role is namespace-scoped to default)
>
> # List all permissions:
> kubectl auth can-i --list --as=system:serviceaccount:default:pod-reader-sa
> ```
>
> **Deploy a pod using the ServiceAccount:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: pod-reader-test
> spec:
>   serviceAccountName: pod-reader-sa
>   containers:
>     - name: kubectl
>       image: bitnami/kubectl:latest
>       command: ["sh", "-c", "kubectl get pods; echo '---'; kubectl get deployments 2>&1; sleep 3600"]
>   restartPolicy: Never
> ```
>
> ```bash
> kubectl apply -f pod-reader-test.yaml
> kubectl wait --for=condition=Ready pod/pod-reader-test --timeout=60s
> kubectl logs pod-reader-test
> # Should show pods list but fail on deployments
> ```

---

## P11: ClusterRole and ClusterRoleBinding - Cluster-Wide Access

**Task:** Create a ServiceAccount named `secret-reader-sa` in the `default` namespace. Create a ClusterRole named `secret-reader` that allows `get` and `list` on secrets cluster-wide. Bind it with a ClusterRoleBinding. Verify it can read secrets in ANY namespace.

**Expected Result:**
```bash
kubectl auth can-i get secrets --as=system:serviceaccount:default:secret-reader-sa
# yes

kubectl auth can-i get secrets -n kube-system --as=system:serviceaccount:default:secret-reader-sa
# yes (cluster-wide!)

kubectl auth can-i create secrets --as=system:serviceaccount:default:secret-reader-sa
# no

kubectl auth can-i get pods --as=system:serviceaccount:default:secret-reader-sa
# no
```

**Verification:**
```bash
kubectl auth can-i --list --as=system:serviceaccount:default:secret-reader-sa --namespace=kube-system | grep secrets
# Should show: secrets [] [get list]
```

> [!hint]- Hint
> - `ClusterRole` is cluster-scoped (not namespace-scoped like Role)
> - `ClusterRoleBinding` grants cluster-wide access
> - Difference: Role+RoleBinding = single namespace, ClusterRole+ClusterRoleBinding = all namespaces
> - You can also use a ClusterRole with a RoleBinding to grant access in only one namespace (useful for reusable roles)
> - Secrets are sensitive - in production, limit who gets `get`/`list` on secrets

> [!success]- Solution
> **clusterrole-secret-reader.yaml:**
> ```yaml
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: secret-reader-sa
>   namespace: default
> ---
> apiVersion: rbac.authorization.k8s.io/v1
> kind: ClusterRole
> metadata:
>   name: secret-reader
> rules:
>   - apiGroups: [""]
>     resources: ["secrets"]
>     verbs: ["get", "list"]
> ---
> apiVersion: rbac.authorization.k8s.io/v1
> kind: ClusterRoleBinding
> metadata:
>   name: secret-reader-binding
> roleRef:
>   apiGroup: rbac.authorization.k8s.io
>   kind: ClusterRole
>   name: secret-reader
> subjects:
>   - kind: ServiceAccount
>     name: secret-reader-sa
>     namespace: default
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f clusterrole-secret-reader.yaml
>
> # Verify resources:
> kubectl get serviceaccount secret-reader-sa
> kubectl get clusterrole secret-reader
> kubectl get clusterrolebinding secret-reader-binding
>
> # Test: can read secrets in default namespace
> kubectl auth can-i get secrets --as=system:serviceaccount:default:secret-reader-sa
> # yes
>
> # Test: can read secrets in kube-system (cluster-wide!)
> kubectl auth can-i get secrets -n kube-system --as=system:serviceaccount:default:secret-reader-sa
> # yes
>
> # Test: can read secrets in ANY namespace
> kubectl auth can-i get secrets --all-namespaces --as=system:serviceaccount:default:secret-reader-sa
> # yes
>
> # Test: CANNOT do other things
> kubectl auth can-i create secrets --as=system:serviceaccount:default:secret-reader-sa
> # no
> kubectl auth can-i get pods --as=system:serviceaccount:default:secret-reader-sa
> # no
> kubectl auth can-i delete secrets --as=system:serviceaccount:default:secret-reader-sa
> # no
>
> # Compare: Role (P10) is namespace-scoped vs ClusterRole (this) is cluster-scoped
> kubectl auth can-i get pods -n kube-system --as=system:serviceaccount:default:pod-reader-sa
> # no (namespace-scoped Role from P10 only works in default)
> kubectl auth can-i get secrets -n kube-system --as=system:serviceaccount:default:secret-reader-sa
> # yes (cluster-scoped ClusterRole works everywhere)
> ```

---

## P12: Pod Security Standards - Restricted Namespace Policy

**Task:** Apply the `restricted` Pod Security Standard to a namespace called `secure-ns`. Try to deploy a privileged pod (should be rejected). Then deploy a compliant pod that follows restricted standards (non-root, read-only root filesystem, drop all capabilities).

**Expected Result:**
```bash
# Privileged pod is rejected:
Error from server (Forbidden): pods "privileged-pod" is forbidden:
violates PodSecurity "restricted:latest"

# Compliant pod succeeds:
kubectl get pods -n secure-ns
NAME             READY   STATUS    RESTARTS   AGE
compliant-pod    1/1     Running   0          15s
```

**Verification:**
```bash
kubectl get namespace secure-ns -o yaml | grep pod-security
# Should show the pod-security labels
kubectl describe pod compliant-pod -n secure-ns | grep -E "securityContext|runAsNonRoot"
```

> [!hint]- Hint
> - Pod Security Standards are enforced via namespace labels (since K8s 1.25)
> - Three levels: `privileged` (unrestricted), `baseline` (common restrictions), `restricted` (hardened)
> - Three modes: `enforce` (reject), `audit` (log), `warn` (warning)
> - Label format: `pod-security.kubernetes.io/<mode>=<level>`
> - Restricted requires: `runAsNonRoot: true`, `seccompProfile`, drop `ALL` capabilities, no privilege escalation
> - Use `warn` mode first to test, then switch to `enforce`

> [!success]- Solution
> **Create and label the namespace:**
> ```bash
> kubectl create namespace secure-ns
>
> # Apply restricted Pod Security Standard in enforce mode:
> kubectl label namespace secure-ns \
>   pod-security.kubernetes.io/enforce=restricted \
>   pod-security.kubernetes.io/enforce-version=latest \
>   pod-security.kubernetes.io/warn=restricted \
>   pod-security.kubernetes.io/warn-version=latest
>
> kubectl get namespace secure-ns --show-labels
> ```
>
> **Try a privileged pod (should FAIL):**
> ```bash
> cat <<'EOF' | kubectl apply -f -
> apiVersion: v1
> kind: Pod
> metadata:
>   name: privileged-pod
>   namespace: secure-ns
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
>       securityContext:
>         privileged: true
> EOF
> # Error from server (Forbidden): violates PodSecurity "restricted:latest"
> ```
>
> **Try a non-compliant pod (no security context - should also FAIL):**
> ```bash
> cat <<'EOF' | kubectl apply -f -
> apiVersion: v1
> kind: Pod
> metadata:
>   name: noncompliant-pod
>   namespace: secure-ns
> spec:
>   containers:
>     - name: nginx
>       image: nginx:1.27
> EOF
> # Also fails - restricted requires explicit security settings
> ```
>
> **Deploy a compliant pod:**
> ```yaml
> # compliant-pod.yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: compliant-pod
>   namespace: secure-ns
> spec:
>   securityContext:
>     runAsNonRoot: true
>     runAsUser: 1000
>     seccompProfile:
>       type: RuntimeDefault
>   containers:
>     - name: app
>       image: busybox:1.36
>       command: ["sh", "-c", "echo 'Running as non-root in restricted namespace'; id; sleep 3600"]
>       securityContext:
>         allowPrivilegeEscalation: false
>         capabilities:
>           drop:
>             - ALL
>         readOnlyRootFilesystem: true
>       resources:
>         requests:
>           cpu: 50m
>           memory: 32Mi
>         limits:
>           cpu: 100m
>           memory: 64Mi
>   restartPolicy: Never
> ```
>
> **Apply and verify:**
> ```bash
> kubectl apply -f compliant-pod.yaml
> kubectl wait --for=condition=Ready pod/compliant-pod -n secure-ns --timeout=60s
>
> kubectl get pods -n secure-ns
> kubectl logs compliant-pod -n secure-ns
> # Should show: uid=1000 (non-root)
>
> # Verify security context:
> kubectl describe pod compliant-pod -n secure-ns | grep -A 10 "Security"
> ```

---

## P13: Production-Ready Stateful Application (Capstone)

**Task:** Deploy a production-ready Redis cluster combining everything learned: StatefulSet (3 replicas) + headless Service + PVCs (via volumeClaimTemplates) + ConfigMap (custom redis.conf) + HPA + RBAC ServiceAccount. The application should be deployed in its own namespace with proper resource management.

**Expected Result:**
```bash
kubectl get all -n redis-prod
# Should show:
# - 3 StatefulSet pods (redis-cluster-0, 1, 2)
# - Headless Service (redis-cluster-headless)
# - Client Service (redis-cluster)
# - HPA targeting the StatefulSet
# - ServiceAccount, Role, RoleBinding

kubectl get pvc -n redis-prod
# 3 PVCs bound to dynamic PVs

kubectl get configmap -n redis-prod
# redis-config ConfigMap

kubectl get hpa -n redis-prod
# HPA targeting redis-cluster StatefulSet
```

**Verification:**
```bash
# Verify Redis is running with custom config:
kubectl exec redis-cluster-0 -n redis-prod -- redis-cli CONFIG GET maxmemory
# Should show custom value from ConfigMap

# Verify RBAC:
kubectl auth can-i get pods -n redis-prod \
  --as=system:serviceaccount:redis-prod:redis-sa
# yes

# Verify stable DNS:
kubectl run dns-check -n redis-prod --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup redis-cluster-0.redis-cluster-headless.redis-prod.svc.cluster.local

# Verify data persistence:
kubectl exec redis-cluster-0 -n redis-prod -- redis-cli SET capstone "complete"
kubectl delete pod redis-cluster-0 -n redis-prod
kubectl wait --for=condition=Ready pod/redis-cluster-0 -n redis-prod --timeout=120s
kubectl exec redis-cluster-0 -n redis-prod -- redis-cli GET capstone
# Output: complete
```

> [!hint]- Hint
> - Start by creating the namespace, then RBAC, then ConfigMap, then Services, then StatefulSet, then HPA
> - Redis config can set `maxmemory`, `maxmemory-policy`, `appendonly`, `save` directives
> - Mount the ConfigMap at `/etc/redis/` and use `redis-server /etc/redis/redis.conf` as the command
> - HPA for StatefulSets works the same as Deployments
> - Remember that HPA needs metrics-server and resource requests defined

> [!success]- Solution
> **redis-prod-complete.yaml:**
> ```yaml
> # 1. Namespace
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: redis-prod
> ---
> # 2. ServiceAccount
> apiVersion: v1
> kind: ServiceAccount
> metadata:
>   name: redis-sa
>   namespace: redis-prod
> ---
> # 3. Role - manage redis pods and services
> apiVersion: rbac.authorization.k8s.io/v1
> kind: Role
> metadata:
>   name: redis-role
>   namespace: redis-prod
> rules:
>   - apiGroups: [""]
>     resources: ["pods", "services", "endpoints"]
>     verbs: ["get", "list", "watch"]
>   - apiGroups: [""]
>     resources: ["configmaps"]
>     verbs: ["get"]
> ---
> # 4. RoleBinding
> apiVersion: rbac.authorization.k8s.io/v1
> kind: RoleBinding
> metadata:
>   name: redis-rolebinding
>   namespace: redis-prod
> roleRef:
>   apiGroup: rbac.authorization.k8s.io
>   kind: Role
>   name: redis-role
> subjects:
>   - kind: ServiceAccount
>     name: redis-sa
>     namespace: redis-prod
> ---
> # 5. ConfigMap - custom redis.conf
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: redis-config
>   namespace: redis-prod
> data:
>   redis.conf: |
>     # Memory
>     maxmemory 128mb
>     maxmemory-policy allkeys-lru
>
>     # Persistence
>     appendonly yes
>     appendfsync everysec
>     save 900 1
>     save 300 10
>     save 60 10000
>
>     # Networking
>     bind 0.0.0.0
>     protected-mode no
>     tcp-keepalive 300
>
>     # Logging
>     loglevel notice
> ---
> # 6. Headless Service (for StatefulSet DNS)
> apiVersion: v1
> kind: Service
> metadata:
>   name: redis-cluster-headless
>   namespace: redis-prod
>   labels:
>     app: redis-cluster
> spec:
>   clusterIP: None
>   selector:
>     app: redis-cluster
>   ports:
>     - port: 6379
>       targetPort: 6379
>       name: redis
> ---
> # 7. Client Service (for application access)
> apiVersion: v1
> kind: Service
> metadata:
>   name: redis-cluster
>   namespace: redis-prod
>   labels:
>     app: redis-cluster
> spec:
>   selector:
>     app: redis-cluster
>   ports:
>     - port: 6379
>       targetPort: 6379
>       name: redis
> ---
> # 8. StatefulSet
> apiVersion: apps/v1
> kind: StatefulSet
> metadata:
>   name: redis-cluster
>   namespace: redis-prod
> spec:
>   serviceName: redis-cluster-headless
>   replicas: 3
>   selector:
>     matchLabels:
>       app: redis-cluster
>   template:
>     metadata:
>       labels:
>         app: redis-cluster
>     spec:
>       serviceAccountName: redis-sa
>       containers:
>         - name: redis
>           image: redis:7
>           command: ["redis-server", "/etc/redis/redis.conf"]
>           ports:
>             - containerPort: 6379
>               name: redis
>           resources:
>             requests:
>               cpu: 100m
>               memory: 128Mi
>             limits:
>               cpu: 250m
>               memory: 256Mi
>           volumeMounts:
>             - name: redis-data
>               mountPath: /data
>             - name: redis-config
>               mountPath: /etc/redis
>           livenessProbe:
>             exec:
>               command: ["redis-cli", "ping"]
>             initialDelaySeconds: 15
>             periodSeconds: 10
>           readinessProbe:
>             exec:
>               command: ["redis-cli", "ping"]
>             initialDelaySeconds: 5
>             periodSeconds: 5
>       volumes:
>         - name: redis-config
>           configMap:
>             name: redis-config
>   volumeClaimTemplates:
>     - metadata:
>         name: redis-data
>       spec:
>         accessModes:
>           - ReadWriteOnce
>         resources:
>           requests:
>             storage: 1Gi
>         storageClassName: standard
> ---
> # 9. HPA
> apiVersion: autoscaling/v2
> kind: HorizontalPodAutoscaler
> metadata:
>   name: redis-cluster-hpa
>   namespace: redis-prod
> spec:
>   scaleTargetRef:
>     apiVersion: apps/v1
>     kind: StatefulSet
>     name: redis-cluster
>   minReplicas: 3
>   maxReplicas: 6
>   metrics:
>     - type: Resource
>       resource:
>         name: cpu
>         target:
>           type: Utilization
>           averageUtilization: 70
>     - type: Resource
>       resource:
>         name: memory
>         target:
>           type: Utilization
>           averageUtilization: 80
> ```
>
> **Apply everything:**
> ```bash
> kubectl apply -f redis-prod-complete.yaml
> kubectl rollout status statefulset/redis-cluster -n redis-prod --timeout=180s
> ```
>
> **Verify all components:**
> ```bash
> echo "=== Namespace ==="
> kubectl get namespace redis-prod
>
> echo "=== RBAC ==="
> kubectl get serviceaccount,role,rolebinding -n redis-prod
> kubectl auth can-i get pods -n redis-prod \
>   --as=system:serviceaccount:redis-prod:redis-sa
> kubectl auth can-i delete pods -n redis-prod \
>   --as=system:serviceaccount:redis-prod:redis-sa
>
> echo "=== ConfigMap ==="
> kubectl get configmap redis-config -n redis-prod
>
> echo "=== Services ==="
> kubectl get svc -n redis-prod
>
> echo "=== StatefulSet ==="
> kubectl get statefulset redis-cluster -n redis-prod
> kubectl get pods -n redis-prod -l app=redis-cluster
>
> echo "=== PVCs ==="
> kubectl get pvc -n redis-prod
>
> echo "=== HPA ==="
> kubectl get hpa -n redis-prod
>
> echo "=== Redis Config Check ==="
> kubectl exec redis-cluster-0 -n redis-prod -- redis-cli CONFIG GET maxmemory
> kubectl exec redis-cluster-0 -n redis-prod -- redis-cli CONFIG GET appendonly
>
> echo "=== DNS Check ==="
> kubectl run dns-check -n redis-prod --image=busybox:1.36 --rm -it --restart=Never -- \
>   nslookup redis-cluster-0.redis-cluster-headless.redis-prod.svc.cluster.local
>
> echo "=== Data Persistence Test ==="
> kubectl exec redis-cluster-0 -n redis-prod -- redis-cli SET capstone "complete"
> kubectl exec redis-cluster-0 -n redis-prod -- redis-cli GET capstone
> kubectl delete pod redis-cluster-0 -n redis-prod
> kubectl wait --for=condition=Ready pod/redis-cluster-0 -n redis-prod --timeout=120s
> kubectl exec redis-cluster-0 -n redis-prod -- redis-cli GET capstone
> # Should still return: complete
> ```

---

## Cleanup

```bash
# Remove all resources created in these exercises
kubectl delete namespace redis-prod secure-ns --ignore-not-found

kubectl delete job fibonacci-job parallel-job timestamp-manual --ignore-not-found
kubectl delete cronjob timestamp-cron --ignore-not-found
kubectl delete daemonset log-collector --ignore-not-found
kubectl delete statefulset redis --ignore-not-found
kubectl delete svc redis-headless php-app --ignore-not-found
kubectl delete deployment php-app --ignore-not-found
kubectl delete hpa php-hpa --ignore-not-found
kubectl delete pvc -l app=redis --ignore-not-found

kubectl delete pod load-generator pod-reader-test --ignore-not-found
kubectl delete serviceaccount pod-reader-sa secret-reader-sa --ignore-not-found
kubectl delete role pod-reader --ignore-not-found
kubectl delete rolebinding pod-reader-binding --ignore-not-found
kubectl delete clusterrole secret-reader --ignore-not-found
kubectl delete clusterrolebinding secret-reader-binding --ignore-not-found
kubectl delete priorityclass low-priority high-priority --ignore-not-found
```
