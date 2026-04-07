---
title: "K8s Workflow: Advanced Deployment Strategies"
date: 2026-04-07
tags:
  - kubernetes
  - workflow
  - project
  - deployment-strategies
  - blue-green
  - canary
  - hpa
  - monitoring
  - prometheus
  - grafana
parent: "[[Kubernetes Study]]"
---

# Workflow 05: Advanced Deployment Strategies

Implement blue/green, canary, rolling update tuning, autoscaling, pod disruption budgets, and monitoring on minikube. This workflow covers the deployment strategies you will encounter in production Kubernetes environments.

## Architecture Overview

```
               Deployment Strategies Comparison
  ┌─────────────────────────────────────────────────────────┐
  │                                                         │
  │  Blue/Green:    100% ──────► 0%/100% (instant switch)   │
  │                 blue          green                      │
  │                                                         │
  │  Canary:        100% ──► 90%/10% ──► 0%/100% (gradual) │
  │                  v1      v1/v2          v2               │
  │                                                         │
  │  Rolling:       ████████ ──► ██████▓▓ ──► ████████     │
  │                  v1         v1+v2          v2            │
  │                         (overlap period)                 │
  └─────────────────────────────────────────────────────────┘
```

## Prerequisites

- minikube running with sufficient resources
- kubectl configured
- Helm 3 installed

```bash
minikube start --cpus=4 --memory=6144
minikube addons enable ingress
minikube addons enable metrics-server
```

---

## Sample Application

A simple Go application that reports its version. This is used across all deployment strategies.

> [!success]- Full Solution: `app/main.go`
> ```go
> package main
> 
> import (
> 	"encoding/json"
> 	"fmt"
> 	"log"
> 	"math"
> 	"net/http"
> 	"os"
> 	"time"
> )
> 
> var (
> 	version   = getEnv("APP_VERSION", "v1")
> 	color     = getEnv("APP_COLOR", "blue")
> 	startTime = time.Now()
> )
> 
> type Info struct {
> 	Version  string `json:"version"`
> 	Color    string `json:"color"`
> 	Hostname string `json:"hostname"`
> 	Uptime   string `json:"uptime"`
> }
> 
> func main() {
> 	port := getEnv("PORT", "8080")
> 
> 	http.HandleFunc("/", infoHandler)
> 	http.HandleFunc("/health", healthHandler)
> 	http.HandleFunc("/ready", readyHandler)
> 	http.HandleFunc("/stress", stressHandler)  // For HPA testing
> 
> 	log.Printf("App %s (%s) starting on port %s", version, color, port)
> 	log.Fatal(http.ListenAndServe(":"+port, nil))
> }
> 
> func infoHandler(w http.ResponseWriter, r *http.Request) {
> 	hostname, _ := os.Hostname()
> 	w.Header().Set("Content-Type", "application/json")
> 	json.NewEncoder(w).Encode(Info{
> 		Version:  version,
> 		Color:    color,
> 		Hostname: hostname,
> 		Uptime:   time.Since(startTime).Round(time.Second).String(),
> 	})
> }
> 
> func healthHandler(w http.ResponseWriter, r *http.Request) {
> 	w.WriteHeader(http.StatusOK)
> 	w.Write([]byte("ok"))
> }
> 
> func readyHandler(w http.ResponseWriter, r *http.Request) {
> 	w.WriteHeader(http.StatusOK)
> 	w.Write([]byte("ready"))
> }
> 
> // stressHandler generates CPU load for HPA testing
> func stressHandler(w http.ResponseWriter, r *http.Request) {
> 	start := time.Now()
> 	// Burn CPU for ~100ms
> 	for time.Since(start) < 100*time.Millisecond {
> 		_ = math.Sqrt(12345.6789)
> 	}
> 	w.Header().Set("Content-Type", "application/json")
> 	json.NewEncoder(w).Encode(map[string]string{
> 		"status":  "stressed",
> 		"elapsed": time.Since(start).String(),
> 	})
> }
> 
> func getEnv(key, fallback string) string {
> 	if v, ok := os.LookupEnv(key); ok {
> 		return v
> 	}
> 	return fallback
> }
> ```

> [!success]- Full Solution: `app/Dockerfile`
> ```dockerfile
> FROM golang:1.21-alpine AS builder
> WORKDIR /app
> COPY main.go .
> RUN CGO_ENABLED=0 go build -o /server main.go
> 
> FROM alpine:3.19
> COPY --from=builder /server /server
> EXPOSE 8080
> ENTRYPOINT ["/server"]
> ```

