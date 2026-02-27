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

> **Also asked as:** "How do you find errors in the pipelines?"

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

> **Also asked as:** "How do you handle rollback in case of a failed release?" — covered above (git revert, versioned artifacts, Blue/Green switch-back, feature flags).

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

---

## 6. How do you design a CI/CD pipeline for a secure deployment?

> **Also asked as:** "How do you create a secure CI/CD pipeline?"

A secure CI/CD pipeline isn't just "add a security scan." It's designing the pipeline so that insecure code, vulnerable dependencies, misconfigured infrastructure, and leaked secrets **cannot** reach production. Security is a gate at every stage, not an afterthought.

**The pipeline stages with security at each gate:**

```
Code Commit → Build → Test → Security Scan → Stage Deploy → Approval → Prod Deploy
     ↑           ↑       ↑         ↑              ↑            ↑           ↑
  Branch       Deps    Unit     SAST/DAST/       Smoke +     Manual      Canary/
  protection   lock    + int    Image scan      observe      gate       Blue-Green
  + signed     file    tests                    window
  commits
```

**Stage 1 — Secure the code before it enters the pipeline.**

```yaml
# GitHub branch protection — enforce before merge
Branch protection rules:
  ✓ Require pull request reviews (2 reviewers minimum for prod services)
  ✓ Require status checks to pass (CI must succeed)
  ✓ Require signed commits (GPG-signed — proves who wrote the code)
  ✓ Restrict who can push to main (only CI bot or release managers)
  ✓ No force pushes to main
```

No direct commits to `main`. Every change goes through a PR with code review. The reviewer is responsible for catching insecure patterns — hardcoded secrets, overly permissive IAM policies, SQL injection vectors.

**Stage 2 — Dependency scanning and lockfile enforcement.**

```groovy
// Jenkinsfile — fail the build if dependencies have critical CVEs
stage('Dependency Audit') {
  steps {
    sh 'npm audit --audit-level=critical'       // Node.js
    sh 'pip-audit --require-hashes -r requirements.txt'  // Python
    sh 'trivy fs --scanners vuln --severity CRITICAL .'  // Universal
  }
}
```

Lock files (`package-lock.json`, `requirements.txt` with hashes, `go.sum`) must be committed. Without a lockfile, `npm install` in CI might pull a different (compromised) version than what was tested locally.

**Stage 3 — Static Application Security Testing (SAST).**

```groovy
stage('SAST') {
  parallel {
    stage('SonarQube') {
      steps {
        // Scans source code for: SQL injection, XSS, hardcoded secrets,
        // insecure deserialization, command injection
        sh 'sonar-scanner -Dsonar.projectKey=order-service'
      }
    }
    stage('Secrets Detection') {
      steps {
        // Catches API keys, passwords, tokens accidentally committed
        sh 'trufflehog filesystem --directory . --only-verified'
        // Or: gitleaks detect --source .
      }
    }
  }
}
```

SonarQube with quality gates: if new code introduces any Critical security hotspot, the pipeline fails. No exceptions. We caught a developer who hardcoded a Stripe test key — `trufflehog` flagged it before it reached `main`.

**Stage 4 — Container image scanning.**

```groovy
stage('Build & Scan Image') {
  steps {
    sh 'docker build -t $ECR_REPO:$GIT_COMMIT .'

    // Scan the built image for OS and library CVEs
    sh '''
      trivy image \
        --exit-code 1 \
        --severity CRITICAL,HIGH \
        --ignore-unfixed \
        $ECR_REPO:$GIT_COMMIT
    '''

    // Lint the Dockerfile for best practices
    sh 'hadolint Dockerfile'
    // Catches: running as root, no healthcheck, unpinned base image
  }
}
```

