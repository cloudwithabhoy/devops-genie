# Docker — Medium Questions

---

## 1. How do you optimize Docker images and reduce their size?

This comes up in every DevOps interview, and most people give a textbook answer — "use Alpine, use multi-stage builds." The follow-up question is always "did you actually do this in production and what was the impact?" Here's the real answer.

**Start by understanding what's making your image large.**

Before optimizing, measure:
```bash
docker image inspect my-app:latest --format='{{.Size}}'
docker history my-app:latest     # Shows the size contribution of each layer
dive my-app:latest               # Tool that gives a layer-by-layer breakdown interactively
```

We had a Python Flask app at 1.4 GB. After the steps below, we got it to 180 MB. This reduced ECR storage costs, but more importantly, cold starts on new pod scheduling dropped from 45 seconds (image pull) to 8 seconds. That's a real operational benefit.

---

**Strategy 1: Multi-stage builds — the biggest win.**

The build environment (compilers, test tools, build dependencies) doesn't need to be in the production image.

**Before (single stage, 1.4 GB):**
```dockerfile
FROM python:3.11
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt     # Includes dev deps like pytest, coverage
COPY . .
CMD ["python", "app.py"]
```

**After (multi-stage, 210 MB):**
```dockerfile
# Stage 1: Build dependencies
FROM python:3.11 AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Stage 2: Production image — only runtime artifacts come here
FROM python:3.11-slim AS production
WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app/ ./app/
ENV PATH=/root/.local/bin:$PATH
USER nobody
CMD ["python", "-m", "app"]
```

The final image has no pip cache, no build tools, no test dependencies. The `builder` stage is discarded entirely.

---

**Strategy 2: Use slim or Alpine base images.**

```dockerfile
# Full image: 900 MB base
FROM python:3.11

# Slim image: 125 MB base (removes build tools, docs, unused libs)
FROM python:3.11-slim

# Alpine image: 50 MB base (uses musl libc instead of glibc)
FROM python:3.11-alpine
```

**Caveat with Alpine:** Some Python packages (especially those with C extensions like `cryptography`, `psycopg2`) need to be compiled from source on Alpine because pre-built wheels assume glibc. This means `pip install` takes much longer on Alpine and sometimes fails. We switched from Alpine to `slim` after wasting half a day getting `cryptography` to compile on Alpine.

**Rule of thumb:** Use `-slim` for Python/Node apps. Use Alpine only if your app has no C extension dependencies.

---

**Strategy 3: Order layers by change frequency.**

Docker caches layers. If you copy your source code before installing dependencies, every code change invalidates the dependency layer and forces a full reinstall.

**Wrong order:**
```dockerfile
COPY . .                    # Code changes → cache miss
RUN pip install -r requirements.txt  # Reinstalls everything on every build
```

**Right order:**
```dockerfile
COPY requirements.txt .     # Only changes when deps change
RUN pip install -r requirements.txt  # Cached unless requirements.txt changes
COPY . .                    # Code changes only invalidate this layer
```

This single change cut our CI build time from 4 minutes to 40 seconds for most commits.

---

**Strategy 4: Use `.dockerignore` to exclude junk.**

Without `.dockerignore`, Docker sends your entire build context to the daemon — including `node_modules`, `.git`, test fixtures, and local env files.

```
# .dockerignore
.git
.github
node_modules
__pycache__
*.pyc
.env
.env.local
tests/
docs/
*.log
```

We found a Node app whose build context was 800 MB because `node_modules` was being copied in before the layer was rebuilt. With `.dockerignore`, the context dropped to 2 MB and `docker build` started in under a second instead of 30 seconds.

---

**Strategy 5: Clean up in the same RUN layer.**

Each `RUN` instruction creates a new layer. If you install packages and then delete the cache in separate instructions, the deleted files still exist in the earlier layer.

**Wrong:**
```dockerfile
RUN apt-get update && apt-get install -y curl
RUN rm -rf /var/lib/apt/lists/*   # Too late — the cache is in the previous layer
```

**Right:**
```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*   # Same layer — cache is never committed
```

---

**Strategy 6: Run as non-root.**

Not a size optimization, but a security one that every production image should have:

```dockerfile
RUN useradd --create-home --shell /bin/bash appuser
USER appuser
```