Build both versions:

```bash
docker build -t myapp:v1 ./app/
docker build -t myapp:v2 ./app/

minikube image load myapp:v1
minikube image load myapp:v2
```

---

## Strategy 1: Blue/Green Deployment

> [!abstract] Key Concept: Blue/Green
> Two identical environments run side by side. Only one serves production traffic at a time. Switching is instant by updating the Service selector. Rollback is equally instant.
>
> ```
>   Before switch:                After switch:
>   Service ──► Blue (v1)         Service ──► Green (v2)
>               Green (v2) idle               Blue (v1) idle
> ```

> [!success]- Full Solution: `blue-green/namespace.yaml`
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: blue-green
> ```

> [!success]- Full Solution: `blue-green/deployment-blue.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-blue
>   namespace: blue-green
>   labels:
>     app: myapp
>     version: blue
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: myapp
>       version: blue
>   template:
>     metadata:
>       labels:
>         app: myapp
>         version: blue
>     spec:
>       containers:
>         - name: app
>           image: myapp:v1
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           env:
>             - name: APP_VERSION
>               value: "v1"
>             - name: APP_COLOR
>               value: "blue"
>           livenessProbe:
>             httpGet:
>               path: /health
>               port: 8080
>             initialDelaySeconds: 3
>             periodSeconds: 10
>           readinessProbe:
>             httpGet:
>               path: /ready
>               port: 8080
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

> [!success]- Full Solution: `blue-green/deployment-green.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-green
>   namespace: blue-green
>   labels:
>     app: myapp
>     version: green
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: myapp
>       version: green
>   template:
>     metadata:
>       labels:
>         app: myapp
>         version: green
>     spec:
>       containers:
>         - name: app
>           image: myapp:v2
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           env:
>             - name: APP_VERSION
>               value: "v2"
>             - name: APP_COLOR
>               value: "green"
>           livenessProbe:
>             httpGet:
>               path: /health
>               port: 8080
>             initialDelaySeconds: 3
>             periodSeconds: 10
>           readinessProbe:
>             httpGet:
>               path: /ready
>               port: 8080
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

> [!success]- Full Solution: `blue-green/service.yaml`
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: myapp-svc
>   namespace: blue-green
> spec:
>   selector:
>     app: myapp
>     version: blue    # <-- Switch this to "green" to cutover
>   ports:
>     - port: 80
>       targetPort: 8080
>   type: ClusterIP
> ```

### Run the Blue/Green Demo

```bash
# Deploy namespace and both versions
kubectl apply -f blue-green/namespace.yaml
kubectl apply -f blue-green/deployment-blue.yaml
kubectl apply -f blue-green/deployment-green.yaml
kubectl apply -f blue-green/service.yaml

# Verify both deployments are running
kubectl get pods -n blue-green -l app=myapp --show-labels

# Test: all traffic goes to blue (v1)
kubectl port-forward svc/myapp-svc 8080:80 -n blue-green &
for i in $(seq 1 5); do curl -s http://localhost:8080 | jq .version; done
# Output: all "v1"

# SWITCH: Change service selector to green
kubectl patch svc myapp-svc -n blue-green \
  -p '{"spec":{"selector":{"version":"green"}}}'

# Test: all traffic now goes to green (v2)
for i in $(seq 1 5); do curl -s http://localhost:8080 | jq .version; done
# Output: all "v2"

# ROLLBACK: Switch back to blue
kubectl patch svc myapp-svc -n blue-green \
  -p '{"spec":{"selector":{"version":"blue"}}}'
```

> [!tip] Blue/Green Trade-offs
> **Pros**: Instant switch, instant rollback, zero downtime, easy to test green before switching.
> **Cons**: Requires 2x the resources (both versions running simultaneously).

---

## Strategy 2: Canary Deployment

> [!abstract] Key Concept: Canary
> Route a small percentage of traffic to the new version. Monitor for errors. Gradually increase traffic until 100% or rollback if issues are detected.
>
> ```
>   Phase 1: v1 (90%) + v2 (10%)     -- Canary
>   Phase 2: v1 (50%) + v2 (50%)     -- Expanding
>   Phase 3: v2 (100%)                -- Promoted
> ```

The simplest canary approach uses replica counts to control traffic distribution. With 9 v1 pods and 1 v2 pod behind the same Service, approximately 10% of traffic goes to v2.

