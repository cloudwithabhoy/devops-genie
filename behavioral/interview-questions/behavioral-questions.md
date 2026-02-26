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

> Also asked as: "What are your roles and responsibilities in your current position?"

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

> Also asked as: "Can you explain your current project?"
> **Also asked as:** "What's the architecture of your current infra and what do you own?"
> **Also asked as:** "Explain AWS services and how you used in your previous projects ?"

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

> Also asked as: "What deployment strategies are used in your organization?"

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

---

## 4. Tell me about yourself.

> **Also asked as:** "Tell me about yourself."

This is not a biography. It's a **90-second pitch** structured as: who you are → what you do → what makes you relevant to this role.

**The framework: Present → Past → Future.**

```
Present:  What you're doing now (role, company, tech stack)
Past:     Key experience that's relevant (1-2 highlights)
Future:   Why you're here (what excites you about this role)
```

---

## 5. Walk me through your day-to-day activities as a DevOps Engineer.

> **Also asked as:** "How your day to day activities as a DevOps Engineer."

The interviewer wants to see if you are proactive or just a "ticket taker." A good answer shows a mix of automation, monitoring, and collaboration.

**Morning: Health Check & Triage**
- **Check Slack/Monitoring:** Review alerts from Prometheus/Grafana or PagerDuty. Did any 2 AM cron jobs fail? Did the production error rate spike?
- **Pipeline Review:** Check for failed nightly builds or integration tests. If a build failed, communicate with the dev team to resolve blocks.

**Mid-Day: Project Work (The "Engineering" part)**
- **Infrastructure as Code (IaC):** Developing new Terraform modules or Ansible playbooks (e.g., migrating a service to a new AWS region or optimizing EKS node groups).
- **CI/CD Optimization:** Improving pipeline speed (caching, parallel testing) or integrating a new security tool (SAST/DAST).
- **Containerization:** Debugging Dockerfiles for the dev team or fine-tuning K8s resource limits.

**Afternoon: Collaboration & Meetings**
- **Stand-ups:** Sync with developers to understand upcoming infrastructure needs for new features.
- **Root Cause Analysis (RCA):** If there was an issue, document the fix to prevent it from happening again.
- **Code Reviews:** Reviewing PRs for IaC or Jenkinsfiles from other team members.

**Wrap-up: Documentation & Training**
- Updating the internal Wiki/Docs (e.g., recording the steps for a new deployment flow).
- Mentoring junior devs on how to use the new logging platform.

**Example (customize with your details):**

"I'm currently working as a DevOps Engineer at [Company], where I manage the infrastructure for a microservices-based platform running on AWS EKS. My day-to-day involves maintaining CI/CD pipelines with Jenkins, managing infrastructure through Terraform, and running our monitoring stack with Prometheus and Grafana.

Before this, I worked at [Previous Company] where I led the migration from EC2-based deployments to containerized workloads on Kubernetes. That experience gave me deep hands-on knowledge of Docker, Helm, and cluster operations.

One of my recent wins was redesigning our CI/CD pipeline — we reduced deployment time from 35 minutes to 9 minutes by implementing parallel builds and Docker layer caching. This increased team deployment frequency from 10 to 40+ deploys per week.

I'm excited about this role because [reason specific to the company — their scale, tech challenges, product]."

**What NOT to do:**
- Don't recite your resume chronologically — the interviewer already has it
- Don't go beyond 2 minutes — keep it tight
- Don't include personal details (hobbies, family) unless asked
- Don't be vague — "I work with cloud technologies" says nothing. "I manage 3 EKS clusters across 2 AWS regions" is specific and credible

> **Also asked as:** "What is your role in your company?" — covered in Q2 above (roles and responsibilities structured around CI/CD, infrastructure, and reliability). "Explain about your current project" — covered in Q3 above.

---

## 5. What branching strategy does your team follow?

"We follow a trunk-based development strategy with short-lived feature branches, optimized for fast CI/CD feedback. 

**The Workflow:**
- We have one primary branch: `main`. It always represents the deployable state of production.
- When an engineer picks up a Jira ticket, they create a feature branch off `main` (e.g., `feature/JIRA-123-add-redis-caching`).
- This feature branch is short-lived — ideally merged back within 1-2 days. 
- We don't use long-running `develop` or `release` branches. What I found in previous projects is that long-living branches lead to massive merge conflicts and slow down the release cadence.

