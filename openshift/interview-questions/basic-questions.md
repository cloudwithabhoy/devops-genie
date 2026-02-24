# OpenShift — Basic Questions

---

## 1. What's your experience with OpenShift? How is it different from vanilla Kubernetes?

OpenShift is Red Hat's enterprise Kubernetes distribution. It runs Kubernetes underneath but adds enterprise features — integrated CI/CD, built-in image registry, stricter security defaults, and a developer-friendly web console.

**The key differences:**

| Feature | Vanilla Kubernetes | OpenShift |
|---|---|---|
| **Installation** | You build the cluster (kubeadm, kops, EKS) | Installer handles everything (IPI/UPI) |
| **Container runtime** | Any (containerd, CRI-O) | CRI-O only |
| **Image registry** | You deploy one (Harbor, ECR) | Built-in registry included |
| **Builds** | External (Docker build, Kaniko, CI tool) | Built-in BuildConfigs (S2I, Docker, Custom) |
| **Routing** | Ingress + controller (NGINX, ALB) | Routes (HAProxy-based, built-in) |
| **Security** | Permissive by default | Strict by default (SCCs, no root containers) |
| **Web console** | Dashboard (basic) | Rich web console with developer + admin views |
| **CLI** | `kubectl` | `oc` (superset of kubectl — all kubectl commands work) |
| **Projects** | Namespaces | Projects (namespaces + RBAC + quotas bundled) |
| **Updates** | Manual, risky | OTA updates via Cluster Version Operator |

**Security — the biggest practical difference:**

Vanilla Kubernetes lets containers run as root by default. OpenShift blocks this with Security Context Constraints (SCCs):

```bash
# In vanilla K8s — this works fine
containers:
  - name: app
    image: myapp:v1
    securityContext:
      runAsUser: 0    # root — allowed by default

# In OpenShift — this FAILS unless you explicitly grant the 'anyuid' SCC
# Default SCC: restricted — assigns a random UID, no root, no privilege escalation
oc adm policy add-scc-to-user anyuid -z my-service-account
```

This means many Docker Hub images that work on vanilla K8s fail on OpenShift because they expect root permissions (nginx official image, for example). You either use Red Hat's UBI-based images or configure SCCs.

**CLI — oc vs kubectl:**

```bash
# oc is a superset of kubectl — everything kubectl does, oc does too
oc get pods          # Same as kubectl get pods
oc get routes        # OpenShift-specific — no equivalent in kubectl
oc new-app nginx     # Creates deployment + service + route in one command
oc start-build myapp # Triggers a BuildConfig
oc login https://api.cluster.example.com  # Authenticates to the cluster
```

**When to choose OpenShift vs vanilla K8s:**

| Scenario | Choose |
|---|---|
| Cloud-native, AWS-first, cost-sensitive | EKS (vanilla K8s) |
| Enterprise, regulated, on-prem, Red Hat ecosystem | OpenShift |
| Team needs built-in CI/CD, registry, developer portal | OpenShift |
| Team has strong K8s expertise, wants flexibility | Vanilla K8s |
| Hybrid cloud (on-prem + cloud) | OpenShift (runs anywhere) |

---

## 2. How do you manage deployments in OpenShift (DeploymentConfigs, Routes, Projects)?

OpenShift has its own deployment and routing resources on top of standard Kubernetes ones. Understanding when to use the OpenShift-specific resources vs the standard K8s resources is key.

**DeploymentConfig vs Deployment:**

DeploymentConfig is OpenShift's original deployment resource. Modern OpenShift (4.x) recommends standard Kubernetes `Deployment` instead, but you'll encounter DeploymentConfigs in existing clusters.

```yaml
# OpenShift DeploymentConfig (legacy but still common)
apiVersion: apps.openshift.io/v1
kind: DeploymentConfig
metadata:
  name: order-service
spec:
  replicas: 3
  triggers:
    - type: ImageChange          # Auto-deploy when image changes in registry
      imageChangeParams:
        automatic: true
        containerNames: [order-service]
        from:
          kind: ImageStreamTag
          name: order-service:latest
    - type: ConfigChange         # Auto-deploy when config changes
  strategy:
    type: Rolling
    rollingParams:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
        - name: order-service
          image: order-service:latest
```