> [!success]- Full Solution: `canary/namespace.yaml`
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: canary
> ```

> [!success]- Full Solution: `canary/deployment-stable.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-stable
>   namespace: canary
>   labels:
>     app: myapp
>     track: stable
> spec:
>   replicas: 9    # 90% of traffic
>   selector:
>     matchLabels:
>       app: myapp
>       track: stable
>   template:
>     metadata:
>       labels:
>         app: myapp
>         track: stable
>     spec:
>       containers:
>         - name: app
>           image: myapp:v1
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           env:
>             - name: APP_VERSION
>               value: "v1"
>             - name: APP_COLOR
>               value: "stable"
>           readinessProbe:
>             httpGet:
>               path: /ready
>               port: 8080
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

> [!success]- Full Solution: `canary/deployment-canary.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-canary
>   namespace: canary
>   labels:
>     app: myapp
>     track: canary
> spec:
>   replicas: 1    # 10% of traffic
>   selector:
>     matchLabels:
>       app: myapp
>       track: canary
>   template:
>     metadata:
>       labels:
>         app: myapp
>         track: canary
>     spec:
>       containers:
>         - name: app
>           image: myapp:v2
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           env:
>             - name: APP_VERSION
>               value: "v2"
>             - name: APP_COLOR
>               value: "canary"
>           readinessProbe:
>             httpGet:
>               path: /ready
>               port: 8080
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

> [!success]- Full Solution: `canary/service.yaml`
> ```yaml
> # The Service selects BOTH stable and canary pods via the shared "app: myapp" label.
> # Traffic distribution is proportional to the number of pods.
> apiVersion: v1
> kind: Service
> metadata:
>   name: myapp-svc
>   namespace: canary
> spec:
>   selector:
>     app: myapp    # Matches both stable and canary
>   ports:
>     - port: 80
>       targetPort: 8080
>   type: ClusterIP
> ```

### Run the Canary Demo

```bash
# Deploy
kubectl apply -f canary/namespace.yaml
kubectl apply -f canary/deployment-stable.yaml
kubectl apply -f canary/deployment-canary.yaml
kubectl apply -f canary/service.yaml

# Verify: 9 stable + 1 canary = 10 total pods
kubectl get pods -n canary -l app=myapp --show-labels

# Test traffic distribution (expect ~10% v2)
kubectl port-forward svc/myapp-svc 8080:80 -n canary &
for i in $(seq 1 20); do curl -s http://localhost:8080 | jq -r .version; done | sort | uniq -c
# Expected: ~18 v1, ~2 v2

# Phase 2: Increase canary to 50%
kubectl scale deployment app-stable -n canary --replicas=5
kubectl scale deployment app-canary -n canary --replicas=5

# Test again
for i in $(seq 1 20); do curl -s http://localhost:8080 | jq -r .version; done | sort | uniq -c
# Expected: ~10 v1, ~10 v2

# Phase 3: Promote canary to 100%
kubectl scale deployment app-stable -n canary --replicas=0
kubectl scale deployment app-canary -n canary --replicas=10