Or use the existing `nobody` user for simple cases. Running as root inside a container is a security risk — if someone escapes the container, they have root on the host. We failed a security audit because 6 of our production images ran as root.

---

**Summary of what we actually did and the impact:**

| Technique | Size Before | Size After |
|-----------|-------------|------------|
| Multi-stage build | 1.4 GB | 210 MB |
| Switch to slim base | 210 MB | 180 MB |
| Layer ordering | (no size change) | Build time: 4 min → 40 sec |
| .dockerignore | 800 MB context | 2 MB context |

**Real production impact:** Smaller images = faster ECR pulls = faster pod startup during scaling events. When Karpenter adds a new node and 10 pods need to pull the image simultaneously, a 180 MB image pulls in 6 seconds vs 45 seconds for 1.4 GB. During a traffic spike, that 39-second difference matters.

---

## 2. What are the stages involved in building a Docker image?

Most people say "docker build" and move on. The real answer explains what happens inside that command — and more importantly, what a production-grade image build pipeline looks like end to end.

**What happens during `docker build`:**

Docker reads the Dockerfile top to bottom. Each instruction creates a new layer:

1. **Base image pull** — `FROM python:3.11-slim`. Docker checks the local cache first. If not cached, pulls from the registry.

2. **Layer execution** — Each `RUN`, `COPY`, `ADD` instruction executes and commits a new read-only layer. Docker uses a union filesystem (OverlayFS) — layers stack on top of each other.

3. **Cache evaluation** — Before executing each instruction, Docker checks if an identical layer already exists in cache. Cache key = instruction text + parent layer hash. If the cache hits, Docker reuses the layer instead of rebuilding it. This is why layer ordering matters.

4. **Image ID generation** — After all layers are built, Docker generates a SHA256 hash of the final image. This is the image ID.

**The full build pipeline we run in CI (not just `docker build`):**

```
Stage 1: Lint Dockerfile
  hadolint Dockerfile
  # Catches common mistakes: apt-get without --no-install-recommends,
  # COPY . before dependency install (breaks cache), using ADD instead of COPY, etc.

Stage 2: Build image
  docker build \
    --build-arg APP_VERSION=$GIT_COMMIT \
    --label "git.commit=$GIT_COMMIT" \
    --label "build.date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    -t $ECR_REPO:$GIT_COMMIT .

Stage 3: Scan image
  trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed $ECR_REPO:$GIT_COMMIT

Stage 4: Test image
  docker run --rm $ECR_REPO:$GIT_COMMIT python -c "import app; print('import OK')"
  # Basic smoke test — does the app actually start?

Stage 5: Push to registry
  docker push $ECR_REPO:$GIT_COMMIT
  # Also tag as :latest for convenience (but we never deploy :latest)
  docker tag $ECR_REPO:$GIT_COMMIT $ECR_REPO:latest
  docker push $ECR_REPO:latest
```

**Labels** — we add `--label` to every image so we can trace any running container back to its source:

```bash
docker inspect <container> | jq '.[0].Config.Labels'
# {
#   "git.commit": "abc123",
#   "build.date": "2024-01-15T14:32:00Z"
# }
```

This saved us during a security audit — we could prove exactly which commit every running container came from.

**Real scenario:** We had a build pipeline that skipped the `hadolint` stage. A developer wrote `RUN apt-get update && apt-get install curl` without `--no-install-recommends`. The final image was 200MB larger than necessary. `hadolint` would have flagged this on the first build. After adding it, we caught 3 similar issues across different services in the first week.

---

## 3. What is the difference between ENTRYPOINT and CMD in Docker?

This is a classic trap question. Most people get the surface-level answer right but miss the interaction between them — which is what interviewers are testing.

**CMD — default arguments, easily overridden:**

```dockerfile
CMD ["python", "app.py"]
```

`CMD` defines what runs when the container starts — but it's completely replaceable. If you do `docker run my-image python debug.py`, the CMD is ignored and `python debug.py` runs instead.

**ENTRYPOINT — fixed executable, not easily overridden:**

```dockerfile
ENTRYPOINT ["python"]
CMD ["app.py"]
```

Now the container always runs `python`. `CMD` becomes the default argument to the entrypoint. Running `docker run my-image debug.py` executes `python debug.py` — you're passing a new argument, not replacing the command. To override the entrypoint entirely, you need `--entrypoint`: `docker run --entrypoint bash my-image`.

