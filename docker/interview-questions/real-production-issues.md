# Docker — Real Production Issues

---

## 1. A container works locally but fails in Kubernetes — how do you narrow down the difference?

"Works on my machine" is the most frustrating Docker problem. The image is the same, but K8s introduces constraints and context that local Docker doesn't have. Here's the systematic diff.

**Step 1 — Check if the image actually runs in the K8s environment at all.**

```bash
# Get the exact error from the pod
kubectl describe pod <pod-name> -n prod
kubectl logs <pod-name> -n prod
kubectl logs <pod-name> -n prod --previous   # Logs from the crashed container
```

The error in `kubectl describe` Events section tells you the category:
- `ErrImagePull` / `ImagePullBackOff` → image can't be pulled (auth, wrong tag, private registry)
- `OOMKilled` → container hit memory limit
- `CrashLoopBackOff` → container starts then crashes (check `--previous` logs)
- `CreateContainerConfigError` → bad env var, missing secret, missing ConfigMap

**Step 2 — Diff the environment variables.**

Locally you run `docker run -e DB_HOST=localhost -e ENV=dev`. In K8s, the pod gets env vars from ConfigMaps, Secrets, and Downward API. Missing or wrong env vars are the most common cause of "works locally, fails in K8s."

```bash
# See exactly what env vars the pod gets
kubectl exec <pod> -n prod -- env | sort

# Compare against what your local container gets
docker run --rm my-app env | sort
```

Differences to look for:
- `DB_HOST` pointing to `localhost` locally but missing or pointing to a wrong service name in K8s
- `ENV=dev` locally vs `ENV=prod` in K8s — some apps have code paths that activate differently in prod
- Missing required env vars that your local setup has in `.env` file but K8s doesn't have in its ConfigMap

**Step 3 — Check resource limits.**

Locally: no limits. In K8s: CPU and memory limits apply. A container that runs fine with 2GB locally will OOMKill if the pod has `memory: 512Mi` limit.

```bash
kubectl describe pod <pod> -n prod | grep -A 10 "Limits:"
# If memory limit is much lower than your local Docker run → OOMKill risk
```

Also check CPU throttling — see if the pod is getting throttled even though average CPU looks fine:

```promql
rate(container_cpu_cfs_throttled_periods_total{pod=~"my-app.*"}[5m])
/ rate(container_cpu_cfs_periods_total{pod=~"my-app.*"}[5m])
```

**Step 4 — Check security context — non-root, read-only filesystem.**

Locally you run as root. In K8s with security policies:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  readOnlyRootFilesystem: true
```

Apps that write to `/tmp`, `/app/logs`, or any local path will fail with "Permission denied" or "Read-only file system" under these constraints. Common culprits: Python writing `.pyc` files to the source directory, apps writing logs to local files instead of stdout.

```bash
# Run the container locally with the same security context
docker run --user 1000:1000 --read-only \
  --tmpfs /tmp \
  my-app
# Does it fail here too? → the container itself is the problem, not K8s
```

**Step 5 — Check network and service discovery.**

Locally: `DB_HOST=localhost`. In K8s: `DB_HOST=postgres-service.prod.svc.cluster.local`. If your app hard-codes `localhost` for any dependency, it fails in K8s.

```bash
# From inside the pod — can it reach its dependencies?
kubectl exec <pod> -n prod -- curl -v http://postgres-service:5432
kubectl exec <pod> -n prod -- nslookup redis-service.prod.svc.cluster.local
```

Also check NetworkPolicy — staging might have no NetworkPolicies (everything can talk to everything). Prod might have `deny-all` by default. A new service dependency works locally and in staging but silently times out in prod because the NetworkPolicy doesn't allow it.

```bash
kubectl get networkpolicy -n prod
# Is there a rule allowing egress from this pod to its new dependency?
```

**Step 6 — Mount paths and volume permissions.**

Locally you might volume-mount a config file. In K8s, ConfigMap mounts create files owned by root. If the app runs as non-root and needs to write to a mounted volume, it fails.

```bash
kubectl exec <pod> -n prod -- ls -la /app/config/
# -rw-r--r-- 1 root root  → file is root-owned, non-root can't write
```

**Real scenario:** A Node.js service worked perfectly locally. In K8s it crashed immediately. `kubectl logs --previous` showed: `EACCES: permission denied, mkdir '/app/.npm'`. The app tried to create a cache directory in `/app/.npm` on startup. In K8s with `readOnlyRootFilesystem: true`, the filesystem was read-only. The app had no way to create the cache directory. Fix: added an `emptyDir` volume mounted at `/app/.npm` — writable ephemeral storage that's not part of the read-only filesystem. 20 minutes to find, 2 lines of YAML to fix. If we'd run `docker run --read-only --tmpfs /app/.npm` locally first, we'd have caught it immediately.

---

## 2. A container is getting OOMKilled — what causes it and how do you fix it?

OOMKilled means the Linux kernel killed the container process because it exceeded the memory limit set in the pod spec. The kernel doesn't negotiate — when the limit is hit, the process is terminated with signal 9.

**Confirm it's OOMKilled:**

```bash
kubectl describe pod <pod-name> -n prod | grep -A 10 "Last State"
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137      ← 128 + 9 (SIGKILL)
#   Started:      ...
#   Finished:     ...
```

Exit code 137 = `128 + SIGKILL(9)` — always means OOMKilled.

**Find the memory usage pattern before the kill:**

```promql
# Memory usage over time — did it spike suddenly or grow gradually?
container_memory_working_set_bytes{pod=~"my-app.*"}

