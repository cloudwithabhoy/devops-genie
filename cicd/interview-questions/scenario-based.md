# CI/CD — Scenario-Based Questions

---

## 1. Explain your project architecture.

I always frame this around the problem we were solving, not the tools we picked.

**Context:** We ran a microservices platform — about 18 services, team of 12 engineers, deploying to AWS EKS. The old setup was manual deployments over SSH, zero rollback capability, and a shared staging environment that was permanently broken because everyone used it at the same time.

**What we built:**

The architecture has three layers — **source control**, **CI (build & test)**, and **CD (deploy)**.

Source control is GitHub with a branch-per-environment strategy: `feature` → `develop` → `staging` → `main` (prod). Every PR requires a passing pipeline and a peer review before merge. No direct pushes to `main`.

CI runs on Jenkins. Every push triggers a pipeline that does:
1. Unit tests + integration tests
2. Static code analysis (SonarQube)
3. Docker image build
4. Image security scan (Trivy — critical CVEs fail the build)
5. Push to Amazon ECR with an immutable tag (`git SHA`, not `latest`)

CD is handled by ArgoCD in a GitOps model. Jenkins never touches the cluster directly. After the image is pushed, Jenkins updates the Helm values file in a separate `k8s-manifests` repo (sets `image.tag` to the new SHA). ArgoCD watches that repo and syncs the change to the cluster. Every deployment is a Git commit — full audit trail, rollback is a `git revert`.

**Infrastructure is Terraform** — VPC, EKS cluster, node groups, ECR repos, IAM roles — all version-controlled. No one clicks in the AWS console.

**Observability:** Prometheus + Grafana (metrics), Loki (logs), Alertmanager → PagerDuty (on-call). Every service has a RED dashboard — Request rate, Error rate, Duration.

**Why ArgoCD over `kubectl apply` in CI:** We once had a deployment with no audit trail. A developer applied a bad manifest on a Friday night, nobody knew what changed, and we spent 2 hours reverting manually. After ArgoCD, rollback is `git revert` and takes 60 seconds. That single incident convinced leadership to invest in GitOps.

---

## 2. Walk me through the CI/CD flow you followed in your project.

End to end — from a developer pushing code to users getting the update — this is the exact flow we ran in production.

**Step 1 — Developer pushes a feature branch and opens a PR.**

GitHub webhook triggers a Jenkins pipeline immediately on the PR branch (not main), so we catch issues before merge.

**Step 2 — Jenkins CI pipeline runs.**

```groovy
pipeline {
  agent any
  environment {
    ECR_REPO = '123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app'
  }
  stages {
    stage('Test')   { steps { sh 'pytest tests/ --cov=app' } }
    stage('Scan')   { steps { sh 'sonar-scanner -Dsonar.projectKey=my-app' } }
    stage('Build')  { steps { sh 'docker build -t $ECR_REPO:$GIT_COMMIT .' } }
    stage('Trivy')  { steps { sh 'trivy image --exit-code 1 --severity CRITICAL $ECR_REPO:$GIT_COMMIT' } }
    stage('Push')   { steps { sh 'docker push $ECR_REPO:$GIT_COMMIT' } }
    stage('Update Manifest') {
      steps {
        sh '''
          git clone https://github.com/our-org/k8s-manifests.git
          yq -i '.image.tag = "'$GIT_COMMIT'"' k8s-manifests/apps/my-app/values-staging.yaml
          cd k8s-manifests && git add . && git commit -m "ci: update my-app to $GIT_COMMIT" && git push
        '''
      }
    }
  }
}
```

We use `GIT_COMMIT` as the image tag — never `latest`. Every image is traceable to an exact commit. If something breaks in production, I know which commit introduced it.

**Step 3 — PR is merged to `develop`.**

Same pipeline runs. Manifest update targets `values-staging.yaml`.

**Step 4 — ArgoCD detects the manifest change and syncs to staging.**

