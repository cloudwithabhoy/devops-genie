# CI/CD — Medium Questions

---

## 1. What deployment strategies have you used, and when would you choose each?

This is not a "list the strategies" question. The interviewer wants to know which you've actually used and why you picked them over the alternatives.

**Rolling Update — what we use for most services.**

New pods replace old pods gradually. K8s does this by default. Zero downtime if your readiness probes are configured correctly.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0      # Never take down a pod before a new one is ready
    maxSurge: 1            # Spin up one extra pod at a time
```

**When we use it:** Stateless services where both old and new versions can serve traffic simultaneously. That's 90% of our microservices.

**When it burned us:** We had a rolling update where the new version changed a database schema. During the rollout, old pods were writing records with the old schema and new pods expected the new schema. We got 15 minutes of data inconsistency. Lesson: schema migrations must be backward-compatible and deployed separately, before the code change.

---

**Blue/Green — used for high-risk releases.**

Two identical environments. Traffic lives on Blue. You deploy to Green, run full smoke tests, then switch traffic. If something breaks, you flip back in seconds.

```
Users → Route53 / ALB → Blue (live)
                      → Green (new version, idle, fully tested)

# After validation:
Users → Route53 / ALB → Green (now live)
                      → Blue (standby for rollback)
```

We implemented this on ALB using weighted target groups — 100% to Blue, 0% to Green during validation, then flip.

**When we use it:** Payments service, auth service — anything where a broken deployment at 2 AM has to be rolled back in under 60 seconds with zero risk. `kubectl rollout undo` takes time and has edge cases. Flipping an ALB weight is instant.

**Cost:** You're running double the infrastructure during the deployment window. For big services, that's real money. We time these releases to off-peak hours and keep the window under 30 minutes.

---

**Canary — used when we're unsure how a change behaves at scale.**

5% of traffic hits the new version. If error rate is normal after 15 minutes, increase to 25%, then 50%, then 100%.

We did this with Argo Rollouts:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: { duration: 15m }
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

The `analysis` block runs a Prometheus query every minute — if error rate on the canary pods goes above 5%, Argo Rollouts automatically aborts and routes all traffic back to the stable version.

**When we use it:** New features with uncertain performance characteristics, database query changes, or anything touching the critical path that we can't test fully in staging because staging traffic is synthetic.

**Real scenario:** We canaried a new caching layer. Staging showed it working fine. In production with real traffic patterns, the cache had an edge case that caused incorrect data for users with specific account types — about 3% of users. The canary caught it at 5% traffic. Only ~0.15% of total users were affected. Without canary, 100% of users would have seen wrong data until we noticed.

---

**Recreate — what we avoid for production, sometimes necessary.**

Stop all old pods, start all new pods. There's a downtime window between stop and start.

```yaml
strategy:
  type: Recreate
```

**When we use it:** Database migration jobs where you absolutely cannot have two versions running simultaneously. Or development environments where downtime doesn't matter and you want a clean slate.

**Key insight for interviews:** The right answer isn't "always use canary" — it's matching the strategy to the risk profile of the change. Low-risk stateless service with backward-compatible changes → rolling update. High-risk auth service change → blue/green. New feature you're not confident in at scale → canary.

---

## 3. How do you manage pipeline configurations and rollback strategies?

**Managing pipeline configurations:**

Pipeline configuration lives in code — not in a CI server's UI. This is non-negotiable in production.

**Approach 1: Jenkinsfile in every application repo (what we use).**

```
order-service/
├── src/
├── Dockerfile
├── Jenkinsfile          ← Pipeline for this service
└── k8s/
    └── values.yaml
```

The Jenkinsfile is reviewed in PRs alongside code changes. If a new stage breaks the pipeline, the PR shows it — same review process as a code bug.

**Approach 2: Shared pipeline templates in a central library.**

For teams with 20+ services, individual Jenkinsfiles become maintenance overhead. We use Shared Libraries so most services' Jenkinsfiles are 5–10 lines:

```groovy
@Library('shared-lib@v2.4.1') _
deployPipeline(
  app: 'order-service',
  ecrRepo: '123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service',
  slackChannel: '#orders-team'
)
```

The actual pipeline logic is in the shared library — versioned, tested independently, shared across all services. A pipeline improvement (adding Trivy scanning, changing the Slack notification format) is a single change in the library, rolled out to all services via a version bump.

**Approach 3: GitOps for the deployment half.**

The "CD" side of the pipeline — what gets deployed where — is managed as Git state. The manifest repo is the source of truth:

```
k8s-manifests/
├── apps/
│   ├── order-service/
│   │   ├── values-staging.yaml    ← "staging is running image abc123"
│   │   └── values-prod.yaml       ← "prod is running image def456"
```

ArgoCD watches this repo. To change what's deployed, you change the manifest repo — via a PR for prod, automated for staging. The entire deployment state is auditable through Git history.

---

**Rollback strategies:**

The right rollback strategy depends on what changed and how it failed.

**Rollback 1: Image tag rollback (fastest — GitOps).**

In our GitOps model, a rollback is reverting the manifest commit:

```bash
# Find the previous commit in the manifest repo
git log --oneline k8s-manifests/apps/order-service/values-prod.yaml
# abc1234 ci: update order-service to new-sha [skip ci]
# def5678 ci: update order-service to old-sha [skip ci]

