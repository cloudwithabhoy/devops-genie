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

---

## 3. How do you create a custom Docker image?

A custom image is built with `docker build` using a `Dockerfile` that defines the instructions. The result is a reusable image you can push to a registry (ECR, Docker Hub) and run anywhere.

**Steps:**

```bash
# 1. Write a Dockerfile (see Q1/Q2 above)
# 2. Build the image
docker build -t my-app:1.0 .
# -t = tag (name:version)
# .  = build context (directory containing Dockerfile)

# 3. Verify the image was created
docker images | grep my-app
# REPOSITORY   TAG   IMAGE ID       SIZE
# my-app       1.0   a1b2c3d4e5f6   245MB

# 4. Run it
docker run -d -p 8080:80 my-app:1.0

# 5. Tag for a registry
docker tag my-app:1.0 123456789012.dkr.ecr.ap-south-1.amazonaws.com/my-app:1.0

# 6. Push to ECR
aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-south-1.amazonaws.com
docker push 123456789012.dkr.ecr.ap-south-1.amazonaws.com/my-app:1.0
```

**The build process — what happens internally:**

Each instruction in the Dockerfile creates a new **layer** (intermediate image). Docker caches each layer — if a layer hasn't changed, it uses the cache. This makes subsequent builds fast.

```
FROM python:3.12-slim         → Layer 1: base OS + Python
WORKDIR /app                  → Layer 2: set working directory
COPY requirements.txt .       → Layer 3: copy requirements
RUN pip install -r requirements.txt  → Layer 4: install packages (CACHED if requirements.txt unchanged)
COPY . .                      → Layer 5: copy app code (invalidates cache on any code change)
CMD ["python", "app.py"]      → Layer 6: set default command
```

**Why `COPY requirements.txt` before `COPY . .` matters:**

Put slow, infrequently-changing steps first. If you did `COPY . . && RUN pip install`, every code change would invalidate the pip install layer — reinstalling all packages on every build. By copying `requirements.txt` first, pip installs are cached as long as requirements don't change.

---

## 4. What are the prerequisites of the `docker build` command?

```bash
docker build -t my-app:latest .
```

For this command to work, you need:

**1. Docker daemon running.**

```bash
# Check if Docker is running
systemctl status docker
docker info    # If this works, daemon is up
```

**2. A `Dockerfile` in the build context directory (or specified explicitly).**

```bash
# Default: looks for "Dockerfile" in the current directory (.)
docker build -t my-app .

# Specify a different Dockerfile location:
docker build -t my-app -f docker/Dockerfile.prod .
```

**3. A build context** — the directory (`.` in the command) that Docker sends to the daemon. All files in this directory are available for `COPY`/`ADD` instructions.

```bash
# The . means: send all files in the current directory as the build context
# Docker compresses and uploads this to the daemon before building
# Large contexts slow down builds — use .dockerignore to exclude unnecessary files
```

**`.dockerignore` — what to exclude from context:**

```
# .dockerignore
node_modules/
.git/
.env
*.log
__pycache__/
```

Without `.dockerignore`, `docker build .` sends `node_modules/` (often 500MB+) to the daemon as context — slow and pointless since `RUN npm install` creates its own `node_modules` inside the image.

**4. The base image accessible** — `FROM python:3.12-slim` must be pullable from Docker Hub or an internal registry. If the daemon can't pull it, the build fails.

```bash
# Pre-pull the base image to ensure it's available (useful in CI/CD)
docker pull python:3.12-slim
docker build -t my-app .
```

**5. Sufficient disk space and permissions.**

```bash
# Check available disk space
df -h /var/lib/docker    # Docker stores images here by default

# User must be in the docker group (or run as root)
sudo usermod -aG docker $USER
```

---

## 5. How do you copy a file from a container to the host?

```bash
# Syntax: docker cp <container_id_or_name>:<path_in_container> <path_on_host>

# Copy a specific file
docker cp my-container:/app/logs/error.log ./error.log

# Copy a directory
docker cp my-container:/app/logs/ ./container-logs/

# The container doesn't need to be running — works on stopped containers too
docker cp stopped-container:/etc/nginx/nginx.conf ./nginx-backup.conf
```

**Reverse: copy from host into a running container:**

```bash
# Copy a config file into a running container
docker cp ./nginx.conf my-container:/etc/nginx/nginx.conf
```

**Practical use cases:**

```bash
# Grab a log file from a crashed container (before it's removed)
docker cp $(docker ps -lq):/var/log/app/error.log ./debug-error.log
# $(docker ps -lq) = ID of the most recently created container

# Export a database dump from a running DB container
docker exec my-postgres pg_dump mydb > /tmp/dump.sql
docker cp my-postgres:/tmp/dump.sql ./db-backup.sql

# Copy a compiled binary out of a build container
docker cp builder-container:/app/bin/myapp ./myapp
```

