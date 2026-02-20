# CI/CD — Basic Questions

---

## 1. What is Blue/Green deployment?

Blue/Green is a deployment strategy where you run two identical environments — Blue (live) and Green (new version) — and switch traffic between them. At any moment, only one environment serves users.

**How it works:**

```
Before deployment:
  Users → Load Balancer → Blue (v1 — live, serving 100% traffic)
                        → Green (v1 — idle)

Deployment:
  Deploy v2 → Green
  Run smoke tests against Green
  Green passes → switch load balancer

After deployment:
  Users → Load Balancer → Green (v2 — live, serving 100% traffic)
                        → Blue (v1 — standby for rollback)
```

**Why teams use it:**

- **Zero downtime** — users never hit the new version until it's been validated
- **Instant rollback** — if Green breaks, flip the load balancer back to Blue in seconds. No redeployment needed
- **Clean cutover** — no period where half your users get v1 and half get v2

**The cost:** You're running double the infrastructure during the deployment window. Two full environments means twice the EC2 instances, twice the container resources. We time these deployments to off-peak hours and keep the Green environment alive for ~30 minutes (long enough to confirm nothing is wrong), then spin it down.

**In AWS:** We implement this with ALB weighted target groups — 100% to Blue during deployment, smoke tests against the Green target group directly, then flip weights. The switch is instant.

**When we choose Blue/Green:** High-risk releases on critical services — payments, auth, anything where a bad deployment at 2 AM needs a rollback in under 60 seconds with zero risk of partial state.

---

## 2. How does a Rolling Update work?

A rolling update replaces old instances of an application with new ones gradually, without taking all of them down at once. Users are always served by a mix of old and new pods during the rollout.

**How it works in Kubernetes:**

```
Start: 4 pods running v1
       [v1] [v1] [v1] [v1]

Step 1: Spin up 1 new v2 pod, wait for readiness
       [v1] [v1] [v1] [v1] [v2]

Step 2: Terminate 1 old v1 pod
       [v1] [v1] [v1] [v2]

Step 3: Repeat until all pods are v2
       [v2] [v2] [v2] [v2]
```

**Configuration:**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0   # Never take down a pod before a replacement is ready
    maxSurge: 1         # Allow 1 extra pod beyond the desired count
