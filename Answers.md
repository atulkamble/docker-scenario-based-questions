**answers + quick explanations** for every scenario in your set. I kept them crisp, command-first, and CI/CD-friendly so you can drop them into your Cloudnautic training kit.

---

# 🧩 Beginner-Level Scenarios — Answers

1. **Basic container start**
   **Answer:**

```bash
docker inspect --format '{{.Config.Entrypoint}} {{.Config.Cmd}}' <image>
# or
docker image inspect <image> | jq '.[0].Config | {Entrypoint, Cmd}'
```

**Why:** Shows the image’s default ENTRYPOINT/CMD so you know what runs if you don’t override it.

2. **Container restart policy**
   **Answer:**

```bash
docker run --restart unless-stopped ...
# or later
docker update --restart unless-stopped <container>
# Compose:
# restart: unless-stopped
```

**Why:** Ensures auto-restart on crashes while still allowing manual stops.

3. **Inspect logs**
   **Answer:**

```bash
docker ps -a
docker logs -f <container>
docker inspect --format '{{.State.ExitCode}} {{.State.Error}}' <container>
docker inspect <container> | jq '.[0].State'
# Optional interactive debug:
docker run -it --entrypoint sh <image>
```

**Why:** Check history, live logs, exit code, and drop into a shell to reproduce locally.

4. **Networking**
   **Answer:**

```bash
docker network create appnet
docker run -d --network appnet --name web <image>
docker run -d --network appnet --name db  <image>
# Compose: put both services on the same user-defined network
```

**Why:** User-defined networks give automatic DNS by service/container name.

5. **Port conflicts (8080 in use)**
   **Answer:**

```bash
# Find the process
lsof -i :8080   # macOS/Linux
netstat -tulpn | grep :8080  # Linux
# Use a different host port
docker run -p 8081:8080 <image>
# Or stop the offender
docker ps --filter publish=8080
docker stop <container>
```

**Why:** Free the port or remap it to avoid collision.

6. **Tagging images**
   **Answer:** Default tag = `latest`.

```bash
docker images
docker tag <image_or_id> repo/app:v1.0
docker push repo/app:v1.0
```

**Why:** Explicit version tags make releases reproducible.

7. **Container cleanup**
   **Answer:**

```bash
# Safe, targeted
docker container prune        # stopped containers
docker image prune            # dangling layers
docker volume prune           # dangling volumes
docker network prune          # unused networks
# One-shot (be careful)
docker system prune -a --volumes
```

**Why:** Reclaims space without nuking in-use artifacts.

---

# ⚙️ Intermediate-Level Scenarios — Answers

1. **Dockerfile optimization**
   **Answer (tactics):**

* Multi-stage builds
* Order RUN/COPY to maximize layer cache
* Use `.dockerignore`
* Smaller base images (alpine/distroless/slim)
* BuildKit cache mounts (apt/npm/pip):

  ```Dockerfile
  RUN --mount=type=cache,target=/var/cache/apt apt-get update && apt-get install -y ...
  ```
* Build with cache:

  ```bash
  docker buildx build --cache-from=type=registry,ref=repo/app:cache \
                      --cache-to=type=registry,ref=repo/app:cache,mode=max ...
  ```

2. **Sensitive credentials**
   **Answer:**

* **Never** bake secrets into images or `ENV` in Dockerfile
* Pass at runtime via env/`--env-file` managed by CI secret store
* Use **BuildKit secrets** for build-time creds:

  ```bash
  docker build --secret id=npmrc,src=$HOME/.npmrc .
  # Dockerfile
  RUN --mount=type=secret,id=npmrc cat /run/secrets/npmrc > ~/.npmrc
  ```
* In orchestration: Docker Swarm/K8s Secrets.

3. **Persist logs**
   **Answer:**

```bash
docker volume create app-logs
docker run -v app-logs:/app/logs <image>
# or host bind
docker run -v /var/log/myapp:/app/logs <image>
```

**Why:** Volumes survive container deletion.

4. **Compose dependency (web waits for db)**
   **Answer:** Add a **healthcheck** to `db`, then:

```yaml
services:
  db:
    image: postgres
    healthcheck:
      test: ["CMD-SHELL","pg_isready -U postgres"]
      interval: 5s
      retries: 10
  web:
    depends_on:
      db:
        condition: service_healthy
```

**Why:** Start ordering + readiness, not just “starts before.”

5. **Shell access**
   **Answer:**

```bash
docker exec -it <container> /bin/bash   # or /bin/sh
```

**Why:** Exec gives an interactive shell without attaching STDIN to main process.

6. **Container → host networking**
   **Answer:**

* Expose host service on `0.0.0.0`
* Use `host.docker.internal` (Docker Desktop/macOS/Win)
* Linux (Engine ≥20.10):

  ```bash
  docker run --add-host=host.docker.internal:host-gateway ...
  ```
* Last resort (Linux only): `--network host` (has caveats).

7. **Version mismatch (Jenkins vs local)**
   **Answer:**

* Compare versions: `docker version && docker info` on both
* Standardize builds in CI using a **containerized builder** (e.g., `docker:26-dind` or `buildx`)
* Pin engine/CLI versions on agents or use a **tooling container**
* Document features (BuildKit) and set env (`DOCKER_BUILDKIT=1`).

---

# 🧠 Advanced / Real-World Scenarios — Answers

1. **Docker in CI/CD (build & push on git push)**
   **GitHub Actions (minimal):**

```yaml
name: build-and-push
on: [push]
jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: docker.io
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: dockerhubuser/app:${{ github.sha }},dockerhubuser/app:latest
```

**Jenkins (declarative):**

```groovy
pipeline {
  agent any
  environment {
    IMAGE = "dockerhubuser/app"
  }
  stages {
    stage('Build & Push') {
      steps {
        script {
          docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-creds') {
            def img = docker.build("${IMAGE}:${env.BUILD_NUMBER}")
            img.push()
            img.push('latest')
          }
        }
      }
    }
  }
}
```

2. **Security hardening / scanning**
   **Answer:**

* Scan in CI with **Trivy/Grype/Snyk** and fail on high CVEs:

  ```bash
  trivy image --exit-code 1 --severity HIGH,CRITICAL repo/app:latest
  ```
* Add Dockerfile best practices: non-root user, minimal base, drop caps, read-only FS.

3. **Custom base images (consistency & updates)**
   **Answer:**

* Central base repo (`org/base:<version>`), labels + SBOM
* Dependabot/Renovate to bump `FROM org/base:x.y` in downstreams
* Registry webhook → trigger rebuilds of dependents
* Periodic rebuild + scan pipeline; deprecate old tags with policy.

4. **Multi-container debugging (timeouts)**
   **Answer:**

* `docker compose ps`, `docker compose logs -f`
* Check healthchecks; verify service DNS:

  ```bash
  docker exec -it web sh
  getent hosts db
  nc -zv db 5432
  ```
* Verify ports/env; try from each container to others; isolate the slow/unhealthy service.

5. **Resource limits**
   **Answer:**

```bash
docker run --cpus=1.5 --memory=512m --memory-swap=512m <image>
```

**Compose (Swarm mode):**

```yaml
deploy:
  resources:
    limits:
      cpus: '1.5'
      memory: 512M
```

**Note:** Non-Swarm Compose ignores `deploy.*`; use `docker run` flags or move to Swarm/K8s.

6. **Private registries (auth in CI)**
   **Answer:**

* **GitHub Actions:** `docker/login-action` with repo secrets
* **Jenkins:** `withCredentials([usernamePassword(...)])` + `docker.withRegistry(...)`
* Use least-privileged robot accounts/tokens; avoid plain passwords; rotate creds.

7. **Rolling updates (no downtime)**
   **Answer (Compose/Swarm):**

```yaml
deploy:
  replicas: 3
  update_config:
    order: start-first
    parallelism: 1
    delay: 10s
  healthcheck: { ... }
```

**Why:** Start new task before stopping the old; rely on healthchecks and multiple replicas.

8. **DNS issues on custom network**
   **Answer:**