ArgoCD polls the manifest repo every 3 minutes (or instantly via webhook). It shows a diff in the UI, then applies it. New pods roll out. Old pods terminate after a `preStop: sleep 10` to avoid 502s during the rollout.

**Step 5 — QA signs off on staging. PR from `develop` → `main` is raised.**

Pipeline runs one more time, now updating `values-prod.yaml`. ArgoCD syncs to the production cluster.

**Step 6 — Post-deploy smoke test.**

Jenkins hits the `/health` endpoint of the service after the manifest update. If it returns non-200, Jenkins sends a Slack alert and creates a Jira ticket. Rollback is a human call, not automatic — we've been burned by auto-rollbacks that masked the real issue.

**What the old flow looked like:** Shell scripts that SSH'd into EC2 instances and ran `git pull && systemctl restart app`. No rollback. No image scanning. No audit trail. We once deployed a container with a known CVE that had been in the base image for 6 months — nobody noticed because nobody was scanning. That incident started this whole migration.

**Key insight:** Jenkins has no cluster access. If Jenkins credentials are compromised, an attacker can only write to Git — they can't run arbitrary kubectl commands. Security is a real reason to adopt GitOps, not just an architectural preference.

---

## 3. What types of applications have you deployed using CI/CD pipelines?

This question is checking breadth of experience. The answer should cover different tech stacks and deployment targets — not just "Node.js to Kubernetes."

**What we've deployed:**

- **Python microservices (Flask/FastAPI)** → Docker → EKS. Most of our backend services. Pipeline runs pytest, SonarQube, Trivy, pushes to ECR, updates Helm values for ArgoCD.

- **Node.js services** → Same Docker + EKS flow. One nuance: npm audit runs in CI to catch vulnerable dependencies before the image is even built.

- **Java Spring Boot** → Docker multi-stage build (builder stage with Maven, final stage with JRE only). Spring Boot jars are large — multi-stage build cut our image from 900MB to 280MB. JVM services also need longer `initialDelaySeconds` in readiness probes — we've been burned by this twice.

- **React/Next.js frontend** → Build artifacts (static files) pushed to S3, CloudFront invalidation triggered by Jenkins. This isn't a container deploy — it's `npm run build` → `aws s3 sync dist/ s3://my-bucket` → `aws cloudfront create-invalidation`.

- **Lambda functions** → Python ZIP packages. Pipeline runs tests, packages dependencies with the function code, pushes to S3, triggers `aws lambda update-function-code`. For larger deployments, we use Terraform to manage the Lambda resource and the CI pipeline only updates the artifact.

- **Terraform itself** → We have a pipeline that runs `terraform plan` on PRs (output posted as a PR comment) and `terraform apply` on merge to `main`. Every infra change goes through code review.

**The key thing interviewers want to hear:** You've deployed more than one type. Containerized microservices, static frontends, serverless functions, and IaC all have different pipeline shapes. Knowing how to adapt the pipeline to the deployment target is the skill.

---

## 4. Which deployment tools have you used — Docker, Kubernetes, Helm, Terraform?

This is a "show me you've actually used these together, not just in isolation" question.

**How these tools fit together in our stack:**

**Docker** is the packaging layer. Every application we deploy is a Docker image. The pipeline builds the image, scans it, and pushes it to ECR with an immutable tag. We never use `latest` in production — every image tag is the Git commit SHA. This means every running container in production is traceable to an exact commit.

**Helm** is the Kubernetes packaging layer. We write Helm charts for every service — one chart per service with `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`. The chart templates the Deployment, Service, Ingress, HPA, PDB, and ServiceAccount. When the CI pipeline updates the image tag, it updates the Helm values file. ArgoCD uses Helm to render and apply the manifests.

**Kubernetes (EKS)** is the runtime. Manages scheduling, scaling, health checks, service discovery. We manage the cluster itself with Terraform. We use Karpenter for node autoscaling — it replaced Cluster Autoscaler and reduced new node provisioning from 3 minutes to under 60 seconds.

