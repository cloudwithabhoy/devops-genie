# Kubernetes Project Ideas

> Hands-on projects to build real Kubernetes skills.

## Beginner

- **Deploy a Web App** — Write Deployment and Service manifests to run a containerised web app on a local cluster (minikube or kind).
- **ConfigMap & Secret Lab** — Inject configuration and secrets into a Pod using ConfigMaps and Secrets.

## Intermediate

<!-- Add intermediate project ideas here -->

- **Ingress Controller** — Deploy an Nginx Ingress controller and route traffic to multiple services by hostname/path.
- **Helm Chart** — Package your application as a Helm chart with configurable values.
- **Horizontal Pod Autoscaler** — Configure HPA based on CPU usage and load-test to observe scaling.

## Advanced

<!-- Add advanced project ideas here -->

- **GitOps with ArgoCD** — Set up ArgoCD to sync Kubernetes manifests from a Git repository automatically.
- **Multi-Tenant Cluster** — Configure namespaces, RBAC, and ResourceQuotas for multiple teams.
- **Stateful Application** — Deploy a PostgreSQL StatefulSet with persistent volumes and verify data survives pod restarts.

## Project Template

For each project, document:
1. **Goal** — What you're building and why
2. **Prerequisites** — Tools (kubectl, helm, etc.) and cluster type
3. **Manifests / Commands** — YAML files and step-by-step commands
4. **Verification** — How to confirm it works
5. **Key Concepts** — K8s concepts demonstrated
