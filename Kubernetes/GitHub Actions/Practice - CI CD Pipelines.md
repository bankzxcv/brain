---
title: "K8s Practice: CI/CD Pipelines with GitHub Actions"
date: 2026-04-07
tags:
  - kubernetes
  - practice
  - github-actions
  - cicd
parent: "[[Kubernetes Study]]"
---

# Practice - CI/CD Pipelines with GitHub Actions

Progressive hands-on exercises for building CI/CD pipelines with GitHub Actions, from basic workflows to a full GitOps deployment pipeline.

## Prerequisites

- A GitHub account with a repository for this practice (e.g., `cicd-practice`)
- A simple Go application in the repo (see setup below)
- Docker installed locally for testing
- Basic familiarity with YAML and Git

## Setting Up the Practice Repo

```bash
mkdir cicd-practice && cd cicd-practice
git init
```

Create a minimal Go application:

**go.mod:**
```go
module github.com/<your-username>/cicd-practice

go 1.22
```

**main.go:**
```go
package main

import (
	"fmt"
	"net/http"
	"os"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	http.HandleFunc("/", handler)
	http.HandleFunc("/health", healthHandler)

	fmt.Printf("Server starting on port %s\n", port)
	http.ListenAndServe(":"+port, nil)
}

func handler(w http.ResponseWriter, r *http.Request) {
	version := os.Getenv("APP_VERSION")
	if version == "" {
		version = "dev"
	}
	fmt.Fprintf(w, "Hello from CI/CD Practice v%s\n", version)
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprintln(w, "ok")
}
```

**main_test.go:**
```go
package main

import (
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestHandler(t *testing.T) {
	req := httptest.NewRequest("GET", "/", nil)
	w := httptest.NewRecorder()
	handler(w, req)

	if w.Code != http.StatusOK {
		t.Errorf("expected status 200, got %d", w.Code)
	}
}

func TestHealthHandler(t *testing.T) {
	req := httptest.NewRequest("GET", "/health", nil)
	w := httptest.NewRecorder()
	healthHandler(w, req)

	if w.Code != http.StatusOK {
		t.Errorf("expected status 200, got %d", w.Code)
	}
}
```