# Revert to old image
git revert abc1234
git push    # ArgoCD detects the change, rolls back to old-sha image
```

The image itself (old-sha) is still in ECR — we have a retention policy that keeps the last 30 images per service. Rollback takes the same time as a forward deploy (~2 minutes).

**Rollback 2: Kubernetes rolling rollback (fastest for live incidents).**

```bash
kubectl rollout undo deployment/order-service -n prod
# Takes 30–60 seconds — K8s rolls pods back to the previous ReplicaSet
# No manifest change needed

# Verify
kubectl rollout status deployment/order-service -n prod
kubectl rollout history deployment/order-service -n prod   # See revision history
```

`kubectl rollout undo` uses the previous ReplicaSet already in the cluster — no image pull needed if the old image is cached.

**Rollback 3: Helm rollback (for chart changes, not just image changes).**

```bash
helm history order-service -n prod       # See all releases
helm rollback order-service 3 -n prod   # Roll back to release revision 3
```

Use this when the Helm values or chart itself changed (not just the image tag). `helm rollback` applies the entire previous chart state.

**Rollback 4: Infrastructure rollback (Terraform).**

For infrastructure changes that caused service failures:

```bash
# Revert the Terraform commit that caused the issue
git revert <commit-sha>
git push    # CI pipeline runs terraform plan + apply
# Or directly:
terraform apply -target=aws_security_group.app   # Reapply specific resource
```

For accidental destroys: restore from S3 state versioning.

**What we decided: never roll forward during an incident.**

Roll forward (deploy a fix instead of reverting) is tempting — "I can fix it in 10 minutes." But during an incident, under pressure, fixes introduce new bugs. Our policy: **rollback first, investigate after.** If rolling back isn't safe (DB migration already ran), we roll forward — but that's the exception, not the default.

**Preventing the need for rollback:**

The best rollback strategy is not needing to roll back. We add gates before prod:
1. Staging deploy + ArgoCD health check
2. Staging smoke tests (automated)
3. 30-minute observation window
4. Manual approval gate
5. Prod deploy + ArgoCD health check
6. Prod smoke test + 10-minute alert silence removed

Most regressions are caught at step 2 or 3. Rollbacks happen, but rarely.

---

## 2. What is a webhook, and how is it used in a CI/CD pipeline?

A webhook is an HTTP callback — instead of a CI server polling every 5 minutes asking "did anything change?", the source control system pushes a notification the moment something happens. It's the difference between checking your email every 5 minutes and getting a notification when a new email arrives.

**How it works:**

1. You register a URL with GitHub (the CI server's webhook endpoint)
2. When a push, PR open, or merge event happens, GitHub sends a POST request to that URL with a JSON payload describing the event
3. The CI server receives the payload, identifies which repo/branch triggered it, and starts the relevant pipeline

**Setup:**

In GitHub: Repo → Settings → Webhooks → Add webhook
- Payload URL: `http://ci-server.company.internal/github-webhook/`
- Content type: `application/json`
- Events: "Just the push event" or "Let me select individual events" (PRs, pushes, etc.)

**What the payload looks like (simplified):**

```json
{
  "ref": "refs/heads/feature/my-feature",
  "after": "abc123def456",
  "repository": {
    "full_name": "our-org/my-app"
  },
  "pusher": {
    "name": "abhoy"
  }
}
```

The CI server parses this, matches `our-org/my-app` to a pipeline job, and triggers the build for branch `feature/my-feature` at commit `abc123def456`.

**Webhook vs polling — the real difference:**