**Terraform** manages everything that isn't application code — EKS cluster, VPC, subnets, security groups, IAM roles, ECR repos, RDS, ElastiCache, S3 buckets. We use the `terraform-aws-modules/eks/aws` module as a base. State is in S3 with DynamoDB locking. Every Terraform change goes through a pipeline: `plan` on PR, `apply` on merge.

**How they connect in a real deploy:**

```
Developer merges code
    → CI: docker build → trivy scan → docker push ECR
    → CI: yq update image tag in Helm values → git push
    → ArgoCD: detects values change → helm template → kubectl apply
    → EKS: pods roll out on Karpenter-managed nodes
    → Prometheus: scrapes metrics → Grafana alert if error rate spikes
```

Terraform sits outside this loop — it provisions the infrastructure that this flow runs on. If we need a new EKS node group or a new ECR repo, that's a Terraform PR, not a CI deploy.

---

## 6. How do you ensure stability and efficiency across your CI/CD and cloud environments?

This is a systems-thinking question. The answer spans process, tooling, and culture — not just listing tools.

**Stability: preventing and recovering from failures.**

**1. Environment parity — staging must mirror production.**

The most common cause of "works in staging, breaks in prod" is environmental differences. We enforce parity via:

```hcl
# Same Terraform module for staging and prod — only variables differ
module "eks" {
  source         = "../modules/eks"
  cluster_name   = var.cluster_name       # "staging-cluster" vs "prod-cluster"
  instance_types = var.instance_types     # ["t3.large"] vs ["m5.large"]
  min_size       = var.min_size           # 2 vs 3
}
```

Schema migrations run in staging first. Config changes go through the same Terraform pipeline for both environments. If staging and prod drift, the drift gets caught in Terraform plan.

**2. Progressive delivery — don't ship to all users at once.**

Every significant change goes through: staging → canary (5% prod traffic) → prod (100%). The canary stage catches regressions that staging misses because staging traffic is synthetic.

```yaml
# Argo Rollouts canary with auto-abort
analysis:
  templates:
    - templateName: success-rate   # PromQL: error rate < 2%
  args:
    - name: service-name
      value: order-service
# If error rate exceeds 2% at 5% traffic → auto-abort → 0 users affected
```

**3. Health checks and automatic rollback in the pipeline.**

The pipeline only succeeds when the deployment is confirmed healthy:

```groovy
sh 'argocd app wait order-service --health --timeout 300'
sh 'curl -f https://api.prod.com/health'     // Smoke test
// If either fails → kubectl rollout undo → Slack alert → pipeline marked failed
```

**4. Resource limits and PodDisruptionBudgets to prevent cascading failures.**

```yaml
# Every service has memory limits (prevent OOM cascade)
resources:
  limits:
    memory: "512Mi"

# PDB prevents all pods being evicted simultaneously during node maintenance
apiVersion: policy/v1
kind: PodDisruptionBudget
spec:
  minAvailable: 2   # Always keep at least 2 pods running
```

---

**Efficiency: getting value out of CI/CD without wasting time or money.**

**1. Parallel pipeline stages — cut build time.**

Sequential: test → lint → build → scan = 18 minutes
Parallel (test + lint + scan simultaneously, then build): 7 minutes

```groovy
stage('Validate') {
  parallel {
    stage('Tests') { steps { sh 'pytest tests/' } }
    stage('Lint')  { steps { sh 'flake8 app/' } }
    stage('SAST')  { steps { sh 'bandit -r app/' } }
  }
}
```

**2. Layer caching in Docker builds — don't reinstall dependencies every build.**

```dockerfile
# Dependencies change rarely → cache them
COPY requirements.txt .
RUN pip install -r requirements.txt   ← cached unless requirements.txt changes

# Code changes frequently → copy it last
COPY src/ ./src/
```

Without proper layer ordering: 4-minute Docker build every commit. With caching: 45-second build if only source code changed.

**3. Spot instances for CI agents — 70% cost reduction.**

Build agents don't need to be on-demand. A PR build that gets interrupted by spot interruption just restarts — a minor inconvenience, not an outage. On-demand for production deploys; spot for PR builds and test runs.

