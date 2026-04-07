---
title: "K8s Practice: Docker Fundamentals"
date: 2026-04-07
tags:
  - kubernetes
  - practice
  - docker
parent: "[[Kubernetes Study]]"
---

# Practice - Docker Fundamentals

14 progressive exercises building Docker fluency from basic containers to microservice architectures.

## Prerequisites

- Docker Desktop installed and running (`docker info` should work)
- Go installed locally (for writing the app code)
- Basic familiarity with terminal commands
- A working directory for exercise files: `mkdir -p ~/docker-practice && cd ~/docker-practice`

---

## P1: Simple Go HTTP Server in Docker

**Task:** Write a Dockerfile for a simple Go "Hello World" HTTP server. Build the image and run it, then verify you can reach the server from the host.

**Create the Go source file first** (`main.go`):

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello, Docker!")
	})
	fmt.Println("Server starting on :8080")
	http.ListenAndServe(":8080", nil)
}
```

**Expected result:**
- `curl http://localhost:8080` returns `Hello, Docker!`

**Verification:**

```bash
curl http://localhost:8080
docker ps  # Should show the running container
```

> [!hint]- Hint
> - Use `golang:1.22` as your base image
> - Set a `WORKDIR`, copy your source, build with `go build`
> - `EXPOSE 8080` documents the port, but you still need `-p 8080:8080` at runtime
> - Use `CMD` to specify the binary to run

> [!success]- Solution
> **Dockerfile:**
> ```dockerfile
> FROM golang:1.22
> 
> WORKDIR /app
> COPY main.go .
> RUN go build -o server main.go
> 
> EXPOSE 8080
> CMD ["./server"]
> ```
> 
> **Commands:**
> ```bash
> docker build -t hello-docker .
> docker run -d -p 8080:8080 --name hello hello-docker
> curl http://localhost:8080
> # Hello, Docker!
> 
> # Cleanup
> docker stop hello && docker rm hello
> ```

---

## P2: Dockerignore

**Task:** Create a `.dockerignore` file that excludes unnecessary files from the build context. Add some dummy files (README.md, .git directory, test files), rebuild, and verify they are NOT inside the image.

**Expected result:**
- Running `docker exec <container> ls /app` should only show `main.go` and the built binary, not `README.md`, `.git`, etc.

**Verification:**

```bash
docker exec <container> ls -la /app
```

> [!hint]- Hint
> - `.dockerignore` uses the same syntax as `.gitignore`
> - Create some dummy files first: `touch README.md notes.txt && mkdir .git`
> - Common exclusions: `.git`, `*.md`, `Dockerfile`, `.dockerignore`, `*_test.go`

> [!success]- Solution
> **Create dummy files:**
> ```bash
> touch README.md notes.txt
> mkdir -p .git
> echo "test" > app_test.go
> ```
> 
> **.dockerignore:**
> ```
> .git
> *.md
> Dockerfile
> .dockerignore
> *_test.go
> notes.txt
> ```
> 
> **Rebuild and verify:**
> ```bash
> docker build -t hello-docker .
> docker run -d -p 8080:8080 --name hello hello-docker
> docker exec hello ls -la /app
> # Should only show: main.go, server (binary)
> 
> # Cleanup
> docker stop hello && docker rm hello
> ```

---

## P3: Multi-Stage Build

**Task:** Rewrite the Dockerfile as a multi-stage build. Stage 1 compiles the Go app using `golang:1.22`. Stage 2 copies only the binary into `alpine:3.19`. Compare the image sizes.

**Expected result:**
- The multi-stage image should be ~10-15 MB vs ~800+ MB for the single-stage image
- The app still works: `curl http://localhost:8080` returns `Hello, Docker!`

**Verification:**

```bash
docker images | grep hello
```

> [!hint]- Hint
> - Name the first stage: `FROM golang:1.22 AS builder`
> - Use `CGO_ENABLED=0` so the binary is statically linked (required for alpine)
> - In stage 2: `COPY --from=builder /app/server /server`
> - Alpine needs no Go toolchain installed

