# OpenShift — Medium Questions

---

## 1. How do you handle image builds and rollbacks in OpenShift?

OpenShift has a built-in build system — you don't need an external CI tool to build images. BuildConfigs define how to build images, and ImageStreams track the built images.

**BuildConfig — three build strategies:**

| Strategy | How it works | Use case |
|---|---|---|
| **Source-to-Image (S2I)** | Injects source code into a builder image | Java, Node.js, Python apps — no Dockerfile needed |
| **Docker** | Uses a Dockerfile in your repo | Full control over the image |
| **Custom** | Runs a custom builder image | Complex builds, proprietary toolchains |

**S2I Build (OpenShift's signature feature):**

```bash
# Create a build from a Git repo — OpenShift detects the language and builds
oc new-app nodejs~https://github.com/my-org/order-service.git

# This creates:
# - BuildConfig (how to build)
# - ImageStream (where to store the image)
# - Deployment (how to run it)
# - Service (how to access it)
```

```yaml
# BuildConfig — explicit definition
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: order-service
spec:
  source:
    type: Git
    git:
      uri: https://github.com/my-org/order-service.git
      ref: main
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: nodejs:18-ubi8       # Red Hat UBI Node.js builder
        namespace: openshift
  output:
    to:
      kind: ImageStreamTag
      name: order-service:latest   # Push to internal registry
  triggers:
    - type: GitHub                 # Webhook triggers build on push
      github:
        secret: my-secret
    - type: ImageChange            # Rebuild when base image updates
```

```bash
# Trigger a build manually
oc start-build order-service

# Watch build logs
oc logs -f build/order-service-3

# List builds
oc get builds
# NAME               TYPE     STATUS     STARTED
# order-service-1    Source   Complete   5m ago
# order-service-2    Source   Complete   2m ago
# order-service-3    Source   Running    10s ago
```

**ImageStreams — OpenShift's image tracking:**

ImageStreams are a layer on top of container images. They track image tags and notify DeploymentConfigs/Deployments when images change.

```bash
# Check what's in an ImageStream
oc get is order-service -o yaml
# Shows all tags (latest, v1.2.0, etc.) and which image SHA they point to

# Tag an image for promotion
oc tag order-service:latest order-service:prod
# Now the "prod" tag points to whatever "latest" currently is
```

**Rollbacks:**

```bash
# Rollback a DeploymentConfig to the previous version
oc rollback order-service

# Rollback to a specific revision
oc rollback order-service --to-revision=3

# For standard Deployments — same as kubectl
oc rollout undo deployment/order-service

# Check rollout history
oc rollout history deployment/order-service

# Rollback an ImageStream tag (point "prod" back to previous SHA)
oc tag order-service@sha256:abc123 order-service:prod
```

**The ImageStream-based promotion workflow:**

```
Build → order-service:latest → Test passes → Tag as :staging → Test passes → Tag as :prod
         (dev)                                  (staging)                        (prod)
```

Each environment watches a different tag. Promotion is just retagging — no rebuild needed. Rollback is retagging back to the previous SHA.

---

## 2. Can you explain how RBAC works in OpenShift?

OpenShift RBAC is Kubernetes RBAC + additional default roles and cluster-wide policies. The model is the same: **Users/Groups → RoleBindings → Roles → Resources.**

**The RBAC hierarchy:**

```
ClusterRole    → ClusterRoleBinding → applies cluster-wide
Role           → RoleBinding        → applies within a project/namespace
```

**Built-in OpenShift roles (more than vanilla K8s):**

| Role | Scope | Permissions |
|---|---|---|
| `cluster-admin` | Cluster | Full access to everything |
| `admin` | Project | Full access within a project (manage roles, quotas, resources) |
| `edit` | Project | Create/edit/delete most resources (no role management) |
| `view` | Project | Read-only access to project resources |
| `self-provisioner` | Cluster | Can create new projects |
| `basic-user` | Cluster | Can see own project list, basic discovery |

**Common RBAC operations:**

```bash
# Grant a user admin access to a project
oc adm policy add-role-to-user admin john -n order-team

# Grant a group view access
oc adm policy add-role-to-group view dev-team -n order-team

# Grant a service account a specific role
oc adm policy add-role-to-user edit -z my-service-account -n order-team

# Grant cluster-level roles
oc adm policy add-cluster-role-to-user cluster-reader jane

# Remove a role
oc adm policy remove-role-from-user admin john -n order-team

# Check who has access to a project
oc adm policy who-can get pods -n order-team
```

**Security Context Constraints (SCCs) — OpenShift-specific:**

SCCs control what pods can do at the OS level. They're more powerful than PodSecurityPolicies/Standards:

```bash
# List available SCCs
oc get scc
# NAME                 PRIV    CAPS   SELINUX     RUNASUSER          FSGROUP
# restricted           false   []     MustRunAs   MustRunAsRange     MustRunAs
# anyuid               false   []     MustRunAs   RunAsAny           RunAsAny
# privileged           true    [*]    RunAsAny    RunAsAny           RunAsAny
# hostnetwork          false   []     MustRunAs   MustRunAsRange     MustRunAs

# Grant a service account the 'anyuid' SCC (needed for legacy images that require root)
oc adm policy add-scc-to-user anyuid -z my-service-account -n order-team

# Check which SCC a pod is using
oc get pod order-service-xyz -o yaml | grep scc
# openshift.io/scc: restricted
```

**The `restricted` SCC (default) enforces:**
- No root containers (random UID assigned)
- No privilege escalation
- No host networking/ports
- Read-only root filesystem (configurable)
- SELinux labels enforced

**Real scenario:** A team migrated an nginx-based app from vanilla K8s to OpenShift. Pods kept crashing with "permission denied." The nginx official image runs as root and binds to port 80. OpenShift's `restricted` SCC blocks both. Fix: switched to `nginxinc/nginx-unprivileged` (runs as UID 8080, listens on port 8080) — no SCC changes needed.

---

## 3. How do you monitor and troubleshoot pods and nodes in OpenShift?

OpenShift 4.x includes a pre-installed monitoring stack — you don't deploy Prometheus separately. It's built in.

**Built-in monitoring stack:**

```
OpenShift Monitoring Stack (installed by default):
├── Prometheus            → metrics collection
├── Alertmanager          → alert routing
├── Grafana (read-only)   → dashboards
├── Thanos Querier        → multi-cluster metrics queries
├── Node Exporter         → node-level metrics
└── kube-state-metrics    → Kubernetes object metrics
```

**Troubleshooting pods:**

```bash
# Check pod status
oc get pods -n order-team
# STATUS tells you: Running, CrashLoopBackOff, ImagePullBackOff, Pending, Error

# Pod details and events
oc describe pod order-service-xyz -n order-team

# Logs (current container)
oc logs order-service-xyz -n order-team

# Logs (previous crashed container)
oc logs order-service-xyz -n order-team --previous

# Follow logs in real-time
oc logs -f order-service-xyz -n order-team

# Debug a running pod (start a debug container)
oc debug pod/order-service-xyz
# Opens a shell in a copy of the pod — great for investigating filesystem, env vars, network

# Debug a node (start a privileged pod on the node)
oc debug node/worker-1
# chroot /host
# systemctl status kubelet
# journalctl -u crio --since "30 min ago"
```

**`oc debug` — OpenShift's killer troubleshooting feature:**

This doesn't exist in vanilla K8s. It creates a copy of a pod (or a privileged pod on a node) for investigation without affecting the running workload.

```bash
# Debug a deployment (uses the same image and env vars)
oc debug deployment/order-service -n order-team

# Debug with a different image (e.g., busybox for networking tools)
oc debug deployment/order-service --image=busybox -n order-team

# Debug a node (get a root shell on the host)
oc debug node/worker-2
sh-4.4# chroot /host
sh-4.4# crictl ps                    # List containers on this node
sh-4.4# journalctl -u kubelet        # Check kubelet logs
sh-4.4# df -h                        # Check disk usage
sh-4.4# top                          # Check resource usage
```

**Monitoring through the web console:**

OpenShift's web console has built-in metrics dashboards:
- **Observe → Metrics** — PromQL queries directly in the console
- **Observe → Dashboards** — pre-built dashboards for etcd, API server, nodes, pods
- **Observe → Alerting** — view and silence active alerts

```bash
# Check cluster operator health (OpenShift-specific)
oc get clusteroperators
# Every operator should show Available=True, Degraded=False

# Check node health
oc get nodes
oc describe node worker-1 | grep -A 10 "Conditions:"
# Look for: MemoryPressure, DiskPressure, PIDPressure — all should be False

# Check resource usage
oc adm top nodes
oc adm top pods -n order-team
```

**Enabling user workload monitoring (for your app metrics):**

By default, OpenShift monitors only platform components. To scrape your app's Prometheus metrics:

```yaml
# Enable user workload monitoring
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```

Then create a `ServiceMonitor` in your project — same as vanilla Prometheus Operator:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service
  namespace: order-team
spec:
  endpoints:
    - port: metrics
      interval: 30s
  selector:
    matchLabels:
      app: order-service
```

---

## 4. What CI/CD tools have you used to deploy to OpenShift?

OpenShift supports multiple CI/CD patterns. The choice depends on whether you want an OpenShift-native solution or integrate with existing tools.

**Option 1: Jenkins on OpenShift (common in enterprises)**

OpenShift includes a certified Jenkins image with the OpenShift plugin pre-installed. Jenkins pipelines use the `openshift-client` plugin to interact with the cluster.

```groovy
// Jenkinsfile — deploying to OpenShift
pipeline {
  agent { label 'maven' }
  stages {
    stage('Build') {
      steps {
        sh 'mvn clean package'
      }
    }
    stage('Build Image') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject('order-team') {
              // Trigger OpenShift BuildConfig
              def build = openshift.startBuild('order-service', '--from-dir=.')
              build.logs('-f')
              // Wait for build to complete
              def bc = openshift.selector('bc', 'order-service')
              bc.related('builds').untilEach(1) {
                return it.object().status.phase == 'Complete'
              }
            }
          }
        }
      }
    }
    stage('Deploy') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject('order-team') {
              openshift.selector('dc', 'order-service').rollout().latest()
              openshift.selector('dc', 'order-service').rollout().status()
            }
          }
        }
      }
    }
  }
}
```

**Option 2: OpenShift Pipelines (Tekton — cloud-native CI/CD)**

OpenShift Pipelines is based on Tekton — Kubernetes-native CI/CD where pipelines run as pods.

```yaml
# Tekton Pipeline
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: build-and-deploy
spec:
  params:
    - name: git-url
    - name: image-name
  tasks:
    - name: clone
      taskRef:
        name: git-clone
      params:
        - name: url
          value: $(params.git-url)
    - name: build-image
      taskRef:
        name: buildah           # Builds container images without Docker
      runAfter: [clone]
      params:
        - name: IMAGE
          value: $(params.image-name)
    - name: deploy
      taskRef:
        name: openshift-client
      runAfter: [build-image]
      params:
        - name: ARGS
          value: ["rollout", "latest", "dc/order-service"]