* Check `/etc/resolv.conf` inside container; Docker’s embedded DNS is `127.0.0.11`
* Ensure user-defined network (not `bridge`)
* Test resolution: `getent hosts service`
* If needed provide upstream DNS: `--dns 1.1.1.1` on network or daemon
* Restart Docker if daemon DNS is stale.

9. **Cross-platform builds (buildx)**
   **Answer:**

```bash
docker buildx create --use --name xbuilder
docker buildx build --platform linux/amd64,linux/arm64 \
  -t repo/app:1.0 --push .
```

**Why:** QEMU emulation + multi-arch manifest in one push.

10. **Container crash loop**
    **Answer:**

```bash
docker inspect --format '{{.RestartCount}} {{.State.Error}}' <container>
docker logs <container>
docker run -it --entrypoint sh <image>  # debug entrypoint/cmd
```

**Fix:** Remove bad restart policy, fix app/env/entrypoint, validate required services are reachable.

---

# 🧰 Expert-Level (DevOps & Cloud) — Answers

1. **ECR / ACR via CI/Terraform**
   **AWS ECR (CLI snippet):**

```bash
aws ecr create-repository --repository-name app || true
aws ecr get-login-password | docker login --username AWS --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
docker build -t <acct>.dkr.ecr.<region>.amazonaws.com/app:latest .
docker push <acct>.dkr.ecr.<region>.amazonaws.com/app:latest
```

**Terraform (ECR minimal):**

```hcl
resource "aws_ecr_repository" "app" { name = "app" image_scanning_configuration { scan_on_push = true } }
```

**Azure ACR (CLI snippet):**

```bash
az acr create -n myacr -g rg --sku Basic
az acr login -n myacr
docker build -t myacr.azurecr.io/app:latest .
docker push myacr.azurecr.io/app:latest
```

Integrate with Actions/Jenkins using cloud credential actions/credentials bindings.

2. **Compose → Kubernetes**
   **Answer:**

* Quick path: **kompose**

  ```bash
  kompose convert -f docker-compose.yml
  kubectl apply -f .
  ```
* Manual: map services→Deployments/Services, env→ConfigMaps/Secrets, volumes→PVCs, healthchecks→probes.

3. **Multi-stage pipelines + cache reuse (Jenkins)**
   **Answer:**

* Use **buildx** with registry cache:

  ```bash
  docker buildx build \
    --cache-from=type=registry,ref=repo/app:cache \
    --cache-to=type=registry,ref=repo/app:cache,mode=max \
    -t repo/app:${BUILD_NUMBER} --push .
  ```
* Or mount a persistent Docker layer cache volume on agents.

4. **Container logging & monitoring**
   **Answer:**

* **CloudWatch driver:**

  ```bash
  docker run --log-driver awslogs \
    --log-opt awslogs-group=mygroup \
    --log-opt awslogs-region=ap-south-1 \
    --log-opt awslogs-stream=app \
    <image>
  ```
* **Prometheus/Grafana:** run **cAdvisor** + node-exporter; app exposes `/metrics`; scrape via Prometheus → Grafana dashboards.

5. **Image signing & verification**
   **Answer (Sigstore Cosign):**

```bash
cosign sign docker.io/user/app:1.0
cosign verify docker.io/user/app:1.0
```

Enforce via admission policies (Kyverno/OPA) or cosigned verifier in K8s.

6. **Base image CVE patching at scale**
   **Answer:**

* Rebuild with **new base tag**; avoid `latest`
* Automate PRs with Renovate/Dependabot for `FROM ...`
* Registry webhook → trigger downstream image rebuilds
* Gate deploy with scanners (Trivy/Snyk) + policy (fail on HIGH/CRITICAL).

7. **Remote Docker layer caching in CI**
   **Answer (GitHub Actions example):**

```yaml
- uses: docker/build-push-action@v6
  with:
    push: true
    tags: repo/app:${{ github.sha }}
    cache-from: type=registry,ref=repo/app:cache
    cache-to:   type=registry,ref=repo/app:cache,mode=max
```

**Jenkins:** same `buildx build` flags; or mount a shared cache volume across runs.

---