> [!success]- Solution
> **Dockerfile:**
> ```dockerfile
> # Stage 1: Build
> FROM golang:1.22 AS builder
> WORKDIR /app
> COPY main.go .
> RUN CGO_ENABLED=0 GOOS=linux go build -o server main.go
> 
> # Stage 2: Run
> FROM alpine:3.19
> COPY --from=builder /app/server /server
> EXPOSE 8080
> CMD ["/server"]
> ```
> 
> **Commands:**
> ```bash
> # Build single-stage for comparison
> docker build -t hello-single -f- . <<'EOF'
> FROM golang:1.22
> WORKDIR /app
> COPY main.go .
> RUN go build -o server main.go
> EXPOSE 8080
> CMD ["./server"]
> EOF
> 
> # Build multi-stage
> docker build -t hello-multi .
> 
> # Compare sizes
> docker images | grep hello
> # hello-single   latest   ...   ~850MB
> # hello-multi    latest   ...   ~12MB
> 
> # Verify it works
> docker run -d -p 8080:8080 --name hello hello-multi
> curl http://localhost:8080
> 
> # Cleanup
> docker stop hello && docker rm hello
> ```

---

## P4: Environment Variables

**Task:** Modify the Go app to read a greeting message from an environment variable `GREETING` (default to "Hello"). Pass the variable at runtime with `-e` and also set a default with `ENV` in the Dockerfile.

**Update `main.go`:**

```go
package main

import (
	"fmt"
	"net/http"
	"os"
)

func main() {
	greeting := os.Getenv("GREETING")
	if greeting == "" {
		greeting = "Hello"
	}
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "%s, Docker!\n", greeting)
	})
	fmt.Println("Server starting on :8080")
	http.ListenAndServe(":8080", nil)
}
```

**Expected result:**
- Default: `curl localhost:8080` returns `Hello, Docker!`
- With `-e GREETING=Hola`: `curl localhost:8080` returns `Hola, Docker!`

**Verification:**

```bash
curl http://localhost:8080
```

> [!hint]- Hint
> - Add `ENV GREETING="Hello"` in the Dockerfile to set a default
> - Override at runtime: `docker run -e GREETING=Hola ...`
> - The `-e` flag takes precedence over `ENV` in the Dockerfile

> [!success]- Solution
> **Dockerfile:**
> ```dockerfile
> FROM golang:1.22 AS builder
> WORKDIR /app
> COPY main.go .
> RUN CGO_ENABLED=0 GOOS=linux go build -o server main.go
> 
> FROM alpine:3.19
> ENV GREETING="Hello"
> COPY --from=builder /app/server /server
> EXPOSE 8080
> CMD ["/server"]
> ```
> 
> **Commands:**
> ```bash
> docker build -t hello-env .
> 
> # Run with default
> docker run -d -p 8080:8080 --name hello-default hello-env
> curl http://localhost:8080
> # Hello, Docker!
> docker stop hello-default && docker rm hello-default
> 
> # Run with override
> docker run -d -p 8080:8080 -e GREETING=Hola --name hello-custom hello-env
> curl http://localhost:8080
> # Hola, Docker!
> docker stop hello-custom && docker rm hello-custom
> ```

---

## P5: Docker Volumes

**Task:** Mount a local directory into a container. Create a local `data/` directory with a file, mount it into the container, and verify the container can read and write to it.

**Expected result:**
- Container can read files from the mounted directory
- Files written by the container appear on the host

**Verification:**

```bash
ls ./data/
cat ./data/container-output.txt
```

> [!hint]- Hint
> - Use `-v $(pwd)/data:/data` to bind mount
> - You can use a simple `alpine` container with a shell command to test
> - Write from inside: `docker exec <container> sh -c "echo 'written by container' > /data/container-output.txt"`

> [!success]- Solution
> ```bash
> # Setup
> mkdir -p data
> echo "hello from host" > data/host-file.txt
> 
> # Run with volume mount
> docker run -d --name vol-test -v $(pwd)/data:/data alpine:3.19 sleep 3600
> 
> # Verify container can read host file
> docker exec vol-test cat /data/host-file.txt
> # hello from host
> 
> # Write from container
> docker exec vol-test sh -c "echo 'written by container' > /data/container-output.txt"
> 
> # Verify on host
> cat data/container-output.txt
> # written by container
> 
> # Cleanup
> docker stop vol-test && docker rm vol-test
> ```

