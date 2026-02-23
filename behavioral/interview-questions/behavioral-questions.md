# Behavioral Questions

---

## 1. A critical production release is scheduled tomorrow. During final validation, you discover a configuration issue that may impact live customer transactions. Business still wants to go live. What will you do as a DevOps Engineer?

This question checks one thing: can you make the right technical decision under production pressure?

**Situation:**

In one of my previous projects, we had a major production release scheduled the next day. During final validation in staging, we identified a configuration mismatch in the deployment pipeline that could potentially affect transaction processing for a subset of users.

Timelines were tight. Business was pushing hard to release.

**Task:**

As a DevOps Engineer, my responsibility was to:
- Assess the production risk
- Validate the technical severity
- Provide clear release options
- Ensure system stability and rollback readiness

**Action:**

**1. Immediate Impact Analysis**
- Identified affected services and dependent components
- Checked logs, monitoring dashboards, and test results
- Assessed whether issue was data-impacting or availability-impacting

**2. Technical Validation**
- Reproduced the issue in staging
- Coordinated with developers and QA to confirm scope
- Verified if configuration fix could be safely applied before release

**3. Prepared Controlled Options for Leadership**

I presented two structured choices:
- Proceed with release + mitigation plan + enhanced monitoring
- Delay release until configuration fix is tested and validated

Additionally, I ensured:
- Rollback strategy was tested
- Backup snapshots were ready
- Deployment was automated and reversible
- All risks were documented formally

**Result:**

Leadership decided to go for a controlled release with:
- Temporary mitigation
- Real-time monitoring alerts enabled
- War-room support during release window
- Permanent fix planned in next patch release

---

## 2. Explain your roles and responsibilities.

Structure this around the three pillars of DevOps work: build/release, infrastructure, and reliability.

"As a DevOps Engineer, my responsibilities span three main areas:

**CI/CD and Release Management:**
- Own and maintain the CI/CD pipelines using Jenkins and GitHub Actions. This includes writing and maintaining Jenkinsfiles and GitHub Actions workflows, managing shared libraries, configuring build agents, and ensuring deployments to Dev, QA, Staging, and Production are fully automated.
- I'm responsible for release coordination — scheduling production deployments, managing rollback procedures, and ensuring deployment windows are respected.

**Infrastructure and Cloud:**
- Manage our AWS infrastructure using Terraform. I write and maintain modules for EKS clusters, RDS instances, ALB/NLB, VPCs, IAM roles, and security groups. All infrastructure changes go through code review and a Terraform plan review before apply.
- Monitor and optimize cloud costs. I review the AWS Cost Explorer monthly and implement rightsizing recommendations from Compute Optimizer.

**Monitoring, Reliability, and On-Call:**
- Set up and maintain our observability stack — Prometheus for metrics, Loki for logs, Grafana for dashboards and alerting, Alertmanager for notification routing.
- Participate in the on-call rotation. When incidents occur I lead triage, communicate status to stakeholders, and write the post-incident review.
- I work closely with developers to improve deployment frequency and reduce the feedback loop — code review on Dockerfiles, helping debug failing pipelines, and running internal workshops on CI/CD best practices."

---

## 3. Explain about your current project.

Frame it as: what the product does → tech stack → your specific contribution → challenges solved.

"I'm working at [Company] on a B2B SaaS platform — it's a supply chain management product used by mid-sized manufacturers to track inventory, orders, and logistics.

On the infrastructure side, we run on AWS with EKS for container orchestration, RDS PostgreSQL for the database, and ElastiCache Redis for caching. The platform has about 15 microservices, each owned by a different squad.

My team (platform engineering — 3 engineers) is responsible for the shared infrastructure that all squads depend on: the EKS cluster, the CI/CD pipelines, the monitoring stack, and the Terraform modules.

**What I specifically own:**
- The Jenkins-based CI/CD pipelines. Each microservice has a Jenkinsfile, and we maintain a shared library with common stages (build, scan, push to ECR, deploy via Helm).
- The Terraform modules for EKS, ALB, and RDS. When a squad needs a new service, they use our module and get a production-ready setup in minutes rather than days.
- The Prometheus + Grafana + Alertmanager monitoring stack. I built the initial setup and maintain the alerting rules and Grafana dashboards.

**Recent challenge I solved:** Our deployment pipeline was taking 35 minutes per service because builds were sequential and we had no Docker layer caching. I implemented parallel builds across services and enabled BuildKit caching in Jenkins, reducing average pipeline time to 9 minutes. This unblocked squads from deploying more frequently — we went from 10 deployments/week to 40+."

---

## 14. Explain the deployment strategy used in your organisation.

"In our organisation we use different strategies depending on the risk profile of the change.

**For most services — Rolling Update (default):**
Our Kubernetes deployments use a rolling update strategy. New pods come up with the new version while old pods are gradually terminated. At no point are all old pods down at the same time.

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # One extra pod allowed during rollout
    maxUnavailable: 0  # Never reduce below desired count
```

Zero downtime, simple to configure. Rollback is a `kubectl rollout undo` or a re-deploy of the previous image tag via ArgoCD.

**For high-risk or database-schema changes — Blue/Green:**
We maintain two identical environments (Blue = live, Green = new version). Deploy to Green, run smoke tests, then switch the ALB target group from Blue to Green. If issues arise, we flip back to Blue in under 2 minutes — no Kubernetes rollout needed.

**For new features with user impact — Canary:**
We use Argo Rollouts for canary deployments on critical services. 5% of traffic goes to the canary version first. We watch error rate and latency for 15 minutes. If metrics are healthy, we progress to 25%, 50%, 100%. If at any stage the error rate exceeds our SLO threshold, Argo Rollouts automatically rolls back.

**The decision flow:**
- Low-risk bug fix or config change → Rolling Update
- New feature with schema change → Blue/Green
- High-risk feature impacting checkout or payments → Canary
- Hotfix → Rolling Update with shortened validation window"
