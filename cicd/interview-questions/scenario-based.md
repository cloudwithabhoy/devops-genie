# CI/CD — Scenario-Based Questions

---

## 1. Explain your project architecture.

> **Also asked as:** "What are microservices and explain you used in your previous company projects ?"

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

> **Also asked as:** "How to configure CI/CD using Jenkins ?"
> **Also asked as:** "How is the CI/CD pipeline is setup in your project? What are the security tools integrated?"

End to end — from a developer pushing code to users getting the update — this is the exact flow we ran in production.

---

## 3. How do you manage the security tools integrated in your pipeline?

> **Also asked as:** "How do you manage them?" (following the security tools question)

Managing security tools isn't just about adding a line to a Jenkinsfile. It requires a strategy for **onboarding**, **policy enforcement**, and **remediation**.

**1. Centralized Configuration (Policy as Code):**
We don't configure scan rules in individual pipelines. Instead, we use centralized configuration files (e.g., `sonar-project.properties` or `.trivyignore`) stored in a central repository or managed via UI (like SonarQube Quality Gates). This ensures every team follows the same security bar.

**2. Quality Gates & Failure Thresholds:**
We set "Quality Gates" in our CI/CD. If a SAST scan finds a **Critical** vulnerability or code coverage drops below 80%, the pipeline fails automatically. 
*Rule:* You cannot merge to `main` until the security gate is green.