# ROLLBACK: If canary shows problems, scale it down
# kubectl scale deployment app-canary -n canary --replicas=0
# kubectl scale deployment app-stable -n canary --replicas=10
```

> [!success]- Full Solution: Canary Automation Script (`canary/promote.sh`)
> ```bash
> #!/bin/bash
> # Canary promotion script with health checking
> set -e
> 
> NAMESPACE="canary"
> STABLE_DEPLOY="app-stable"
> CANARY_DEPLOY="app-canary"
> SERVICE_URL="http://localhost:8080"
> TOTAL_REPLICAS=10
> ERROR_THRESHOLD=5  # Max errors allowed during canary check
> 
> # Canary percentages to step through
> CANARY_STEPS=(10 25 50 75 100)
> 
> for PERCENT in "${CANARY_STEPS[@]}"; do
>     CANARY_REPLICAS=$((TOTAL_REPLICAS * PERCENT / 100))
>     STABLE_REPLICAS=$((TOTAL_REPLICAS - CANARY_REPLICAS))
> 
>     echo "=== Canary Phase: ${PERCENT}% (stable=${STABLE_REPLICAS}, canary=${CANARY_REPLICAS}) ==="
> 
>     kubectl scale deployment ${STABLE_DEPLOY} -n ${NAMESPACE} --replicas=${STABLE_REPLICAS}
>     kubectl scale deployment ${CANARY_DEPLOY} -n ${NAMESPACE} --replicas=${CANARY_REPLICAS}
> 
>     # Wait for rollout
>     kubectl rollout status deployment/${CANARY_DEPLOY} -n ${NAMESPACE} --timeout=60s
> 
>     # Health check: send 50 requests and count errors
>     echo "Running health check..."
>     ERRORS=0
>     for i in $(seq 1 50); do
>         HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" ${SERVICE_URL}/ 2>/dev/null || echo "000")
>         if [ "${HTTP_CODE}" != "200" ]; then
>             ERRORS=$((ERRORS + 1))
>         fi
>     done
> 
>     echo "Health check: ${ERRORS} errors out of 50 requests"
> 
>     if [ ${ERRORS} -gt ${ERROR_THRESHOLD} ]; then
>         echo "ERROR: Too many errors. Rolling back!"
>         kubectl scale deployment ${CANARY_DEPLOY} -n ${NAMESPACE} --replicas=0
>         kubectl scale deployment ${STABLE_DEPLOY} -n ${NAMESPACE} --replicas=${TOTAL_REPLICAS}
>         echo "Rollback complete."
>         exit 1
>     fi
> 
>     if [ ${PERCENT} -lt 100 ]; then
>         echo "Canary at ${PERCENT}% looks healthy. Proceeding in 10 seconds..."
>         sleep 10
>     fi
> done
> 
> echo "=== Canary promotion complete! v2 is now at 100% ==="
> ```

---

## Strategy 3: Rolling Update Tuning

> [!abstract] Key Concept: Rolling Update Parameters
> Kubernetes' default deployment strategy is `RollingUpdate`. You can tune it with:
> - `maxSurge`: Max pods above desired count during update (extra capacity)
> - `maxUnavailable`: Max pods that can be unavailable during update
> - `minReadySeconds`: Wait time after a pod is ready before considering it available
>
> ```
>   maxSurge=1, maxUnavailable=0 (safest, slowest):
>   v1 v1 v1 -> v1 v1 v1 v2 -> v1 v1 v2 v2 -> v1 v2 v2 v2 -> v2 v2 v2
>
>   maxSurge=0, maxUnavailable=1 (resource-efficient):
>   v1 v1 v1 -> v1 v1 __ -> v1 v1 v2 -> v1 __ v2 -> v1 v2 v2 -> __ v2 v2 -> v2 v2 v2
>
>   maxSurge=100%, maxUnavailable=100% (fastest = recreate):
>   v1 v1 v1 -> __ __ __ -> v2 v2 v2
> ```

> [!success]- Full Solution: `rolling/deployment-safe.yaml`
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: rolling
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-safe-rolling
>   namespace: rolling
> spec:
>   replicas: 5
>   strategy:
>     type: RollingUpdate
>     rollingUpdate:
>       maxSurge: 1           # Allow 1 extra pod during update
>       maxUnavailable: 0     # Never have fewer than desired pods
>   minReadySeconds: 10       # Wait 10s after ready before proceeding
>   selector:
>     matchLabels:
>       app: myapp-safe
>   template:
>     metadata:
>       labels:
>         app: myapp-safe
>     spec:
>       containers:
>         - name: app
>           image: myapp:v1
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           env:
>             - name: APP_VERSION
>               value: "v1"
>           readinessProbe:
>             httpGet:
>               path: /ready
>               port: 8080
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
>   name: myapp-safe-svc
>   namespace: rolling
> spec:
>   selector:
>     app: myapp-safe
>   ports:
>     - port: 80
>       targetPort: 8080
> ```

> [!success]- Full Solution: `rolling/deployment-fast.yaml`
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-fast-rolling
>   namespace: rolling
> spec:
>   replicas: 5
>   strategy:
>     type: RollingUpdate
>     rollingUpdate:
>       maxSurge: 3           # Allow 3 extra pods
>       maxUnavailable: 2     # Allow 2 pods to be down
>   minReadySeconds: 0        # No wait after ready
>   selector:
>     matchLabels:
>       app: myapp-fast
>   template:
>     metadata:
>       labels:
>         app: myapp-fast
>     spec:
>       containers:
>         - name: app
>           image: myapp:v1
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           env:
>             - name: APP_VERSION
>               value: "v1"
>           readinessProbe:
>             httpGet:
>               path: /ready
>               port: 8080
>             initialDelaySeconds: 1
>             periodSeconds: 2
>           resources:
>             requests:
>               memory: "32Mi"
>               cpu: "50m"
>             limits:
>               memory: "64Mi"
>               cpu: "100m"
> ```

### Run the Rolling Update Demo

```bash
kubectl apply -f rolling/deployment-safe.yaml