**The interaction — this is where most answers fall short:**

When both are defined, they combine: `ENTRYPOINT` + `CMD` = the full command.

```dockerfile
ENTRYPOINT ["python"]       # Fixed: always runs python
CMD ["app.py"]              # Default arg: python app.py
```

`docker run my-image` → runs `python app.py`
`docker run my-image debug.py` → runs `python debug.py` (CMD replaced, ENTRYPOINT kept)
`docker run --entrypoint bash my-image` → runs `bash` (ENTRYPOINT replaced entirely)

**Shell form vs exec form — the part that trips people up in production:**

```dockerfile
# Exec form (recommended): runs the process directly, PID 1
ENTRYPOINT ["python", "app.py"]

# Shell form: runs via /bin/sh -c, your process is NOT PID 1
ENTRYPOINT python app.py
```

Shell form means your process runs as a child of `/bin/sh`. When Kubernetes sends `SIGTERM` to gracefully shut down the container, `/bin/sh` receives it — but it doesn't forward signals to child processes by default. Your app never receives SIGTERM, never shuts down gracefully, and gets `SIGKILL`'d after the grace period. We had 504 errors during every deployment because of this — old pods were getting force-killed mid-request instead of finishing existing requests.

Fix: always use exec form for `ENTRYPOINT` and `CMD`.

**What we use in production:**

```dockerfile
# For long-running services
ENTRYPOINT ["python", "-m", "gunicorn"]
CMD ["--bind", "0.0.0.0:8080", "--workers", "4", "app:create_app()"]

# Running locally with different args:
docker run my-app --workers 1 --reload   # Overrides CMD, keeps ENTRYPOINT
```

**Real scenario:** A Node.js service had `CMD node server.js` (shell form). During rolling deployments, terminating pods took the full 30-second grace period every time — Node.js never received SIGTERM. Switching to `CMD ["node", "server.js"]` (exec form) meant the app received SIGTERM, gracefully closed connections, and terminated in under 2 seconds. Deployments that used to take 10 minutes now complete in 3.

---

## 4. Which container registry do you use, and how do you manage images?

We use **Amazon ECR (Elastic Container Registry)** as our primary registry. Here's why we chose it and how we manage it — not just "we use ECR."

**Why ECR over Docker Hub or other registries:**

- **IAM-based authentication** — no registry username/password to manage. EKS nodes authenticate via their IAM instance profile. CI authenticates via an IAM role. No credentials to rotate, no secrets to store.
- **Private by default** — no accidental public exposure. Docker Hub repos are public unless you explicitly make them private (and free tier limits private repos).
- **Same VPC** — EKS nodes pull images without leaving the AWS network. No NAT Gateway costs for image pulls (with a VPC endpoint for ECR). Fast pull speeds within the region.
- **Integrated with Trivy and other scanners** — ECR has its own basic vulnerability scanning (uses Clair) that runs automatically on push. We also run Trivy in CI before pushing for more comprehensive results.

**How we structure ECR repos:**

One repo per service. We use immutable image tags — once an image with tag `abc123` is pushed, you can't overwrite it with a different image using the same tag. This prevents silent tag overwrites.

```bash
# Each service gets its own ECR repo
123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service
123456789.dkr.ecr.ap-south-1.amazonaws.com/payment-service
123456789.dkr.ecr.ap-south-1.amazonaws.com/auth-service
```

**Lifecycle policies — critical for cost control:**

Without lifecycle policies, ECR accumulates images forever. We had one service with 4,000 untagged images from 18 months of CI builds. ECR charges per GB stored.

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Delete untagged images older than 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
```

We manage these policies in Terraform so every new ECR repo gets lifecycle rules automatically.

**Authenticating from Jenkins:**

```bash
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.ap-south-1.amazonaws.com
```

The ECR token expires after 12 hours — we always run this command at the start of the push stage, never cache the token.

---

## 5. How do you pass environment variables during Docker build and runtime?

There are two different mechanisms — `ARG` for build time, `ENV` for runtime — and mixing them up causes real production issues.

**Build-time variables: `ARG`**

`ARG` variables are only available during the `docker build` process. They're passed with `--build-arg` and don't persist in the final image.

```dockerfile
ARG APP_VERSION=dev
ARG BUILD_DATE