```yaml
# Standard Kubernetes Deployment (recommended for new workloads)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
```

Key difference: DeploymentConfig has **triggers** (auto-deploy on image change) and uses the **replication controller** internally. Standard Deployment uses **replica sets** and is managed by `kubectl rollout`. For new projects, use Deployment.

**Routes — OpenShift's alternative to Ingress:**

Routes expose services externally via the built-in HAProxy router. They existed before Kubernetes Ingress and are still the primary way to expose services in OpenShift.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: order-service
spec:
  host: orders.myapp.com
  to:
    kind: Service
    name: order-service
    weight: 100
  port:
    targetPort: 8080
  tls:
    termination: edge                   # TLS at the router
    certificate: |
      -----BEGIN CERTIFICATE-----
      ...
    key: |
      -----BEGIN PRIVATE KEY-----
      ...
    insecureEdgeTerminationPolicy: Redirect   # HTTP → HTTPS redirect
```

```bash
# Quick route creation
oc expose service order-service --hostname=orders.myapp.com

# Check all routes
oc get routes
# NAME            HOST                    PATH   SERVICES        PORT
# order-service   orders.myapp.com               order-service   8080
```

Routes vs Ingress: Routes support TLS re-encryption (edge → passthrough → re-encrypt), weighted traffic splitting for A/B testing (Route + multiple services with weights), and are managed by the OpenShift router (HAProxy). Ingress works too — OpenShift converts Ingress resources to Routes internally.

**Projects — namespaces with guardrails:**

A Project is a namespace + default RBAC + resource quotas + network policies. When you create a Project, OpenShift automatically sets up isolation.

```bash
# Create a project (namespace + RBAC + template)
oc new-project order-team --display-name="Order Team" --description="Order microservice"

# Switch between projects
oc project order-team

# List all projects you have access to
oc get projects
```

A project template can enforce company-wide defaults:

```yaml
# Every new project automatically gets:
# - ResourceQuota (CPU/memory limits)
# - LimitRange (default container limits)
# - NetworkPolicy (deny all inter-namespace traffic by default)
# - RoleBinding (project admin role for the creator)
```

---

## 3. How do you handle secrets and ConfigMaps in OpenShift?

Secrets and ConfigMaps in OpenShift work identically to vanilla Kubernetes — same YAML, same `kubectl`/`oc` commands. The difference is in the enterprise tooling around them.

```bash
# Create a secret
oc create secret generic db-creds \
  --from-literal=username=admin \
  --from-literal=password=s3cur3p@ss \
  -n order-team

# Create a ConfigMap
oc create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-file=app.properties=./config/app.properties \
  -n order-team

# Mount in a DeploymentConfig or Deployment — same as K8s
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: password
```

**OpenShift-specific enhancements:**

**1. Sealed Secrets or External Secrets for GitOps:**

Since OpenShift secrets are base64-encoded (not encrypted), you never commit them to Git. Instead:

```bash
# Option 1: Sealed Secrets (encrypt secrets for Git)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml
# sealed-secret.yaml is safe to commit — only the cluster can decrypt it

# Option 2: External Secrets Operator (sync from HashiCorp Vault)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-creds
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: db-creds
  data:
    - secretKey: password
      remoteRef:
        key: secret/data/order-service
        property: db_password
```

**2. Service account token secrets — auto-managed:**

OpenShift automatically creates and manages service account tokens as secrets. Each service account gets a token that pods use for API authentication.

**3. Image pull secrets — integrated with the internal registry:**

```bash
# OpenShift automatically configures pull secrets for the internal registry
# For external registries (ECR, DockerHub):
oc create secret docker-registry ecr-pull-secret \
  --docker-server=123456.dkr.ecr.ap-south-1.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$(aws ecr get-login-password)

# Link to the default service account
oc secrets link default ecr-pull-secret --for=pull
```

> **Also asked as:** "How are secrets managed differently in OpenShift vs Kubernetes?" — the API is identical. The difference is operational: OpenShift's stricter RBAC defaults mean fewer people have access to secrets by default, and the web console has a dedicated secrets management UI.