---

## P6: Docker Networking

**Task:** Create a custom Docker network. Run two containers on it: an nginx server and an alpine container. From the alpine container, curl the nginx server by container name.

**Expected result:**
- The alpine container can reach nginx by name (e.g., `curl http://web`)

**Verification:**

```bash
docker exec alpine-client wget -qO- http://web
```

> [!hint]- Hint
> - `docker network create mynet`
> - Attach containers with `--network mynet`
> - Docker's built-in DNS resolves container names on custom networks
> - Use `--name web` to give the nginx container a DNS-resolvable name

> [!success]- Solution
> ```bash
> # Create network
> docker network create mynet
> 
> # Run nginx with a name
> docker run -d --name web --network mynet nginx:alpine
> 
> # Run alpine client on same network
> docker run -d --name alpine-client --network mynet alpine:3.19 sleep 3600
> 
> # Curl nginx from alpine by container name
> docker exec alpine-client wget -qO- http://web
> # Should return the nginx welcome page HTML
> 
> # Cleanup
> docker stop web alpine-client
> docker rm web alpine-client
> docker network rm mynet
> ```

---

## P7: Docker Compose - Multi-Service Stack

**Task:** Write a `docker-compose.yml` that defines a stack with three services:
1. A Go HTTP app (builds from local Dockerfile)
2. PostgreSQL 16
3. Redis 7

The app should be able to reach both Postgres and Redis by service name.

**Expected result:**
- `docker compose up -d` starts all three services
- `docker compose ps` shows all services running
- The app container can ping `postgres` and `redis` by hostname

**Verification:**

```bash
docker compose ps
docker compose exec app ping -c 1 postgres
docker compose exec app ping -c 1 redis
```

> [!hint]- Hint
> - Use `build: .` for the app service
> - Postgres needs `POSTGRES_PASSWORD` environment variable
> - All services on the same Compose network can resolve each other by service name
> - Add `depends_on` so the app starts after its dependencies

> [!success]- Solution
> **docker-compose.yml:**
> ```yaml
> services:
>   app:
>     build: .
>     ports:
>       - "8080:8080"
>     environment:
>       - DATABASE_URL=postgres://postgres:secret@postgres:5432/mydb?sslmode=disable
>       - REDIS_URL=redis://redis:6379
>     depends_on:
>       - postgres
>       - redis
> 
>   postgres:
>     image: postgres:16-alpine
>     environment:
>       POSTGRES_PASSWORD: secret
>       POSTGRES_DB: mydb
>     volumes:
>       - pgdata:/var/lib/postgresql/data
> 
>   redis:
>     image: redis:7-alpine
> 
> volumes:
>   pgdata:
> ```
> 
> **Commands:**
> ```bash
> docker compose up -d
> docker compose ps
> 
> # Verify connectivity (install ping first since alpine image is minimal)
> docker compose exec app sh -c "apk add --no-cache busybox-extras && ping -c 1 postgres"
> docker compose exec app sh -c "ping -c 1 redis"
> 
> # Cleanup
> docker compose down -v
> ```

---

## P8: Docker Compose Health Checks

**Task:** Add a health check to the Postgres service. Configure the app service to only start after Postgres is healthy using `depends_on` with `condition: service_healthy`.

**Expected result:**
- `docker compose up` shows the app waiting for Postgres to become healthy
- `docker compose ps` shows the health status column for Postgres

**Verification:**

```bash
docker compose ps
# Postgres should show (healthy)
```

> [!hint]- Hint
> - Postgres health check: `pg_isready -U postgres`
> - Set `interval`, `timeout`, `retries` in the healthcheck block
> - In `depends_on`, use the extended syntax: `condition: service_healthy`