RUN echo "Building version $APP_VERSION on $BUILD_DATE"
LABEL version="$APP_VERSION"
```

```bash
docker build \
  --build-arg APP_VERSION=$GIT_COMMIT \
  --build-arg BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ) \
  -t my-app:$GIT_COMMIT .
```

**`ARG` values are NOT in the final image** (except via `LABEL` or `ENV`). If you inspect the image, you won't find them. This is intentional — you don't want build-time credentials (npm tokens, pip credentials) baked into the image.

**Common trap:** If you set a `ARG` and then do `ENV MY_VAR=$MY_ARG`, the value IS baked into the image and visible in `docker history`. Never pass secrets this way.

**Runtime variables: `ENV`**

`ENV` variables are baked into the image and available to the running process.

```dockerfile
ENV APP_PORT=8080
ENV LOG_LEVEL=info
ENV PYTHONUNBUFFERED=1    # Common for Python — ensures stdout isn't buffered
```

Override at runtime:
```bash
docker run -e LOG_LEVEL=debug -e APP_PORT=9090 my-app
```

In Kubernetes:
```yaml
env:
  - name: LOG_LEVEL
    value: "debug"
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: db-host
```

**Secrets at runtime — what we actually do in production:**

Never put secrets in `ENV` in the Dockerfile (baked into image, visible in `docker history`). Never pass them as `--build-arg` unless the arg is consumed and not stored.

For runtime secrets, we use one of:

1. **Kubernetes Secrets** (for K8s deployments) — mounted as env vars via `valueFrom.secretKeyRef`. The secret value is base64-encoded in etcd, not in the image.

2. **AWS Secrets Manager via External Secrets Operator** — the secret lives in AWS, ESO syncs it to a K8s Secret, the pod mounts it. Automatic rotation, no manual updates.

3. **Environment files** (for local development only):
```bash
docker run --env-file .env my-app    # .env file never committed to git
```

**Real incident caused by mixing these up:** A developer needed an NPM token to install private packages during `docker build`. They passed it as `--build-arg NPM_TOKEN=...` and then did `ENV NPM_TOKEN=$NPM_TOKEN`. The token was baked into every image layer. A security scan flagged it — the token was visible in `docker history my-app`. We had to rotate the NPM token and rebuild all images. Fix: use multi-stage builds — the `ARG` and npm install happen in the builder stage, the final stage doesn't copy the token.

---

## 6. How do you perform Docker image vulnerability scanning during build time and at the registry level?

Scanning at build time catches issues before they're deployed. Scanning at the registry level catches new CVEs that appear after an image is already in use. You need both.

**Build-time scanning with Trivy:**

Trivy is what we use. It's fast, has a good CVE database, and has sensible defaults.

```bash
# In Jenkins CI, after docker build, before docker push
trivy image \
  --exit-code 1 \               # Fail the build if vulnerabilities found
  --severity CRITICAL,HIGH \    # Only fail on serious issues
  --ignore-unfixed \            # Skip CVEs with no fix available
  --format table \              # Human-readable output in CI logs
  $ECR_REPO:$GIT_COMMIT