| | Webhook | Polling |
|---|---|---|
| Trigger speed | Instant | Up to 5 min delay |
| CI server load | Zero until event | Constant HTTP requests to GitHub |
| GitHub API rate limit | Not affected | Can exhaust rate limits at scale |
| Requires CI server reachable from internet | Yes | No |

**When polling is used instead of webhooks:**

The CI server is inside a private VPC with no public ingress. GitHub.com can't reach it. Polling with a 2-minute interval (`H/2 * * * *`) is used as a fallback. For enterprise GitHub (self-hosted in the same VPC), webhooks always work because the network is private but GitHub can reach the CI server directly.

**Real scenario:** All 18 pipelines failed to trigger for 3 hours. Root cause: a plugin update broke the GitHub webhook endpoint — it returned 500. GitHub retried the webhook 3 times, then stopped. The failure was silent from GitHub's side. Fix: configured GitHub webhook delivery alerts and added a polling fallback for production pipelines as a safety net.

---

## 4. How do you manage environment-specific configurations in CI/CD?

Environment-specific config is one of the most common sources of bugs that "work in staging but break in prod." The goal is: same pipeline logic, different values per environment — and those values are never hardcoded in the pipeline itself.

**Approach 1: Per-environment values files (Helm) — what we use.**

The same Helm chart is deployed to every environment. Only the values file changes:

```
k8s-manifests/apps/order-service/
├── values-dev.yaml       ← replicas: 1, DB: dev-db.internal, LOG_LEVEL: debug
├── values-staging.yaml   ← replicas: 2, DB: staging-db.internal, LOG_LEVEL: info
└── values-prod.yaml      ← replicas: 5, DB: prod-db.internal, LOG_LEVEL: warn
```

The pipeline picks the right values file based on the branch being deployed:

```groovy
def valuesFile = "values-${env.ENVIRONMENT}.yaml"
// ENVIRONMENT is set per pipeline: 'staging' when merging to develop, 'prod' when merging to main

sh "yq -i '.image.tag = \"${env.IMAGE_TAG}\"' k8s-manifests/apps/order-service/${valuesFile}"
sh "git push"   // ArgoCD applies values-staging.yaml to staging cluster, values-prod.yaml to prod
```

**Approach 2: Kubernetes ConfigMaps per namespace.**

For config that doesn't need to be in the chart (app-level settings that ops manages separately):

```yaml
# dev namespace
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: dev
data:
  LOG_LEVEL: "debug"
  CACHE_TTL: "30"
  EXTERNAL_API_URL: "https://sandbox.payment-api.com"

# prod namespace — same ConfigMap name, different values
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: prod
data:
  LOG_LEVEL: "warn"
  CACHE_TTL: "300"
  EXTERNAL_API_URL: "https://api.payment-api.com"
```

The pod mounts `app-config` without specifying namespace — it automatically picks up the namespace-specific version. No pipeline changes needed when config values change.

**Approach 3: AWS Parameter Store / Secrets Manager with env-prefixed paths.**

For values that include secrets (DB URLs with credentials, API keys):

```
/dev/order-service/db-url          → postgres://user:pass@dev-db:5432/orders
/staging/order-service/db-url      → postgres://user:pass@staging-db:5432/orders
/prod/order-service/db-url         → postgres://user:pass@prod-db:5432/orders
```

The pipeline passes `ENVIRONMENT` as a variable. The app constructs the SSM path at runtime:

```python
import boto3
env = os.environ['ENVIRONMENT']  # 'dev', 'staging', 'prod'
ssm = boto3.client('ssm')
db_url = ssm.get_parameter(Name=f'/{env}/order-service/db-url', WithDecryption=True)['Parameter']['Value']
```

No environment-specific secrets in the pipeline. The IAM role controls which environment the app can read.

**Approach 4: GitHub Actions environments with protected secrets.**

GitHub Actions has an "Environments" feature — secrets scoped to an environment, with protection rules:

```yaml
jobs:
  deploy-prod:
    environment: production    # Triggers approval gate + uses production secrets
    steps:
      - name: Deploy
        env:
          DB_URL: ${{ secrets.DB_URL }}      # From "production" environment secrets
          API_KEY: ${{ secrets.API_KEY }}    # Different value than staging's API_KEY
        run: ./deploy.sh
```

The `production` environment has its own set of secrets — `DB_URL` in staging points to the staging DB, `DB_URL` in production points to the prod DB. Same secret name, different values per environment. Also: the `production` environment can require manual approval before jobs run.

**What we never do: hardcode environment differences in the Jenkinsfile/workflow.**