**Result:**
Over three months, "ops requests" from the dev team dropped by 70%. Developers felt empowered to manage their own service lifecycles from code commit to production monitoring, allowing the DevOps team to focus on migrating our database backend to a new region instead of fighting daily operational fires.

---

## 9. Share a time prod broke and how you led the fix

> **Also asked as:** "Share a time prod broke and how you led the fix"

This question isn't about how good you are at typing commands; it's about how you manage chaos, communicate under pressure, and drive a resolution.

**Situation:**
Two days before a major marketing push, our primary user authentication service went down. Customers couldn't log in, and the support queue was blowing up. Our synthetic monitoring triggered a P1 Page.

**Task:**
As the primary on-call engineer, I had to triage the issue, mitigate the customer impact as quickly as possible, and coordinate the engineering response.

**Action:**
1. **Acknowledge and Commandeer:** I immediately acknowledged the page, jumped into the `#incident-auth` Slack channel, and stated clearly: *"I am the Incident Commander. Investigating Auth API 5xx errors."* This stopped 5 other engineers from duplicating efforts.
2. **Triage The Bleeding:** I checked the Grafana dashboards. The DB connections for the auth service were maxed out. CPU was saturated. 
3. **Mitigate First, Investigate Later:** I didn't try to find the bad query yet. I needed the site back up. I instructed the database engineer to scale the RDS instance up to the next instance class (a 5-minute operation) to buy us breathing room, while I temporarily scaled up the auth service pods.
4. **Isolate:** Once the site was limping along (slow but working), I dug into Datadog APM. I found a single new endpoint introduced in a deployment 3 hours earlier that was performing a full table scan without an index.
5. **Remediate:** I asked the developer who authored the PR to verify the endpoint wasn't critical for login. It wasn't (it was a background profile sync). We immediately reverted that PR and pushed the hotfix through the CI pipeline.

**Result:**
The system stabilized 22 minutes after the page. I ran the post-mortem the next day, which resulted in two action items: adding query cost analysis to our CI pipeline, and separating the auth read path from the heavy background sync path. The marketing push went off without a hitch.

---

## 10. How do you learn tools under pressure?

> **Also asked as:** "How do you learn tools under pressure?"

DevOps requires constantly touching technologies you've never seen before. Interviewers want to know your framework for rapid comprehension when the stakes are high.

**Situation:**
I was tasked with debugging a critical latency issue in a legacy project that used Elasticsearch. I had never managed an Elasticsearch cluster before, and the production site was painfully slow.

**Action:**
When learning under pressure, I abandon "bottom-up" learning (reading the full manual) and switch to "top-down, problem-oriented" learning:
1. **Find the Error, Not the Tutorial:** I don't Google "How does Elasticsearch work." I go straight to the logs, find the exact error code (e.g., `es_rejected_execution_exception`), and research that specific symptom.
2. **Isolate in a Sandbox:** I never test theories in Prod. I spun up a local Docker container of the same Elasticsearch version. I deliberately tried to reproduce the error by flooding it with requests to understand the breakage mechanism.
3. **Documentation over Medium Articles:** I rely strictly on official documentation for configuration architecture (the "RTFM" rule). I realized our issue was thread pool exhaustion from bulk inserts blocking read queries.
4. **Consult an Expert:** I found the original architect who implemented it 3 years ago and asked a highly specific question: *"Our search thread pool is maxing out during batch ingests. Why wasn't a dedicated ingest node configured?"* rather than saying *"ES is broken, help."*

**Result:**
Within 4 hours of touching the technology for the first time, I implemented a change to the client application to chunk the bulk requests and added a dedicated coordinating node to the cluster. The latency dropped back to normal.

---

## 11. A tech decision you regret — and why

> **Also asked as:** "A tech decision you regret — and why"

Interviewers are looking for humility, self-awareness, and a bias against over-engineering (Resume Driven Development).

**Situation:**
In an early startup role, we had a monolith that we were breaking into three microservices. I decided we needed a Service Mesh (Istio) to manage the network traffic between them securely.

**Action:**
I spent three weeks configuring Istio, setting up mutual TLS (mTLS), configuring complex Envoy sidecar routing rules, and writing custom Gateways. 