# Peak memory vs limit
container_memory_working_set_bytes{pod=~"my-app.*"}
/
container_spec_memory_limit_bytes{pod=~"my-app.*"}
```

**Root cause 1: Memory limit too low.**

The app legitimately needs more memory than the limit allows. The fix is simple — raise the limit.

But don't just double it blindly. Use VPA to understand actual usage:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"   # Recommendation only
```

After a day of traffic, `kubectl describe vpa my-app-vpa` shows the recommended memory request and limit based on actual usage. Set your limit to `recommended upper bound × 1.2` for headroom.

**Root cause 2: Memory leak.**

Usage grows steadily over hours/days until it hits the limit. Every pod restart buys a few more hours, then it fills up again.

```bash
# Memory trend — is it growing linearly over time?
# In Grafana: graph container_memory_working_set_bytes over 24 hours
# Linear upward slope = memory leak
```

Investigation:
- **Java:** Heap dump when memory is high: `kubectl exec <pod> -- jmap -dump:format=b,file=/tmp/heap.hprof <java-pid>`. Analyze with Eclipse MAT or VisualVM
- **Node.js:** `--expose-gc` flag + heap snapshot via `v8.writeHeapSnapshot()`
- **Python:** Use `tracemalloc` or `memory_profiler` to find the allocating code path

Common leaks: unclosed database connections, event listeners not removed, in-memory caches without eviction, growing queues that never drain.

**Root cause 3: Memory spike on specific requests.**

A particular operation (large file upload, complex query, generating a report) allocates far more than typical. 99% of requests use 200MB; 1% of requests (generating a PDF of all orders) use 2GB.

```bash
# Correlate OOMKill time with specific request types in logs
kubectl logs <pod> -n prod --previous | tail -100
# What was the app doing right before the kill?
```

Fix: Streaming instead of buffering (stream large S3 objects instead of loading into memory), async job queue for memory-intensive operations (separate worker pods with higher limits), pagination instead of fetching all records.

**Root cause 4: JVM not respecting container limits.**

Java's GC uses `Runtime.getRuntime().maxMemory()` to size the heap. On old JVM versions, this reads the host machine's memory — not the container limit. A JVM on a 64GB node with a 512MB container limit thinks it has 64GB available and allocates accordingly.

```bash
# Check JVM version and flags
kubectl exec <pod> -n prod -- java -version
# Java 8 before 8u191: doesn't respect container limits
# Java 11+: respects cgroups limits by default

# For older Java: add JVM flags
# -XX:+UseContainerSupport   (Java 10+)
# -XX:MaxRAMPercentage=75    (75% of container memory limit)
```

**Fix: the full approach**

```yaml
# 1. Set meaningful limits based on observed usage
resources:
  requests:
    memory: "512Mi"    # Scheduling — reserve this much on the node
  limits:
    memory: "768Mi"    # OOMKill threshold — 50% headroom over request

# 2. Set JVM heap within the limit (for Java apps)
env:
  - name: JAVA_OPTS
    value: "-XX:MaxRAMPercentage=75 -XX:+UseContainerSupport"
    # 75% of 768Mi = ~576Mi for heap, 192Mi for non-heap (metaspace, threads)

# 3. Set Node.js memory limit
env:
  - name: NODE_OPTIONS
    value: "--max-old-space-size=512"    # In MB — below the container limit
```

**Real scenario:** Our Java Spring Boot service OOMKilled every 6 hours like clockwork. Memory graph showed linear growth — classic leak. Heap dump revealed a `Caffeine` cache configured with `maximumSize(10_000)` items — but each item was a 100KB serialized response object. 10,000 × 100KB = 1GB heap usage, then OOMKill. The cache had no size limit in bytes, only in item count. Fix: added `maximumWeight` with a byte-based weigher. Cache stabilized at 200MB. No OOMKill since.