```groovy
// NEVER
if (env.BRANCH_NAME == 'main') {
  DB_URL = 'postgres://prod-db:5432/orders'
} else {
  DB_URL = 'postgres://staging-db:5432/orders'
}
// → Credentials in pipeline code → in Git history → security incident
// → Config changes require pipeline code changes → messy
```

**Real scenario:** Our payment service had three different Stripe API keys — test, staging, production. Early on, these were hardcoded in the Jenkinsfile in a big `if/else` block. When Stripe rotated our production key (security incident on their side), we had to update the Jenkinsfile, open a PR, get it reviewed, merge it — 45 minutes to rotate a key. After migrating to Secrets Manager with env-prefixed paths: key rotation is a 2-minute SSM Parameter Store update, no pipeline change, no PR needed.

---

## 5. How do you implement manual approval before production deployment?

Manual approval is a deliberate gate — it says "a human has reviewed the staging deploy and is comfortable pushing this to production." The implementation varies by CI tool.

**Jenkins: `input` step.**

```groovy
stage('Deploy to Production') {
  steps {
    timeout(time: 30, unit: 'MINUTES') {
      // Pipeline pauses here and waits for a human to click Proceed/Abort
      input(
        message: "Deploy order-service ${env.IMAGE_TAG} to production?",
        ok: 'Deploy to Prod',
        submitter: 'release-managers,team-leads',  // Only these users can approve
        parameters: [
          string(name: 'REASON', description: 'Reason for deployment')
        ]
      )
    }
    // Timeout: if nobody approves in 30 minutes, pipeline aborts automatically
    // This prevents pipelines from hanging indefinitely on weekends
  }
}
```

The Jenkins pipeline pauses at `input`. An email/Slack notification goes out. A release manager clicks "Proceed" or "Abort" in Jenkins UI. The `submitter` parameter restricts who can approve — junior engineers can't approve production deploys.

**GitHub Actions: Environment protection rules.**

```yaml
jobs:
  deploy-staging:
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh staging

  deploy-production:
    needs: deploy-staging
    environment: production     # ← "production" environment has required reviewers
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh production
```

In GitHub Settings → Environments → production → "Required reviewers" — add specific people or teams. When the `deploy-production` job is reached, it pauses and GitHub sends a review request to the required reviewers. A reviewer approves in the GitHub UI (or via Slack with the GitHub integration). The job then runs.

```yaml
# GitHub Actions deployment protection also supports:
# - Wait timer: delay X minutes after staging before production is allowed
# - Branch restrictions: only 'main' can deploy to production environment
environment:
  name: production
  url: https://myapp.com    # Shown in the deployment log as the deployment target
```

**GitLab CI: `when: manual` gates.**

```yaml
deploy-staging:
  stage: deploy
  script:
    - ./deploy.sh staging
  environment: staging

deploy-production:
  stage: deploy
  script:
    - ./deploy.sh production
  environment: production
  when: manual          # ← Job appears in pipeline but doesn't run automatically
  allow_failure: false  # If this job is skipped, pipeline shows as blocked (not success)
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

In GitLab UI, the production deploy job shows as a "play button" — someone clicks it to trigger. GitLab Enterprise supports protected environments with specific role requirements for who can click that button.

**ArgoCD: sync policy without auto-sync for production.**

In a GitOps setup, the deployment gate can be at the ArgoCD level rather than the pipeline:

```yaml
# ArgoCD Application for staging — auto-syncs when manifest changes
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-staging
spec:
  syncPolicy:
    automated:
      prune: true
      selfHeal: true   # Auto-deploys on manifest change

---
# ArgoCD Application for prod — NO auto-sync
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service-prod
spec:
  syncPolicy: {}    # Empty — sync only happens when someone clicks "Sync" in ArgoCD UI
                    # or runs: argocd app sync order-service-prod
```

When the CI pipeline merges to main and updates `values-prod.yaml`, ArgoCD shows the prod app as "OutOfSync" but doesn't deploy. A release manager reviews the diff in ArgoCD UI and clicks "Sync" to approve the deployment. This is our preferred pattern — the approval UI shows exactly what will change in the cluster, not just "do you want to deploy?"

**Real scenario:** Before we added approval gates, a developer accidentally merged a PR to `main` that had `LOG_LEVEL=debug` hardcoded (should have been an env var). The CI pipeline auto-deployed to production. Log volume increased 40x. CloudWatch costs spiked. After 20 minutes someone noticed and we rolled back. After adding a manual gate: that same PR would have been caught during the 30-minute staging observation window before anyone clicked "Approve for Prod."
