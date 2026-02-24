# Docker — Hard Questions

---

## 1. You need live-patching of a Docker host kernel without downtime — how do you achieve it?

Kernel patching on a Docker host requires rebooting the host — containers on that host stop. The goal is zero downtime for the running workloads, not zero reboots on the host.

**The approach: cordon → drain → patch → reboot → uncordon.**

This is the same node maintenance pattern used in Kubernetes, but applied to standalone Docker hosts.

**Step 1 — Remove the host from the load balancer (if standalone, not Kubernetes).**

If Docker hosts are behind an ALB target group:

```bash
# Deregister the host from the target group — it stops receiving new connections
aws elbv2 deregister-targets \
  --target-group-arn arn:aws:elasticloadbalancing:ap-south-1:123456:targetgroup/app-tg/abc123 \
  --targets Id=<instance-id>

# Wait for in-flight connections to drain (connection draining = 30s by default)
sleep 30

# Verify: no active connections to this host
aws elbv2 describe-target-health \
  --target-group-arn <tg-arn> \
  --targets Id=<instance-id>
# State should be "draining" then "unused"
```

**Step 2 — Stop containers gracefully.**

```bash
# List running containers
docker ps --format "{{.ID}}: {{.Names}}"

# Stop each container with a graceful signal (SIGTERM + 30s timeout)
docker stop --time 30 <container-id>

# Or stop all at once (if you want zero grace period, use kill)
docker stop $(docker ps -q)
```

For Docker Compose services:

```bash
docker compose stop    # Sends SIGTERM, waits for stop_grace_period (default 10s)
```

**Step 3 — Apply the kernel patch and reboot.**

```bash
# Update kernel packages
sudo apt-get update && sudo apt-get upgrade -y linux-image-generic linux-headers-generic

# Reboot to activate the new kernel
sudo reboot
```

After reboot, verify the new kernel is running:

```bash
uname -r
# Should show the new kernel version
```

**Step 4 — Restart containers and re-register with the load balancer.**

```bash
# Start containers (or use --restart=always flag to auto-start on reboot)
docker start $(docker ps -aq)

# Or Docker Compose:
docker compose up -d

# Re-register with ALB
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:ap-south-1:123456:targetgroup/app-tg/abc123 \
  --targets Id=<instance-id>

# Verify health
aws elbv2 describe-target-health --target-group-arn <tg-arn>
# State should return to "healthy"
```

**For Kubernetes (EKS/K8s managed Docker hosts):**

In Kubernetes, the process is cleaner because the control plane handles rescheduling:

```bash
# 1. Cordon — prevent new pods from being scheduled on this node
kubectl cordon <node-name>

# 2. Drain — evict all pods gracefully (respects PodDisruptionBudgets)
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60

# 3. Patch and reboot the node (out-of-band via SSH or SSM)
aws ssm send-command \
  --instance-ids <instance-id> \
  --document-name "AWS-RunShellScript" \
  --parameters commands=["sudo apt-get upgrade -y linux-image-generic && sudo reboot"]

# 4. Wait for node to come back online
kubectl wait --for=condition=Ready node/<node-name> --timeout=300s

# 5. Uncordon — allow pods to be scheduled again
kubectl uncordon <node-name>
```

Kubernetes PodDisruptionBudgets ensure that drain doesn't take too many pods down at once. If your PDB says `minAvailable: 2`, drain will wait until pods reschedule to other nodes before evicting the next one.

**Live kernel patching without reboot (kpatch / livepatch):**

For CVE-critical patches that can't wait for a reboot window, Linux kernel live patching applies the patch to the running kernel without a reboot:

```bash
# Ubuntu Livepatch — kernel security patches applied live (requires Ubuntu Pro)
sudo ua enable livepatch
sudo canonical-livepatch enable <token>

# Check live patch status
canonical-livepatch status
# Shows: currently applied patches, CVEs covered, next check time
```

Live patching covers security CVEs but not all kernel changes — major kernel version upgrades still require a reboot. Livepatch is a bridge: it keeps you secure until your next maintenance window, not a permanent reboot replacement.

**EKS managed node groups — automatic node rotation:**

For EKS, AWS handles kernel patching through node group updates. When a new optimised AMI is released:

```bash
# Update the node group to use the new AMI (triggers rolling replacement)
aws eks update-nodegroup-version \
  --cluster-name prod-cluster \
  --nodegroup-name general \
  --release-version latest

# EKS drains old nodes, launches new nodes (with patched kernel), drains old
# Zero downtime if PodDisruptionBudgets are configured correctly
```

**Real scenario:** A critical kernel CVE (privilege escalation) required patching all Docker hosts within 24 hours. We had 8 EC2 instances running Docker directly (pre-Kubernetes migration). We rolled through them one by one: deregister from ALB → drain containers → patch → reboot → start containers → re-register. Each host took ~8 minutes total. All 8 hosts patched in 64 minutes with zero user-visible downtime. After migrating to EKS, the same type of CVE was handled by `aws eks update-nodegroup-version` — automated rolling replacement in 20 minutes with no manual intervention.

---

## 2. How do you enforce policy-as-code for Docker container security?

Policy-as-code means defining security rules in version-controlled, machine-readable policies that are automatically enforced — not just documented guidelines that developers may or may not follow.

**Layer 1: Enforce at build time — Dockerfile linting with Hadolint.**

Hadolint checks Dockerfiles against security best practices before the image is ever built:

```bash
# In CI pipeline — fail the build if Dockerfile has issues
hadolint Dockerfile

# Common rules:
# DL3006: Always tag the version of FROM images — FROM ubuntu:22.04 not FROM ubuntu
# DL3007: Don't use :latest tag — unpredictable builds
# DL3008: Pin apt-get package versions
# DL3025: Use COPY instead of ADD
# DL3045: COPY with root context when USER is non-root
```