**Alternative for structured data: volumes.**

For ongoing file sharing between host and container, `docker cp` is a one-time operation. Volumes (`-v /host/path:/container/path`) are better for persistent sharing — but `docker cp` is ideal for ad-hoc extraction.

---

## 6. What happens when you write `COPY . .` in a Dockerfile?

```dockerfile
WORKDIR /app
COPY . .
```

**What this does:**

- **Source:** `.` = the build context directory (the directory you passed to `docker build .` on the host)
- **Destination:** `.` = the current `WORKDIR` inside the container (in this case, `/app`)

So `COPY . .` copies **everything in the build context** into `/app` inside the container.

```
Host build context (.)          Container /app
├── app.py              →       /app/app.py
├── requirements.txt    →       /app/requirements.txt
├── static/             →       /app/static/
└── templates/          →       /app/templates/
```

**The background process:**

1. `docker build .` sends the entire build context to the Docker daemon as a tar archive
2. The daemon extracts the archive
3. `COPY . .` copies from that extracted archive into the image filesystem at `WORKDIR`

**Why it can be problematic without `.dockerignore`:**

```
Host build context WITHOUT .dockerignore:
├── app.py
├── requirements.txt
├── node_modules/          ← 500MB of dependencies
├── .git/                  ← Git history, potentially large
├── .env                   ← SECRETS — would be baked into the image!
└── __pycache__/
```

If you `COPY . .` without a `.dockerignore`, you copy `node_modules` (large), `.env` (secrets leak), `.git/` (unnecessary metadata), and `__pycache__` (Python bytecache, unnecessary) into the image.

**Best practice:**

```dockerfile
# Copy only specific files/directories instead of everything
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY src/ ./src/
COPY config/ ./config/
# Only copy what the application actually needs to run
```

Or use `.dockerignore`:

```
.git/
.env
__pycache__/
*.pyc
node_modules/
.dockerignore
README.md
tests/
```

With `.dockerignore` in place, `COPY . .` is safe and copies only the relevant application files.

---

## 7. What is the difference between a Docker image and a Docker container?

This is one of the most fundamental Docker concepts and is asked in almost every interview.

**Docker Image — the blueprint (read-only template).**

An image is a layered, read-only filesystem snapshot. It contains the OS base, dependencies, application code, and the default command to run. An image by itself does nothing — it just sits on disk.

```bash
# Images exist on disk — they don't "run"
docker images
# REPOSITORY   TAG       IMAGE ID       SIZE
# nginx        latest    a1b2c3d4       187MB
# myapp        1.0       e5f6g7h8       245MB

# An image is built from a Dockerfile
docker build -t myapp:1.0 .

# An image can be pulled from a registry
docker pull nginx:latest
```

**Docker Container — a running instance of an image.**

A container is a live, running process created from an image. When you start a container, Docker adds a thin **writable layer** on top of the read-only image layers. The container can write files, but those changes exist only in that container's writable layer — they don't touch the image.

```bash
# Create and start a container from an image
docker run -d -p 8080:80 --name web nginx:latest
# This creates a container named "web" from the nginx image
# The image is the template; this container is the instance

# List running containers
docker ps
# CONTAINER ID   IMAGE          STATUS         NAMES
# a1b2c3d4       nginx:latest   Up 2 minutes   web

# The same image can have multiple containers simultaneously
docker run -d -p 8081:80 --name web2 nginx:latest
docker run -d -p 8082:80 --name web3 nginx:latest
# Three containers, one image
```

**The key relationship:**

```
Dockerfile → docker build → Image → docker run → Container(s)
  (recipe)                (cake mold)             (actual cakes)

One image → many containers
Each container is isolated — changes in one don't affect others
Stop/delete a container → the image is unchanged
```

**What happens to data when a container stops:**

```bash
# Write a file inside a running container
docker exec web touch /tmp/test.txt

# Stop and remove the container
docker stop web && docker rm web

# Start a new container from the same image
docker run -d -p 8080:80 --name web nginx:latest

# The file is gone — /tmp/test.txt does not exist
# The container's writable layer was destroyed when it was removed
```

This is why **volumes** exist — to persist data beyond the container lifecycle:

```bash
docker run -d -p 8080:80 -v /host/data:/container/data nginx:latest
# /host/data persists even when the container is removed
```

**Summary table:**

| | Image | Container |
|---|---|---|
| State | Static, read-only | Dynamic, running process |
| Storage | Exists on disk as layers | Image layers + thin writable layer |
| Lifecycle | Created by `docker build` | Created by `docker run` |
| Multiplicity | One image | Many containers from one image |
| Data | Immutable | Changes lost when container removed (use volumes) |
