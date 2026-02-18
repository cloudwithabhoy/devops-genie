# CI/CD Project Ideas

> Hands-on projects to build real CI/CD skills.

## Beginner

- **GitHub Actions Hello World** — Create a workflow that runs tests and prints a build summary on every push.
- **Docker Build & Push** — Pipeline that builds a Docker image and pushes it to Docker Hub or GHCR on merge to main.

## Intermediate

<!-- Add intermediate project ideas here -->

- **Full CI Pipeline** — Build, test, lint, security scan, and package a Node.js or Python app using GitHub Actions.
- **Multi-Environment Deployment** — Deploy to staging automatically and require manual approval before promoting to production.
- **Reusable Workflows** — Refactor a monolithic workflow into reusable GitHub Actions workflow files.

## Advanced

<!-- Add advanced project ideas here -->

- **GitOps with ArgoCD** — Store K8s manifests in Git; ArgoCD auto-syncs on merge. Roll back by reverting a commit.
- **Matrix Testing** — Run tests across multiple OS and language runtime versions in parallel.
- **Self-Hosted Runner** — Spin up a self-hosted GitHub Actions runner on a cloud VM and use it in a pipeline.

## Project Template

For each project, document:
1. **Goal** — What the pipeline does
2. **Tools Used** — CI platform, testing framework, registry
3. **Pipeline YAML** — Full workflow/pipeline file
4. **Steps** — How to set it up from scratch
5. **Verification** — How to trigger and verify the pipeline runs