# Watch the rolling update in real-time
# Terminal 1: watch pods
kubectl get pods -n rolling -l app=myapp-safe -w

# Terminal 2: trigger the update
kubectl set image deployment/app-safe-rolling -n rolling app=myapp:v2

# Observe: pods are replaced one at a time, with 10s pause between each
# The update takes ~60 seconds for 5 replicas

# Compare with fast rolling update
kubectl apply -f rolling/deployment-fast.yaml
kubectl set image deployment/app-fast-rolling -n rolling app=myapp:v2
# This completes much faster
```

---

## Strategy 4: Horizontal Pod Autoscaler (HPA)

> [!abstract] Key Concept: HPA
> HPA automatically adjusts the number of pod replicas based on CPU/memory utilization or custom metrics. It checks metrics every 15 seconds by default and scales within configured min/max bounds.
>
> ```
>   Load increases ──► CPU > target ──► HPA adds pods
>   Load decreases ──► CPU < target ──► HPA removes pods (after cooldown)
> ```

> [!success]- Full Solution: `hpa/deployment.yaml`
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: hpa-demo
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-hpa
>   namespace: hpa-demo
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: myapp-hpa
>   template:
>     metadata:
>       labels:
>         app: myapp-hpa
>     spec:
>       containers:
>         - name: app
>           image: myapp:v1
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           env:
>             - name: APP_VERSION
>               value: "v1"
>           resources:
>             requests:
>               memory: "32Mi"
>               cpu: "100m"     # Must set requests for HPA to work
>             limits:
>               memory: "64Mi"
>               cpu: "200m"
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: myapp-hpa-svc
>   namespace: hpa-demo
> spec:
>   selector:
>     app: myapp-hpa
>   ports:
>     - port: 80
>       targetPort: 8080
> ```

> [!success]- Full Solution: `hpa/hpa.yaml`
> ```yaml
> apiVersion: autoscaling/v2
> kind: HorizontalPodAutoscaler
> metadata:
>   name: app-hpa
>   namespace: hpa-demo
> spec:
>   scaleTargetRef:
>     apiVersion: apps/v1
>     kind: Deployment
>     name: app-hpa
>   minReplicas: 1
>   maxReplicas: 10
>   metrics:
>     - type: Resource
>       resource:
>         name: cpu
>         target:
>           type: Utilization
>           averageUtilization: 50    # Scale when CPU > 50% of request
>   behavior:
>     scaleUp:
>       stabilizationWindowSeconds: 30
>       policies:
>         - type: Pods
>           value: 2
>           periodSeconds: 60        # Add max 2 pods per minute
>     scaleDown:
>       stabilizationWindowSeconds: 300   # Wait 5 min before scaling down
>       policies:
>         - type: Pods
>           value: 1
>           periodSeconds: 60        # Remove max 1 pod per minute
> ```

### Run the HPA Demo

```bash
# Ensure metrics-server is running
minikube addons enable metrics-server

# Deploy
kubectl apply -f hpa/deployment.yaml
kubectl apply -f hpa/hpa.yaml

# Verify HPA is working (may show <unknown> initially, wait ~30s for metrics)
kubectl get hpa -n hpa-demo -w
```

> [!success]- Full Solution: Load Generation Script (`hpa/load-gen.sh`)
> ```bash
> #!/bin/bash
> # Generate load to trigger HPA scaling
> 
> echo "Starting load generation against the /stress endpoint..."
> echo "Watch HPA in another terminal: kubectl get hpa -n hpa-demo -w"
> echo "Watch pods: kubectl get pods -n hpa-demo -w"
> echo ""
> echo "Press Ctrl+C to stop."
> 
> # Start port-forward in background
> kubectl port-forward svc/myapp-hpa-svc 8080:80 -n hpa-demo &
> PF_PID=$!
> sleep 2
> 
> # Generate load with concurrent requests
> while true; do
>     # Send 20 concurrent requests
>     for i in $(seq 1 20); do
>         curl -s http://localhost:8080/stress > /dev/null &
>     done
>     wait
> done
> 
> # Cleanup on exit
> trap "kill $PF_PID 2>/dev/null" EXIT
> ```