**The Regret:**
It was a massive mistake. For a team of four engineers and three services, the operational overhead of Istio completely crippled our velocity. 
- When a pod couldn't talk to another pod, debugging became a nightmare because we had to parse Envoy proxy logs instead of just standard application logs.
- The control plane consumed more cluster resources than our actual applications.
- I had introduced extreme complexity to solve a problem we didn't actually have yet (we didn't need granular L7 routing or strict zero-trust yet; a simple Kubernetes NetworkPolicy and standard Service DNS would have sufficed).

**Result & Lesson Learned:**
After two months of fighting the tool, I ripped it out and replaced it with standard Kubernetes Services. The lesson I learned is the principle of **Minimum Viable Architecture**. You earn complexity; you don't start with it. Default to the simplest boring technology until the business requirements force you to adopt something more complex.

---

## 12. Convincing leadership to migrate infra

> **Also asked as:** "Convincing leadership to migrate infra"

Technical arguments rarely work on business leaders. This question tests if you can translate engineering problems into business ROI (Return on Investment).

**Situation:**
At my last company, we were running on bare-metal VMs hosted in a traditional colo data center. Deployments took days, and scaling for peak traffic required ordering physical hardware weeks in advance. The engineering team wanted to move to AWS (EKS), but leadership denied the request because "the cloud is too expensive."

**Action:**
I didn't argue about Kubernetes, Docker, or "modern tech." I built a business case focused on three things: **Time to Market**, **Opportunity Cost**, and **Risk**.
1. **Quantified the Friction:** I pulled Jira data showing that our developers spent 30% of their sprint waiting on the Ops team to provision VMs or fix environment drift. I translated that into engineering hours wasted per month (which equals dollars).
2. **Highlight the Revenue Blockers:** I documented two specific instances where marketing campaigns failed because the site crashed under load and we couldn't scale. I calculated the lost revenue from those outages.
3. **The Prototype:** I took a low-risk, internal backend service, migrated it to AWS in my spare time, and proved we could deploy it 10x faster and scale it automatically during load tests. I created a cost projection showing that while cloud hosting costs *are* higher than bare metal, the total cost of ownership (factoring in the 30% regained developer time and auto-scaling down at night) was actually lower.

**Result:**
By framing the migration as a way to increase product velocity and prevent lost sales, leadership approved a phased migration. We moved 100% to AWS over the next 8 months.

---

## 13. Handling on-call burnout

> **Also asked as:** "Handling on-call burnout"

Burnout isn't solved by "taking a deep breath" — it's solved by systemic engineering changes.

**Situation:**
When I joined my previous team, the on-call rotation was brutal. Engineers were getting paged 15-20 times a week, mostly at night. Morale was destroyed, and people were afraid of their shifts. It was pure alert fatigue.

**Action:**
I initiated an "Alert Bankruptcy" and systemic overhaul project:
1. **Kill the Noise:** I audited PagerDuty. 70% of our pages were CPU spikes that auto-resolved in 3 minutes. I changed those thresholds: if the system auto-recovers, it doesn't page a human; it generates a Slack message or a Jira ticket for next-day review. We only page for user-facing degradation (e.g., Error Rate > 2%).
2. **Enforce Runbooks:** I pushed a rule: Every alert *must* have a runbook linked in the notification. If an alert fires and there is no documented troubleshooting step, we delete the alert. Nobody should be guessing at 3 AM.
3. **Rotate fairly and Compensate Time:** I worked with management to implement "comp time." If you got paged at 2 AM, you are explicitly told to log on late the next morning. Hero culture is toxic; operational work requires rest.
4. **Fix the Root Cause, Don't Just Ack:** I started a weekly "Page Review" meeting. We looked at the top 3 noisiest alerts and created engineering sprint cards to permanently fix the underlying flakiness, rather than just clicking "Acknowledge" forever.

**Result:**
In two months, we reduced after-hours pages from 20/week to about 2/week. Engineers stopped fearing the rotation, and time spent firefighting dropped, giving us more time to write actual code.

---

## 6. SRE Behavioral: Arguing for uptime over feature speed

> **Also asked as:** "SRE Behavioral: Arguing for uptime over feature speed"

This question tests your ability to push back constructively against Product/Development teams when system reliability is threatened.

**Situation:**
At my last company, the product team was pushing to release a major new payment integration before Black Friday. However, our error budgets were already depleted from two previous minor outages that week, and the new checkout service lacked proper load testing and circuit breakers.