```

`--ignore-unfixed` is important — without it, Trivy fails on CVEs that don't have a fix yet (e.g., a vulnerability in a library where the maintainer hasn't released a patch). This blocks every build even though there's nothing you can do about it. We fail only on fixable critical/high CVEs.

**In the Jenkinsfile as a parallel stage:**

```groovy
stage('Security Scan') {
  parallel {
    stage('Trivy - OS packages') {
      steps {
        sh '''
          trivy image \
            --exit-code 1 \
            --severity CRITICAL,HIGH \
            --ignore-unfixed \
            --vuln-type os \
            $ECR_REPO:$IMAGE_TAG
        '''
      }
    }
    stage('Trivy - App dependencies') {
      steps {
        sh '''
          trivy image \
            --exit-code 1 \
            --severity CRITICAL \
            --ignore-unfixed \
            --vuln-type library \
            $ECR_REPO:$IMAGE_TAG
        '''
      }
    }
  }
}
```

Separating OS and library scans lets us have different severity thresholds — CRITICAL+HIGH for OS packages, CRITICAL-only for libraries (library vulnerabilities are often theoretical and the actual exploit path might not apply).

**Registry-level scanning with ECR:**

ECR has built-in scanning (powered by Clair/Snyk). Enable enhanced scanning:

```hcl
# Terraform
resource "aws_ecr_registry_scanning_configuration" "main" {
  scan_type = "ENHANCED"   # Uses Snyk for more comprehensive scanning
  rule {
    scan_frequency = "CONTINUOUS_SCAN"   # Re-scans existing images when new CVEs are published
    repository_filter {
      filter      = "*"
      filter_type = "WILDCARD"
    }
  }
}
```

`CONTINUOUS_SCAN` means even if an image passed Trivy at build time, ECR will flag it if a new CVE is published against a package in that image. We get an EventBridge event → SNS → Slack alert when a critical CVE appears in a deployed image.

**The gap between build-time and registry scanning:**

Build-time Trivy runs against the CVE database at the moment of the build. An image built 3 months ago passed all checks — but a new CVE published last week might affect it. Registry-level continuous scanning covers this gap. We had an image in production for 6 weeks that was clean at build time. ECR's continuous scan flagged a critical CVE in OpenSSL that was published 2 weeks after the image was built. We rebuilt and redeployed within the hour.

**Tools summary:**

| Tool | Where | What it scans |
|------|-------|---------------|
| Trivy | CI pipeline (build time) | OS packages, language dependencies, config files |
| ECR Enhanced Scanning | Registry (post-push + continuous) | OS packages, new CVEs on existing images |
| Snyk (optional) | CI or IDE | Language dependencies (deep SBOM analysis) |
| Checkov | CI pipeline | Dockerfile misconfigurations, IaC |

---

## 7. How do you manage multi-container dependencies using Docker Compose?

Docker Compose lets you define multi-container apps in a single `docker-compose.yml`. Managing dependencies means controlling startup order and health-based readiness.

**Basic dependency: `depends_on`.**

```yaml
version: "3.9"
services:
  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - db
      - redis

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret

  redis:
    image: redis:7-alpine
```

`depends_on` only waits for the container to start — not for the service inside to be ready. Postgres takes 2–3 seconds to accept connections after starting.

**Production-ready: `depends_on` with `condition: service_healthy`.**

```yaml
services:
  app:
    build: .
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 10s     # Give Postgres time to start before first check

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
```

With `condition: service_healthy`, the `app` container only starts after both `db` and `redis` pass their health checks. No more "connection refused on startup" errors.

**Useful Compose commands:**

```bash
docker compose up -d              # Start all services in background
docker compose up --build         # Rebuild images before starting
docker compose ps                 # Show service status + health
docker compose logs -f app        # Follow logs for one service
docker compose restart app        # Restart a single service
docker compose down -v            # Stop + remove containers and volumes
```

---

## 8. How do you monitor container performance in production?

Container monitoring has three layers: host-level metrics, container-level metrics, and application-level metrics.

**Layer 1: Container-level metrics — Docker stats.**

```bash
# Real-time resource usage for all containers:
docker stats

# CONTAINER ID   NAME     CPU %   MEM USAGE / LIMIT   MEM %   NET I/O   BLOCK I/O
# a1b2c3d4e5f6   myapp    12.5%   256MiB / 512MiB     50%     1.2MB/s   45kB/s
```

**Layer 2: cAdvisor — container metrics for Prometheus.**

cAdvisor runs as a container on each host and exposes Docker metrics for Prometheus to scrape:

```yaml
# docker-compose.yml
cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  ports:
    - "8080:8080"
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
```

Key metrics exposed:
- `container_cpu_usage_seconds_total` — CPU usage per container
- `container_memory_usage_bytes` — memory usage
- `container_network_receive_bytes_total` — network in
- `container_fs_writes_bytes_total` — disk I/O

**Layer 3: Prometheus + Grafana dashboard.**

```yaml
# Prometheus scrape config for cAdvisor
scrape_configs:
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
```

Grafana dashboard shows: per-container CPU/memory trend, OOM events, restart count, network throughput.

**Layer 4: Log aggregation.**

```bash
# Forward Docker logs to a central system:
docker run --log-driver=awslogs \
  --log-opt awslogs-group=/prod/myapp \
  --log-opt awslogs-region=ap-south-1 \
  myapp:1.0

# Or use Fluentd/Fluent Bit as a sidecar or log driver
```

**Key alerts to set up:**

```yaml
# Container memory near limit → risk of OOMKill
alert: ContainerMemoryHigh
expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.85

# Container restarting frequently
alert: ContainerRestartLoop
expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
```