```bash
# Terminal 1: Watch HPA and pods
kubectl get hpa -n hpa-demo -w

# Terminal 2: Watch pods
kubectl get pods -n hpa-demo -w

# Terminal 3: Generate load
chmod +x hpa/load-gen.sh
./hpa/load-gen.sh

# After ~1-2 minutes of load:
# - HPA should show CPU > 50%
# - Replica count increases from 1 toward 10
# Stop the load generator (Ctrl+C)
# After ~5 minutes (scaleDown stabilization window), replicas decrease back to 1
```

> [!warning] Metrics Delay
> HPA reads metrics every 15 seconds, but metrics-server itself has a collection interval of ~60 seconds. After deploying, wait 1-2 minutes before expecting HPA to show real CPU values instead of `<unknown>`.

---

## Strategy 5: Pod Disruption Budget (PDB)

> [!abstract] Key Concept: PDB
> A PDB limits how many pods in a Deployment can be voluntarily disrupted at once (e.g., during node drain, cluster upgrades). It does **not** protect against involuntary disruptions (node crashes).
>
> - `minAvailable: 2` means at least 2 pods must always be running
> - `maxUnavailable: 1` means at most 1 pod can be down at a time
> - You can use absolute numbers or percentages

> [!success]- Full Solution: `pdb/deployment.yaml`
> ```yaml
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: pdb-demo
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-pdb
>   namespace: pdb-demo
> spec:
>   replicas: 5
>   selector:
>     matchLabels:
>       app: myapp-pdb
>   template:
>     metadata:
>       labels:
>         app: myapp-pdb
>     spec:
>       containers:
>         - name: app
>           image: myapp:v1
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           readinessProbe:
>             httpGet:
>               path: /ready
>               port: 8080
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

> [!success]- Full Solution: `pdb/pdb.yaml`
> ```yaml
> apiVersion: policy/v1
> kind: PodDisruptionBudget
> metadata:
>   name: app-pdb
>   namespace: pdb-demo
> spec:
>   # At least 3 pods must always be available during voluntary disruptions
>   minAvailable: 3
>   # OR use: maxUnavailable: 2  (same effect with 5 replicas)
>   selector:
>     matchLabels:
>       app: myapp-pdb
> ```

### Run the PDB Demo

```bash
# Deploy
kubectl apply -f pdb/deployment.yaml
kubectl apply -f pdb/pdb.yaml

# Verify PDB
kubectl get pdb -n pdb-demo
# NAME      MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# app-pdb   3               N/A               2                     10s

# Simulate node drain (in minikube, all pods are on the same node)
# This will attempt to evict pods respecting the PDB
kubectl drain minikube --ignore-daemonsets --delete-emptydir-data

# The drain will proceed but only evict 2 pods at a time (5 - minAvailable 3 = 2)
# Since minikube has only 1 node, the drain will eventually get stuck
# because evicted pods cannot reschedule elsewhere.

# Uncordon the node to restore scheduling
kubectl uncordon minikube
```

> [!tip] PDB in Multi-Node Clusters
> PDB is most useful in multi-node clusters where pods can reschedule to other nodes during a drain. In single-node minikube, drained pods have nowhere to go, so the drain may hang until you uncordon.

---

## Strategy 6: Monitoring with Prometheus and Grafana

### Install Prometheus + Grafana via Helm

```bash
# Add Helm repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install the kube-prometheus-stack (includes Prometheus, Grafana, and Alertmanager)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

### Access Grafana

```bash
# Port-forward Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring &

# Login at http://localhost:3000
# Username: admin
# Password: admin
```

### Create a ServiceMonitor for the Application

> [!success]- Full Solution: `monitoring/service-monitor.yaml`
> ```yaml
> apiVersion: monitoring.coreos.com/v1
> kind: ServiceMonitor
> metadata:
>   name: myapp-monitor
>   namespace: monitoring
>   labels:
>     release: monitoring
> spec:
>   namespaceSelector:
>     matchNames:
>       - canary
>       - blue-green
>       - hpa-demo
>   selector:
>     matchLabels:
>       app: myapp
>   endpoints:
>     - port: http
>       interval: 15s
>       path: /metrics
> ```

### Useful Grafana Dashboards

Pre-built dashboards to import (Grafana UI -> Dashboards -> Import):

| Dashboard | ID | Purpose |
|---|---|---|
| Kubernetes Cluster Monitoring | 6417 | Cluster-wide resource overview |
| Kubernetes Deployment Metrics | 8588 | Deployment-level CPU, memory, replicas |
| Kubernetes Pod Metrics | 6336 | Pod-level resource usage and restarts |