> [!success]- Solution
> **docker-compose.yml:**
> ```yaml
> services:
>   app:
>     build: .
>     ports:
>       - "8080:8080"
>     environment:
>       - DATABASE_URL=postgres://postgres:secret@postgres:5432/mydb?sslmode=disable
>       - REDIS_URL=redis://redis:6379
>     depends_on:
>       postgres:
>         condition: service_healthy
>       redis:
>         condition: service_started
> 
>   postgres:
>     image: postgres:16-alpine
>     environment:
>       POSTGRES_PASSWORD: secret
>       POSTGRES_DB: mydb
>     volumes:
>       - pgdata:/var/lib/postgresql/data
>     healthcheck:
>       test: ["CMD-SHELL", "pg_isready -U postgres"]
>       interval: 5s
>       timeout: 5s
>       retries: 5
>       start_period: 10s
> 
>   redis:
>     image: redis:7-alpine
>     healthcheck:
>       test: ["CMD", "redis-cli", "ping"]
>       interval: 5s
>       timeout: 5s
>       retries: 5
> 
> volumes:
>   pgdata:
> ```
> 
> **Commands:**
> ```bash
> docker compose up -d
> 
> # Watch health status
> watch docker compose ps
> 
> # After a few seconds, Postgres should show (healthy)
> docker compose ps
> 
> # Cleanup
> docker compose down -v
> ```

---

## P9: Docker Layer Caching Optimization

**Task:** Optimize a Dockerfile for maximum layer cache hits. The key insight: copy dependency files first, install dependencies, THEN copy application code. This way, changing your code does not invalidate the dependency cache.

**Create a Go module first:**

```bash
go mod init myapp
```

**Update `main.go` to import a dependency:**

```go
package main

import (
	"fmt"
	"net/http"
	"github.com/gorilla/mux"
)

func main() {
	r := mux.NewRouter()
	r.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintln(w, "Hello with mux!")
	})
	fmt.Println("Server starting on :8080")
	http.ListenAndServe(":8080", r)
}
```

```bash
go mod tidy
```

**Expected result:**
- First build downloads dependencies (slow)
- Changing `main.go` and rebuilding reuses the dependency download layer (fast)
- You should see "CACHED" for the `go mod download` step on second build

**Verification:**

```bash
# Build, change main.go slightly, rebuild - observe cache behavior
docker build -t cache-test . 2>&1 | grep -i cached
```

> [!hint]- Hint
> - Copy `go.mod` and `go.sum` BEFORE copying `main.go`
> - Run `go mod download` as a separate layer after copying mod files
> - Then `COPY . .` for the rest of the source
> - This pattern works for any language: copy lockfile -> install deps -> copy source

> [!success]- Solution
> **Dockerfile (UNOPTIMIZED - bad caching):**
> ```dockerfile
> FROM golang:1.22 AS builder
> WORKDIR /app
> COPY . .
> RUN go mod download
> RUN CGO_ENABLED=0 go build -o server .
> 
> FROM alpine:3.19
> COPY --from=builder /app/server /server
> CMD ["/server"]
> ```
> 
> **Dockerfile (OPTIMIZED - good caching):**
> ```dockerfile
> FROM golang:1.22 AS builder
> WORKDIR /app
> 
> # Copy dependency files first
> COPY go.mod go.sum ./
> RUN go mod download
> 
> # Then copy source code
> COPY . .
> RUN CGO_ENABLED=0 go build -o server .
> 
> FROM alpine:3.19
> COPY --from=builder /app/server /server
> EXPOSE 8080
> CMD ["/server"]
> ```
> 
> **Test the caching:**
> ```bash
> # First build (downloads dependencies)
> docker build -t cache-test .
> 
> # Change main.go (e.g., change the greeting text)
> sed -i '' 's/Hello with mux!/Goodbye with mux!/' main.go
> 
> # Rebuild - "go mod download" layer should be CACHED
> docker build -t cache-test .
> # Look for: CACHED [builder 3/5] RUN go mod download
> ```

---

## P10: Local Registry

**Task:** Run a local Docker registry. Build your image, tag it for the local registry, and push it. Then pull it from the registry to verify.

**Expected result:**
- Local registry running on port 5000
- Image pushed to and pullable from `localhost:5000/hello:v1`

**Verification:**

```bash
curl http://localhost:5000/v2/_catalog
docker pull localhost:5000/hello:v1
```

> [!hint]- Hint
> - Start a registry: `docker run -d -p 5000:5000 --name registry registry:2`
> - Tag format: `localhost:5000/<name>:<tag>`
> - Use `docker tag` to create the registry-prefixed tag, then `docker push`
> - List images in registry: `curl http://localhost:5000/v2/_catalog`