**3. Suppression & Exception Handling:**
If a vulnerability is found but is a false positive or has a mitigation (e.g., the service isn't exposed to the internet), we use a formal exception process. We add the CVE ID to an allowlist file with a justification and an expiration date.

**4. Continuous Monitoring & Dashboarding:**
We aggregate scan results into a central dashboard (like SonarQube or AWS Security Hub). This allows us to track "Security Debt" across all microservices and prioritize which teams need to focus on patching.

**5. Secret Management:**
For the tools itself (API keys for SonarQube/Snyk), we store them in **Jenkins Credentials** or **HashiCorp Vault**. They are injected into the pipeline at runtime and never printed to logs.

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

> **Also asked as:** "How do you deploy microservices in AWS ?"

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

---

## 7. You have an `index.html` in a GitHub repository. The requirement is to deploy it to AWS behind a load balancer, using the repository's username and token. How do you achieve this?

This is a common beginner scenario that combines GitHub (source), a CI/CD pipeline (automation), AWS EC2 (server), and an ALB (load balancer). The answer walks through each component.

**Architecture:**

```
GitHub repo (index.html)
    ↓ push trigger
GitHub Actions (CI/CD pipeline)
    ↓ SSH or AWS CLI
EC2 Instance (Apache/Nginx serving index.html)
    ↓
ALB (Application Load Balancer)
    ↓
User's browser
```

**Step 1: AWS infrastructure — EC2 + ALB.**

```
VPC
└── Public Subnet
    ├── EC2 instance (Apache installed, port 80)
    └── ALB (listener port 80 → Target Group → EC2)
```

The EC2 instance runs Apache (or Nginx) and serves files from `/var/www/html/`. The ALB sits in front and distributes traffic.

**Step 2: Set up Apache on the EC2 instance.**

```bash
# On the EC2 instance (Amazon Linux 2 / Ubuntu)
sudo apt update && sudo apt install apache2 -y   # Ubuntu
# or
sudo yum install httpd -y && sudo systemctl start httpd   # Amazon Linux

# Web root — this is where index.html must live
/var/www/html/
```

**Step 3: Store credentials as GitHub Secrets.**

In your GitHub repository:

```
Settings → Secrets and variables → Actions → New repository secret

EC2_HOST      = <EC2 public IP or DNS>
EC2_USER      = ubuntu   (or ec2-user for Amazon Linux)
EC2_SSH_KEY   = <contents of the .pem private key file>
```

You do NOT store GitHub username/token in the repo itself — you use GitHub Secrets for all sensitive values. The token mentioned in the question is used to authenticate GitHub Actions access to the repository if it's private.

**Step 4: GitHub Actions workflow.**

```yaml
# .github/workflows/deploy.yml
name: Deploy index.html to AWS EC2

on:
  push:
    branches: [main]   # Trigger on push to main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      # 1. Check out the code (uses GITHUB_TOKEN automatically for public/same repo)
      - name: Checkout repository
        uses: actions/checkout@v4
        # For a private repo accessed by another service, you'd use:
        # with:
        #   token: ${{ secrets.GH_PAT }}

      # 2. Copy index.html to EC2 via SCP
      - name: Deploy index.html via SCP
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          source: "index.html"
          target: "/var/www/html/"

      # 3. Restart Apache to ensure it's serving the latest file
      - name: Restart Apache
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            sudo systemctl restart apache2
            echo "Deployment complete. index.html served from /var/www/html/"
```

**Alternative: pull from GitHub directly on the EC2 instance (using username + token).**

If you want the EC2 instance to `git pull` the repo itself (rather than CI pushing files to it):

```yaml
- name: SSH and pull latest from GitHub
  uses: appleboy/ssh-action@v1.0.3
  with:
    host: ${{ secrets.EC2_HOST }}
    username: ${{ secrets.EC2_USER }}
    key: ${{ secrets.EC2_SSH_KEY }}
    script: |
      cd /var/www/html
      # Pull using GitHub username and token embedded in URL
      git pull https://${{ secrets.GH_USERNAME }}:${{ secrets.GH_TOKEN }}@github.com/org/repo.git main
      sudo systemctl reload apache2
```

**Step 5: ALB Health Check.**

The ALB Target Group needs a health check path. Since you're serving `index.html`, configure:

```
Health check path: /index.html  (or just /)
Health check port: 80
Expected response: 200
```

The EC2 instance is healthy → ALB routes traffic to it → users access `http://<ALB-DNS>/` and see `index.html`.

**What the username and token are used for:**

- **GitHub username** — identifies who is making the API/Git request
- **GitHub token (PAT)** — authenticates the request (replaces password). Used when the repository is private, when the EC2 does a `git pull` on startup, or when a script clones the repo. In GitHub Actions itself, `GITHUB_TOKEN` is auto-provided for same-repo operations — you only need a PAT for cross-repo access or external scripts.

**Security note:** Never hardcode the token in the workflow YAML or in scripts committed to the repo. Always store it in GitHub Secrets or AWS Secrets Manager.

---

## 8. Blue/Green vs Canary rollout — when to pick what?

> **Also asked as:** "Blue/Green vs Canary rollout — when to pick what?"
> **Also asked as:** "How do you ensure continuous deployment and zero downtime for microservices ?"

Both reduce deployment risk compared to a standard Rolling Update, but they optimize for completely different things.

### Blue/Green Deployment (The "Switch")
You run two identical environments. Blue is serving prod traffic. You deploy the new code to Green, run tests on Green safely, and then flip the load balancer router to send 100% of traffic to Green.
- **When to use it:** Legacy monoliths, heavy stateful applications, or deployments involving major breaking schema changes.
- **The Tradeoff:** You need 200% of the infrastructure capacity during the deployment. It's expensive.
- **The Rollback:** Instantaneous. Flip the router back to Blue.

### Canary Deployment (The "Blast Radius")
You deploy the new version alongside the old version. You route just 1% of live user traffic to the new version. You monitor error rates and latency. If metrics are healthy, you scale up to 10%, 25%, 50%, 100%. If metrics spike, you automatically abort and route back to the old version.
- **When to use it:** Modern microservices, SaaS platforms, high-traffic consumer apps.
- **The Tradeoff:** Requires excellent observability infrastructure (Prometheus/Datadog) to make automated go/no-go decisions. You are testing in production, which means that 1% of users *will* experience errors if the code is broken.
- **The Rollback:** Gradual routing rollback, but automated.

**Summary:** If your app takes 10 minutes to boot up and connects to a legacy database, use Blue/Green. If you deploy a lightweight Go microservice 10 times a day, use Canary via Argo Rollouts.

---

## 9. Describe Vault + GitHub Actions integration securely

> **Also asked as:** "Describe Vault + GitHub Actions integration securely"

Integrating HashiCorp Vault with GitHub Actions is a classic "Secret Zero" problem. How does the pipeline authenticate to Vault without us hardcoding a Vault token inside a GitHub Secret?

**The Anti-Pattern:** Generating a long-lived Vault AppRole Token and saving it in GitHub Secrets. If that secret leaks, the attacker has permanent access to Vault.

**The Modern Solution: OIDC (OpenID Connect)**
You establish a trust relationship between Vault and GitHub based on identity, not passwords.

1. **GitHub Context:** Every run of a GitHub Action generates a short-lived JSON Web Token (JWT) signed by GitHub's private key. This token contains metadata about the job (`repository: my-org/my-repo`, `ref: refs/heads/main`).
2. **Vault Configuration:** You configure Vault's OIDC auth method to trust GitHub's public OIDC endpoints (`https://token.actions.githubusercontent.com`).
3. **The Bind:** You create a Vault Role that says: "If a JWT comes in, verified by GitHub's public key, and the repository is exactly `my-org/my-repo` and the branch is `main`, assign them the `prod-deployer` Vault policy."
4. **The Pipeline Run:** 
   - GitHub Actions requests the Vault JWT.
   - It sends the JWT to Vault.
   - Vault cryptographically verifies the signature with GitHub.
   - Vault issues a short-lived, ephemeral Vault Token to the pipeline (valid for maybe 5 minutes).
   - The pipeline fetches the secrets and exits. The token expires immediately.

**Why this matters in an interview:** It proves you understand identity federation. There are no passwords to rotate, no long-lived tokens to steal, and access is tied definitively to the execution of code on the `main` branch.
