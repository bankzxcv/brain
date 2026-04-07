---
title: "K8s Practice: Helm Charts"
date: 2026-04-07
tags:
  - kubernetes
  - practice
  - helm
parent: "[[Kubernetes Study]]"
---

# Practice - Helm Charts

Progressive hands-on exercises for mastering Helm chart creation, templating, and distribution.

## Prerequisites

- minikube running (`minikube start`)
- Helm 3 installed (`brew install helm` or [official docs](https://helm.sh/docs/intro/install/))
- kubectl configured to use minikube context
- Add the Bitnami repo:
  ```bash
  helm repo add bitnami https://charts.bitnami.com/bitnami
  helm repo update
  ```

---

## P1: Install nginx from Bitnami Helm Chart

**Task:** Install the `bitnami/nginx` chart with the release name `my-nginx`. Set the replica count to 2 and service type to `NodePort`.

**Expected Result:**
- `helm list` shows the `my-nginx` release in `deployed` status
- `kubectl get pods` shows 2 nginx pods running
- `kubectl get svc my-nginx` shows a `NodePort` service

**Verification:**
```bash
helm list
kubectl get pods -l app.kubernetes.io/instance=my-nginx
kubectl get svc my-nginx
```

> [!hint]- Hint
> Use `--set` flags to override default values:
> ```bash
> helm install <release-name> <chart> --set key1=val1 --set key2=val2
> ```
> Check the chart's available values with `helm show values bitnami/nginx` to find the correct keys for replica count and service type.

> [!success]- Solution
> ```bash
> # View available values first
> helm show values bitnami/nginx | grep -A 5 replicaCount
> helm show values bitnami/nginx | grep -A 10 "service:"
>
> # Install with custom values
> helm install my-nginx bitnami/nginx \
>   --set replicaCount=2 \
>   --set service.type=NodePort
>
> # Verify
> helm list
> kubectl get pods -l app.kubernetes.io/instance=my-nginx
> kubectl get svc my-nginx
> ```

---

## P2: Inspect a Release

**Task:** Using the `my-nginx` release from P1, perform the following inspections:
1. List all releases in the current namespace
2. View the user-supplied values for the release
3. View the full computed values (defaults + overrides)
4. View the rendered Kubernetes manifests that were deployed

**Expected Result:** You can see the release status, your custom values (`replicaCount: 2`, `service.type: NodePort`), and the full YAML manifests (Deployment, Service, etc.).

**Verification:**
```bash
helm list
helm get values my-nginx
helm get values my-nginx --all
helm get manifest my-nginx
```

> [!hint]- Hint
> Key `helm get` subcommands:
> - `helm get values <release>` -- only user-supplied overrides
> - `helm get values <release> --all` -- all computed values
> - `helm get manifest <release>` -- rendered K8s YAML
> - `helm get notes <release>` -- post-install notes
> - `helm status <release>` -- release status summary

> [!success]- Solution
> ```bash
> # List all releases
> helm list
>
> # View only user-supplied values
> helm get values my-nginx
> # Output shows: replicaCount: 2, service.type: NodePort
>
> # View all computed values (defaults merged with overrides)
> helm get values my-nginx --all
>
> # View the rendered Kubernetes manifests
> helm get manifest my-nginx
>
> # View release status
> helm status my-nginx
>
> # View release history
> helm history my-nginx
> ```

---

## P3: Upgrade and Rollback a Release

**Task:**
1. Upgrade `my-nginx` to use 3 replicas and add a custom annotation `team=platform` on the pods
2. Verify the upgrade created revision 2
3. Rollback to revision 1
4. Verify the rollback restored 2 replicas

**Expected Result:**
- After upgrade: 3 pods running, revision 2
- After rollback: 2 pods running, revision 3 (rollback creates a new revision)

**Verification:**
```bash
helm history my-nginx
kubectl get pods -l app.kubernetes.io/instance=my-nginx
```

> [!hint]- Hint
> - `helm upgrade <release> <chart> --set key=val` upgrades a release
> - Pod annotations can be set with `--set podAnnotations.team=platform`
> - `helm rollback <release> <revision>` rolls back
> - After rollback, the revision number increments (it's a new revision that restores old config)

> [!success]- Solution
> ```bash
> # Upgrade to 3 replicas with annotation
> helm upgrade my-nginx bitnami/nginx \
>   --set replicaCount=3 \
>   --set service.type=NodePort \
>   --set podAnnotations.team=platform
>
> # Verify upgrade
> helm history my-nginx
> kubectl get pods -l app.kubernetes.io/instance=my-nginx
> kubectl get pods -l app.kubernetes.io/instance=my-nginx -o jsonpath='{.items[0].metadata.annotations}'
>
> # Rollback to revision 1
> helm rollback my-nginx 1
>
> # Verify rollback - should be revision 3 with 2 replicas
> helm history my-nginx
> kubectl get pods -l app.kubernetes.io/instance=my-nginx
>
> # Clean up
> helm uninstall my-nginx
> ```

---

## P4: Create a New Helm Chart

**Task:** Create a new Helm chart called `myapp` using `helm create`. Explore and understand every file in the generated structure.

**Expected Result:** A directory `myapp/` with the standard Helm chart structure. You should be able to explain the purpose of each file.

**Verification:**
```bash
ls -R myapp/
cat myapp/Chart.yaml
cat myapp/values.yaml
```

> [!hint]- Hint
> ```bash
> helm create myapp
> ```
> The generated structure includes:
> - `Chart.yaml` -- chart metadata
> - `values.yaml` -- default values
> - `templates/` -- Kubernetes manifest templates
> - `templates/_helpers.tpl` -- reusable template definitions
> - `templates/NOTES.txt` -- post-install message template
> - `charts/` -- dependencies (subcharts)
> - `.helmignore` -- files to exclude from packaging

> [!success]- Solution
> ```bash
> # Create the chart
> helm create myapp
>
> # Explore the structure
> tree myapp/
> # myapp/
> # ├── .helmignore
> # ├── Chart.yaml          # Chart metadata (name, version, appVersion)
> # ├── values.yaml         # Default configuration values
> # ├── charts/             # Chart dependencies
> # ├── templates/          # Template files
> # │   ├── NOTES.txt       # Post-install notes (templated)
> # │   ├── _helpers.tpl    # Named templates / partials
> # │   ├── deployment.yaml # Deployment template
> # │   ├── hpa.yaml        # HorizontalPodAutoscaler template
> # │   ├── ingress.yaml    # Ingress template
> # │   ├── service.yaml    # Service template
> # │   ├── serviceaccount.yaml # ServiceAccount template
> # │   └── tests/
> # │       └── test-connection.yaml # Helm test
>
> # Examine key files
> cat myapp/Chart.yaml
> cat myapp/values.yaml
> cat myapp/templates/deployment.yaml
> cat myapp/templates/_helpers.tpl
>
> # Render the templates without installing (dry run)
> helm template myapp ./myapp
> ```

---

## P5: Customize values.yaml and Install Your Chart

**Task:** Modify the `myapp` chart's `values.yaml`:
1. Change the image to `hashicorp/http-echo` with tag `latest`
2. Set replica count to 2
3. Set service type to `NodePort`
4. Add container args: `["-text=Hello from Helm"]`

Install the chart and verify it works.

**Expected Result:**
- 2 pods running `hashicorp/http-echo`
- NodePort service accessible
- Curling the service returns "Hello from Helm"

**Verification:**
```bash
kubectl get pods -l app.kubernetes.io/instance=myapp
minikube service myapp --url
curl $(minikube service myapp --url)
```

> [!hint]- Hint
> In `values.yaml`, modify:
> ```yaml
> image:
>   repository: hashicorp/http-echo
>   tag: "latest"
> ```
> For container args, you will need to edit `templates/deployment.yaml` to add an `args` field, or add it through values. The simplest approach is to edit the deployment template directly for now.

> [!success]- Solution
> Edit `myapp/values.yaml`:
> ```yaml
> replicaCount: 2
>
> image:
>   repository: hashicorp/http-echo
>   pullPolicy: IfNotPresent
>   tag: "latest"
>
> service:
>   type: NodePort
>   port: 5678
>
> # Add custom args
> containerArgs:
>   - "-text=Hello from Helm"
> ```
>
> Edit `myapp/templates/deployment.yaml` -- add args to the container spec:
> ```yaml
>           ports:
>             - name: http
>               containerPort: {{ .Values.service.port }}
>               protocol: TCP
>           {{- if .Values.containerArgs }}
>           args:
>             {{- toYaml .Values.containerArgs | nindent 12 }}
>           {{- end }}
> ```
>
> Also update the container port in `deployment.yaml` to use `.Values.service.port` and fix the probes to use the correct port.
>
> Install and verify:
> ```bash
> # Dry run first to check rendered output
> helm template myapp ./myapp
>
> # Install
> helm install myapp ./myapp
>
> # Verify pods
> kubectl get pods -l app.kubernetes.io/instance=myapp
>
> # Test the service
> minikube service myapp --url
> curl $(minikube service myapp --url)
> # Should print: Hello from Helm
> ```

---

## P6: Add a ConfigMap Template

**Task:** Add a ConfigMap template to the `myapp` chart:
1. Create `templates/configmap.yaml`
2. The ConfigMap should contain values from `values.yaml` under a new `config` key
3. Add `APP_ENV`, `LOG_LEVEL`, and `APP_NAME` to the config
4. Mount the ConfigMap as environment variables in the Deployment

**Expected Result:**
- `kubectl get configmap myapp-config` shows the ConfigMap
- Pods have the environment variables set from the ConfigMap

**Verification:**
```bash
kubectl get configmap
kubectl describe configmap myapp-config
kubectl exec <pod-name> -- env | grep APP_
```

> [!hint]- Hint
> ConfigMap template structure:
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: {{ include "myapp.fullname" . }}-config
> data:
>   KEY: {{ .Values.config.key | quote }}
> ```
> In the Deployment, use `envFrom` with `configMapRef` to inject all keys as env vars.

> [!success]- Solution
> Add to `myapp/values.yaml`:
> ```yaml
> config:
>   APP_ENV: "development"
>   LOG_LEVEL: "info"
>   APP_NAME: "myapp"
> ```
>
> Create `myapp/templates/configmap.yaml`:
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: {{ include "myapp.fullname" . }}-config
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
> data:
>   {{- range $key, $value := .Values.config }}
>   {{ $key }}: {{ $value | quote }}
>   {{- end }}
> ```
>
> Edit `myapp/templates/deployment.yaml` -- add `envFrom` to the container spec:
> ```yaml
>           env:
>             {{- toYaml .Values.env | nindent 12 }}
>           envFrom:
>             - configMapRef:
>                 name: {{ include "myapp.fullname" . }}-config
> ```
>
> Upgrade and verify:
> ```bash
> helm upgrade myapp ./myapp
>
> kubectl get configmap
> kubectl describe configmap myapp-config
>
> # Check env vars in pod
> POD=$(kubectl get pod -l app.kubernetes.io/instance=myapp -o jsonpath='{.items[0].metadata.name}')
> kubectl exec $POD -- env | grep -E "APP_|LOG_"
> ```

---

## P7: Use Template Functions

**Task:** Practice using Helm template functions in your chart:
1. Use `default` to provide a fallback for `.Values.config.APP_ENV` if not set
2. Use `quote` to ensure string values are properly quoted in YAML
3. Use `toYaml` with `nindent` to render a block of labels
4. Use `upper` to transform the APP_ENV value to uppercase
5. Use `required` to make `image.repository` mandatory

**Expected Result:** `helm template myapp ./myapp` renders correctly with all functions applied. Missing required values cause a clear error.

**Verification:**
```bash
helm template myapp ./myapp
helm template myapp ./myapp --set image.repository=""  # Should fail
```

> [!hint]- Hint
> Common template functions:
> ```yaml
> {{ .Values.key | default "fallback" }}
> {{ .Values.key | quote }}
> {{ .Values.block | toYaml | nindent 8 }}
> {{ .Values.key | upper }}
> {{ required "image.repository is required" .Values.image.repository }}
> ```
> The `nindent` function adds a newline then indents by N spaces -- crucial for YAML formatting.

> [!success]- Solution
> Edit `myapp/templates/configmap.yaml`:
> ```yaml
> apiVersion: v1
> kind: ConfigMap
> metadata:
>   name: {{ include "myapp.fullname" . }}-config
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
> data:
>   APP_ENV: {{ .Values.config.APP_ENV | default "production" | upper | quote }}
>   LOG_LEVEL: {{ .Values.config.LOG_LEVEL | default "warn" | quote }}
>   APP_NAME: {{ .Values.config.APP_NAME | default (include "myapp.fullname" .) | quote }}
> ```
>
> Edit `myapp/templates/deployment.yaml` -- use `required` for the image:
> ```yaml
>           image: "{{ required "image.repository is required" .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
> ```
>
> Add custom labels using `toYaml` + `nindent` in `deployment.yaml`:
> ```yaml
>   template:
>     metadata:
>       labels:
>         {{- include "myapp.selectorLabels" . | nindent 8 }}
>         {{- if .Values.podLabels }}
>         {{- toYaml .Values.podLabels | nindent 8 }}
>         {{- end }}
> ```
>
> Add to `values.yaml`:
> ```yaml
> podLabels:
>   team: platform
>   tier: frontend
> ```
>
> Verify:
> ```bash
> # Render templates -- should succeed
> helm template myapp ./myapp
>
> # Test required -- should fail with clear error
> helm template myapp ./myapp --set image.repository=""
> # Error: execution error at (myapp/templates/deployment.yaml:XX):
> # image.repository is required
>
> # Check the rendered ConfigMap -- APP_ENV should be "DEVELOPMENT"
> helm template myapp ./myapp | grep -A 5 "kind: ConfigMap"
> ```

---

## P8: Create a Named Template in _helpers.tpl

**Task:**
1. Create a named template called `myapp.commonAnnotations` in `_helpers.tpl` that generates a standard set of annotations (managed-by, chart version, team)
2. Create a named template called `myapp.resourceLabels` that combines selector labels with custom resource-specific labels
3. Use both templates with `include` in `deployment.yaml` and `service.yaml`

**Expected Result:** Both Deployment and Service have consistent annotations and labels generated from the shared templates.

**Verification:**
```bash
helm template myapp ./myapp | grep -A 10 "annotations:"
```

> [!hint]- Hint
> Named template definition:
> ```yaml
> {{- define "myapp.commonAnnotations" -}}
> managed-by: helm
> chart: {{ .Chart.Name }}-{{ .Chart.Version }}
> {{- end }}
> ```
> Use it with:
> ```yaml
>   annotations:
>     {{- include "myapp.commonAnnotations" . | nindent 4 }}
> ```
> Note: `include` (not `template`) lets you pipe the output through functions like `nindent`.

> [!success]- Solution
> Add to `myapp/templates/_helpers.tpl`:
> ```yaml
> {{/*
> Common annotations for all resources
> */}}
> {{- define "myapp.commonAnnotations" -}}
> app.kubernetes.io/managed-by: {{ .Release.Service }}
> helm.sh/chart: {{ include "myapp.chart" . }}
> app.kubernetes.io/part-of: {{ .Release.Name }}
> {{- if .Values.commonAnnotations }}
> {{- toYaml .Values.commonAnnotations | nindent 0 }}
> {{- end }}
> {{- end }}
>
> {{/*
> Resource labels combining selector labels with custom per-resource labels
> */}}
> {{- define "myapp.resourceLabels" -}}
> {{- include "myapp.selectorLabels" . }}
> app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
> app.kubernetes.io/managed-by: {{ .Release.Service }}
> {{- if .Values.podLabels }}
> {{- toYaml .Values.podLabels | nindent 0 }}
> {{- end }}
> {{- end }}
> ```
>
> Add to `values.yaml`:
> ```yaml
> commonAnnotations:
>   team: platform
>   contact: platform@company.com
> ```
>
> Edit `myapp/templates/deployment.yaml`:
> ```yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: {{ include "myapp.fullname" . }}
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
>   annotations:
>     {{- include "myapp.commonAnnotations" . | nindent 4 }}
> spec:
>   replicas: {{ .Values.replicaCount }}
>   selector:
>     matchLabels:
>       {{- include "myapp.selectorLabels" . | nindent 6 }}
>   template:
>     metadata:
>       labels:
>         {{- include "myapp.resourceLabels" . | nindent 8 }}
>       annotations:
>         {{- include "myapp.commonAnnotations" . | nindent 8 }}
> ```
>
> Edit `myapp/templates/service.yaml` similarly:
> ```yaml
> metadata:
>   name: {{ include "myapp.fullname" . }}
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
>   annotations:
>     {{- include "myapp.commonAnnotations" . | nindent 4 }}
> ```
>
> Verify:
> ```bash
> helm template myapp ./myapp | grep -A 10 "annotations:"
> helm upgrade myapp ./myapp
> kubectl get deploy myapp -o jsonpath='{.metadata.annotations}' | jq .
> ```

---

## P9: Add Conditional Ingress

**Task:** Configure the Ingress template so it is only created when `.Values.ingress.enabled` is `true`:
1. Set `ingress.enabled: false` by default in `values.yaml`
2. Verify that `helm template` does NOT render an Ingress resource
3. Override with `--set ingress.enabled=true` and verify the Ingress IS rendered
4. Configure the Ingress with a hostname and path from values

**Expected Result:**
- Default install: no Ingress resource created
- With `ingress.enabled=true`: Ingress resource rendered with correct host and path

**Verification:**
```bash
helm template myapp ./myapp | grep "kind: Ingress"                       # No output
helm template myapp ./myapp --set ingress.enabled=true | grep "kind: Ingress"  # Found
```

> [!hint]- Hint
> The `helm create` scaffold already includes conditional Ingress logic. Look at the generated `templates/ingress.yaml` -- it wraps everything in:
> ```yaml
> {{- if .Values.ingress.enabled -}}
> ...
> {{- end }}
> ```
> Make sure `values.yaml` has the right structure under `ingress:`.

> [!success]- Solution
> Edit `myapp/values.yaml`:
> ```yaml
> ingress:
>   enabled: false
>   className: "nginx"
>   annotations:
>     nginx.ingress.kubernetes.io/rewrite-target: /
>   hosts:
>     - host: myapp.local
>       paths:
>         - path: /
>           pathType: Prefix
>   tls: []
> ```
>
> The default `myapp/templates/ingress.yaml` from `helm create` already has the conditional:
> ```yaml
> {{- if .Values.ingress.enabled -}}
> apiVersion: networking.k8s.io/v1
> kind: Ingress
> metadata:
>   name: {{ include "myapp.fullname" . }}
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
>   {{- with .Values.ingress.annotations }}
>   annotations:
>     {{- toYaml . | nindent 4 }}
>   {{- end }}
> spec:
>   {{- if .Values.ingress.className }}
>   ingressClassName: {{ .Values.ingress.className }}
>   {{- end }}
>   {{- if .Values.ingress.tls }}
>   tls:
>     {{- toYaml .Values.ingress.tls | nindent 4 }}
>   {{- end }}
>   rules:
>     {{- range .Values.ingress.hosts }}
>     - host: {{ .host | quote }}
>       http:
>         paths:
>           {{- range .paths }}
>           - path: {{ .path }}
>             pathType: {{ .pathType }}
>             backend:
>               service:
>                 name: {{ include "myapp.fullname" $ }}
>                 port:
>                   number: {{ $.Values.service.port }}
>           {{- end }}
>     {{- end }}
> {{- end }}
> ```
>
> Verify:
> ```bash
> # Default -- no Ingress
> helm template myapp ./myapp | grep "kind: Ingress"
> # (no output)
>
> # Enabled -- Ingress present
> helm template myapp ./myapp --set ingress.enabled=true | grep -A 30 "kind: Ingress"
>
> # Full render with custom host
> helm template myapp ./myapp \
>   --set ingress.enabled=true \
>   --set "ingress.hosts[0].host=myapp.example.com" \
>   --set "ingress.hosts[0].paths[0].path=/" \
>   --set "ingress.hosts[0].paths[0].pathType=Prefix"
> ```

---

## P10: Use Range to Loop Over Environment Variables

**Task:**
1. Define a list of environment variables in `values.yaml` under `env:`
2. Use `range` in `deployment.yaml` to iterate and render each as an `env` entry in the container spec
3. Include at least 3 environment variables with different value sources (literal value, fieldRef, secretKeyRef)

**Expected Result:** The rendered Deployment has all environment variables properly set on the container.

**Verification:**
```bash
helm template myapp ./myapp | grep -A 20 "env:"
```

> [!hint]- Hint
> In `values.yaml`:
> ```yaml
> env:
>   - name: APP_PORT
>     value: "5678"
>   - name: POD_NAME
>     valueFrom:
>       fieldRef:
>         fieldPath: metadata.name
> ```
> In the template, use:
> ```yaml
> env:
>   {{- range .Values.env }}
>   - name: {{ .name }}
>     ...
>   {{- end }}
> ```
> The simplest approach is `toYaml` on the entire env list.

> [!success]- Solution
> Edit `myapp/values.yaml`:
> ```yaml
> env:
>   - name: APP_PORT
>     value: "5678"
>   - name: APP_ENV
>     value: "development"
>   - name: LOG_FORMAT
>     value: "json"
>   - name: POD_NAME
>     valueFrom:
>       fieldRef:
>         fieldPath: metadata.name
>   - name: POD_IP
>     valueFrom:
>       fieldRef:
>         fieldPath: status.podIP
>   - name: NODE_NAME
>     valueFrom:
>       fieldRef:
>         fieldPath: spec.nodeName
> ```
>
> Edit `myapp/templates/deployment.yaml` container spec:
> ```yaml
>       containers:
>         - name: {{ .Chart.Name }}
>           image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
>           imagePullPolicy: {{ .Values.image.pullPolicy }}
>           {{- if .Values.containerArgs }}
>           args:
>             {{- toYaml .Values.containerArgs | nindent 12 }}
>           {{- end }}
>           env:
>             {{- range .Values.env }}
>             - name: {{ .name }}
>               {{- if .value }}
>               value: {{ .value | quote }}
>               {{- else if .valueFrom }}
>               valueFrom:
>                 {{- toYaml .valueFrom | nindent 16 }}
>               {{- end }}
>             {{- end }}
>           envFrom:
>             - configMapRef:
>                 name: {{ include "myapp.fullname" . }}-config
> ```
>
> Alternative simpler approach using `toYaml` on the whole list:
> ```yaml
>           {{- if .Values.env }}
>           env:
>             {{- toYaml .Values.env | nindent 12 }}
>           {{- end }}
> ```
>
> Verify:
> ```bash
> helm template myapp ./myapp | grep -A 30 "env:"
>
> helm upgrade myapp ./myapp
>
> POD=$(kubectl get pod -l app.kubernetes.io/instance=myapp -o jsonpath='{.items[0].metadata.name}')
> kubectl exec $POD -- env | sort
> ```

---

## P11: Add a Chart Dependency (PostgreSQL Subchart)

**Task:**
1. Add the Bitnami PostgreSQL chart as a dependency to `myapp`
2. Configure the PostgreSQL subchart via `myapp/values.yaml` (set password, database name)
3. Run `helm dependency update` and install
4. Verify both myapp and PostgreSQL pods are running

**Expected Result:**
- `myapp/charts/` contains the PostgreSQL chart archive
- Both myapp pods and a PostgreSQL pod are running
- The PostgreSQL instance has the configured database

**Verification:**
```bash
ls myapp/charts/
kubectl get pods
kubectl exec <postgres-pod> -- psql -U myuser -d myappdb -c '\l'
```

> [!hint]- Hint
> In `Chart.yaml`, add:
> ```yaml
> dependencies:
>   - name: postgresql
>     version: "~15.0"
>     repository: "https://charts.bitnami.com/bitnami"
> ```
> Then run `helm dependency update ./myapp`.
> Configure the subchart in `values.yaml` under the dependency name:
> ```yaml
> postgresql:
>   auth:
>     username: myuser
>     password: mypassword
>     database: myappdb
> ```

> [!success]- Solution
> Edit `myapp/Chart.yaml` -- add dependencies:
> ```yaml
> apiVersion: v2
> name: myapp
> description: A Helm chart for Kubernetes
> type: application
> version: 0.2.0
> appVersion: "1.0.0"
>
> dependencies:
>   - name: postgresql
>     version: "~15.0"
>     repository: "https://charts.bitnami.com/bitnami"
>     condition: postgresql.enabled
> ```
>
> Edit `myapp/values.yaml` -- add PostgreSQL config:
> ```yaml
> postgresql:
>   enabled: true
>   auth:
>     username: myuser
>     password: mypassword
>     database: myappdb
>     postgresPassword: adminpassword
>   primary:
>     persistence:
>       enabled: false  # Disable persistence for minikube
> ```
>
> Build dependencies and install:
> ```bash
> # Update dependencies -- downloads postgresql chart to charts/
> helm dependency update ./myapp
>
> # Verify the dependency was downloaded
> ls myapp/charts/
> # postgresql-15.x.x.tgz
>
> # Uninstall previous release first
> helm uninstall myapp
>
> # Install with dependencies
> helm install myapp ./myapp
>
> # Verify pods
> kubectl get pods
> # NAME                     READY   STATUS    RESTARTS   AGE
> # myapp-xxxxx              1/1     Running   0          1m
> # myapp-postgresql-0       1/1     Running   0          1m
>
> # Test PostgreSQL connection
> kubectl exec myapp-postgresql-0 -- \
>   env PGPASSWORD=mypassword psql -U myuser -d myappdb -c '\l'
> ```

---

## P12: Multiple Value Files for Different Environments

**Task:**
1. Create `values-dev.yaml` with development settings (1 replica, debug logging, no persistence)
2. Create `values-prod.yaml` with production settings (3 replicas, warn logging, persistence enabled)
3. Install the same chart twice with different release names and value files
4. Verify each environment has the expected configuration

**Expected Result:**
- `myapp-dev` release with 1 replica and debug logging
- `myapp-prod` release with 3 replicas and warn logging

**Verification:**
```bash
helm list
kubectl get pods -l app.kubernetes.io/instance=myapp-dev
kubectl get pods -l app.kubernetes.io/instance=myapp-prod
```

> [!hint]- Hint
> Use `-f` flag to specify value files:
> ```bash
> helm install myapp-dev ./myapp -f values-dev.yaml
> helm install myapp-prod ./myapp -f values-prod.yaml
> ```
> You can also combine multiple value files -- later files override earlier ones:
> ```bash
> helm install myapp-dev ./myapp -f values.yaml -f values-dev.yaml
> ```

> [!success]- Solution
> Create `myapp/values-dev.yaml`:
> ```yaml
> replicaCount: 1
>
> config:
>   APP_ENV: "development"
>   LOG_LEVEL: "debug"
>   APP_NAME: "myapp-dev"
>
> env:
>   - name: APP_PORT
>     value: "5678"
>   - name: APP_ENV
>     value: "development"
>
> postgresql:
>   enabled: true
>   auth:
>     username: devuser
>     password: devpassword
>     database: myappdb_dev
>     postgresPassword: devadmin
>   primary:
>     persistence:
>       enabled: false
>
> resources:
>   limits:
>     cpu: 200m
>     memory: 256Mi
>   requests:
>     cpu: 100m
>     memory: 128Mi
> ```
>
> Create `myapp/values-prod.yaml`:
> ```yaml
> replicaCount: 3
>
> config:
>   APP_ENV: "production"
>   LOG_LEVEL: "warn"
>   APP_NAME: "myapp-prod"
>
> env:
>   - name: APP_PORT
>     value: "5678"
>   - name: APP_ENV
>     value: "production"
>
> postgresql:
>   enabled: true
>   auth:
>     username: produser
>     password: prodpassword123
>     database: myappdb_prod
>     postgresPassword: prodadmin123
>   primary:
>     persistence:
>       enabled: false  # Set true in real prod
>
> resources:
>   limits:
>     cpu: 500m
>     memory: 512Mi
>   requests:
>     cpu: 250m
>     memory: 256Mi
> ```
>
> Install both:
> ```bash
> # Clean up any existing release
> helm uninstall myapp 2>/dev/null
>
> # Install dev environment
> helm install myapp-dev ./myapp -f myapp/values-dev.yaml
>
> # Install prod environment (in a different namespace for isolation)
> kubectl create namespace prod
> helm install myapp-prod ./myapp -f myapp/values-prod.yaml -n prod
>
> # Verify dev
> helm list
> kubectl get pods -l app.kubernetes.io/instance=myapp-dev
>
> # Verify prod
> helm list -n prod
> kubectl get pods -n prod -l app.kubernetes.io/instance=myapp-prod
>
> # Check config values differ
> kubectl get configmap myapp-dev-config -o yaml
> kubectl get configmap myapp-prod-config -n prod -o yaml
> ```

---

## P13: Add Helm Hooks

**Task:**
1. Create a PreInstall hook Job that simulates a database migration
2. Create a PostInstall hook Job that runs a smoke test (curl the service endpoint)
3. Set appropriate hook weights so migration runs before the smoke test
4. Add `hook-delete-policy` to clean up completed hook Jobs

**Expected Result:**
- On `helm install`, the migration Job runs first, then the app deploys, then the smoke test Job runs
- Hook Jobs are cleaned up after success

**Verification:**
```bash
helm install myapp-hooks ./myapp
kubectl get jobs
kubectl logs job/myapp-migrate
kubectl logs job/myapp-smoketest
```

> [!hint]- Hint
> Hook annotations:
> ```yaml
> annotations:
>   "helm.sh/hook": pre-install
>   "helm.sh/hook-weight": "-5"
>   "helm.sh/hook-delete-policy": hook-succeeded
> ```
> Hook types: `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`
> Hook weights determine order (lower runs first).

> [!success]- Solution
> Create `myapp/templates/pre-install-migration.yaml`:
> ```yaml
> apiVersion: batch/v1
> kind: Job
> metadata:
>   name: {{ include "myapp.fullname" . }}-migrate
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
>   annotations:
>     "helm.sh/hook": pre-install,pre-upgrade
>     "helm.sh/hook-weight": "-5"
>     "helm.sh/hook-delete-policy": hook-succeeded,before-hook-creation
> spec:
>   template:
>     metadata:
>       name: {{ include "myapp.fullname" . }}-migrate
>     spec:
>       restartPolicy: Never
>       containers:
>         - name: migrate
>           image: busybox:1.36
>           command:
>             - /bin/sh
>             - -c
>             - |
>               echo "Starting database migration..."
>               echo "Running migration 001_create_users_table..."
>               sleep 2
>               echo "Running migration 002_add_email_column..."
>               sleep 1
>               echo "All migrations completed successfully."
>   backoffLimit: 1
> ```
>
> Create `myapp/templates/post-install-smoketest.yaml`:
> ```yaml
> apiVersion: batch/v1
> kind: Job
> metadata:
>   name: {{ include "myapp.fullname" . }}-smoketest
>   labels:
>     {{- include "myapp.labels" . | nindent 4 }}
>   annotations:
>     "helm.sh/hook": post-install,post-upgrade
>     "helm.sh/hook-weight": "5"
>     "helm.sh/hook-delete-policy": hook-succeeded,before-hook-creation
> spec:
>   template:
>     metadata:
>       name: {{ include "myapp.fullname" . }}-smoketest
>     spec:
>       restartPolicy: Never
>       containers:
>         - name: smoketest
>           image: curlimages/curl:8.5.0
>           command:
>             - /bin/sh
>             - -c
>             - |
>               echo "Running smoke test..."
>               echo "Waiting for service to be ready..."
>               sleep 10
>               echo "Testing HTTP endpoint..."
>               HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/ || echo "000")
>               echo "HTTP Status: $HTTP_CODE"
>               if [ "$HTTP_CODE" = "200" ]; then
>                 echo "Smoke test PASSED"
>                 exit 0
>               else
>                 echo "Smoke test FAILED (got $HTTP_CODE)"
>                 exit 1
>               fi
>   backoffLimit: 2
> ```
>
> Install and verify:
> ```bash
> # Clean up previous releases
> helm uninstall myapp-dev 2>/dev/null
> helm uninstall myapp-prod -n prod 2>/dev/null
>
> # Install and watch hooks execute
> helm install myapp-hooks ./myapp --debug
>
> # Watch jobs
> kubectl get jobs -w
>
> # Check migration logs
> kubectl logs job/myapp-hooks-migrate
>
> # Check smoke test logs
> kubectl logs job/myapp-hooks-smoketest
>
> # Verify hook Jobs were cleaned up (since delete-policy is hook-succeeded)
> kubectl get jobs
>
> # Clean up
> helm uninstall myapp-hooks
> ```

---

## P14: Package and Serve a Local Helm Repo

**Task:**
1. Package the `myapp` chart into a `.tgz` archive
2. Create an `index.yaml` for the chart repository
3. Serve the chart as a local Helm repository using a simple HTTP server
4. Add the local repo to Helm and install the chart from it

**Expected Result:**
- `myapp-0.2.0.tgz` exists
- A local HTTP server serves the chart repository
- `helm search repo localrepo` shows your chart
- You can install `localrepo/myapp` successfully

**Verification:**
```bash
helm search repo localrepo
helm install from-local-repo localrepo/myapp
helm list
```

> [!hint]- Hint
> ```bash
> helm package ./myapp             # Creates .tgz
> helm repo index .                # Creates index.yaml
> python3 -m http.server 8080 &   # Serve files
> helm repo add localrepo http://localhost:8080
> ```
> Make sure `Chart.yaml` has the correct `version` field before packaging.

> [!success]- Solution
> ```bash
> # Step 1: Package the chart
> mkdir -p ~/helm-repo
> helm package ./myapp -d ~/helm-repo
> # Creates ~/helm-repo/myapp-0.2.0.tgz
>
> # Step 2: Generate the repository index
> helm repo index ~/helm-repo --url http://localhost:8080
> # Creates ~/helm-repo/index.yaml
>
> # Inspect the index
> cat ~/helm-repo/index.yaml
>
> # Step 3: Start a local HTTP server
> cd ~/helm-repo && python3 -m http.server 8080 &
> cd -
>
> # Step 4: Add the local repo to Helm
> helm repo add localrepo http://localhost:8080
> helm repo update
>
> # Search for your chart
> helm search repo localrepo
> # NAME              CHART VERSION   APP VERSION   DESCRIPTION
> # localrepo/myapp   0.2.0           1.0.0         A Helm chart for Kubernetes
>
> # Step 5: Install from the local repo
> helm uninstall myapp-hooks 2>/dev/null
> helm install from-local localrepo/myapp
>
> # Verify
> helm list
> kubectl get pods -l app.kubernetes.io/instance=from-local
>
> # Clean up
> helm uninstall from-local
> helm repo remove localrepo
> kill %1  # Stop the HTTP server
> rm -rf ~/helm-repo
> ```