> [!success]- Solution
> ```bash
> # Start local registry
> docker run -d -p 5000:5000 --name registry registry:2
> 
> # Build the image
> docker build -t hello-app .
> 
> # Tag for local registry (multiple versions)
> docker tag hello-app localhost:5000/hello:v1
> docker tag hello-app localhost:5000/hello:v2
> docker tag hello-app localhost:5000/hello:latest
> 
> # Push all tags
> docker push localhost:5000/hello:v1
> docker push localhost:5000/hello:v2
> docker push localhost:5000/hello:latest
> 
> # Verify - list repos in registry
> curl http://localhost:5000/v2/_catalog
> # {"repositories":["hello"]}
> 
> # List tags
> curl http://localhost:5000/v2/hello/tags/list
> # {"name":"hello","tags":["latest","v1","v2"]}
> 
> # Remove local image and pull from registry
> docker rmi localhost:5000/hello:v1
> docker pull localhost:5000/hello:v1
> 
> # Cleanup
> docker stop registry && docker rm registry
> ```

---

## P11: Image Security Scanning

**Task:** Scan your Docker image for vulnerabilities using `docker scout`. Identify a vulnerability and fix it by changing the base image.

**Expected result:**
- `docker scout` shows vulnerability report
- After switching to a more secure base image, vulnerability count decreases

**Verification:**

```bash
docker scout quickview hello-app
```

> [!hint]- Hint
> - Run `docker scout quickview <image>` for a summary
> - Run `docker scout cves <image>` for detailed CVEs
> - Alpine-based images generally have fewer vulnerabilities than Debian-based
> - Try switching from `alpine:3.19` to `alpine:3.20` or using `scratch` for Go binaries
> - `scratch` is the most secure - it's an empty image with zero OS packages

> [!success]- Solution
> ```bash
> # Scan the current image
> docker scout quickview hello-app
> docker scout cves hello-app
> 
> # Note the vulnerability count and CVE IDs
> ```
> 
> **Dockerfile using scratch (most secure for Go):**
> ```dockerfile
> FROM golang:1.22 AS builder
> WORKDIR /app
> COPY go.mod go.sum ./
> RUN go mod download
> COPY . .
> RUN CGO_ENABLED=0 GOOS=linux go build -o server .
> 
> FROM scratch
> COPY --from=builder /app/server /server
> EXPOSE 8080
> CMD ["/server"]
> ```
> 
> ```bash
> # Rebuild with scratch
> docker build -t hello-secure .
> 
> # Scan again
> docker scout quickview hello-secure
> # Should show 0 vulnerabilities (no OS packages)
> 
> # Compare sizes
> docker images | grep hello
> # hello-secure is even smaller than alpine-based
> ```

---

## P12: Non-Root User

**Task:** Create a non-root user in the Dockerfile and run the application as that user. Verify the container process is NOT running as root.

**Expected result:**
- `docker exec <container> whoami` returns `appuser` (not `root`)
- `docker exec <container> id` shows a non-zero UID

**Verification:**

```bash
docker exec <container> whoami
docker exec <container> id
docker top <container>
```

> [!hint]- Hint
> - In Alpine: `RUN addgroup -S appgroup && adduser -S appuser -G appgroup`
> - Place the `USER` directive AFTER copying files but BEFORE `CMD`
> - The user needs read/execute permissions on the binary
> - You cannot bind to ports below 1024 as non-root (8080 is fine)

> [!success]- Solution
> **Dockerfile:**
> ```dockerfile
> FROM golang:1.22 AS builder
> WORKDIR /app
> COPY go.mod go.sum ./
> RUN go mod download
> COPY . .
> RUN CGO_ENABLED=0 GOOS=linux go build -o server .
> 
> FROM alpine:3.19
> 
> # Create non-root user
> RUN addgroup -S appgroup && adduser -S appuser -G appgroup
> 
> # Copy binary
> COPY --from=builder /app/server /server
> 
> # Switch to non-root user
> USER appuser
> 
> EXPOSE 8080
> CMD ["/server"]
> ```
> 
> **Commands:**
> ```bash
> docker build -t hello-nonroot .
> docker run -d -p 8080:8080 --name hello hello-nonroot
> 
> # Verify non-root
> docker exec hello whoami
> # appuser
> 
> docker exec hello id
> # uid=100(appuser) gid=101(appgroup) groups=101(appgroup)
> 
> # Check process owner
> docker top hello
> # USER should be "appuser" not "root"
> 
> # App still works
> curl http://localhost:8080
> 
> # Cleanup
> docker stop hello && docker rm hello
> ```