### Key Prometheus Queries for Deployments

```promql
# Current replica count
kube_deployment_spec_replicas{namespace="canary"}

# Available replicas
kube_deployment_status_replicas_available{namespace="canary"}

# Pod restart count (canary health signal)
sum(kube_pod_container_status_restarts_total{namespace="canary"}) by (pod)

# CPU usage by pod
sum(rate(container_cpu_usage_seconds_total{namespace="canary"}[5m])) by (pod)

# Memory usage by pod
sum(container_memory_working_set_bytes{namespace="canary"}) by (pod)

# Request rate (if app exposes /metrics)
sum(rate(http_requests_total{namespace="canary"}[5m])) by (version)

# Error rate
sum(rate(http_requests_total{namespace="canary",code=~"5.."}[5m])) by (version)
```

---

## Strategy 7: Full Scenario -- Canary with Monitoring

Put everything together: deploy v1, canary v2 to 10%, monitor metrics, decide whether to promote or rollback.

> [!success]- Full Solution: Full Scenario Script (`full-scenario/run.sh`)
> ```bash
> #!/bin/bash
> set -e
> 
> NAMESPACE="canary-full"
> 
> echo "============================================"
> echo "  Full Canary Deployment Scenario"
> echo "============================================"
> echo ""
> 
> # Step 1: Create namespace and deploy v1 (stable)
> echo ">>> Step 1: Deploy v1 (stable) with 10 replicas"
> cat <<EOF | kubectl apply -f -
> apiVersion: v1
> kind: Namespace
> metadata:
>   name: ${NAMESPACE}
> ---
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-stable
>   namespace: ${NAMESPACE}
> spec:
>   replicas: 10
>   selector:
>     matchLabels:
>       app: myapp
>       track: stable
>   template:
>     metadata:
>       labels:
>         app: myapp
>         track: stable
>     spec:
>       containers:
>         - name: app
>           image: myapp:v1
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           env:
>             - name: APP_VERSION
>               value: "v1"
>             - name: APP_COLOR
>               value: "stable"
>           readinessProbe:
>             httpGet:
>               path: /ready
>               port: 8080
>             initialDelaySeconds: 2
>             periodSeconds: 3
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
>   name: myapp-svc
>   namespace: ${NAMESPACE}
> spec:
>   selector:
>     app: myapp
>   ports:
>     - port: 80
>       targetPort: 8080
> ---
> apiVersion: policy/v1
> kind: PodDisruptionBudget
> metadata:
>   name: app-pdb
>   namespace: ${NAMESPACE}
> spec:
>   minAvailable: 8
>   selector:
>     matchLabels:
>       app: myapp
> EOF
> 
> kubectl rollout status deployment/app-stable -n ${NAMESPACE} --timeout=120s
> echo "v1 stable deployment ready."
> echo ""
> 
> # Step 2: Deploy canary (v2) at 10%
> echo ">>> Step 2: Deploy canary (v2) at 10% traffic"
> cat <<EOF | kubectl apply -f -
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: app-canary
>   namespace: ${NAMESPACE}
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: myapp
>       track: canary
>   template:
>     metadata:
>       labels:
>         app: myapp
>         track: canary
>     spec:
>       containers:
>         - name: app
>           image: myapp:v2
>           imagePullPolicy: Never
>           ports:
>             - containerPort: 8080
>           env:
>             - name: APP_VERSION
>               value: "v2"
>             - name: APP_COLOR
>               value: "canary"
>           readinessProbe:
>             httpGet:
>               path: /ready
>               port: 8080
>             initialDelaySeconds: 2
>             periodSeconds: 3
>           resources:
>             requests:
>               memory: "32Mi"
>               cpu: "50m"
>             limits:
>               memory: "64Mi"
>               cpu: "100m"
> EOF
> 
> kubectl rollout status deployment/app-canary -n ${NAMESPACE} --timeout=60s
> 
> # Reduce stable to maintain total at ~10 pods
> kubectl scale deployment app-stable -n ${NAMESPACE} --replicas=9
> echo "Canary deployed: 9 stable + 1 canary = ~10% canary traffic"
> echo ""
> 
> # Step 3: Start port-forward and test
> echo ">>> Step 3: Testing traffic distribution"
> kubectl port-forward svc/myapp-svc 8080:80 -n ${NAMESPACE} &
> PF_PID=$!
> sleep 2
> 
> echo "Sending 100 requests..."
> V1_COUNT=0
> V2_COUNT=0
> ERRORS=0
> for i in $(seq 1 100); do
>     RESULT=$(curl -s http://localhost:8080/ 2>/dev/null)
>     VERSION=$(echo "$RESULT" | grep -o '"version":"[^"]*"' | cut -d'"' -f4)
>     case "$VERSION" in
>         v1) V1_COUNT=$((V1_COUNT + 1)) ;;
>         v2) V2_COUNT=$((V2_COUNT + 1)) ;;
>         *)  ERRORS=$((ERRORS + 1)) ;;
>     esac
> done
> 
> echo "Results: v1=${V1_COUNT}, v2=${V2_COUNT}, errors=${ERRORS}"
> echo ""
> 
> # Step 4: Decision point
> if [ ${ERRORS} -gt 5 ]; then
>     echo ">>> Too many errors! Rolling back canary..."
>     kubectl scale deployment app-canary -n ${NAMESPACE} --replicas=0
>     kubectl scale deployment app-stable -n ${NAMESPACE} --replicas=10
>     echo "ROLLBACK COMPLETE"
> else
>     echo ">>> Canary looks healthy! Promoting..."
> 
>     # Phase 2: 50%
>     echo "  Scaling to 50/50..."
>     kubectl scale deployment app-stable -n ${NAMESPACE} --replicas=5
>     kubectl scale deployment app-canary -n ${NAMESPACE} --replicas=5
>     sleep 5
> 
>     # Phase 3: 100% v2
>     echo "  Promoting to 100% v2..."
>     kubectl scale deployment app-stable -n ${NAMESPACE} --replicas=0
>     kubectl scale deployment app-canary -n ${NAMESPACE} --replicas=10
> 
>     echo "PROMOTION COMPLETE: v2 is now serving 100% of traffic"
> fi
> 
> # Cleanup port-forward
> kill $PF_PID 2>/dev/null
> 
> echo ""
> echo ">>> Final state:"
> kubectl get pods -n ${NAMESPACE} -l app=myapp --no-headers | awk '{print $1}' | while read pod; do
>     VERSION=$(kubectl get pod "$pod" -n ${NAMESPACE} -o jsonpath='{.spec.containers[0].env[?(@.name=="APP_VERSION")].value}')
>     echo "  $pod -> $VERSION"
> done
> ```

