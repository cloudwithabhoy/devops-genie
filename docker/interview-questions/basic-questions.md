# Docker — Basic Questions

---

## 1. Write a basic Dockerfile.

This is a trick question — a "basic" Dockerfile that works in production is not what most people write in tutorials. Here's the difference.

**What most people write (works, but don't use in production):**

```dockerfile
FROM python:3.12
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
EXPOSE 8080
CMD ["python", "app.py"]
```

This works, but the image is ~1.2GB, runs as root, and rebuilds all pip dependencies even if only `app.py` changed.

**What we actually deploy (production-grade, same app):**

```dockerfile
# ---- Builder stage ----
FROM python:3.12-slim AS builder

WORKDIR /app

# Copy requirements first — Docker caches this layer
# Only rebuilds pip install when requirements.txt changes
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ---- Final stage ----
FROM python:3.12-slim

WORKDIR /app

# Copy installed packages from builder
COPY --from=builder /install /usr/local

# Copy only application code — not tests, docs, .git, etc.
COPY src/ ./src/

# Create a non-root user and switch to it
RUN adduser --disabled-password --gecos "" appuser
USER appuser

EXPOSE 8080

# Use exec form — process gets PID 1, receives SIGTERM cleanly
ENTRYPOINT ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**Why each line matters:**

- `FROM python:3.12-slim` not `python:3.12` — slim variant cuts ~700MB by removing non-essential system packages
- `COPY requirements.txt .` before `COPY src/` — Docker layer cache means pip install only re-runs when requirements change, not every time code changes. On every rebuild this saves 2–3 minutes
- `--no-cache-dir` on pip — prevents pip from storing downloaded packages inside the image (~50MB savings)
- Multi-stage build — the builder stage has build tools; final stage has none. Smaller attack surface, smaller image
- `adduser` + `USER appuser` — never run as root inside a container. If the container is compromised, the attacker doesn't have root on the host
- `ENTRYPOINT` exec form `["python", ...]` not shell form `python ...` — exec form means your process is PID 1 and receives SIGTERM directly. Shell form wraps it in `/bin/sh -c`, which doesn't forward signals → container ignores SIGTERM → Docker waits 30 seconds and sends SIGKILL → slow shutdowns, request drops during rolling updates

**The `.dockerignore` file (equally important):**

```
.git
.gitignore
**/__pycache__
**/*.pyc
*.md
tests/
.env
node_modules/
```

Without `.dockerignore`, `COPY . .` sends everything to the Docker daemon — including your `.git` directory, test files, and any `.env` with secrets. The `.dockerignore` is the Docker equivalent of `.gitignore`.

**Real impact:** In our Python microservices, following these patterns cut average image size from 1.4GB to 180MB. CI build time dropped from 8 minutes to 3 minutes (pip cache layer hits on most builds). Security scan findings went from 12 CVEs per image to 2 (smaller base image = fewer installed packages = fewer CVEs).

---

## 2. Write a basic Dockerfile for a Node.js application.

```dockerfile
FROM node:20-alpine AS builder

WORKDIR /app

# Install dependencies first — cached separately from app code
COPY package*.json ./
RUN npm ci --only=production

# ---- Final stage ----
FROM node:20-alpine

WORKDIR /app

# Copy only what we need — not node_modules from builder
COPY --from=builder /app/node_modules ./node_modules
COPY src/ ./src/

RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 3000

ENTRYPOINT ["node", "src/index.js"]
```

**Key difference from the Python example:** `npm ci` instead of `npm install`. `npm ci` installs exactly the versions in `package-lock.json` — reproducible builds. `npm install` can update patch versions and give you different packages on different build days.

**`node:20-alpine` vs `node:20`:** Alpine is ~100MB vs ~1GB for the full Debian-based image. Alpine works for most Node.js apps. Exception: if your app uses native addons compiled against glibc, Alpine (which uses musl) will break them. In that case, use `node:20-slim` (Debian slim) instead.