**4. Fail fast — expensive stages run last.**

```
Order: lint (5s) → unit tests (30s) → integration tests (2m) → build image (3m) → push (1m)
```

Lint failure aborts before the 6-minute build+push. A developer gets feedback in 35 seconds instead of 7 minutes.

**5. Artifact reuse — build once, deploy everywhere.**

The same Docker image (same SHA) goes through staging, canary, and prod. It's built once in CI, not rebuilt per environment. This ensures what passed staging tests is exactly what runs in prod.

---

**Stability + Efficiency together: GitOps.**

GitOps (ArgoCD watching the manifest repo) ties both together. Stability: every deployment is a Git commit — full audit trail, rollback is a `git revert`. Efficiency: ArgoCD continuously reconciles desired vs actual state — if a pod gets manually modified in prod, ArgoCD corrects it automatically. Zero manual `kubectl apply` in production.

**Real metric:** Before this architecture, our mean time to deploy (code merged → in production) was 45 minutes. Mean time to detect a regression was 2 hours (manual monitoring). After: deploy time 12 minutes, regression detection 4 minutes (automated smoke tests + alert correlation in Grafana). MTTR dropped from 47 minutes average to 8 minutes (auto-rollback catches most issues before users notice).

---

## 5. A deployment just caused widespread failures across the platform — how do you respond?

"Widespread failures" means the deployment didn't just break one service — it broke something shared. This could be a common library, a shared config, or a central service that everything depends on.

**First 60 seconds — rollback, don't investigate.**

When the blast radius is wide (multiple services failing), the priority is restoration, not root cause. Every minute of investigation while users experience widespread failure costs more than a rollback.

```bash
# Rollback the deployment immediately
kubectl rollout undo deployment/<service-name> -n prod

# If it's a shared Helm release
helm rollback <release-name> 1 -n prod

# In ArgoCD — revert the Git commit that caused the change
git revert <bad-commit-sha>
git push    # ArgoCD detects the change and syncs rollback
```

**Communicate before you've fully diagnosed:**

```
🔴 INCIDENT: Widespread failures following deployment of [service] at [time]
Impact: [X] services affected, error rates elevated
Action: Rolling back [service] deployment now
ETA: 2-3 minutes for rollback to complete
Next update: [current time + 10 minutes]
```

**After rollback — verify it actually fixed it:**

```bash
# Watch error rates in Grafana — should drop within 2 minutes of rollback
# Check pod readiness
kubectl get pods -n prod -w | grep -v Running

# Hit key endpoints manually
curl -s https://api.yourapp.com/health
curl -s https://api.yourapp.com/checkout/health
```

If error rates don't drop after rollback — the rollback didn't fix the issue, or there's an additional compounding factor. Check if the rollback itself introduced a problem (sometimes rolling back a schema migration breaks things too).

**After stabilization — root cause:**

Now that users aren't affected, investigate what in the deployment caused widespread failures.

```bash
# What changed in this deployment?
git show <bad-commit-sha>
kubectl diff -f manifests/

# What do the logs show at the time of failure?
kubectl logs -n prod -l app=<service> --since=30m | grep ERROR
```

Common causes of deployment-triggered widespread failures:
- **Shared library update with breaking change** — updated a common package that changed an interface all services use
- **Centralized auth/discovery service deployed** — every service calls auth; auth breaks; every service fails
- **Config change in a shared ConfigMap** — one ConfigMap change affected all pods that mount it
- **Database schema migration** — migration ran, old pods can't read the new schema
- **ECR image tag mismatch** — ArgoCD applied a values file pointing to a wrong image tag (empty string, wrong SHA)

**Document and prevent recurrence:**

For widespread deployment failures specifically, the postmortem should ask:
- Was this tested in staging? Why didn't staging catch it?
- Was there a deployment freeze for high-risk changes?
- Do we have canary deployments for this service to limit blast radius?
- Can we make this change backward-compatible so rollback is safer?