### Run the Full Scenario

```bash
chmod +x full-scenario/run.sh
./full-scenario/run.sh
```

Monitor in Grafana while the scenario runs:
1. Open `http://localhost:3000`
2. Navigate to Explore -> Prometheus
3. Query: `kube_deployment_spec_replicas{namespace="canary-full"}`
4. Watch replica counts change as the canary progresses

---

## Cleanup

```bash
kubectl delete namespace blue-green canary rolling hpa-demo pdb-demo canary-full
helm uninstall monitoring -n monitoring
kubectl delete namespace monitoring
```

---

## Strategy Comparison

| Strategy | Downtime | Rollback Speed | Resource Cost | Complexity | Best For |
|---|---|---|---|---|---|
| **Rolling Update** | None | Moderate (re-rollout) | 1x + surge | Low | Default, most workloads |
| **Blue/Green** | None | Instant (switch selector) | 2x | Low | Databases, stateful apps |
| **Canary** | None | Fast (scale down canary) | 1x + canary | Medium | High-traffic, risk-sensitive |
| **Recreate** | Brief | Fast (redeploy old) | 1x | Lowest | Dev environments, breaking changes |

> [!abstract] Key Takeaways
> - **Blue/Green**: Instant switch and rollback at the cost of 2x resources. Best when you need zero-risk cutover.
> - **Canary**: Gradual traffic shift with real-world validation. Use replica ratios for simple traffic splitting.
> - **Rolling Update Tuning**: `maxSurge` and `maxUnavailable` let you trade speed for safety. `minReadySeconds` prevents fast-fail scenarios.
> - **HPA**: Auto-scales based on CPU/memory metrics. Always set resource requests. Use scale-down stabilization to prevent flapping.
> - **PDB**: Protects availability during voluntary disruptions (drains, upgrades). Use `minAvailable` or `maxUnavailable`.
> - **Monitoring**: Prometheus + Grafana provide the observability needed to make promotion/rollback decisions during canary deployments.
