# Kubernetes — Medium Questions

---

## 1. How do you troubleshoot CrashLoopBackOff or other deployment errors?

CrashLoopBackOff means the container starts, crashes, Kubernetes restarts it, it crashes again — and the backoff delay increases each time (10s, 20s, 40s...). The fix depends on **why** it's crashing.

**Step 1 — Check what the pod status tells you:**

```bash
kubectl get pods -n prod
# NAME                        READY   STATUS             RESTARTS   AGE
# order-service-7d9-xk2       0/1     CrashLoopBackOff   5          3m
# order-service-7d9-ab3       0/1     ImagePullBackOff   0          3m
# order-service-7d9-cd4       0/1     Pending            0          3m
```

Each status points to a different root cause:

| Status | Meaning | First command |
|---|---|---|
| `CrashLoopBackOff` | Container starts then crashes | `kubectl logs <pod> --previous` |
| `ImagePullBackOff` | Can't pull the Docker image | `kubectl describe pod <pod>` |
| `Pending` | Pod can't be scheduled | `kubectl describe pod <pod>` (check Events) |
| `OOMKilled` | Container exceeded memory limit | `kubectl describe pod <pod>` (check Last State) |
| `CreateContainerConfigError` | Missing ConfigMap/Secret | `kubectl describe pod <pod>` |

**Step 2 — For CrashLoopBackOff, check the logs:**

```bash
# Logs from the CURRENT container (might be empty if it crashed instantly)
kubectl logs order-service-7d9-xk2 -n prod

# Logs from the PREVIOUS crashed container (this is what you need)
kubectl logs order-service-7d9-xk2 -n prod --previous
```

Common findings:
- `Error: connect ECONNREFUSED 10.0.3.45:5432` → database not reachable (wrong host, SG blocking, DB down)
- `java.lang.OutOfMemoryError` → memory limit too low or memory leak
- `FileNotFoundException: /config/app.yaml` → ConfigMap not mounted correctly
- `permission denied` → container running as wrong user, or filesystem is read-only

**Step 3 — Check events and describe:**

```bash
kubectl describe pod order-service-7d9-xk2 -n prod

# Look at:
# Events:    → scheduling errors, image pull errors, probe failures
# Last State: → exit code, reason (OOMKilled, Error), finished at
# Containers: → readiness/liveness probe config, resource limits
```

**Step 4 — Common fixes:**

```bash
# OOMKilled → increase memory limit
kubectl set resources deployment/order-service -n prod --limits=memory=512Mi

# ImagePullBackOff → check image name, ECR auth, imagePullSecrets
kubectl get secret ecr-creds -n prod    # Does the pull secret exist?

# Pending → check node resources
kubectl describe nodes | grep -A 5 "Allocated resources"
# If CPU/memory requested > available → scale the node group

# CreateContainerConfigError → check if referenced ConfigMap/Secret exists
kubectl get configmap app-config -n prod
kubectl get secret db-credentials -n prod
```

**Step 5 — Debug inside the container:**

```bash
# If the pod keeps crashing, override the command to keep it alive
kubectl run debug-pod --image=<same-image> --command -- sleep 3600
kubectl exec -it debug-pod -- /bin/sh
# Now manually run the start command and see what fails
```

---

## 2. What is a PDB (PodDisruptionBudget) in Kubernetes?

A PDB tells Kubernetes: "During voluntary disruptions (node drain, cluster upgrade, scaling down), never let available pods drop below this threshold." It prevents cluster operations from accidentally taking down all replicas of a service.

**The problem without PDB:**

```bash
# Cluster upgrade drains nodes one by one
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
# If all 3 pods of order-service happen to be on node-1 → all evicted → downtime
```

**With PDB:**

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service-pdb
  namespace: prod
spec:
  minAvailable: 2          # At least 2 pods must always be running
  selector:
    matchLabels:
      app: order-service
```

Now if you drain a node, Kubernetes checks the PDB first. If evicting a pod would violate `minAvailable: 2`, the drain **waits** until another pod is healthy elsewhere.

**Two ways to define it:**

```yaml
# Option 1: minAvailable — at least N pods must be running
minAvailable: 2        # Or "60%" — at least 60% of replicas

# Option 2: maxUnavailable — at most N pods can be down
maxUnavailable: 1      # Only 1 pod can be down at a time
```

For a 3-replica service: `minAvailable: 2` = same as `maxUnavailable: 1`.

**When PDB matters:**
- **EKS node group upgrades** — nodes are drained and replaced. Without PDB, all pods on a node are evicted at once
- **Spot instance interruptions** — AWS reclaims the node with 2-min notice. PDB ensures Kubernetes moves pods safely
- **Cluster autoscaler scale-down** — autoscaler won't remove a node if it would violate a PDB

**Real scenario:** During an EKS upgrade, the node group replaced all 3 nodes simultaneously (default behavior). All pods were evicted at once — 90 seconds of downtime. After adding PDBs with `maxUnavailable: 1`, the next upgrade drained one node at a time, waited for pods to reschedule, then drained the next. Zero downtime.

---

## 3. What is HPA (Horizontal Pod Autoscaler)?

HPA automatically adjusts the number of pod replicas based on observed metrics — CPU, memory, or custom metrics. When load increases, HPA scales out. When load decreases, it scales in.

**Basic HPA on CPU:**

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 15
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale out when avg CPU > 70%
```

When average CPU across all pods exceeds 70%, HPA adds replicas. When it drops below, HPA removes replicas (with a 5-minute cooldown to avoid thrashing).

**HPA on custom metrics (requests per second):**

```yaml
metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: 100    # Scale when each pod handles > 100 req/s
```

This requires a metrics adapter (Prometheus Adapter) that exposes app metrics to the Kubernetes metrics API.

**Important requirement:** HPA only works if your pods have `resources.requests` set:

```yaml
resources:
  requests:
    cpu: 200m       # HPA calculates utilization relative to this
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

Without `requests`, HPA can't calculate "70% of what" — it won't scale.

**HPA commands:**

```bash
# Check HPA status
kubectl get hpa -n prod
# NAME                  REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS
# order-service-hpa     Deployment/order-svc     45%/70%   3         15        3

# Describe for events and scaling history
kubectl describe hpa order-service-hpa -n prod
```

**Real scenario:** We set HPA to scale on CPU but our service was I/O-bound (waiting on database), not CPU-bound. CPU stayed at 20% while response times climbed to 3 seconds. HPA never scaled because CPU was fine. We switched to scaling on `http_requests_per_second` via Prometheus Adapter — now it scales based on actual load, not just CPU.