---

## P13: Build Arguments (ARG)

**Task:** Parameterize the Go version at build time using `ARG`. Build the image with different Go versions and verify each one.

**Expected result:**
- `docker build --build-arg GO_VERSION=1.21 -t hello-go121 .` builds with Go 1.21
- `docker build --build-arg GO_VERSION=1.22 -t hello-go122 .` builds with Go 1.22
- Both images work correctly

**Verification:**

```bash
docker images | grep hello-go
```

> [!hint]- Hint
> - `ARG GO_VERSION=1.22` declares a build argument with a default
> - Use it in `FROM`: `FROM golang:${GO_VERSION} AS builder`
> - `ARG` must come BEFORE `FROM` if used in `FROM`
> - Build arguments are not available at runtime (unlike `ENV`)

> [!success]- Solution
> **Dockerfile:**
> ```dockerfile
> ARG GO_VERSION=1.22
> 
> FROM golang:${GO_VERSION} AS builder
> WORKDIR /app
> COPY go.mod go.sum ./
> RUN go mod download
> COPY . .
> RUN CGO_ENABLED=0 GOOS=linux go build -o server .
> 
> FROM alpine:3.19
> RUN addgroup -S appgroup && adduser -S appuser -G appgroup
> COPY --from=builder /app/server /server
> USER appuser
> EXPOSE 8080
> CMD ["/server"]
> ```
> 
> **Commands:**
> ```bash
> # Build with default Go version (1.22)
> docker build -t hello-go122 .
> 
> # Build with Go 1.21
> docker build --build-arg GO_VERSION=1.21 -t hello-go121 .
> 
> # Build with Go 1.23
> docker build --build-arg GO_VERSION=1.23 -t hello-go123 .
> 
> # Verify all images exist
> docker images | grep hello-go
> 
> # Test each
> docker run -d -p 8080:8080 --name test121 hello-go121
> curl http://localhost:8080
> docker stop test121 && docker rm test121
> 
> docker run -d -p 8080:8080 --name test122 hello-go122
> curl http://localhost:8080
> docker stop test122 && docker rm test122
> ```

---

## P14: Microservice Architecture with Docker Compose

**Task:** Create a `docker-compose.yml` that simulates a microservice architecture with:
1. **API Gateway** (nginx) - routes `/api/users` to user-service and `/api/orders` to order-service
2. **User Service** (Go HTTP server on port 8081)
3. **Order Service** (Go HTTP server on port 8082)
4. **PostgreSQL** database

Requirements:
- Proper networking: frontend network (gateway) and backend network (services + DB)
- Dependency ordering with health checks
- The gateway should be the only service exposed to the host

**Expected result:**
- `curl http://localhost/api/users` is proxied to user-service
- `curl http://localhost/api/orders` is proxied to order-service
- Database is not directly accessible from the host

**Verification:**

```bash
docker compose up -d
docker compose ps
curl http://localhost/api/users
curl http://localhost/api/orders
```

> [!hint]- Hint
> - Create separate directories: `user-service/`, `order-service/`
> - nginx config: use `proxy_pass` in location blocks
> - Define two networks: `frontend` and `backend`
> - Gateway connects to `frontend`; services connect to both; DB connects to `backend` only
> - Only the gateway maps ports to the host