**Task:**
I had to convince the engineering director and product manager to delay the feature release or significantly alter the rollout plan, without being perceived as a "blocker."

**Action:**
I didn't argue using opinions; I used data. I pulled the Grafana dashboards showing our current 99.5% uptime (below our 99.9% SLO) and the latency spikes we were seeing.
I explained the specific risks: "If the new third-party payment API times out, we don't have circuit breakers in place. It will consume all our threading resources, and the entire checkout page will crash for *all* customers, not just those using the new payment method."
I proposed a compromise: Instead of delaying the feature entirely, we launch it wrapped in a feature flag, initially rolled out to only 2% of users for a 48-hour canary period, while the development team urgently added the missing circuit breaker pattern.

**Result:**
The product manager agreed to the canary approach. During the 2% rollout, the third-party API actually did experience latency spikes. The impact was contained to a tiny fraction of users instead of causing a total outage. The developers added the circuit breaker, and we safely rolled out to 100% two days later. Uptime was protected, and the feature still launched.

---

## 7. How to run a blameless post-mortem

> **Also asked as:** "How to run a blameless post-mortem"

Interviewers want to know if you understand that human error is a symptom of a system failure, not the root cause.

**Situation:**
A junior engineer ran a Terraform apply that accidentally dropped a production database index, causing 30 minutes of severe degraded performance for the main application.

**Task:**
I was tasked with leading the post-incident review (post-mortem). The goal was to find out *why* this happened and prevent it, without making the junior engineer feel targeted or afraid to deploy in the future.

**Action:**
I scheduled the meeting with the involved engineers, clearly stating in the invite that the goal was strictly systemic improvement, not assigning blame.
During the meeting, I focused the language entirely on the system and the timeline instead of the person. Instead of asking, "Why did you run that command?", I asked, "What information did the Terraform plan output show, and did it clearly indicate a destructive change?"
We used the "5 Whys" methodology:
1. Why did the database slow down? The index was deleted.
2. Why was the index deleted? Terraform destroyed it during an apply.
3. Why did Terraform destroy it? A recent refactor changed the resource identifier name.
4. Why wasn't this caught? The `terraform plan` output was a 5,000-line textual diff, and the destruction was buried on line 4,200.
5. Why did we apply a 5,000-line diff directly to prod? We lacked a staging environment that mirrored production closely enough to run this safely first.

**Result:**
The takeaway was not "Engineer needs to read the plan more closely." The takeaway was an infrastructure failure. We implemented a tool (Infracost/Terraform Cloud) to parse `terraform plan` outputs and automatically inject a summary (e.g., "🔴 1 Resource to Destroy") securely into the GitHub PR comments, requiring two senior approvals if the destroy count is > 0. The engineer felt supported, and the system became structurally safer.

---

## 8. Mentorship: How do you upskill developers on DevOps?

> **Also asked as:** "Mentorship: How do you upskill developers on DevOps?"

DevOps is a culture, not a team. If the DevOps engineer is doing all the deployments, it's an anti-pattern.

**Situation:**
In a previous role, my team of three DevOps engineers was a bottleneck. We were receiving 20+ Jira tickets a week from developers asking us to update environment variables, restart pods, or tweak Dockerfiles.

**Task:**
I needed to shift operations left, empowering the developers to handle their own day-to-day deployment and configuration tasks safely.

**Action:**
I couldn't just give them production access, so I focused on tooling and education.
1. **Tooling (Paved Paths):** We moved all Kubernetes manifests into a GitOps model (ArgoCD) and put the repository in front of the developers. If they needed to change an environment variable, they didn't need `kubectl` access; they just opened a Pull Request against the manifest repo.
2. **Standardization:** I created a "Golden Path" Dockerfile template and CI/CD pipeline template. Developers no longer had to write them from scratch; they just included the shared Jenkins library.
3. **Education:** I started a bi-weekly "DevOps for Devs" 30-minute lunch-and-learn. I taught them how to read ArgoCD dashboards, how to write efficient multi-stage Docker builds, and how to query their own logs in Loki instead of asking us to grep servers.

**Result:**
Over three months, "ops requests" from the dev team dropped by 70%. Developers felt empowered to manage their own service lifecycles from code commit to production monitoring, allowing the DevOps team to focus on migrating our database backend to a new region instead of fighting daily operational fires.
