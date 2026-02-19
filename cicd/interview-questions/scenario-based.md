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

## 3. Which type of Jenkins pipeline do you use and why?

We use **Declarative pipelines** for all production pipelines. The answer isn't "Declarative is better" — it's understanding why it fits the team.

**Declarative is what we chose because:**

- **It's reviewable.** Every engineer on the team can read and understand a Declarative Jenkinsfile without knowing Groovy. When a pipeline fails at 2 AM, the on-call engineer can read the file and understand exactly what each stage does. With Scripted pipelines, you often end up with nested Groovy loops and closures that only the original author understands.

- **It has guardrails.** Declarative syntax validates the pipeline structure before it runs. If you have a syntax error in a `stage` block, Jenkins tells you immediately. Scripted pipelines fail at runtime — halfway through a build.

- **`post {}` blocks are clean.** Declarative has built-in `post { success {} failure {} always {} }` at both the pipeline and stage level. In Scripted, you have to wrap everything in try/catch/finally manually. We had a Scripted pipeline where a developer forgot the `finally` block — docker-compose containers leaked on the agent for weeks.

- **Shared Library compatibility.** When we moved to Shared Libraries, Declarative made it easy to call library functions as steps. The pipeline structure remained readable; complex logic moved to the library.

**When Scripted makes sense:**

Scripted is Groovy — fully programmable. If you need dynamic stage generation (e.g., loop over 15 microservices and create a stage per service), Declarative can't do it cleanly. We have one legacy pipeline that generates stages dynamically from a config file — that one stays Scripted.

```groovy
// Scripted — dynamic stages
def services = ['auth', 'orders', 'payments', 'notifications']
node {
  stage('Build All') {
    def parallelStages = [:]
    services.each { svc ->
      parallelStages[svc] = {
        sh "docker build -t ecr//${svc}:${GIT_COMMIT} ./${svc}"
      }
    }
    parallel parallelStages
  }
}
```

**Real scenario:** We inherited a Scripted monolith pipeline from a previous team — 600 lines of Groovy, multiple nested closures, no comments. When the original author left, nobody could safely modify it. Every change was trial-and-error. We rewrote it in Declarative over a weekend. New pipeline: 80 lines. Every engineer can modify it confidently.

---

## 4. What types of applications have you deployed using Jenkins pipelines?

This question is checking breadth of experience. The answer should cover different tech stacks and deployment targets — not just "Node.js to Kubernetes."

**What we've deployed:**

- **Python microservices (Flask/FastAPI)** → Docker → EKS. Most of our backend services. Pipeline runs pytest, SonarQube, Trivy, pushes to ECR, updates Helm values for ArgoCD.

- **Node.js services** → Same Docker + EKS flow. One nuance: npm audit runs in CI to catch vulnerable dependencies before the image is even built.

- **Java Spring Boot** → Docker multi-stage build (builder stage with Maven, final stage with JRE only). Spring Boot jars are large — multi-stage build cut our image from 900MB to 280MB. JVM services also need longer `initialDelaySeconds` in readiness probes — we've been burned by this twice.

- **React/Next.js frontend** → Build artifacts (static files) pushed to S3, CloudFront invalidation triggered by Jenkins. This isn't a container deploy — it's `npm run build` → `aws s3 sync dist/ s3://my-bucket` → `aws cloudfront create-invalidation`.

- **Lambda functions** → Python ZIP packages. Pipeline runs tests, packages dependencies with the function code, pushes to S3, triggers `aws lambda update-function-code`. For larger deployments, we use Terraform to manage the Lambda resource and Jenkins only updates the artifact.

- **Terraform itself** → We have a pipeline that runs `terraform plan` on PRs (output posted as a PR comment) and `terraform apply` on merge to `main`. Every infra change goes through code review.

**The key thing interviewers want to hear:** You've deployed more than one type. Containerized microservices, static frontends, serverless functions, and IaC all have different pipeline shapes. Knowing how to adapt the pipeline to the deployment target is the skill.

---

## 5. Which deployment tools have you used — Docker, Kubernetes, Helm, Terraform?

This is a "show me you've actually used these together, not just in isolation" question.

**How these tools fit together in our stack:**

**Docker** is the packaging layer. Every application we deploy is a Docker image. The pipeline builds the image, scans it, and pushes it to ECR with an immutable tag. We never use `latest` in production — every image tag is the Git commit SHA. This means every running container in production is traceable to an exact commit.

**Helm** is the Kubernetes packaging layer. We write Helm charts for every service — one chart per service with `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`. The chart templates the Deployment, Service, Ingress, HPA, PDB, and ServiceAccount. When Jenkins updates the image tag, it updates the Helm values file. ArgoCD uses Helm to render and apply the manifests.

**Kubernetes (EKS)** is the runtime. Manages scheduling, scaling, health checks, service discovery. We manage the cluster itself with Terraform. We use Karpenter for node autoscaling — it replaced Cluster Autoscaler and reduced new node provisioning from 3 minutes to under 60 seconds.

**Terraform** manages everything that isn't application code — EKS cluster, VPC, subnets, security groups, IAM roles, ECR repos, RDS, ElastiCache, S3 buckets. We use the `terraform-aws-modules/eks/aws` module as a base. State is in S3 with DynamoDB locking. Every Terraform change goes through a Jenkins pipeline: `plan` on PR, `apply` on merge.

**How they connect in a real deploy:**

```
Developer merges code
    → Jenkins: docker build → trivy scan → docker push ECR
    → Jenkins: yq update image tag in Helm values → git push
    → ArgoCD: detects values change → helm template → kubectl apply
    → EKS: pods roll out on Karpenter-managed nodes
    → Prometheus: scrapes metrics → Grafana alert if error rate spikes
```

Terraform sits outside this loop — it provisions the infrastructure that this flow runs on. If we need a new EKS node group or a new ECR repo, that's a Terraform PR, not a Jenkins deploy.

---

## 6. A deployment just caused widespread failures across the platform — how do you respond?

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
