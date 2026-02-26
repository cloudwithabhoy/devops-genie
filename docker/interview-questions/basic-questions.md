# Docker — Basic Interview Questions

---

## 1. What is a Docker image vs. a container?

> **Also asked as:** "Docker image vs container"

An **Image** is a read-only template that contains the instructions for creating a Docker container. It includes the application code, libraries, dependencies, and other files needed for the application to run. Think of it as a **Class** in programming.

A **Container** is a runnable instance of an image. You can start, stop, move, or delete a container using the Docker API or CLI. It is isolated from other containers and the host machine. Think of it as an **Object** (instance) of that class.

---

## 2. What is a multi-stage Docker build?

> **Also asked as:** "Why do we use multi-stage builds?"

**Multi-stage builds** allow you to use multiple `FROM` statements in a single Dockerfile. Each `FROM` instruction begins a new stage of the build. You can selectively copy artifacts from one stage to another, leaving behind everything you don't want in the final image.

**Key Benefits:**
- **Smaller Image Size:** You don't include build tools (like compilers, Maven, or Go toolchains) in the final production image. 
- **Security:** Reduced attack surface because the production image only contains the minimal runtime requirements.
- **Simplicity:** You don't need separate Dockerfiles for "build" and "runtime".

**Example (Go application):**
```dockerfile
# Stage 1: Build stage
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o main .

# Stage 2: Final production stage
FROM alpine:latest
WORKDIR /root/
# Copy only the compiled binary from the builder stage
COPY --from=builder /app/main .
CMD ["./main"]
```
In this example, the final image doesn't contain the Go compiler or source code, only the 10MB binary.