```

**Option 3: GitHub Actions + `oc` CLI**

```yaml
# .github/workflows/deploy.yml
name: Deploy to OpenShift
on:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login to OpenShift
        uses: redhat-actions/oc-login@v1
        with:
          openshift_server_url: ${{ secrets.OPENSHIFT_SERVER }}
          openshift_token: ${{ secrets.OPENSHIFT_TOKEN }}
      - name: Deploy
        run: |
          oc project order-team
          oc start-build order-service --from-dir=. --follow
          oc rollout status dc/order-service --timeout=120s
```

**Option 4: ArgoCD on OpenShift (GitOps)**

OpenShift GitOps is the official ArgoCD distribution for OpenShift. Install it from the OperatorHub:

```bash
# Install OpenShift GitOps operator (via web console or CLI)
# It installs ArgoCD in the openshift-gitops namespace

# Same ArgoCD Application pattern as vanilla K8s
# ArgoCD watches a Git manifest repo → syncs to OpenShift
```

**Decision guide:**

| Tool | Best for |
|---|---|
| Jenkins + OpenShift plugin | Existing Jenkins shops, enterprise approvals |
| OpenShift Pipelines (Tekton) | Cloud-native, K8s-native CI/CD, no external tool |
| GitHub Actions + oc CLI | GitHub-centric, simple pipelines |
| OpenShift GitOps (ArgoCD) | GitOps, declarative deployments, multi-cluster |

