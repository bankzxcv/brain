---
title: "K8s Practice: Services and Networking"
date: 2026-04-07
tags:
  - kubernetes
  - practice
  - services
  - networking
parent: "[[Kubernetes Study]]"
---

# Practice - Services and Networking

13 progressive exercises covering Kubernetes Services, DNS discovery, Ingress, and NetworkPolicies. All exercises run on minikube.

## Prerequisites

- minikube installed and running: `minikube start`
- kubectl configured: `kubectl cluster-info`
- Create a working namespace: `kubectl create namespace practice`
- Set it as default: `kubectl config set-context --current --namespace=practice`
- Enable ingress addon (needed for P7+): `minikube addons enable ingress`

---

## P1: ClusterIP Service

**Task:** Create an nginx Deployment with 3 replicas and a ClusterIP Service. Verify connectivity by `curl`ing the Service from another Pod inside the cluster.

**Expected result:**
- Service is created with a ClusterIP
- Requests from inside the cluster reach the nginx pods

**Verification:**

```bash
kubectl get svc
kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://nginx-svc
```

> [!hint]- Hint
> - A ClusterIP Service is the default type (you don't need to specify `type: ClusterIP`)
> - The Service `spec.selector` must match the Deployment's pod labels
> - The Service `port` is what clients connect to; `targetPort` is the container port
> - To test: run a temporary pod with curl

> [!success]- Solution
> **nginx-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx
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
> **nginx-svc.yaml:**
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: nginx-svc
>   namespace: practice
> spec:
>   selector:
>     app: nginx
>   ports:
>     - port: 80
>       targetPort: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f nginx-deployment.yaml
> kubectl apply -f nginx-svc.yaml
> 
> # Check the service
> kubectl get svc nginx-svc
> # NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
> # nginx-svc   ClusterIP   10.96.x.x     <none>        80/TCP    5s
> 
> # Verify endpoints (should show 3 pod IPs)
> kubectl get endpoints nginx-svc
> 
> # Test from inside the cluster
> kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://nginx-svc
> # Should return nginx welcome page HTML
> 
> # Cleanup
> kubectl delete deployment nginx
> kubectl delete svc nginx-svc
> ```

---

## P2: NodePort Service

**Task:** Create a NodePort Service for an nginx Deployment. Access the app from your host machine using minikube's IP.

**Expected result:**
- Service exposes a port on the node (30000-32767 range)
- `curl http://<minikube-ip>:<nodePort>` returns the nginx page

**Verification:**

```bash
minikube service nginx-nodeport --url
curl $(minikube service nginx-nodeport --url)
```

> [!hint]- Hint
> - Set `type: NodePort` in the Service spec
> - You can specify a `nodePort` (30000-32767) or let Kubernetes pick one
> - Get minikube IP: `minikube ip`
> - Shortcut: `minikube service <svc-name> --url` gives you the full URL

> [!success]- Solution
> **nginx-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx
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
> **nginx-nodeport.yaml:**
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: nginx-nodeport
>   namespace: practice
> spec:
>   type: NodePort
>   selector:
>     app: nginx
>   ports:
>     - port: 80
>       targetPort: 80
>       nodePort: 30080
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f nginx-deployment.yaml
> kubectl apply -f nginx-nodeport.yaml
> 
> # Check the service
> kubectl get svc nginx-nodeport
> # NAME              TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
> # nginx-nodeport    NodePort   10.96.x.x    <none>        80:30080/TCP   5s
> 
> # Access from host
> minikube service nginx-nodeport --url -n practice
> # http://192.168.49.2:30080
> 
> curl $(minikube service nginx-nodeport --url -n practice)
> # Should return nginx welcome page
> 
> # Or manually
> curl http://$(minikube ip):30080
> 
> # Cleanup
> kubectl delete deployment nginx
> kubectl delete svc nginx-nodeport
> ```

---

## P3: DNS Discovery Between Services

**Task:** Deploy two applications:
1. **backend** - a simple HTTP server that returns JSON
2. **frontend** - a pod that calls the backend by its Service DNS name

Verify that the frontend can reach the backend using the Kubernetes DNS name.

**Expected result:**
- frontend can call `http://backend-svc` (short name) or `http://backend-svc.practice.svc.cluster.local` (FQDN)

**Verification:**

```bash
kubectl exec frontend -- wget -qO- http://backend-svc
```

> [!hint]- Hint
> - Kubernetes DNS format: `<service-name>.<namespace>.svc.cluster.local`
> - Within the same namespace, just `<service-name>` works
> - For cross-namespace: `<service-name>.<namespace>`
> - Deploy backend with a Service, then test from frontend pod

> [!success]- Solution
> **backend-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: backend
>   namespace: practice
> spec:
>   replicas: 2
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
>           image: hashicorp/http-echo
>           args:
>             - "-text=Hello from backend!"
>             - "-listen=:8080"
>           ports:
>             - containerPort: 8080
> ```
> 
> **backend-svc.yaml:**
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: backend-svc
>   namespace: practice
> spec:
>   selector:
>     app: backend
>   ports:
>     - port: 80
>       targetPort: 8080
> ```
> 
> **frontend-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: frontend
>   namespace: practice
> spec:
>   containers:
>     - name: frontend
>       image: busybox:1.36
>       command: ["sleep", "3600"]
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f backend-deployment.yaml
> kubectl apply -f backend-svc.yaml
> kubectl apply -f frontend-pod.yaml
> 
> # Test DNS discovery - short name
> kubectl exec frontend -- wget -qO- http://backend-svc
> # Hello from backend!
> 
> # Test DNS discovery - FQDN
> kubectl exec frontend -- wget -qO- http://backend-svc.practice.svc.cluster.local
> # Hello from backend!
> 
> # Verify DNS resolution
> kubectl exec frontend -- nslookup backend-svc
> # Name:      backend-svc
> # Address 1: 10.96.x.x backend-svc.practice.svc.cluster.local
> 
> # Cleanup
> kubectl delete deployment backend
> kubectl delete svc backend-svc
> kubectl delete pod frontend
> ```

---

## P4: Headless Service

**Task:** Create a headless Service (`clusterIP: None`) for a StatefulSet-like use case. Inspect the DNS entries to see individual pod IPs returned instead of a single virtual IP.

**Expected result:**
- The Service has no ClusterIP (`None`)
- DNS lookup returns individual pod IPs instead of a single VIP

**Verification:**

```bash
kubectl exec dns-test -- nslookup headless-svc
# Should return multiple A records (one per pod IP)
```

> [!hint]- Hint
> - Set `clusterIP: None` in the Service spec
> - Headless services return pod IPs directly in DNS (no load balancing by kube-proxy)
> - This is used by StatefulSets for stable network identities
> - Regular service: DNS returns 1 ClusterIP; Headless: DNS returns N pod IPs

> [!success]- Solution
> **nginx-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: nginx-headless
>   namespace: practice
> spec:
>   replicas: 3
>   selector:
>     matchLabels:
>       app: nginx-headless
>   template:
>     metadata:
>       labels:
>         app: nginx-headless
>     spec:
>       containers:
>         - name: nginx
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 80
> ```
> 
> **headless-svc.yaml:**
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: headless-svc
>   namespace: practice
> spec:
>   clusterIP: None
>   selector:
>     app: nginx-headless
>   ports:
>     - port: 80
>       targetPort: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f nginx-deployment.yaml
> kubectl apply -f headless-svc.yaml
> 
> # Check the service - ClusterIP should be None
> kubectl get svc headless-svc
> # NAME           TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
> # headless-svc   ClusterIP   None         <none>        80/TCP    5s
> 
> # DNS lookup from a test pod - returns individual pod IPs
> kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup headless-svc.practice.svc.cluster.local
> # Name:      headless-svc.practice.svc.cluster.local
> # Address 1: 10.244.0.5
> # Address 2: 10.244.0.6
> # Address 3: 10.244.0.7
> 
> # Compare with a regular ClusterIP service
> kubectl expose deployment nginx-headless --name=regular-svc --port=80 --target-port=80
> kubectl run dns-test2 --image=busybox:1.36 --rm -it --restart=Never -- nslookup regular-svc.practice.svc.cluster.local
> # Returns single ClusterIP instead of pod IPs
> 
> # Cleanup
> kubectl delete deployment nginx-headless
> kubectl delete svc headless-svc regular-svc
> ```

---

## P5: Multi-Port Service

**Task:** Create a Deployment that serves HTTP on port 80 and metrics on port 9090. Create a single Service that exposes both ports.

**Expected result:**
- Service exposes port 80 (HTTP) and port 9090 (metrics)
- Both ports are accessible from within the cluster

**Verification:**

```bash
kubectl get svc multi-port-svc
kubectl describe svc multi-port-svc
```

> [!hint]- Hint
> - List multiple entries under `spec.ports` in the Service
> - Each port entry needs a unique `name` when there are multiple ports
> - You can use `nginx` for HTTP and a sidecar for metrics, or simulate with a single container

> [!success]- Solution
> **multi-port-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: multi-port-app
>   namespace: practice
> spec:
>   replicas: 2
>   selector:
>     matchLabels:
>       app: multi-port
>   template:
>     metadata:
>       labels:
>         app: multi-port
>     spec:
>       containers:
>         - name: web
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 80
>         - name: metrics
>           image: hashicorp/http-echo
>           args:
>             - "-text=metrics_total 42"
>             - "-listen=:9090"
>           ports:
>             - containerPort: 9090
> ```
> 
> **multi-port-svc.yaml:**
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: multi-port-svc
>   namespace: practice
> spec:
>   selector:
>     app: multi-port
>   ports:
>     - name: http
>       port: 80
>       targetPort: 80
>     - name: metrics
>       port: 9090
>       targetPort: 9090
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f multi-port-deployment.yaml
> kubectl apply -f multi-port-svc.yaml
> 
> # Check the service
> kubectl get svc multi-port-svc
> kubectl describe svc multi-port-svc
> # Should show both ports: 80/TCP and 9090/TCP
> 
> # Test HTTP port
> kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://multi-port-svc:80
> # Returns nginx welcome page
> 
> # Test metrics port
> kubectl run curl-test2 --image=curlimages/curl --rm -it --restart=Never -- curl http://multi-port-svc:9090
> # Returns: metrics_total 42
> 
> # Cleanup
> kubectl delete deployment multi-port-app
> kubectl delete svc multi-port-svc
> ```

---

## P6: Named Ports

**Task:** Create a Deployment where containers define named ports. Create a Service that references these ports by name instead of number.

**Expected result:**
- The Service uses `targetPort: http` (name) instead of `targetPort: 80` (number)
- This decouples the Service from the exact container port number

**Verification:**

```bash
kubectl describe svc named-port-svc
# targetPort should show the name, not a number
```

> [!hint]- Hint
> - In the container spec: `ports: [{name: http, containerPort: 80}]`
> - In the Service spec: `targetPort: http` (references the name)
> - Benefit: you can change the container port without updating the Service
> - This is a best practice for production

> [!success]- Solution
> **named-port-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: named-port-app
>   namespace: practice
> spec:
>   replicas: 2
>   selector:
>     matchLabels:
>       app: named-port
>   template:
>     metadata:
>       labels:
>         app: named-port
>     spec:
>       containers:
>         - name: web
>           image: nginx:1.25-alpine
>           ports:
>             - name: http
>               containerPort: 80
>             - name: metrics
>               containerPort: 9113
> ```
> 
> **named-port-svc.yaml:**
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: named-port-svc
>   namespace: practice
> spec:
>   selector:
>     app: named-port
>   ports:
>     - name: http
>       port: 80
>       targetPort: http
>     - name: metrics
>       port: 9113
>       targetPort: metrics
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f named-port-deployment.yaml
> kubectl apply -f named-port-svc.yaml
> 
> # Verify named ports in service
> kubectl describe svc named-port-svc
> # Port:         http  80/TCP
> # TargetPort:   http/TCP
> # Port:         metrics  9113/TCP
> # TargetPort:   metrics/TCP
> 
> # Test connectivity
> kubectl run curl-test --image=curlimages/curl --rm -it --restart=Never -- curl http://named-port-svc:80
> 
> # Cleanup
> kubectl delete deployment named-port-app
> kubectl delete svc named-port-svc
> ```

---

## P7: Ingress with Path Routing

**Task:** Enable the minikube ingress addon. Create two Deployments (frontend and backend) with ClusterIP Services. Create an Ingress that routes `/` to the frontend and `/api` to the backend.

**Expected result:**
- `curl http://<minikube-ip>/` returns the frontend response
- `curl http://<minikube-ip>/api` returns the backend response

**Verification:**

```bash
kubectl get ingress
curl http://$(minikube ip)/
curl http://$(minikube ip)/api
```

> [!hint]- Hint
> - Enable ingress: `minikube addons enable ingress`
> - Ingress API: `networking.k8s.io/v1`
> - Use `pathType: Prefix` for path matching
> - The `backend.service.name` and `backend.service.port.number` fields point to your Services
> - Minikube ingress may take a minute to get an address

> [!success]- Solution
> ```bash
> # Ensure ingress addon is enabled
> minikube addons enable ingress
> ```
> 
> **frontend-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: frontend
>   namespace: practice
> spec:
>   replicas: 2
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
>           image: hashicorp/http-echo
>           args:
>             - "-text=Frontend Response"
>             - "-listen=:8080"
>           ports:
>             - containerPort: 8080
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: frontend-svc
>   namespace: practice
> spec:
>   selector:
>     app: frontend
>   ports:
>     - port: 80
>       targetPort: 8080
> ```
> 
> **backend-deployment.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: backend
>   namespace: practice
> spec:
>   replicas: 2
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
>           image: hashicorp/http-echo
>           args:
>             - "-text=Backend API Response"
>             - "-listen=:8080"
>           ports:
>             - containerPort: 8080
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: backend-svc
>   namespace: practice
> spec:
>   selector:
>     app: backend
>   ports:
>     - port: 80
>       targetPort: 8080
> ```
> 
> **ingress.yaml:**
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: app-ingress
>   namespace: practice
>   annotations:
>     nginx.ingress.kubernetes.io/rewrite-target: /
> spec:
>   rules:
>     - http:
>         paths:
>           - path: /
>             pathType: Prefix
>             backend:
>               service:
>                 name: frontend-svc
>                 port:
>                   number: 80
>           - path: /api
>             pathType: Prefix
>             backend:
>               service:
>                 name: backend-svc
>                 port:
>                   number: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f frontend-deployment.yaml
> kubectl apply -f backend-deployment.yaml
> kubectl apply -f ingress.yaml
> 
> # Wait for ingress to get an address
> kubectl get ingress app-ingress -w
> # Wait until ADDRESS column shows the minikube IP
> 
> # Test routing
> curl http://$(minikube ip)/
> # Frontend Response
> 
> curl http://$(minikube ip)/api
> # Backend API Response
> 
> # Cleanup
> kubectl delete ingress app-ingress
> kubectl delete deployment frontend backend
> kubectl delete svc frontend-svc backend-svc
> ```

---

## P8: Host-Based Ingress Routing

**Task:** Create an Ingress that routes based on hostname:
- `app-a.local` goes to service-a
- `app-b.local` goes to service-b

**Expected result:**
- `curl -H "Host: app-a.local" http://<minikube-ip>` returns response from service-a
- `curl -H "Host: app-b.local" http://<minikube-ip>` returns response from service-b

**Verification:**

```bash
curl -H "Host: app-a.local" http://$(minikube ip)
curl -H "Host: app-b.local" http://$(minikube ip)
```

> [!hint]- Hint
> - Each `rules[]` entry can have a `host` field
> - The Ingress controller matches the `Host` HTTP header to route traffic
> - You can test with `curl -H "Host: ..."` or add entries to `/etc/hosts`
> - Each host has its own set of paths

> [!success]- Solution
> **service-a.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: service-a
>   namespace: practice
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: service-a
>   template:
>     metadata:
>       labels:
>         app: service-a
>     spec:
>       containers:
>         - name: app
>           image: hashicorp/http-echo
>           args: ["-text=Response from Service A", "-listen=:8080"]
>           ports:
>             - containerPort: 8080
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: service-a
>   namespace: practice
> spec:
>   selector:
>     app: service-a
>   ports:
>     - port: 80
>       targetPort: 8080
> ```
> 
> **service-b.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: service-b
>   namespace: practice
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: service-b
>   template:
>     metadata:
>       labels:
>         app: service-b
>     spec:
>       containers:
>         - name: app
>           image: hashicorp/http-echo
>           args: ["-text=Response from Service B", "-listen=:8080"]
>           ports:
>             - containerPort: 8080
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: service-b
>   namespace: practice
> spec:
>   selector:
>     app: service-b
>   ports:
>     - port: 80
>       targetPort: 8080
> ```
> 
> **host-ingress.yaml:**
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: host-ingress
>   namespace: practice
> spec:
>   rules:
>     - host: app-a.local
>       http:
>         paths:
>           - path: /
>             pathType: Prefix
>             backend:
>               service:
>                 name: service-a
>                 port:
>                   number: 80
>     - host: app-b.local
>       http:
>         paths:
>           - path: /
>             pathType: Prefix
>             backend:
>               service:
>                 name: service-b
>                 port:
>                   number: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f service-a.yaml
> kubectl apply -f service-b.yaml
> kubectl apply -f host-ingress.yaml
> 
> # Wait for ingress
> kubectl get ingress host-ingress -w
> 
> # Test with Host header
> curl -H "Host: app-a.local" http://$(minikube ip)
> # Response from Service A
> 
> curl -H "Host: app-b.local" http://$(minikube ip)
> # Response from Service B
> 
> # Optional: add to /etc/hosts for browser testing
> # echo "$(minikube ip) app-a.local app-b.local" | sudo tee -a /etc/hosts
> 
> # Cleanup
> kubectl delete ingress host-ingress
> kubectl delete deployment service-a service-b
> kubectl delete svc service-a service-b
> ```

---

## P9: Ingress with Path Rewriting

**Task:** Create an Ingress where requests to `/app/v1/(.*)` are rewritten to `/$1` before being forwarded to the backend. This is useful when the backend doesn't know about the `/app/v1` prefix.

**Expected result:**
- `curl http://<minikube-ip>/app/v1/` is rewritten to `/` and returns the backend's root response
- The backend only sees `/`, not `/app/v1/`

**Verification:**

```bash
curl http://$(minikube ip)/app/v1/
```

> [!hint]- Hint
> - Use the annotation: `nginx.ingress.kubernetes.io/rewrite-target: /$1`
> - The path should use a regex capture group: `/app/v1/(.*)`
> - Also need: `nginx.ingress.kubernetes.io/use-regex: "true"`
> - The `$1` in rewrite-target references the first capture group

> [!success]- Solution
> **rewrite-backend.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: rewrite-backend
>   namespace: practice
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: rewrite-backend
>   template:
>     metadata:
>       labels:
>         app: rewrite-backend
>     spec:
>       containers:
>         - name: app
>           image: hashicorp/http-echo
>           args: ["-text=Backend root response", "-listen=:8080"]
>           ports:
>             - containerPort: 8080
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: rewrite-backend-svc
>   namespace: practice
> spec:
>   selector:
>     app: rewrite-backend
>   ports:
>     - port: 80
>       targetPort: 8080
> ```
> 
> **rewrite-ingress.yaml:**
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: rewrite-ingress
>   namespace: practice
>   annotations:
>     nginx.ingress.kubernetes.io/rewrite-target: /$1
>     nginx.ingress.kubernetes.io/use-regex: "true"
> spec:
>   rules:
>     - http:
>         paths:
>           - path: /app/v1/(.*)
>             pathType: ImplementationSpecific
>             backend:
>               service:
>                 name: rewrite-backend-svc
>                 port:
>                   number: 80
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f rewrite-backend.yaml
> kubectl apply -f rewrite-ingress.yaml
> 
> # Wait for ingress
> kubectl get ingress rewrite-ingress -w
> 
> # Test - /app/v1/ is rewritten to /
> curl http://$(minikube ip)/app/v1/
> # Backend root response
> 
> # /app/v1/health would be rewritten to /health
> curl http://$(minikube ip)/app/v1/health
> 
> # Cleanup
> kubectl delete ingress rewrite-ingress
> kubectl delete deployment rewrite-backend
> kubectl delete svc rewrite-backend-svc
> ```

---

## P10: NetworkPolicy

**Task:** Create a NetworkPolicy that:
1. Denies all ingress traffic to pods in the namespace
2. Then create a second policy that allows ingress only from pods with label `role: frontend`

**Expected result:**
- After the deny-all policy: no pod can reach the target pods
- After the allow policy: only frontend pods can reach the target

**Verification:**

```bash
# From unauthorized pod - should fail/timeout
kubectl exec unauthorized-pod -- wget --timeout=3 -qO- http://target-svc

# From authorized pod - should succeed
kubectl exec frontend-pod -- wget --timeout=3 -qO- http://target-svc
```

> [!hint]- Hint
> - Minikube default CNI may not enforce NetworkPolicies. Start minikube with Calico: `minikube start --cni=calico`
> - Deny all: empty `ingress: []` in the policy spec with `podSelector: {}` (matches all pods)
> - Allow specific: use `ingress.from.podSelector.matchLabels`
> - NetworkPolicies are additive - if any policy allows traffic, it's allowed

> [!success]- Solution
> ```bash
> # Restart minikube with Calico CNI (if not already using it)
> minikube start --cni=calico
> kubectl create namespace practice
> kubectl config set-context --current --namespace=practice
> ```
> 
> **target-app.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: target-app
>   namespace: practice
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: target
>   template:
>     metadata:
>       labels:
>         app: target
>     spec:
>       containers:
>         - name: app
>           image: hashicorp/http-echo
>           args: ["-text=Secret data", "-listen=:8080"]
>           ports:
>             - containerPort: 8080
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: target-svc
>   namespace: practice
> spec:
>   selector:
>     app: target
>   ports:
>     - port: 80
>       targetPort: 8080
> ```
> 
> **deny-all.yaml:**
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: deny-all-ingress
>   namespace: practice
> spec:
>   podSelector: {}
>   policyTypes:
>     - Ingress
>   ingress: []
> ```
> 
> **allow-frontend.yaml:**
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: NetworkPolicy
> metadata:
>   name: allow-frontend
>   namespace: practice
> spec:
>   podSelector:
>     matchLabels:
>       app: target
>   policyTypes:
>     - Ingress
>   ingress:
>     - from:
>         - podSelector:
>             matchLabels:
>               role: frontend
>       ports:
>         - port: 8080
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f target-app.yaml
> 
> # Test before any policy - should work
> kubectl run test-before --image=busybox:1.36 --rm -it --restart=Never -- wget --timeout=3 -qO- http://target-svc
> # Secret data
> 
> # Apply deny-all policy
> kubectl apply -f deny-all.yaml
> 
> # Test again - should timeout
> kubectl run test-denied --image=busybox:1.36 --rm -it --restart=Never -- wget --timeout=3 -qO- http://target-svc
> # wget: download timed out
> 
> # Apply allow-frontend policy
> kubectl apply -f allow-frontend.yaml
> 
> # Test from pod WITHOUT the frontend label - should fail
> kubectl run unauthorized --image=busybox:1.36 --rm -it --restart=Never -- wget --timeout=3 -qO- http://target-svc
> # wget: download timed out
> 
> # Test from pod WITH the frontend label - should succeed
> kubectl run authorized --image=busybox:1.36 --labels="role=frontend" --rm -it --restart=Never -- wget --timeout=3 -qO- http://target-svc
> # Secret data
> 
> # Cleanup
> kubectl delete networkpolicy deny-all-ingress allow-frontend
> kubectl delete deployment target-app
> kubectl delete svc target-svc
> ```

---

## P11: ExternalName Service

**Task:** Create an ExternalName Service that acts as a DNS alias for an external service (e.g., `example.com`). Verify that resolving the service name returns the external CNAME.

**Expected result:**
- `nslookup external-svc` returns a CNAME pointing to `example.com`
- No ClusterIP is assigned

**Verification:**

```bash
kubectl get svc external-svc
kubectl exec dns-test -- nslookup external-svc.practice.svc.cluster.local
```

> [!hint]- Hint
> - Set `type: ExternalName` in the Service spec
> - Set `externalName: example.com` (the external DNS name)
> - No `selector` or `ports` needed
> - This creates a CNAME DNS record, not an A record

> [!success]- Solution
> **external-svc.yaml:**
> ```yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: external-svc
>   namespace: practice
> spec:
>   type: ExternalName
>   externalName: example.com
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f external-svc.yaml
> 
> # Check the service - no ClusterIP
> kubectl get svc external-svc
> # NAME           TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
> # external-svc   ExternalName   <none>       example.com   <none>    5s
> 
> # Verify DNS resolution
> kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- nslookup external-svc.practice.svc.cluster.local
> # external-svc.practice.svc.cluster.local
> # canonical name = example.com
> 
> # Use case: your app connects to "external-svc" and K8s resolves it to example.com
> # Later you can change to a ClusterIP service without modifying your app
> 
> # Cleanup
> kubectl delete svc external-svc
> ```

---

## P12: Service Mesh Basics (Manual mTLS with Sidecar)

**Task:** Deploy two services with a manual sidecar pattern that simulates service mesh mTLS. Each pod has a main application container and an nginx sidecar that handles TLS termination. This demonstrates the sidecar proxy pattern used by service meshes like Istio.

**Expected result:**
- Traffic between services goes through the nginx sidecar
- The sidecar handles TLS (using self-signed certs for this exercise)
- The application container only speaks plain HTTP

**Verification:**

```bash
kubectl exec client-pod -c app -- wget -qO- http://localhost:8080
# The sidecar proxies the request to the server's sidecar over TLS
```

> [!hint]- Hint
> - Generate self-signed certs: `openssl req -x509 -nodes -days 365 -newkey rsa:2048 ...`
> - Store certs in a Secret, mount into the sidecar container
> - nginx sidecar config: listen on 443 with SSL, proxy_pass to localhost:8080 (the app)
> - Client sidecar: listen on 8080, proxy_pass to the server's TLS endpoint
> - This is what Istio/Linkerd do automatically with Envoy sidecars

> [!success]- Solution
> **Generate self-signed certificates:**
> ```bash
> # Generate CA and server cert
> openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
>   -keyout server.key -out server.crt \
>   -subj "/CN=server-svc.practice.svc.cluster.local"
> 
> # Create Kubernetes Secret for certs
> kubectl create secret tls server-tls \
>   --cert=server.crt --key=server.key \
>   -n practice
> ```
> 
> **server-nginx.conf (ConfigMap):**
> ```bash
> kubectl create configmap server-nginx-conf -n practice --from-literal=nginx.conf='
> events { worker_connections 1024; }
> http {
>     server {
>         listen 443 ssl;
>         ssl_certificate /etc/tls/tls.crt;
>         ssl_certificate_key /etc/tls/tls.key;
> 
>         location / {
>             proxy_pass http://localhost:8080;
>         }
>     }
> }
> '
> ```
> 
> **server-with-sidecar.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: server
>   namespace: practice
> spec:
>   replicas: 1
>   selector:
>     matchLabels:
>       app: server
>   template:
>     metadata:
>       labels:
>         app: server
>     spec:
>       containers:
>         # Main application - plain HTTP
>         - name: app
>           image: hashicorp/http-echo
>           args: ["-text=Secure response via mTLS sidecar", "-listen=:8080"]
>           ports:
>             - containerPort: 8080
> 
>         # TLS sidecar proxy
>         - name: tls-sidecar
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 443
>           volumeMounts:
>             - name: tls-certs
>               mountPath: /etc/tls
>               readOnly: true
>             - name: nginx-conf
>               mountPath: /etc/nginx/nginx.conf
>               subPath: nginx.conf
> 
>       volumes:
>         - name: tls-certs
>           secret:
>             secretName: server-tls
>         - name: nginx-conf
>           configMap:
>             name: server-nginx-conf
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: server-svc
>   namespace: practice
> spec:
>   selector:
>     app: server
>   ports:
>     - name: https
>       port: 443
>       targetPort: 443
>     - name: http
>       port: 80
>       targetPort: 8080
> ```
> 
> **client-pod.yaml:**
> ```yaml
> apiVersion: v1
> kind: Pod
> metadata:
>   name: client-pod
>   namespace: practice
> spec:
>   containers:
>     - name: app
>       image: curlimages/curl
>       command: ["sleep", "3600"]
> ```
> 
> **Commands:**
> ```bash
> kubectl apply -f server-with-sidecar.yaml
> kubectl apply -f client-pod.yaml
> 
> # Wait for pods to be ready
> kubectl get pods -w
> 
> # Test plain HTTP (bypassing sidecar) - works but not encrypted
> kubectl exec client-pod -c app -- curl -s http://server-svc:80
> # Secure response via mTLS sidecar
> 
> # Test through TLS sidecar - encrypted in transit
> kubectl exec client-pod -c app -- curl -sk https://server-svc:443
> # Secure response via mTLS sidecar
> 
> # Verify TLS is actually working
> kubectl exec client-pod -c app -- curl -sv https://server-svc:443 2>&1 | grep "SSL connection"
> # SSL connection using TLSv1.3
> 
> # In a real service mesh (Istio), the sidecar is injected automatically
> # and all inter-service traffic is encrypted without app changes
> 
> # Cleanup
> kubectl delete pod client-pod
> kubectl delete deployment server
> kubectl delete svc server-svc
> kubectl delete secret server-tls
> kubectl delete configmap server-nginx-conf
> rm -f server.key server.crt
> ```

---

## P13: End-to-End Application Stack

**Task:** Deploy a complete application stack:
1. **Frontend** (nginx serving static content)
2. **Backend API** (HTTP echo service)
3. **Database** (PostgreSQL)

Wire them together with ClusterIP Services and expose the frontend via Ingress with path routing:
- `/` goes to frontend
- `/api` goes to backend

The backend should be able to connect to the database. The database should not be exposed externally.

**Expected result:**
- `curl http://<minikube-ip>/` returns the frontend page
- `curl http://<minikube-ip>/api` returns the backend response
- Database is only reachable from within the cluster

**Verification:**

```bash
kubectl get all -n practice
curl http://$(minikube ip)/
curl http://$(minikube ip)/api
```

> [!hint]- Hint
> - Use three Deployments, three ClusterIP Services, one Ingress
> - Frontend and backend on the same Ingress with different paths
> - Database uses a ClusterIP Service with no Ingress rule (internal only)
> - Backend can reach DB via DNS: `postgres-svc.practice.svc.cluster.local`
> - Consider adding health checks and resource limits for production readiness

> [!success]- Solution
> **database.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: postgres
>   namespace: practice
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
>             - name: POSTGRES_PASSWORD
>               value: secret
>             - name: POSTGRES_DB
>               value: appdb
>           resources:
>             requests:
>               cpu: "100m"
>               memory: "128Mi"
>             limits:
>               cpu: "500m"
>               memory: "256Mi"
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: postgres-svc
>   namespace: practice
> spec:
>   selector:
>     app: postgres
>   ports:
>     - port: 5432
>       targetPort: 5432
> ```
> 
> **backend.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: backend
>   namespace: practice
> spec:
>   replicas: 2
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
>           image: hashicorp/http-echo
>           args:
>             - "-text={\"status\":\"ok\",\"service\":\"backend-api\",\"db_host\":\"postgres-svc\"}"
>             - "-listen=:8080"
>           ports:
>             - containerPort: 8080
>           resources:
>             requests:
>               cpu: "50m"
>               memory: "32Mi"
>             limits:
>               cpu: "200m"
>               memory: "64Mi"
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: backend-svc
>   namespace: practice
> spec:
>   selector:
>     app: backend
>   ports:
>     - port: 80
>       targetPort: 8080
> ```
> 
> **frontend.yaml:**
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: frontend
>   namespace: practice
> spec:
>   replicas: 2
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
>           image: nginx:1.25-alpine
>           ports:
>             - containerPort: 80
>           resources:
>             requests:
>               cpu: "50m"
>               memory: "32Mi"
>             limits:
>               cpu: "200m"
>               memory: "64Mi"
> ---
> apiVersion: v1
> kind: Service
> metadata:
>   name: frontend-svc
>   namespace: practice
> spec:
>   selector:
>     app: frontend
>   ports:
>     - port: 80
>       targetPort: 80
> ```
> 
> **app-ingress.yaml:**
> ```yaml
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: app-ingress
>   namespace: practice
>   annotations:
>     nginx.ingress.kubernetes.io/use-regex: "true"
>     nginx.ingress.kubernetes.io/rewrite-target: /$1
> spec:
>   rules:
>     - http:
>         paths:
>           - path: /api(.*)
>             pathType: ImplementationSpecific
>             backend:
>               service:
>                 name: backend-svc
>                 port:
>                   number: 80
>           - path: /(.*)
>             pathType: ImplementationSpecific
>             backend:
>               service:
>                 name: frontend-svc
>                 port:
>                   number: 80
> ```
> 
> **Commands:**
> ```bash
> # Deploy in order: database -> backend -> frontend -> ingress
> kubectl apply -f database.yaml
> kubectl apply -f backend.yaml
> kubectl apply -f frontend.yaml
> kubectl apply -f app-ingress.yaml
> 
> # Verify all resources
> kubectl get all -n practice
> 
> # Wait for ingress to get an address
> kubectl get ingress app-ingress -w
> 
> # Test frontend
> curl http://$(minikube ip)/
> # Should return nginx welcome page
> 
> # Test backend API
> curl http://$(minikube ip)/api
> # {"status":"ok","service":"backend-api","db_host":"postgres-svc"}
> 
> # Verify backend can reach database
> kubectl exec $(kubectl get pods -l app=backend -o jsonpath='{.items[0].metadata.name}') -- \
>   wget --timeout=3 -qO- --spider http://postgres-svc:5432 2>&1 || \
>   echo "Connection attempted (expected non-HTTP response from Postgres)"
> 
> # Verify database is NOT accessible from outside
> # There's no NodePort or Ingress for postgres - it's internal only
> kubectl get svc postgres-svc
> # TYPE should be ClusterIP
> 
> # Full cleanup
> kubectl delete ingress app-ingress
> kubectl delete deployment frontend backend postgres
> kubectl delete svc frontend-svc backend-svc postgres-svc
> ```