Images that fail the scan don't get pushed to ECR. Period. The `--ignore-unfixed` flag avoids blocking on CVEs that don't have a fix yet (otherwise you'd be stuck indefinitely).

**Stage 5 — Infrastructure as Code scanning.**

```groovy
stage('IaC Security') {
  steps {
    // Scan Terraform for misconfigurations
    sh 'checkov -d terraform/ --framework terraform --check HIGH,CRITICAL'
    // Catches: S3 buckets without encryption, security groups open to 0.0.0.0/0,
    // RDS without backup, Lambda without VPC
  }
}
```

Checkov or `tfsec` runs on every PR that touches Terraform code. A security group rule that opens port 22 to the world is caught here, not in a quarterly audit.

**Stage 6 — Deploy to staging with smoke tests and observation.**

```groovy
stage('Deploy Staging') {
  steps {
    sh 'helm upgrade --install order-service ./chart -f values-staging.yaml -n staging'
    sh 'kubectl rollout status deployment/order-service -n staging --timeout=120s'
  }
}
stage('Staging Smoke Tests') {
  steps {
    sh './scripts/smoke-test.sh staging'
    // Tests: health endpoint, auth flow, core CRUD operations
    // Verifies: no 5xx errors, response times within SLO
  }
}
stage('Observation Window') {
  steps {
    // Wait 30 minutes — watch error rate and latency in staging
    sleep(time: 30, unit: 'MINUTES')
    // Automated check: Prometheus query — error rate < 1% in last 30 min
    sh '''
      ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=..." | jq '.data.result[0].value[1]')
      if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
        echo "Error rate too high in staging: $ERROR_RATE"
        exit 1
      fi
    '''
  }
}
```

**Stage 7 — Manual approval gate for production.**

```groovy
stage('Production Approval') {
  steps {
    timeout(time: 60, unit: 'MINUTES') {
      input(
        message: "Deploy ${env.IMAGE_TAG} to production?",
        ok: 'Deploy to Prod',
        submitter: 'release-managers'  // Only specific roles can approve
      )
    }
  }
}
```

**Stage 8 — Production deployment with rollback capability.**

```groovy
stage('Deploy Production') {
  steps {
    // Update manifest repo (GitOps)
    sh '''
      yq -i '.image.tag = "${GIT_COMMIT}"' k8s-manifests/apps/order-service/values-prod.yaml
      git commit -am "ci: deploy order-service ${GIT_COMMIT}"
      git push
    '''
    // ArgoCD syncs the change to the cluster
    // Canary or blue/green strategy for production
  }
}
stage('Post-Deploy Validation') {
  steps {
    sh './scripts/smoke-test.sh production'
    // If smoke test fails → automatic rollback
  }
  post {
    failure {
      sh 'kubectl rollout undo deployment/order-service -n prod'
      slackSend channel: '#incidents', message: "🔴 Auto-rollback: order-service deploy failed in prod"
    }
  }
}
```

**Secrets management in the pipeline:**

```groovy
// NEVER: hardcoded credentials in Jenkinsfile
// NEVER: credentials in environment variables visible in build logs

// Use Jenkins credential store or HashiCorp Vault
withCredentials([
  string(credentialsId: 'ecr-token', variable: 'ECR_TOKEN'),
  usernamePassword(credentialsId: 'sonar-creds', usernameVariable: 'SONAR_USER', passwordVariable: 'SONAR_PASS')
]) {
  sh 'docker login -u AWS -p $ECR_TOKEN $ECR_REPO'
}
// Credentials are masked in build logs — never printed in plaintext
```

**Pipeline security checklist:**

```
☐ Branch protection on main (2 reviewers, CI must pass)
☐ Dependency audit with lockfile verification
☐ SAST scan (SonarQube) with quality gate
☐ Secret detection (trufflehog / gitleaks)
☐ Container image scan (Trivy) — block CRITICAL/HIGH
☐ Dockerfile lint (Hadolint) — no root, pinned base images
☐ IaC scan (Checkov) — no open security groups, encryption enforced
☐ Staging deploy + 30-min observation window
☐ Manual approval for production
☐ Production deploy with auto-rollback on smoke test failure
☐ Secrets in credential store, never in code or env vars
☐ Build logs don't contain credentials (masked)
☐ Image signing (Cosign) for supply chain integrity
```

**Real scenario:** We had a pipeline that ran image scanning but only scanned the final layer of a multi-stage Docker build. A developer added a `curl` command in a build stage that downloaded a binary from a personal GitHub repo — it passed the scan because the final image didn't contain `curl`. After the incident, we added: (1) Hadolint to catch `ADD` from external URLs, (2) network policies in the CI runner to block outbound internet during builds (only ECR and approved registries), and (3) a policy that all base images must come from our internal ECR registry (mirrors of official images, scanned weekly). No more arbitrary downloads during builds.

> **Also asked as:** "How do you integrate security into your CI/CD pipeline?" — covered above (SAST, dependency scanning, image scanning, IaC scanning, secrets management, approval gates, and auto-rollback).

---

## 7. How will you deploy an application using Argo CD?

ArgoCD follows the GitOps model — the Git repository is the single source of truth for what should be running in the cluster. You don't `kubectl apply` or `helm install` manually. You update the manifest in Git, and ArgoCD syncs the cluster to match.

**The workflow:**

```
Developer merges PR → CI pipeline builds image → CI updates manifest repo → ArgoCD detects change → ArgoCD syncs cluster
```

**Step 1 — Install ArgoCD in the cluster:**

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

**Step 2 — Create an ArgoCD Application that points at your manifest repo:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: order-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/my-org/k8s-manifests.git
    targetRevision: main
    path: apps/order-service         # Folder containing Helm chart or YAML
    helm:
      valueFiles:
        - values-prod.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:                       # Auto-sync for staging
      prune: true                    # Delete resources removed from Git
      selfHeal: true                 # Revert manual kubectl changes
    syncOptions:
      - CreateNamespace=true
```

**Step 3 — Deploy by updating the manifest repo:**

```bash
# CI pipeline updates the image tag in the values file
yq -i '.image.tag = "abc123"' apps/order-service/values-prod.yaml
git commit -am "ci: deploy order-service abc123"
git push

# ArgoCD detects the change within 3 minutes (polling interval)
# Or trigger sync immediately:
argocd app sync order-service
```

**For production — disable auto-sync, require manual approval:**

```yaml
syncPolicy: {}    # Empty — no auto-sync. ArgoCD shows "OutOfSync" until someone clicks Sync
```

A release manager reviews the diff in ArgoCD UI (shows exactly what changed), clicks "Sync", and the deployment rolls out.

**Key commands:**

```bash
argocd app list                              # All apps and their sync status
argocd app get order-service                 # Detailed status
argocd app sync order-service                # Trigger sync
argocd app rollback order-service 2          # Rollback to revision 2
argocd app diff order-service                # Show what would change
```

---

## 8. How do you integrate Argo CD with Jenkins?

Jenkins handles CI (build, test, scan, push image). ArgoCD handles CD (deploy to cluster). They connect through a **manifest Git repository** — Jenkins writes to it, ArgoCD reads from it.

**The integration flow:**

```
Jenkins Pipeline:
  1. Build Docker image → push to ECR
  2. Update image tag in manifest repo (Git commit)
        ↓
ArgoCD:
  3. Detects manifest change → syncs cluster
  4. Rolls out new pods with new image
```

**Jenkins pipeline (the CI part):**

```groovy
pipeline {
  stages {
    stage('Build & Push') {
      steps {
        sh 'docker build -t $ECR_REPO:$GIT_COMMIT .'
        sh 'docker push $ECR_REPO:$GIT_COMMIT'
      }
    }
    stage('Update Manifests') {
      steps {
        // Clone the manifest repo, update the image tag, push
        sh '''
          git clone https://github.com/my-org/k8s-manifests.git
          cd k8s-manifests
          yq -i '.image.tag = "'$GIT_COMMIT'"' apps/order-service/values-prod.yaml
          git config user.email "ci@company.com"
          git commit -am "ci: deploy order-service $GIT_COMMIT [skip ci]"
          git push
        '''
        // [skip ci] prevents Jenkins from triggering on its own commit
      }
    }
  }
}
```

**ArgoCD (the CD part):**

ArgoCD watches the manifest repo. When Jenkins pushes the updated `values-prod.yaml`, ArgoCD detects the change and syncs — no ArgoCD CLI calls needed from Jenkins.

**Why this separation matters:**
- Jenkins doesn't need cluster credentials — it never runs `kubectl` or `helm`
- ArgoCD has cluster access but doesn't need Docker/ECR access
- Separation of concerns: CI team owns Jenkins, platform team owns ArgoCD
- Full audit trail in Git — every deployment is a Git commit with author, timestamp, PR link

**Alternative: Jenkins triggers ArgoCD sync directly:**

```groovy
stage('Trigger ArgoCD Sync') {
  steps {
    sh '''
      argocd app sync order-service \
        --auth-token $ARGOCD_TOKEN \
        --server argocd.internal \
        --grpc-web
    '''
  }
}
```

We avoid this pattern because it couples Jenkins to ArgoCD — if ArgoCD is temporarily unavailable, the Jenkins pipeline fails. With the Git-commit approach, Jenkins always succeeds (it just pushes to Git), and ArgoCD syncs whenever it's ready.

---

## 9. How is the CI/CD pipeline set up in your project? What security tools are integrated and how do you manage them?

> **Also asked as:** "Walk me through your CI/CD pipeline" · "How do you manage security in your pipeline?" · "What DevSecOps tools do you use in CI/CD?" · "Write a rough pipeline script for microservices architecture."

This is a "show your work" question. The interviewer wants to see a real pipeline, not a textbook list of tools.

**Our pipeline: Jenkins + Docker + ECR + ArgoCD on EKS**

```
Developer pushes code to GitHub
    ↓
GitHub Webhook → Jenkins (triggers pipeline)
    ↓
Stage 1: Code Checkout + Dependency Install
Stage 2: Unit Tests + Code Coverage
Stage 3: SAST (SonarQube)           ← security
Stage 4: Docker Build (multi-stage)
Stage 5: Image Scan (Trivy)          ← security
Stage 6: Push to ECR
Stage 7: Update K8s Manifest (GitOps)
Stage 8: ArgoCD auto-syncs → deploys to EKS
Stage 9: Smoke Test (post-deploy validation)
Stage 10: Slack Notification (pass/fail)
```

**Rough Jenkinsfile for a microservices architecture:**

```groovy
pipeline {
  agent any

  environment {
    APP_NAME     = "order-service"
    ECR_REGISTRY = "123456789.dkr.ecr.ap-south-1.amazonaws.com"
    IMAGE_TAG    = "${env.GIT_COMMIT[0..6]}"
    SONAR_URL    = "http://sonarqube.internal:9000"
    K8S_REPO     = "git@github.com:org/k8s-manifests.git"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Unit Tests') {
      steps {
        sh 'mvn test -q'
        junit 'target/surefire-reports/*.xml'
        jacoco execPattern: 'target/jacoco.exec'
      }
    }

    stage('SAST - SonarQube') {
      steps {
        withSonarQubeEnv('sonarqube') {
          sh '''
            mvn sonar:sonar \
              -Dsonar.projectKey=${APP_NAME} \
              -Dsonar.host.url=${SONAR_URL}
          '''
        }
        // Block pipeline if Quality Gate fails
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh "docker build -t ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG} ."
      }
    }

    stage('Image Scan - Trivy') {
      steps {
        sh """
          trivy image \
            --exit-code 1 \
            --severity CRITICAL \
            --ignore-unfixed \
            ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
        """
      }
    }

    stage('Push to ECR') {
      steps {
        withAWS(credentials: 'aws-ecr-creds', region: 'ap-south-1') {
          sh """
            aws ecr get-login-password | docker login \
              --username AWS \
              --password-stdin ${ECR_REGISTRY}
            docker push ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}
          """
        }
      }
    }

    stage('Update Manifest (GitOps)') {
      steps {
        sshagent(['github-deploy-key']) {
          sh """
            git clone ${K8S_REPO} k8s-manifests
            cd k8s-manifests
            sed -i "s|image: .*${APP_NAME}.*|image: ${ECR_REGISTRY}/${APP_NAME}:${IMAGE_TAG}|" \
              apps/${APP_NAME}/deployment.yaml
            git config user.email "jenkins@ci.internal"
            git config user.name "Jenkins"
            git add .
            git commit -m "ci: update ${APP_NAME} to ${IMAGE_TAG}"
            git push origin main
          """
        }
      }
    }

    stage('Smoke Test') {
      steps {
        sleep(30)   // Wait for ArgoCD to sync
        sh """
          curl -sf https://api.prod.internal/orders/health \
            || (echo "Smoke test failed" && exit 1)
        """
      }
    }
  }

  post {
    success {
      slackSend channel: '#deployments',
        color: 'good',
        message: "✅ ${APP_NAME}:${IMAGE_TAG} deployed to prod"
    }
    failure {
      slackSend channel: '#deployments',
        color: 'danger',
        message: "❌ Pipeline failed: ${APP_NAME} — ${env.BUILD_URL}"
    }
  }
}
```

**Security tools integrated and how we manage them:**

| Tool | Stage | What it checks | Fail condition |
|---|---|---|---|
| SonarQube | After unit tests | Code quality, OWASP bugs, code smells, coverage | Quality Gate fails (configurable thresholds) |
| Trivy | After Docker build | OS package CVEs, language dependency CVEs | Any CRITICAL with a fix available |
| OWASP Dependency-Check | During build | Known vulnerable dependencies (CVE database) | CVSS score > 7 |
| Checkov | On Terraform/K8s YAML | IaC misconfigurations (public S3, permissive SGs) | Any HIGH severity finding |
| Gitleaks | Pre-commit + CI | Secrets accidentally committed to Git | Any secret pattern match |

**How we manage these tools:**

**SonarQube:** Self-hosted on a dedicated EC2 instance. Quality Gates configured per project — stricter for payment services (0 critical bugs, coverage > 80%) than internal tooling. Credentials stored in Jenkins as a secret text credential.

**Trivy:** Run as a Docker container in the pipeline — no installation needed. `trivy image` downloads the vulnerability DB on first run and caches it. We update the DB cache daily via a scheduled Jenkins job so pipeline scans are fast.

**Gitleaks:** Added as a GitHub Actions pre-check that runs before Jenkins even starts. Catches leaked secrets before they reach CI — API keys, AWS credentials, JWT secrets. If Gitleaks finds a hit, the push is blocked and Slack gets an alert immediately.

**Secrets in Jenkins:** All credentials (AWS keys, SonarQube tokens, GitHub deploy keys) stored in Jenkins Credentials Store. Accessed via `withCredentials` or `withAWS` blocks — never hardcoded in Jenkinsfiles. Jenkinsfiles themselves are in the application repo under `Jenkinsfile` — version controlled, reviewed in PRs.

**Real scenario:** During a feature branch build, Trivy found a `CRITICAL` CVE in `log4j-core:2.14.0` — this was log4shell. The pipeline failed, blocked the merge. The dev updated the dependency to `2.17.1` (fixed version), pipeline went green. Without the scan, that code would have shipped to prod in 20 minutes. That one pipeline failure prevented a potential RCE vulnerability in production.

> **Also asked as:** "What is GitOps and how do you implement it?" — GitOps = Git as the single source of truth for infrastructure and app deployment. ArgoCD watches Git repos and ensures cluster state matches. See above for the full Jenkins + ArgoCD implementation.