**Dockerfile:**
```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o server .

FROM alpine:3.19
RUN apk --no-cache add ca-certificates
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

```bash
git add . && git commit -m "Initial Go app with tests and Dockerfile"
gh repo create cicd-practice --public --push --source .
```

---

## P1: Basic Workflow - Hello CI

**Task:** Create a GitHub Actions workflow that:
1. Triggers on push to any branch
2. Runs on `ubuntu-latest`
3. Checks out the code
4. Prints "Hello CI" and the commit SHA

**Expected Result:**
- Pushing to the repo triggers the workflow
- The Actions tab shows a successful run with the "Hello CI" message

**Verification:** Check the Actions tab in your GitHub repository.

> [!hint]- Hint
> Workflow files live in `.github/workflows/`. Key structure:
> ```yaml
> name: CI
> on: [push]
> jobs:
>   hello:
>     runs-on: ubuntu-latest
>     steps:
>       - uses: actions/checkout@v4
>       - run: echo "Hello"
> ```
> Use `${{ github.sha }}` to access the commit SHA.

> [!success]- Solution
> Create `.github/workflows/ci.yaml`:
> ```yaml
> name: CI
>
> on:
>   push:
>     branches: ['*']
>
> jobs:
>   hello:
>     name: Hello CI
>     runs-on: ubuntu-latest
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Hello CI
>         run: |
>           echo "Hello CI!"
>           echo "Repository: ${{ github.repository }}"
>           echo "Commit SHA: ${{ github.sha }}"
>           echo "Branch: ${{ github.ref_name }}"
>           echo "Actor: ${{ github.actor }}"
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Add basic CI workflow"
> git push
>
> # Check the Actions tab on GitHub
> gh run list --limit 1
> gh run view --log
> ```

---

## P2: Add Go Test Steps

**Task:** Extend the workflow to:
1. Set up Go 1.22 using `actions/setup-go`
2. Run `go test ./...` with verbose output
3. Run `go vet ./...` for static analysis
4. Run `go build` to verify the code compiles

**Expected Result:**
- All tests pass
- `go vet` reports no issues
- Build succeeds

**Verification:** The workflow run shows green checks for all steps.

> [!hint]- Hint
> Use the `actions/setup-go` action:
> ```yaml
> - uses: actions/setup-go@v5
>   with:
>     go-version: '1.22'
> ```
> Then run standard Go commands in subsequent steps.

> [!success]- Solution
> Update `.github/workflows/ci.yaml`:
> ```yaml
> name: CI
>
> on:
>   push:
>     branches: ['*']
>   pull_request:
>     branches: [main]
>
> jobs:
>   test:
>     name: Test
>     runs-on: ubuntu-latest
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Go
>         uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>
>       - name: Verify Go installation
>         run: go version
>
>       - name: Download dependencies
>         run: go mod download
>
>       - name: Run go vet
>         run: go vet ./...
>
>       - name: Run tests
>         run: go test -v -race -coverprofile=coverage.out ./...
>
>       - name: Display coverage
>         run: go tool cover -func=coverage.out
>
>       - name: Build
>         run: go build -o server .
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Add Go test steps to CI"
> git push
>
> gh run watch
> ```

---

## P3: Add Go Module Caching

**Task:** Add caching for Go modules to speed up builds:
1. Use the built-in caching in `actions/setup-go` (it caches by default)
2. Alternatively, add explicit caching with `actions/cache`
3. Compare run times with and without cache

**Expected Result:**
- First run downloads dependencies (cache miss)
- Second run uses the cache (faster)
- Run logs show "cache hit" messages

**Verification:** Check the "Set up Go" step logs for cache hit/miss messages.

> [!hint]- Hint
> `actions/setup-go@v5` has built-in caching enabled by default. It caches the Go module cache (`~/go/pkg/mod`) and the build cache (`~/.cache/go-build`).
>
> For explicit control, you can use `actions/cache`:
> ```yaml
> - uses: actions/cache@v4
>   with:
>     path: |
>       ~/go/pkg/mod
>       ~/.cache/go-build
>     key: go-${{ runner.os }}-${{ hashFiles('**/go.sum') }}
> ```

> [!success]- Solution
> Update `.github/workflows/ci.yaml`:
> ```yaml
> name: CI
>
> on:
>   push:
>     branches: ['*']
>   pull_request:
>     branches: [main]
>
> jobs:
>   test:
>     name: Test
>     runs-on: ubuntu-latest
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Go (with built-in caching)
>         uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>           cache: true  # This is actually the default
>
>       - name: Download dependencies
>         run: go mod download
>
>       - name: Run go vet
>         run: go vet ./...
>
>       - name: Run tests
>         run: go test -v -race -coverprofile=coverage.out ./...
>
>       - name: Display coverage
>         run: go tool cover -func=coverage.out
>
>       - name: Build
>         run: go build -o server .
> ```
>
> **Alternative with explicit cache control:**
> ```yaml
>       - name: Set up Go
>         uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>           cache: false  # Disable built-in cache
>
>       - name: Cache Go modules
>         uses: actions/cache@v4
>         with:
>           path: |
>             ~/go/pkg/mod
>             ~/.cache/go-build
>           key: go-${{ runner.os }}-${{ hashFiles('**/go.sum') }}
>           restore-keys: |
>             go-${{ runner.os }}-
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Add Go module caching"
> git push
>
> # First run: cache miss
> gh run watch
>
> # Push a trivial change to trigger second run
> echo "" >> main.go
> git add . && git commit -m "Trigger cached build" && git push
>
> # Second run: cache hit (check logs)
> gh run watch
> ```

---

## P4: Matrix Build

**Task:** Configure a matrix strategy to test across:
1. Go versions: 1.21 and 1.22
2. Operating systems: ubuntu-latest and macos-latest
3. This should produce 4 parallel jobs (2 versions x 2 OS)

**Expected Result:**
- 4 jobs run in parallel
- All 4 jobs pass
- The job names include the Go version and OS

**Verification:** The Actions tab shows 4 parallel jobs in the workflow run.

> [!hint]- Hint
> Use the `strategy.matrix` key:
> ```yaml
> strategy:
>   matrix:
>     go-version: ['1.21', '1.22']
>     os: [ubuntu-latest, macos-latest]
> runs-on: ${{ matrix.os }}
> ```
> Access matrix values with `${{ matrix.go-version }}`.

> [!success]- Solution
> Update `.github/workflows/ci.yaml`:
> ```yaml
> name: CI
>
> on:
>   push:
>     branches: ['*']
>   pull_request:
>     branches: [main]
>
> jobs:
>   test:
>     name: Test (Go ${{ matrix.go-version }}, ${{ matrix.os }})
>     runs-on: ${{ matrix.os }}
>
>     strategy:
>       matrix:
>         go-version: ['1.21', '1.22']
>         os: [ubuntu-latest, macos-latest]
>       fail-fast: false  # Don't cancel other jobs if one fails
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Go ${{ matrix.go-version }}
>         uses: actions/setup-go@v5
>         with:
>           go-version: ${{ matrix.go-version }}
>
>       - name: Download dependencies
>         run: go mod download
>
>       - name: Run go vet
>         run: go vet ./...
>
>       - name: Run tests
>         run: go test -v -race -coverprofile=coverage.out ./...
>
>       - name: Build
>         run: go build -o server .
>
>       - name: Print build info
>         run: |
>           echo "Go version: $(go version)"
>           echo "OS: ${{ matrix.os }}"
>           echo "Build successful!"
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Add matrix build for Go versions and OS"
> git push
>
> # Watch all 4 jobs
> gh run watch
> ```

---

## P5: Build a Docker Image

**Task:** Add a step to build a Docker image:
1. Tag the image with the commit SHA (`ghcr.io/<user>/cicd-practice:<sha>`)
2. Also tag with `latest` if on the `main` branch
3. Use `docker/build-push-action` for efficient builds with BuildKit

**Expected Result:**
- Docker image builds successfully
- Image is tagged with the commit SHA
- Build step shows in the workflow run

**Verification:** The workflow run shows a successful Docker build step.

> [!hint]- Hint
> Use these actions:
> ```yaml
> - uses: docker/setup-buildx-action@v3
> - uses: docker/build-push-action@v6
>   with:
>     context: .
>     push: false
>     tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
> ```
> Set `push: false` for now -- we will push in the next exercise.

> [!success]- Solution
> Update `.github/workflows/ci.yaml` -- simplify matrix back to single config and add Docker build:
> ```yaml
> name: CI
>
> on:
>   push:
>     branches: ['*']
>   pull_request:
>     branches: [main]
>
> jobs:
>   test:
>     name: Test
>     runs-on: ubuntu-latest
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Go
>         uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>
>       - name: Run tests
>         run: go test -v -race ./...
>
>       - name: Build binary
>         run: go build -o server .
>
>   build:
>     name: Build Docker Image
>     runs-on: ubuntu-latest
>     needs: test
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Docker Buildx
>         uses: docker/setup-buildx-action@v3
>
>       - name: Docker metadata
>         id: meta
>         uses: docker/metadata-action@v5
>         with:
>           images: ghcr.io/${{ github.repository }}
>           tags: |
>             type=sha,prefix=
>             type=raw,value=latest,enable={{is_default_branch}}
>
>       - name: Build Docker image
>         uses: docker/build-push-action@v6
>         with:
>           context: .
>           push: false
>           tags: ${{ steps.meta.outputs.tags }}
>           labels: ${{ steps.meta.outputs.labels }}
>           cache-from: type=gha
>           cache-to: type=gha,mode=max
>
>       - name: Print image tags
>         run: |
>           echo "Image tags:"
>           echo "${{ steps.meta.outputs.tags }}"
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Add Docker image build step"
> git push
>
> gh run watch
> ```

---

## P6: Push Docker Image to GitHub Container Registry

**Task:** Push the Docker image to GitHub Container Registry (ghcr.io):
1. Authenticate to ghcr.io using `GITHUB_TOKEN`
2. Push the image with SHA and latest tags
3. Verify the image is available in your GitHub package registry

**Expected Result:**
- Image pushed to `ghcr.io/<user>/cicd-practice:<sha>`
- Image visible in the Packages tab of your repository

**Verification:**
```bash
docker pull ghcr.io/<your-username>/cicd-practice:<sha>
```

> [!hint]- Hint
> Use `docker/login-action` with `GITHUB_TOKEN`:
> ```yaml
> - uses: docker/login-action@v3
>   with:
>     registry: ghcr.io
>     username: ${{ github.actor }}
>     password: ${{ secrets.GITHUB_TOKEN }}
> ```
> Then set `push: true` in `docker/build-push-action`.
> Make sure the job has `packages: write` permission.

> [!success]- Solution
> Update `.github/workflows/ci.yaml`:
> ```yaml
> name: CI
>
> on:
>   push:
>     branches: [main]
>   pull_request:
>     branches: [main]
>
> permissions:
>   contents: read
>   packages: write
>
> jobs:
>   test:
>     name: Test
>     runs-on: ubuntu-latest
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Go
>         uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>
>       - name: Run tests
>         run: go test -v -race ./...
>
>       - name: Build binary
>         run: go build -o server .
>
>   build-and-push:
>     name: Build and Push Docker Image
>     runs-on: ubuntu-latest
>     needs: test
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Docker Buildx
>         uses: docker/setup-buildx-action@v3
>
>       - name: Login to GitHub Container Registry
>         uses: docker/login-action@v3
>         with:
>           registry: ghcr.io
>           username: ${{ github.actor }}
>           password: ${{ secrets.GITHUB_TOKEN }}
>
>       - name: Docker metadata
>         id: meta
>         uses: docker/metadata-action@v5
>         with:
>           images: ghcr.io/${{ github.repository }}
>           tags: |
>             type=sha,prefix=
>             type=raw,value=latest,enable={{is_default_branch}}
>
>       - name: Build and push Docker image
>         uses: docker/build-push-action@v6
>         with:
>           context: .
>           push: ${{ github.event_name != 'pull_request' }}
>           tags: ${{ steps.meta.outputs.tags }}
>           labels: ${{ steps.meta.outputs.labels }}
>           cache-from: type=gha
>           cache-to: type=gha,mode=max
>
>       - name: Print image digest
>         run: |
>           echo "Image pushed with tags:"
>           echo "${{ steps.meta.outputs.tags }}"
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Push Docker image to GHCR"
> git push
>
> gh run watch
>
> # Verify the image in the package registry
> gh api user/packages/container/cicd-practice/versions --jq '.[].metadata.container.tags'
>
> # Pull and test locally
> docker pull ghcr.io/<your-username>/cicd-practice:latest
> docker run -p 8080:8080 ghcr.io/<your-username>/cicd-practice:latest
> curl localhost:8080
> ```

---

## P7: Add a Lint Job in Parallel

**Task:** Add a `golangci-lint` job that runs in parallel with the test job:
1. Use the `golangci/golangci-lint-action` action
2. The lint job should run independently (not depend on test)
3. Create a `.golangci.yaml` config enabling common linters

**Expected Result:**
- Lint and test jobs run simultaneously
- Both must pass for the build job to proceed

**Verification:** The Actions tab shows lint and test jobs running in parallel.

> [!hint]- Hint
> ```yaml
> lint:
>   runs-on: ubuntu-latest
>   steps:
>     - uses: actions/checkout@v4
>     - uses: actions/setup-go@v5
>       with:
>         go-version: '1.22'
>     - uses: golangci/golangci-lint-action@v6
>       with:
>         version: latest
> ```
> The build job can depend on both: `needs: [test, lint]`.

> [!success]- Solution
> Create `.golangci.yaml`:
> ```yaml
> linters:
>   enable:
>     - errcheck
>     - govet
>     - staticcheck
>     - unused
>     - ineffassign
>     - gosimple
>
> linters-settings:
>   errcheck:
>     check-type-assertions: true
>
> issues:
>   max-issues-per-linter: 50
>   max-same-issues: 10
> ```
>
> Update `.github/workflows/ci.yaml`:
> ```yaml
> name: CI
>
> on:
>   push:
>     branches: [main]
>   pull_request:
>     branches: [main]
>
> permissions:
>   contents: read
>   packages: write
>
> jobs:
>   lint:
>     name: Lint
>     runs-on: ubuntu-latest
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Go
>         uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>
>       - name: Run golangci-lint
>         uses: golangci/golangci-lint-action@v6
>         with:
>           version: latest
>
>   test:
>     name: Test
>     runs-on: ubuntu-latest
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Go
>         uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>
>       - name: Run tests
>         run: go test -v -race -coverprofile=coverage.out ./...
>
>       - name: Display coverage
>         run: go tool cover -func=coverage.out
>
>   build-and-push:
>     name: Build and Push
>     runs-on: ubuntu-latest
>     needs: [lint, test]  # Depends on BOTH lint and test
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Docker Buildx
>         uses: docker/setup-buildx-action@v3
>
>       - name: Login to GHCR
>         uses: docker/login-action@v3
>         with:
>           registry: ghcr.io
>           username: ${{ github.actor }}
>           password: ${{ secrets.GITHUB_TOKEN }}
>
>       - name: Docker metadata
>         id: meta
>         uses: docker/metadata-action@v5
>         with:
>           images: ghcr.io/${{ github.repository }}
>           tags: |
>             type=sha,prefix=
>             type=raw,value=latest,enable={{is_default_branch}}
>
>       - name: Build and push
>         uses: docker/build-push-action@v6
>         with:
>           context: .
>           push: ${{ github.event_name != 'pull_request' }}
>           tags: ${{ steps.meta.outputs.tags }}
>           labels: ${{ steps.meta.outputs.labels }}
>           cache-from: type=gha
>           cache-to: type=gha,mode=max
> ```
>
> ```bash
> git add .golangci.yaml .github/workflows/ci.yaml
> git commit -m "Add parallel lint job with golangci-lint"
> git push
>
> # Watch -- lint and test should run in parallel
> gh run watch
> ```

---

## P8: Deploy Job with Dependencies

**Task:** Add a deploy job that:
1. Depends on test, lint, AND build-and-push jobs (all must pass)
2. Only runs on the `main` branch (not on PRs)
3. Outputs the deployed image tag
4. Simulates a deployment (echo commands for now)

**Expected Result:**
- Deploy job only runs after lint, test, and build all succeed
- Deploy job is skipped on pull requests
- Deploy job prints the image tag being deployed

**Verification:** On main branch push, all 4 jobs run in sequence. On PR, deploy is skipped.

> [!hint]- Hint
> Use `needs` and `if` conditions:
> ```yaml
> deploy:
>   needs: [lint, test, build-and-push]
>   if: github.ref == 'refs/heads/main' && github.event_name == 'push'
>   runs-on: ubuntu-latest
> ```

> [!success]- Solution
> Add to `.github/workflows/ci.yaml`:
> ```yaml
>   deploy:
>     name: Deploy
>     runs-on: ubuntu-latest
>     needs: [lint, test, build-and-push]
>     if: github.ref == 'refs/heads/main' && github.event_name == 'push'
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Deploy
>         env:
>           IMAGE_TAG: ${{ github.sha }}
>         run: |
>           echo "========================================="
>           echo "  Deploying to Production"
>           echo "========================================="
>           echo "Image: ghcr.io/${{ github.repository }}:${IMAGE_TAG:0:7}"
>           echo "Commit: ${{ github.sha }}"
>           echo "Branch: ${{ github.ref_name }}"
>           echo "Actor: ${{ github.actor }}"
>           echo ""
>           echo "Simulating deployment..."
>           echo "  - Pulling image..."
>           echo "  - Updating deployment..."
>           echo "  - Waiting for rollout..."
>           echo "  - Deployment complete!"
>           echo "========================================="
> ```
>
> The full workflow now has 4 jobs:
> ```
> lint ─────┐
>           ├──> build-and-push ──> deploy
> test ─────┘
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Add deploy job with dependencies"
> git push
>
> gh run watch
> ```

---

## P9: GitHub Environments with Manual Approval

**Task:** Configure a production environment with manual approval:
1. Create a `production` environment in GitHub repository settings
2. Add a required reviewer for the `production` environment
3. Update the deploy job to use the `production` environment
4. Push a change and observe the approval gate

**Expected Result:**
- Deploy job pauses and waits for manual approval
- A reviewer must approve before deployment proceeds
- The environment shows deployment history

**Verification:** The Actions tab shows the deploy job waiting for approval.

> [!hint]- Hint
> 1. Go to Repository Settings > Environments > New environment
> 2. Name it `production`, add yourself as a required reviewer
> 3. In the workflow, add `environment: production` to the deploy job:
>    ```yaml
>    deploy:
>      environment:
>        name: production
>        url: https://myapp.example.com
>    ```

> [!success]- Solution
> **Step 1: Create the environment on GitHub**
> Go to your repository > Settings > Environments > New environment:
> - Name: `production`
> - Required reviewers: add your GitHub username
> - (Optional) Wait timer: 0 minutes
> - (Optional) Deployment branches: `main` only
>
> **Step 2: Update the workflow**
> ```yaml
>   deploy:
>     name: Deploy to Production
>     runs-on: ubuntu-latest
>     needs: [lint, test, build-and-push]
>     if: github.ref == 'refs/heads/main' && github.event_name == 'push'
>     environment:
>       name: production
>       url: https://myapp.example.com
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Deploy to production
>         env:
>           IMAGE_TAG: ${{ github.sha }}
>         run: |
>           echo "Deploying to production environment..."
>           echo "Image: ghcr.io/${{ github.repository }}:${IMAGE_TAG:0:7}"
>           echo "Deployment complete!"
> ```
>
> You can also add a staging environment without approval:
> ```yaml
>   deploy-staging:
>     name: Deploy to Staging
>     runs-on: ubuntu-latest
>     needs: [lint, test, build-and-push]
>     if: github.ref == 'refs/heads/main' && github.event_name == 'push'
>     environment:
>       name: staging
>
>     steps:
>       - name: Deploy to staging
>         run: echo "Deploying to staging..."
>
>   deploy-production:
>     name: Deploy to Production
>     runs-on: ubuntu-latest
>     needs: [deploy-staging]
>     environment:
>       name: production
>
>     steps:
>       - name: Deploy to production
>         run: echo "Deploying to production..."
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Add production environment with approval gate"
> git push
>
> # Watch -- deploy will pause waiting for approval
> gh run watch
>
> # Approve via CLI
> gh run view  # Get the run ID
> # Approve in the GitHub UI: Actions > Run > Review deployments
> ```

---

## P10: Reusable Workflow for Docker Build+Push

**Task:** Extract the Docker build+push logic into a reusable workflow:
1. Create a reusable workflow `.github/workflows/docker-build.yaml`
2. Accept inputs: image name, tag, and whether to push
3. Call the reusable workflow from the main CI workflow

**Expected Result:**
- Reusable workflow is defined with `workflow_call` trigger
- Main workflow calls it with `uses: ./.github/workflows/docker-build.yaml`

**Verification:** The main workflow successfully calls the reusable workflow.

> [!hint]- Hint
> Reusable workflows use `on: workflow_call` with `inputs` and `secrets`:
> ```yaml
> on:
>   workflow_call:
>     inputs:
>       image-name:
>         required: true
>         type: string
> ```
> Call it with:
> ```yaml
> jobs:
>   build:
>     uses: ./.github/workflows/docker-build.yaml
>     with:
>       image-name: my-image
> ```

> [!success]- Solution
> Create `.github/workflows/docker-build.yaml`:
> ```yaml
> name: Docker Build and Push (Reusable)
>
> on:
>   workflow_call:
>     inputs:
>       image-name:
>         description: 'Full image name (e.g., ghcr.io/user/app)'
>         required: true
>         type: string
>       push:
>         description: 'Whether to push the image'
>         required: false
>         type: boolean
>         default: false
>       context:
>         description: 'Docker build context'
>         required: false
>         type: string
>         default: '.'
>     outputs:
>       image-tag:
>         description: 'The image tag (SHA)'
>         value: ${{ jobs.build.outputs.image-tag }}
>       image-digest:
>         description: 'The image digest'
>         value: ${{ jobs.build.outputs.image-digest }}
>
> jobs:
>   build:
>     name: Build Docker Image
>     runs-on: ubuntu-latest
>     outputs:
>       image-tag: ${{ steps.meta.outputs.version }}
>       image-digest: ${{ steps.build.outputs.digest }}
>
>     permissions:
>       contents: read
>       packages: write
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Set up Docker Buildx
>         uses: docker/setup-buildx-action@v3
>
>       - name: Login to GHCR
>         if: inputs.push
>         uses: docker/login-action@v3
>         with:
>           registry: ghcr.io
>           username: ${{ github.actor }}
>           password: ${{ secrets.GITHUB_TOKEN }}
>
>       - name: Docker metadata
>         id: meta
>         uses: docker/metadata-action@v5
>         with:
>           images: ${{ inputs.image-name }}
>           tags: |
>             type=sha,prefix=
>             type=raw,value=latest,enable={{is_default_branch}}
>
>       - name: Build and push
>         id: build
>         uses: docker/build-push-action@v6
>         with:
>           context: ${{ inputs.context }}
>           push: ${{ inputs.push }}
>           tags: ${{ steps.meta.outputs.tags }}
>           labels: ${{ steps.meta.outputs.labels }}
>           cache-from: type=gha
>           cache-to: type=gha,mode=max
> ```
>
> Update `.github/workflows/ci.yaml` to call it:
> ```yaml
> name: CI
>
> on:
>   push:
>     branches: [main]
>   pull_request:
>     branches: [main]
>
> permissions:
>   contents: read
>   packages: write
>
> jobs:
>   lint:
>     name: Lint
>     runs-on: ubuntu-latest
>     steps:
>       - uses: actions/checkout@v4
>       - uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>       - uses: golangci/golangci-lint-action@v6
>         with:
>           version: latest
>
>   test:
>     name: Test
>     runs-on: ubuntu-latest
>     steps:
>       - uses: actions/checkout@v4
>       - uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>       - run: go test -v -race ./...
>
>   build-and-push:
>     name: Build and Push
>     needs: [lint, test]
>     uses: ./.github/workflows/docker-build.yaml
>     with:
>       image-name: ghcr.io/${{ github.repository }}
>       push: ${{ github.event_name != 'pull_request' }}
>     permissions:
>       contents: read
>       packages: write
>
>   deploy:
>     name: Deploy
>     runs-on: ubuntu-latest
>     needs: [build-and-push]
>     if: github.ref == 'refs/heads/main' && github.event_name == 'push'
>     environment:
>       name: production
>
>     steps:
>       - name: Deploy
>         run: |
>           echo "Deploying image tag: ${{ needs.build-and-push.outputs.image-tag }}"
>           echo "Deployment complete!"
> ```
>
> ```bash
> git add .github/workflows/docker-build.yaml .github/workflows/ci.yaml
> git commit -m "Extract reusable Docker build workflow"
> git push
>
> gh run watch
> ```

---

## P11: Manual Trigger with Input Parameters

**Task:** Create a workflow with `workflow_dispatch` that:
1. Accepts an `image-tag` input (default: latest)
2. Accepts a `target-env` choice input (staging, production)
3. Accepts a `dry-run` boolean input
4. Deploys the specified image tag to the specified environment

**Expected Result:**
- The workflow appears in the Actions tab with a "Run workflow" button
- You can select the target environment and image tag
- The workflow deploys (or dry-runs) to the selected environment

**Verification:**
```bash
gh workflow run deploy.yaml -f image-tag=abc1234 -f target-env=staging -f dry-run=true
```

> [!hint]- Hint
> ```yaml
> on:
>   workflow_dispatch:
>     inputs:
>       image-tag:
>         description: 'Image tag to deploy'
>         required: true
>         default: 'latest'
>       target-env:
>         description: 'Target environment'
>         required: true
>         type: choice
>         options:
>           - staging
>           - production
> ```
> Access inputs with `${{ inputs.image-tag }}` or `${{ github.event.inputs.image-tag }}`.

> [!success]- Solution
> Create `.github/workflows/deploy.yaml`:
> ```yaml
> name: Manual Deploy
>
> on:
>   workflow_dispatch:
>     inputs:
>       image-tag:
>         description: 'Image tag to deploy'
>         required: true
>         default: 'latest'
>         type: string
>       target-env:
>         description: 'Target environment'
>         required: true
>         type: choice
>         options:
>           - staging
>           - production
>       dry-run:
>         description: 'Dry run (no actual deployment)'
>         required: false
>         type: boolean
>         default: false
>
> jobs:
>   deploy:
>     name: Deploy to ${{ inputs.target-env }}
>     runs-on: ubuntu-latest
>     environment:
>       name: ${{ inputs.target-env }}
>
>     steps:
>       - name: Checkout code
>         uses: actions/checkout@v4
>
>       - name: Print deployment info
>         run: |
>           echo "========================================="
>           echo "  Deployment Configuration"
>           echo "========================================="
>           echo "Image tag:    ${{ inputs.image-tag }}"
>           echo "Environment:  ${{ inputs.target-env }}"
>           echo "Dry run:      ${{ inputs.dry-run }}"
>           echo "Triggered by: ${{ github.actor }}"
>           echo "========================================="
>
>       - name: Validate image exists
>         run: |
>           echo "Checking if image ghcr.io/${{ github.repository }}:${{ inputs.image-tag }} exists..."
>           # In a real workflow, you would:
>           # docker manifest inspect ghcr.io/${{ github.repository }}:${{ inputs.image-tag }}
>           echo "Image found!"
>
>       - name: Deploy (or dry-run)
>         run: |
>           if [ "${{ inputs.dry-run }}" = "true" ]; then
>             echo "[DRY RUN] Would deploy ghcr.io/${{ github.repository }}:${{ inputs.image-tag }} to ${{ inputs.target-env }}"
>             echo "[DRY RUN] Skipping actual deployment"
>           else
>             echo "Deploying ghcr.io/${{ github.repository }}:${{ inputs.image-tag }} to ${{ inputs.target-env }}..."
>             echo "Updating Kubernetes deployment..."
>             # kubectl set image deployment/myapp myapp=ghcr.io/${{ github.repository }}:${{ inputs.image-tag }}
>             echo "Waiting for rollout..."
>             # kubectl rollout status deployment/myapp
>             echo "Deployment complete!"
>           fi
>
>       - name: Verify deployment
>         if: inputs.dry-run == false
>         run: |
>           echo "Running post-deploy verification..."
>           echo "Health check: OK"
>           echo "Smoke test: OK"
> ```
>
> ```bash
> git add .github/workflows/deploy.yaml
> git commit -m "Add manual deploy workflow with inputs"
> git push
>
> # Trigger via CLI
> gh workflow run deploy.yaml \
>   -f image-tag=abc1234 \
>   -f target-env=staging \
>   -f dry-run=true
>
> gh run watch
>
> # Trigger a real deployment
> gh workflow run deploy.yaml \
>   -f image-tag=latest \
>   -f target-env=production \
>   -f dry-run=false
> ```

---

## P12: GitOps Deploy - Update Image Tag in Config Repo

**Task:** After a successful build, automatically update the image tag in a separate GitOps configuration repository:
1. Create a GitOps repo (e.g., `cicd-practice-gitops`) with Kubernetes manifests
2. After Docker build+push, update the image tag in the GitOps repo
3. Use a Personal Access Token to push to the GitOps repo

**Expected Result:**
- After CI builds and pushes the image, a commit is made to the GitOps repo
- The GitOps repo's deployment manifest has the updated image tag
- ArgoCD (if configured) picks up the change and deploys

**Verification:** Check the GitOps repo for the automated commit updating the image tag.

> [!hint]- Hint
> You need:
> 1. A PAT (Personal Access Token) stored as a repository secret (`GITOPS_PAT`)
> 2. Steps to: clone the GitOps repo, update the image tag (e.g., with `sed` or `yq`), commit, and push
>
> ```yaml
> - name: Update GitOps repo
>   env:
>     GITOPS_PAT: ${{ secrets.GITOPS_PAT }}
>   run: |
>     git clone https://x-access-token:${GITOPS_PAT}@github.com/<user>/gitops-repo.git
>     cd gitops-repo
>     # Update image tag
>     git commit -am "Update image to <new-tag>"
>     git push
> ```

> [!success]- Solution
> **Step 1: Create the GitOps repo**
> ```bash
> mkdir cicd-practice-gitops && cd cicd-practice-gitops
> git init
>
> mkdir -p apps/cicd-practice
>
> cat <<'EOF' > apps/cicd-practice/deployment.yaml
> apiVersion: apps/v1
> kind: Deployment
> metadata:
>   name: cicd-practice
>   labels:
>     app: cicd-practice
> spec:
>   replicas: 2
>   selector:
>     matchLabels:
>       app: cicd-practice
>   template:
>     metadata:
>       labels:
>         app: cicd-practice
>     spec:
>       containers:
>         - name: app
>           image: ghcr.io/<your-username>/cicd-practice:latest  # Updated by CI
>           ports:
>             - containerPort: 8080
> EOF
>
> cat <<'EOF' > apps/cicd-practice/service.yaml
> apiVersion: v1
> kind: Service
> metadata:
>   name: cicd-practice
> spec:
>   selector:
>     app: cicd-practice
>   ports:
>     - port: 80
>       targetPort: 8080
>   type: ClusterIP
> EOF
>
> git add . && git commit -m "Initial GitOps manifests"
> gh repo create cicd-practice-gitops --public --push --source .
> ```
>
> **Step 2: Create a PAT and add as secret**
> - Go to GitHub > Settings > Developer settings > Personal access tokens > Fine-grained tokens
> - Create a token with `contents: write` access to the GitOps repo
> - Add it as a secret in the CI repo: Settings > Secrets > `GITOPS_PAT`
>
> **Step 3: Add the GitOps update job to CI**
>
> Add to `.github/workflows/ci.yaml`:
> ```yaml
>   update-gitops:
>     name: Update GitOps Repo
>     runs-on: ubuntu-latest
>     needs: [build-and-push]
>     if: github.ref == 'refs/heads/main' && github.event_name == 'push'
>
>     steps:
>       - name: Update image tag in GitOps repo
>         env:
>           GITOPS_PAT: ${{ secrets.GITOPS_PAT }}
>           IMAGE_TAG: ${{ github.sha }}
>         run: |
>           # Configure git
>           git config --global user.name "github-actions[bot]"
>           git config --global user.email "github-actions[bot]@users.noreply.github.com"
>
>           # Clone the GitOps repo
>           git clone https://x-access-token:${GITOPS_PAT}@github.com/${{ github.repository_owner }}/cicd-practice-gitops.git
>           cd cicd-practice-gitops
>
>           # Update the image tag using sed
>           SHORT_SHA=${IMAGE_TAG:0:7}
>           sed -i "s|image: ghcr.io/${{ github.repository }}:.*|image: ghcr.io/${{ github.repository }}:${SHORT_SHA}|" \
>             apps/cicd-practice/deployment.yaml
>
>           # Verify the change
>           cat apps/cicd-practice/deployment.yaml
>
>           # Commit and push
>           git add .
>           git diff --staged --quiet || {
>             git commit -m "Update cicd-practice image to ${SHORT_SHA}
>
>           Triggered by: ${{ github.repository }}@${{ github.sha }}
>           Workflow: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
>             git push
>           }
>
>           echo "GitOps repo updated with image tag: ${SHORT_SHA}"
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Add GitOps repo update after build"
> git push
>
> gh run watch
>
> # Check the GitOps repo for the automated commit
> cd ../cicd-practice-gitops
> git pull
> git log --oneline -5
> cat apps/cicd-practice/deployment.yaml
> ```

---

## P13: Full Pipeline - Lint, Test, Build, GitOps, Notify

**Task:** Build the complete end-to-end CI/CD pipeline:
1. **Lint** -- golangci-lint (parallel with test)
2. **Test** -- Go tests with coverage (parallel with lint)
3. **Build+Push** -- Docker build and push to GHCR (depends on lint+test)
4. **Update GitOps** -- Update image tag in GitOps repo (depends on build)
5. **Notify** -- Send a webhook notification with build status (always runs)

The pipeline should:
- Run on push to main and on PRs
- Only push Docker images and update GitOps on main branch
- Send notifications on both success and failure
- Include a build summary as a job output

**Expected Result:**
- Complete pipeline executes end-to-end on push to main
- PR builds skip push/deploy/GitOps steps
- Notification is sent with build status

**Verification:**
```bash
gh run list --limit 5
gh run view <run-id>
```

> [!hint]- Hint
> For the notification step, you can use a simple webhook (e.g., webhook.site for testing) or the `slackapi/slack-github-action` for Slack:
> ```yaml
> - uses: slackapi/slack-github-action@v2.0.0
>   with:
>     webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
>     webhook-type: incoming-webhook
> ```
> Use `if: always()` to ensure the notification runs regardless of previous job status. Use `needs` result checking: `needs.build.result == 'success'`.

> [!success]- Solution
> Create the final `.github/workflows/ci.yaml`:
> ```yaml
> name: CI/CD Pipeline
>
> on:
>   push:
>     branches: [main]
>   pull_request:
>     branches: [main]
>
> permissions:
>   contents: read
>   packages: write
>
> env:
>   IMAGE_NAME: ghcr.io/${{ github.repository }}
>
> jobs:
>   # ==========================================
>   # Stage 1: Quality Checks (parallel)
>   # ==========================================
>   lint:
>     name: Lint
>     runs-on: ubuntu-latest
>     steps:
>       - name: Checkout
>         uses: actions/checkout@v4
>
>       - name: Set up Go
>         uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>
>       - name: Run golangci-lint
>         uses: golangci/golangci-lint-action@v6
>         with:
>           version: latest
>
>   test:
>     name: Test
>     runs-on: ubuntu-latest
>     outputs:
>       coverage: ${{ steps.coverage.outputs.total }}
>     steps:
>       - name: Checkout
>         uses: actions/checkout@v4
>
>       - name: Set up Go
>         uses: actions/setup-go@v5
>         with:
>           go-version: '1.22'
>
>       - name: Run tests
>         run: go test -v -race -coverprofile=coverage.out ./...
>
>       - name: Get coverage percentage
>         id: coverage
>         run: |
>           COVERAGE=$(go tool cover -func=coverage.out | grep total | awk '{print $3}')
>           echo "total=${COVERAGE}" >> "$GITHUB_OUTPUT"
>           echo "Total coverage: ${COVERAGE}"
>
>   # ==========================================
>   # Stage 2: Build and Push
>   # ==========================================
>   build-and-push:
>     name: Build and Push
>     runs-on: ubuntu-latest
>     needs: [lint, test]
>     outputs:
>       image-tag: ${{ steps.short-sha.outputs.sha }}
>
>     steps:
>       - name: Checkout
>         uses: actions/checkout@v4
>
>       - name: Get short SHA
>         id: short-sha
>         run: echo "sha=${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"
>
>       - name: Set up Docker Buildx
>         uses: docker/setup-buildx-action@v3
>
>       - name: Login to GHCR
>         if: github.event_name != 'pull_request'
>         uses: docker/login-action@v3
>         with:
>           registry: ghcr.io
>           username: ${{ github.actor }}
>           password: ${{ secrets.GITHUB_TOKEN }}
>
>       - name: Docker metadata
>         id: meta
>         uses: docker/metadata-action@v5
>         with:
>           images: ${{ env.IMAGE_NAME }}
>           tags: |
>             type=sha,prefix=
>             type=raw,value=latest,enable={{is_default_branch}}
>             type=ref,event=pr
>
>       - name: Build and push
>         uses: docker/build-push-action@v6
>         with:
>           context: .
>           push: ${{ github.event_name != 'pull_request' }}
>           tags: ${{ steps.meta.outputs.tags }}
>           labels: ${{ steps.meta.outputs.labels }}
>           cache-from: type=gha
>           cache-to: type=gha,mode=max
>           build-args: |
>             APP_VERSION=${{ steps.short-sha.outputs.sha }}
>
>   # ==========================================
>   # Stage 3: Update GitOps Config
>   # ==========================================
>   update-gitops:
>     name: Update GitOps
>     runs-on: ubuntu-latest
>     needs: [build-and-push]
>     if: github.ref == 'refs/heads/main' && github.event_name == 'push'
>
>     steps:
>       - name: Update image tag in GitOps repo
>         env:
>           GITOPS_PAT: ${{ secrets.GITOPS_PAT }}
>           IMAGE_TAG: ${{ needs.build-and-push.outputs.image-tag }}
>         run: |
>           git config --global user.name "github-actions[bot]"
>           git config --global user.email "github-actions[bot]@users.noreply.github.com"
>
>           git clone https://x-access-token:${GITOPS_PAT}@github.com/${{ github.repository_owner }}/cicd-practice-gitops.git
>           cd cicd-practice-gitops
>
>           sed -i "s|image: ${IMAGE_NAME}:.*|image: ${IMAGE_NAME}:${IMAGE_TAG}|" \
>             apps/cicd-practice/deployment.yaml
>
>           git add .
>           git diff --staged --quiet || {
>             git commit -m "chore: update cicd-practice to ${IMAGE_TAG}
>
>           Source: ${{ github.repository }}@${{ github.sha }}
>           Run: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
>             git push
>             echo "GitOps repo updated with tag: ${IMAGE_TAG}"
>           }
>
>   # ==========================================
>   # Stage 4: Notify
>   # ==========================================
>   notify:
>     name: Notify
>     runs-on: ubuntu-latest
>     needs: [lint, test, build-and-push, update-gitops]
>     if: always() && github.ref == 'refs/heads/main'
>
>     steps:
>       - name: Determine overall status
>         id: status
>         run: |
>           if [ "${{ needs.lint.result }}" = "failure" ] || \
>              [ "${{ needs.test.result }}" = "failure" ] || \
>              [ "${{ needs.build-and-push.result }}" = "failure" ] || \
>              [ "${{ needs.update-gitops.result }}" = "failure" ]; then
>             echo "status=failure" >> "$GITHUB_OUTPUT"
>             echo "emoji=:x:" >> "$GITHUB_OUTPUT"
>           else
>             echo "status=success" >> "$GITHUB_OUTPUT"
>             echo "emoji=:white_check_mark:" >> "$GITHUB_OUTPUT"
>           fi
>
>       - name: Build summary
>         run: |
>           echo "## Pipeline Summary" >> $GITHUB_STEP_SUMMARY
>           echo "" >> $GITHUB_STEP_SUMMARY
>           echo "| Stage | Status |" >> $GITHUB_STEP_SUMMARY
>           echo "|-------|--------|" >> $GITHUB_STEP_SUMMARY
>           echo "| Lint | ${{ needs.lint.result }} |" >> $GITHUB_STEP_SUMMARY
>           echo "| Test | ${{ needs.test.result }} |" >> $GITHUB_STEP_SUMMARY
>           echo "| Build | ${{ needs.build-and-push.result }} |" >> $GITHUB_STEP_SUMMARY
>           echo "| GitOps | ${{ needs.update-gitops.result }} |" >> $GITHUB_STEP_SUMMARY
>           echo "" >> $GITHUB_STEP_SUMMARY
>           echo "**Image tag:** \`${{ needs.build-and-push.outputs.image-tag }}\`" >> $GITHUB_STEP_SUMMARY
>           echo "**Coverage:** ${{ needs.test.outputs.coverage }}" >> $GITHUB_STEP_SUMMARY
>
>       - name: Send webhook notification
>         if: vars.WEBHOOK_URL != ''
>         env:
>           WEBHOOK_URL: ${{ vars.WEBHOOK_URL }}
>         run: |
>           curl -X POST "$WEBHOOK_URL" \
>             -H "Content-Type: application/json" \
>             -d '{
>               "repository": "${{ github.repository }}",
>               "commit": "${{ github.sha }}",
>               "image_tag": "${{ needs.build-and-push.outputs.image-tag }}",
>               "status": "${{ steps.status.outputs.status }}",
>               "coverage": "${{ needs.test.outputs.coverage }}",
>               "run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}",
>               "actor": "${{ github.actor }}"
>             }'
>
>       # Alternative: Slack notification
>       # - name: Notify Slack
>       #   if: vars.SLACK_WEBHOOK_URL != ''
>       #   uses: slackapi/slack-github-action@v2.0.0
>       #   with:
>       #     webhook: ${{ vars.SLACK_WEBHOOK_URL }}
>       #     webhook-type: incoming-webhook
>       #     payload: |
>       #       {
>       #         "text": "${{ steps.status.outputs.emoji }} *${{ github.repository }}* pipeline ${{ steps.status.outputs.status }}\nCommit: `${{ github.sha }}`\nImage: `${{ needs.build-and-push.outputs.image-tag }}`\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
>       #       }
> ```
>
> The complete pipeline flow:
> ```
>   lint ────┐
>            ├──> build-and-push ──> update-gitops ──┐
>   test ────┘                                       ├──> notify
>                                                    │
>   (all jobs report status) ────────────────────────┘
> ```
>
> ```bash
> git add .github/workflows/ci.yaml
> git commit -m "Complete CI/CD pipeline with GitOps and notifications"
> git push
>
> # Watch the full pipeline
> gh run watch
>
> # View the run summary
> gh run view --log
>
> # Check the GitOps repo was updated
> cd ../cicd-practice-gitops
> git pull
> git log --oneline -3
> cat apps/cicd-practice/deployment.yaml
> ```
>
> **Testing the full flow:**
> ```bash
> # Make a code change
> cd ../cicd-practice
> echo '// v2' >> main.go
> git add . && git commit -m "Trigger full pipeline" && git push
>
> # Watch all stages execute
> gh run watch
>
> # Verify:
> # 1. Lint passed
> # 2. Tests passed with coverage
> # 3. Docker image pushed to GHCR
> # 4. GitOps repo updated with new image tag
> # 5. Notification sent
>
> # If using ArgoCD, it will now detect the GitOps change and deploy
> # argocd app get cicd-practice
> ```