```

- `maxUnavailable: 0` — zero-downtime guarantee. Old pods stay up until the new pod passes its readiness probe
- `maxSurge: 1` — controls pace. Higher values = faster rollout but more temporary resource usage

**Why it's the default for most services:**

- No extra infrastructure cost — you're just using slightly more resources temporarily
- Simple — Kubernetes handles it automatically
- Works for stateless services where old and new versions can coexist

**The gotcha that trips people up:** During a rolling update, both v1 and v2 are serving traffic simultaneously. If your new version has a **breaking database schema change** (drops a column v1 depends on), v1 pods will start failing. Rolling updates require backward-compatible changes. Non-backward-compatible changes need a different strategy (Recreate or Blue/Green).

---

## 4. What do you do during Continuous Delivery (CD)?

CD picks up where CI leaves off. CI builds and validates the artifact. CD takes that validated artifact and moves it through environments until it reaches production — automatically or with a controlled gate.

**What CI produces:** A tested, scanned Docker image pushed to ECR tagged with the Git commit SHA. CI stops there.

**What CD does:**

**1. Environment promotion — staging first, then production.**

After CI pushes the image, CD updates the deployment manifests. In our GitOps setup (ArgoCD), Jenkins updates the Helm values file in the manifest repo:

```bash
# Jenkins runs this after image push:
yq -i '.image.tag = "abc123def456"' k8s-manifests/apps/order-service/values-staging.yaml
git commit -m "ci: update order-service to abc123def456 [skip ci]"
git push
# ArgoCD detects the change → deploys to staging
```

**2. Wait for health — not just manifest update.**

A deployment isn't done when the manifest is updated. It's done when the new pods are Running, Ready, and the service is healthy:

```groovy
sh 'argocd app wait order-service --health --timeout 300'
# Blocks until ArgoCD reports app as Healthy
# If pods crash loop, OOMKill, or fail readiness probes → this fails → pipeline fails
```

**3. Smoke test against the real environment.**

After deployment, hit the actual service endpoint:

```bash
response=$(curl -s -o /dev/null -w "%{http_code}" https://staging.myapp.com/health)
if [ "$response" != "200" ]; then
  echo "Smoke test failed: $response"
  kubectl rollout undo deployment/order-service -n staging
  exit 1
fi
```

**4. Manual gate before production (Continuous Delivery vs Deployment).**

**Continuous Delivery** — every successful staging deploy is ready for production, but a human explicitly approves the production push. Jenkins `input` step or a PR from staging → main.

**Continuous Deployment** — every successful staging deploy automatically goes to production. No human gate.

We use Continuous Delivery (with a gate) for most services. The gate is a 1-click approval in Jenkins after staging smoke tests pass. This gives the team 30 minutes to look at staging before prod gets the change.

```groovy
stage('Approve Prod') {
  steps {
    timeout(time: 30, unit: 'MINUTES') {
      input message: 'Deploy to production?', ok: 'Deploy'
    }
  }
}
stage('Deploy Prod') {
  steps {
    sh 'yq -i \'.image.tag = "$IMAGE_TAG"\' k8s-manifests/apps/order-service/values-prod.yaml'
    sh 'git push'  // ArgoCD syncs to prod
  }
}
```

**5. Post-production verification.**

```bash
# Watch key metrics for 10 minutes after prod deploy
# Error rate (Grafana)
# Latency p99 (Grafana)
# Business metrics (orders placed, checkout completions)
# If anything degrades: rollback
kubectl rollout undo deployment/order-service -n prod
```

**The key difference between CI and CD:**

CI proves the code works. CD proves the code works *in each environment* with real config, real dependencies, and real traffic. A service can pass all CI checks and still fail in production because of a missing secret, a NetworkPolicy that doesn't allow a new dependency, or a database schema that wasn't migrated. CD is about safely getting the artifact to where it runs.

---

## 3. How does a Canary Deployment work?

A canary deployment sends a small percentage of traffic to the new version and gradually increases it while monitoring metrics. If something goes wrong, you abort and all traffic goes back to the stable version — only a fraction of users were affected.

**The concept (from mining — canaries detected gas before miners did):**

```
Before: 100% traffic → v1

Canary step 1:   5% traffic → v2 (canary)
                95% traffic → v1 (stable)
                → Monitor: error rate, latency, conversion metrics

Canary step 2:  25% traffic → v2
                75% traffic → v1
                → Still monitoring

Canary step 3: 100% traffic → v2
                → Rollout complete
```

**What you're looking for during each step:**

- Error rate on v2 ≤ error rate on v1
- Latency (p99) not regressing
- Business metrics (orders placed, checkout completions) not dropping

**In Kubernetes with Argo Rollouts:**

```yaml
strategy:
  canary:
    steps:
      - setWeight: 5
      - pause: { duration: 15m }   # Monitor for 15 minutes at 5%
      - setWeight: 25
      - pause: { duration: 15m }
      - setWeight: 100
    analysis:
      templates:
        - templateName: success-rate
      args:
        - name: service-name
          value: my-app
```

The `analysis` block runs a Prometheus query automatically. If the canary's error rate exceeds the threshold, Argo Rollouts aborts and routes all traffic back to stable — without human intervention.

**When canary is the right choice:** When you're deploying a change you're not confident about at full production scale — new caching logic, changes to a database query, algorithm changes. Staging traffic is synthetic. Canary uses real user traffic as your signal, at a scale where a problem is recoverable.

**When NOT to use canary:** If old and new versions are genuinely incompatible (different API contracts that clients depend on), having 5% of users on the new API version will break their clients. Canary requires backward compatibility, same as rolling updates.