```yaml
# .github/workflows/docker.yml
- name: Lint Dockerfile
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
    failure-threshold: warning    # Fail on warnings and above
    ignore: DL3008                # Ignore specific rules if justified
```

Create `.hadolint.yaml` to codify your team's accepted exceptions with justifications:

```yaml
# .hadolint.yaml
ignore:
  - DL3013   # Allow unpinned pip packages in dev images
trustedRegistries:
  - ghcr.io/myorg    # Only these registries allowed in FROM statements
```

**Layer 2: Enforce at build time — image scanning with Trivy.**

```yaml
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE }}
    format: sarif
    exit-code: '1'                         # Fail the build
    severity: 'CRITICAL,HIGH'
    ignore-unfixed: true                   # Don't fail on CVEs with no fix
```

Block images with critical CVEs from being pushed to the registry.

**Layer 3: Enforce at registry — ECR with image signing (Cosign).**

```bash
# Sign the image after it passes CI scans
cosign sign \
  --key cosign.key \
  --annotations "ci-run=$GITHUB_RUN_ID" \
  --annotations "git-commit=$GITHUB_SHA" \
  ghcr.io/myorg/myapp:$IMAGE_TAG

# Verify signature before running
cosign verify \
  --key cosign.pub \
  ghcr.io/myorg/myapp:$IMAGE_TAG
```

Images without a valid signature from your CI system are rejected. No one can push and run an image built outside the approved pipeline.

**Layer 4: Enforce at runtime in Kubernetes — OPA/Gatekeeper admission control.**

Open Policy Agent (OPA) Gatekeeper intercepts every `kubectl apply` and evaluates policies before allowing the resource to be created:

```yaml
# ConstraintTemplate — defines the policy
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirenonroot
spec:
  crd:
    spec:
      names:
        kind: K8sRequireNonRoot
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirenonroot
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := sprintf("Container '%v' must set runAsNonRoot: true", [container.name])
        }
---
# Constraint — applies the template to all pods in all namespaces
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequireNonRoot
metadata:
  name: require-non-root
spec:
  match:
    kinds: [{"apiGroups": [""], "kinds": ["Pod"]}]
    excludedNamespaces: ["kube-system"]
```

This policy blocks any pod that doesn't set `runAsNonRoot: true`. It applies to every deployment, not just ones where the developer remembered.

**Additional Gatekeeper policies for Docker security:**

```yaml
# Policy: No privileged containers
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  container.securityContext.privileged == true
  msg := sprintf("Container '%v' cannot run as privileged", [container.name])
}

# Policy: Read-only root filesystem required
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not container.securityContext.readOnlyRootFilesystem
  msg := sprintf("Container '%v' must use readOnlyRootFilesystem: true", [container.name])
}

# Policy: Images must come from approved registries
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  not startswith(container.image, "ghcr.io/myorg/")
  not startswith(container.image, "123456.dkr.ecr.ap-south-1.amazonaws.com/")
  msg := sprintf("Container '%v' uses an unapproved image registry", [container.name])
}

# Policy: No containers running as UID 0
violation[{"msg": msg}] {
  container := input.review.object.spec.containers[_]
  container.securityContext.runAsUser == 0
  msg := sprintf("Container '%v' cannot run as root (UID 0)", [container.name])
}
```

**Layer 5: Enforce at runtime — Falco for runtime security monitoring.**

Gatekeeper enforces policies at admission time (before the container starts). Falco monitors running containers for suspicious behaviour:

```yaml
# /etc/falco/falco_rules.yaml
- rule: Container running as root
  desc: Detect containers running as root
  condition: container and proc.vpid=1 and user.uid=0 and not user_known_containers
  output: "Container running as root (user=%user.name container=%container.name image=%container.image)"
  priority: WARNING

- rule: Write below /etc in container
  desc: Detect file writes below /etc in a container
  condition: container and fd.directory startswith /etc and evt.is_open_write=true
  output: "Write below /etc in container (file=%fd.name container=%container.name)"
  priority: ERROR

- rule: Unexpected outbound connection
  desc: Alert on container making unexpected outbound connections
  condition: container and evt.type=connect and not expected_network_traffic
  output: "Unexpected outbound connection from container (dest=%fd.rip container=%container.name)"
  priority: WARNING
```

Falco sends alerts to Slack or PagerDuty when a container does something outside the policy — running a shell, writing to unexpected paths, or making unexpected network connections.

**The full enforcement pipeline:**

```
Code commit
    ↓
Hadolint: Dockerfile must follow security rules (build fails if not)
    ↓
Trivy: No CRITICAL CVEs with available fix (build fails if not)
    ↓
Image pushed to registry
    ↓
Cosign: Image signed with CI private key
    ↓
kubectl apply
    ↓
Gatekeeper: Pod must have runAsNonRoot, readOnlyFS, approved registry (deploy fails if not)
    ↓
Container running
    ↓
Falco: Runtime detection of policy violations (alert if violated)
```

**Real scenario:** A developer unknowingly shipped a container running as root with `securityContext` not set (they copied a quickstart example). In production, this container had root access to the underlying node filesystem. After enabling Gatekeeper with the `K8sRequireNonRoot` constraint, that deployment was rejected at `kubectl apply` with a clear error message. The developer fixed the Dockerfile to use `USER 1000` and added `runAsNonRoot: true` to the pod spec. The policy caught 8 similar violations in existing deployments that were remediated over the next sprint.