---

## 3. How do you handle container security vulnerabilities in production?

Security vulnerabilities in container images are inevitable — base images get new CVEs published daily. The question is: how fast do you know, how fast do you fix, and how do you prevent regressions?

**Layer 1: Catch vulnerabilities before they ship — CI scanning.**

Every Docker build triggers a Trivy scan. Critical/High CVEs with available fixes block the merge:

```bash
trivy image \
  --exit-code 1 \
  --severity CRITICAL,HIGH \
  --ignore-unfixed \
  --format sarif \
  --output trivy-results.sarif \
  $ECR_REPO:$IMAGE_TAG
```

`--ignore-unfixed` is critical — without it, you fail builds on CVEs where no patch exists yet. You can't fix an unfixable CVE; blocking the build just creates noise and engineers start ignoring scan results.

`--format sarif` outputs to GitHub's Code Scanning format — CVEs appear directly in the PR as annotations with file/line context.

**Layer 2: Catch CVEs in already-deployed images — continuous registry scanning.**

A CVE published today affects images you built 6 months ago. ECR Enhanced Scanning re-scans images daily as new CVEs are published:

```hcl
resource "aws_ecr_registry_scanning_configuration" "main" {
  scan_type = "ENHANCED"
  rule {
    scan_frequency = "CONTINUOUS_SCAN"
    repository_filter {
      filter      = "*"
      filter_type = "WILDCARD"
    }
  }
}
```

When a new CVE is found on a deployed image, ECR generates an EventBridge event. We route it to SNS → Slack:

```hcl
resource "aws_cloudwatch_event_rule" "ecr_critical_finding" {
  event_pattern = jsonencode({
    source      = ["aws.inspector2"]
    detail-type = ["Inspector2 Finding"]
    detail = {
      severity = ["CRITICAL"]
      status   = ["ACTIVE"]
    }
  })
}
```

**Layer 3: Understand the actual exposure, not just the CVSS score.**

A CVSS 9.8 CVE sounds catastrophic. But if it's in `curl` and your container never uses `curl`, the exposure is zero. Context matters:

```bash
# Is the vulnerable package actually used by the app?
trivy image --format json $ECR_REPO:$IMAGE_TAG | \
  jq '.Results[].Vulnerabilities[] | select(.Severity == "CRITICAL") | {
    pkg: .PkgName,
    cve: .VulnerabilityID,
    installed: .InstalledVersion,
    fixed: .FixedVersion,
    title: .Title
  }'
```

**Questions to ask for each CVE:**
- Is the vulnerable component in the runtime image or only in the build stage? (multi-stage builds can eliminate build-time vulnerabilities)
- Does the vulnerability require network access? (if the pod has no ingress, a network vulnerability may be unexploitable)
- Is there a fix available? If not, document the accepted risk

**Layer 4: Fix the source, not the symptom.**

The right fix for a CVE in `openssl` in your base image:

```dockerfile
# Wrong: pin to a specific vulnerable version and add a workaround
FROM python:3.12.0   # Has CVE in openssl

# Right: update the base image
FROM python:3.12-slim   # Always get the latest patch version
# OR explicitly pin to a known-good version after CVE fix
FROM python:3.12.3-slim   # 3.12.3 includes the openssl fix
```

Rebuild and redeploy. If `python:3.12-slim` doesn't have the fix yet, temporarily add a manual update in the Dockerfile:

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get upgrade -y && apt-get clean
```

This is a temporary measure — it adds ~50MB to the image and needs to be removed once the base image is updated.

**Layer 5: Don't let old images stay deployed.**

The most dangerous vulnerability is one in an image that was built 6 months ago and is still running in production because nobody noticed. We enforce image age policies:

- ECR lifecycle policy: delete images older than 90 days that aren't tagged as deployed
- Kyverno policy: reject pods using images with a build date > 60 days old

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: check-image-age
spec:
  rules:
    - name: check-build-date
      match:
        resources:
          kinds: ["Pod"]
      validate:
        message: "Image must be built within the last 60 days"
        deny:
          conditions:
            - key: "{{ time_since('', '{{image.labels.build-date}}', '') }}"
              operator: GreaterThan
              value: "1440h"   # 60 days
```

**Real scenario:** We had a `log4shell` (CVE-2021-44228) moment — a critical CVE published on a Tuesday affected every Java image in our entire registry. Without continuous scanning, we'd have found out when a security researcher told us. With ECR Enhanced Scanning, we got a Slack alert within 2 hours of the CVE being published. By end of day, we had rebuilt and redeployed all affected Java services. Zero breach, zero audit finding. The investment in scanning infrastructure paid for itself in one incident.