> [!success]- Solution
> **Directory structure:**
> ```
> microservices/
> ├── docker-compose.yml
> ├── gateway/
> │   └── nginx.conf
> ├── user-service/
> │   ├── Dockerfile
> │   └── main.go
> └── order-service/
>     ├── Dockerfile
>     └── main.go
> ```
> 
> **user-service/main.go:**
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"net/http"
> )
> 
> func main() {
> 	http.HandleFunc("/api/users", func(w http.ResponseWriter, r *http.Request) {
> 		fmt.Fprintln(w, `{"service": "user-service", "users": ["alice", "bob"]}`)
> 	})
> 	http.ListenAndServe(":8081", nil)
> }
> ```
> 
> **user-service/Dockerfile:**
> ```dockerfile
> FROM golang:1.22 AS builder
> WORKDIR /app
> COPY main.go .
> RUN CGO_ENABLED=0 go build -o server main.go
> 
> FROM alpine:3.19
> RUN addgroup -S app && adduser -S app -G app
> COPY --from=builder /app/server /server
> USER app
> CMD ["/server"]
> ```
> 
> **order-service/main.go:**
> ```go
> package main
> 
> import (
> 	"fmt"
> 	"net/http"
> )
> 
> func main() {
> 	http.HandleFunc("/api/orders", func(w http.ResponseWriter, r *http.Request) {
> 		fmt.Fprintln(w, `{"service": "order-service", "orders": [{"id": 1, "item": "widget"}]}`)
> 	})
> 	http.ListenAndServe(":8082", nil)
> }
> ```
> 
> **order-service/Dockerfile:**
> ```dockerfile
> FROM golang:1.22 AS builder
> WORKDIR /app
> COPY main.go .
> RUN CGO_ENABLED=0 go build -o server main.go
> 
> FROM alpine:3.19
> RUN addgroup -S app && adduser -S app -G app
> COPY --from=builder /app/server /server
> USER app
> CMD ["/server"]
> ```
> 
> **gateway/nginx.conf:**
> ```nginx
> events {
>     worker_connections 1024;
> }
> 
> http {
>     upstream user_service {
>         server user-service:8081;
>     }
> 
>     upstream order_service {
>         server order-service:8082;
>     }
> 
>     server {
>         listen 80;
> 
>         location /api/users {
>             proxy_pass http://user_service;
>             proxy_set_header Host $host;
>             proxy_set_header X-Real-IP $remote_addr;
>         }
> 
>         location /api/orders {
>             proxy_pass http://order_service;
>             proxy_set_header Host $host;
>             proxy_set_header X-Real-IP $remote_addr;
>         }
>     }
> }
> ```
> 
> **docker-compose.yml:**
> ```yaml
> services:
>   gateway:
>     image: nginx:alpine
>     ports:
>       - "80:80"
>     volumes:
>       - ./gateway/nginx.conf:/etc/nginx/nginx.conf:ro
>     networks:
>       - frontend
>     depends_on:
>       user-service:
>         condition: service_started
>       order-service:
>         condition: service_started
> 
>   user-service:
>     build: ./user-service
>     networks:
>       - frontend
>       - backend
>     depends_on:
>       postgres:
>         condition: service_healthy
> 
>   order-service:
>     build: ./order-service
>     networks:
>       - frontend
>       - backend
>     depends_on:
>       postgres:
>         condition: service_healthy
> 
>   postgres:
>     image: postgres:16-alpine
>     environment:
>       POSTGRES_PASSWORD: secret
>       POSTGRES_DB: microservices
>     volumes:
>       - pgdata:/var/lib/postgresql/data
>     networks:
>       - backend
>     healthcheck:
>       test: ["CMD-SHELL", "pg_isready -U postgres"]
>       interval: 5s
>       timeout: 5s
>       retries: 5
>       start_period: 10s
> 
> networks:
>   frontend:
>   backend:
> 
> volumes:
>   pgdata:
> ```
> 
> **Commands:**
> ```bash
> # Start everything
> docker compose up -d --build
> 
> # Wait for health checks
> docker compose ps
> 
> # Test routing through gateway
> curl http://localhost/api/users
> # {"service": "user-service", "users": ["alice", "bob"]}
> 
> curl http://localhost/api/orders
> # {"service": "order-service", "orders": [{"id": 1, "item": "widget"}]}
> 
> # Verify DB is NOT accessible from host
> # (No ports mapped for postgres - this should fail)
> psql -h localhost -U postgres 2>&1 || echo "Cannot connect - correct!"
> 
> # Cleanup
> docker compose down -v
> ```
