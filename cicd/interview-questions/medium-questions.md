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

## 2. What is the difference between Declarative and Scripted Jenkins pipelines?

Both are written in a Jenkinsfile, both live in source control alongside the code — the difference is structure, safety, and readability.

**Declarative pipeline:**

```groovy
pipeline {
  agent any
  environment {
    IMAGE_TAG = "${GIT_COMMIT}"
  }
  stages {
    stage('Test') {
      steps {
        sh 'pytest tests/'
      }
    }
    stage('Build') {
      steps {
        sh 'docker build -t my-app:$IMAGE_TAG .'
      }
    }
  }
  post {
    failure {
      slackSend channel: '#alerts', message: "Build failed: ${env.JOB_NAME}"
    }
    always {
      cleanWs()
    }
  }
}
```

- Fixed, predefined structure (`pipeline {}`, `stages {}`, `steps {}`)
- Validated before execution — syntax errors caught immediately
- Built-in `post {}` blocks for success/failure/always
- Limited to what the DSL supports — can't run arbitrary Groovy outside `script {}` blocks

**Scripted pipeline:**

```groovy
node {
  try {
    stage('Test') {
      sh 'pytest tests/'
    }
    stage('Build') {
      sh "docker build -t my-app:${GIT_COMMIT} ."
    }
  } catch (e) {
    slackSend channel: '#alerts', message: "Build failed: ${env.JOB_NAME}"
    throw e
  } finally {
    cleanWs()
  }
}
```

- Full Groovy — any valid Groovy code runs here
- More flexible: dynamic stage generation, complex loops, conditionals
- No structure validation — errors discovered at runtime
- Harder to read, harder to review in PRs

**When I use each:**

Declarative for everything new. It's readable, reviewable, and the `post {}` block means junior engineers don't accidentally skip cleanup on failure.

Scripted only when I need dynamic stage generation — for example, looping over a list of services and creating parallel stages for each. Declarative can't do this cleanly without hacky workarounds.

**Real scenario:** Inherited a 500-line Scripted pipeline. Every change required deep Groovy knowledge. A developer once broke the finally block — docker-compose containers leaked on the agent for 3 weeks causing port conflicts. Rewrote in Declarative: 90 lines, readable by the whole team, `post { always { cleanWs() } }` handles cleanup unconditionally.

---

## 3. How do you define and trigger Jenkins pipelines?

There are three parts: where the pipeline is defined, how it's stored, and what triggers it.

**Defining the pipeline — always in a Jenkinsfile.**

The Jenkinsfile lives in the root of the application repo alongside the code. This means:
- Pipeline changes go through PR review, same as code changes
- If you `git revert` a bad deployment, you revert the pipeline too
- Every version of the code has its corresponding pipeline version

In Jenkins, the job is configured as a "Pipeline" or "Multibranch Pipeline" that points to the repo. Jenkins reads the Jenkinsfile from the branch being built.

**Triggering pipelines:**

**1. Webhook (what we use — instant trigger):**

GitHub sends a POST request to Jenkins every time a push or PR event happens. Jenkins receives it and triggers the relevant pipeline immediately.

Setup: In GitHub repo → Settings → Webhooks → Add webhook → `http://jenkins.company.internal/github-webhook/`

In the Jenkinsfile or job config:
```groovy
triggers {
  githubPush()    // Triggers on any push to the repo
}
```

**2. SCM Polling (fallback when webhooks can't reach Jenkins):**

Jenkins checks GitHub every N minutes for new commits.

```groovy
triggers {
  pollSCM('H/5 * * * *')    // Check every 5 minutes
}
```

We used polling in environments where Jenkins was behind a VPN and GitHub couldn't reach it. Downside: up to 5-minute delay between push and build.

**3. Manual trigger:**

Someone clicks "Build Now" in the Jenkins UI or calls the API:
```bash
curl -X POST http://jenkins.company.internal/job/my-app/build \
  --user "$JENKINS_USER:$JENKINS_TOKEN"
```

We use manual triggers for production deployments after staging sign-off — a deliberate gate, not automatic.

**4. Scheduled (cron):**

```groovy
triggers {
  cron('0 2 * * 1-5')    // Run at 2 AM on weekdays — nightly full test suite
}
```

We run a nightly pipeline that runs integration tests against a live staging environment — too slow for every PR but valuable for catching regressions.

**Multibranch Pipeline — what we actually use:**

Instead of one job per branch, a Multibranch Pipeline automatically discovers branches and PRs, creates a job for each, and runs the Jenkinsfile from that branch. New branch → new job automatically. Branch deleted → job cleaned up automatically. This is how we handle `feature/*` branches without manually creating Jenkins jobs.

---

## 4. What are Jenkins Shared Libraries? How are they structured and used in Jenkinsfiles?

Shared Libraries solve the copy-paste problem. When you have 15 services each with their own Jenkinsfile, and you need to update the Docker build command across all of them, you either edit 15 files or you put that logic in one place — the Shared Library.

**What it is:** A Git repository of reusable Groovy functions that any Jenkinsfile can call.

**Folder structure (this is fixed — Jenkins expects exactly this layout):**

```
jenkins-shared-library/           ← Separate Git repo
├── vars/                         ← Global variables / pipeline steps (called directly from Jenkinsfile)
│   ├── buildAndPushImage.groovy  ← def call() makes it callable as buildAndPushImage(...)
│   ├── runTrivy.groovy
│   ├── updateManifest.groovy
│   └── notifySlack.groovy
├── src/                          ← Groovy classes (for complex reusable logic)
│   └── org/
│       └── company/
│           └── DockerUtils.groovy
└── resources/                    ← Static files (scripts, config templates)
    └── scripts/
        └── smoke-test.sh
```

**A `vars/` file — the most common pattern:**

```groovy
// vars/buildAndPushImage.groovy
def call(Map config) {
  def ecrRepo = config.ecrRepo
  def imageTag = config.imageTag ?: env.GIT_COMMIT

  sh """
    aws ecr get-login-password --region ap-south-1 | \
      docker login --username AWS --password-stdin ${ecrRepo.split('/')[0]}
    docker build -t ${ecrRepo}:${imageTag} .
    docker push ${ecrRepo}:${imageTag}
  """
  return "${ecrRepo}:${imageTag}"
}
```

**Using it in a Jenkinsfile:**

```groovy
@Library('jenkins-shared-library@main') _   // Load the library from its Git repo

pipeline {
  agent any
  stages {
    stage('Build & Push') {
      steps {
        script {
          buildAndPushImage(
            ecrRepo: '123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app',
            imageTag: env.GIT_COMMIT
          )
        }
      }
    }
    stage('Scan') {
      steps {
        script {
          runTrivy(image: "123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app:${env.GIT_COMMIT}")
        }
      }
    }
    stage('Notify') {
      steps {
        script {
          notifySlack(channel: '#deployments', status: 'success')
        }
      }
    }
  }
}
```

**Setting it up in Jenkins:** Global configuration → Manage Jenkins → Configure System → Global Pipeline Libraries → Add library with the Git repo URL and default branch.

**Real impact:** We had 18 Jenkinsfiles, each with a copy of the ECR login + docker build + push block. When AWS changed the ECR login command syntax, I had to update 18 files. After migrating to a Shared Library, that change is one file in one repo. The `@Library` tag in each Jenkinsfile always pulls from `main` — every pipeline automatically gets the update on next run.

**Key pitfall:** Shared Library changes take effect on the next build — they're not versioned per-service unless you pin to a specific tag (`@Library('my-lib@v1.2.0')`). We pin to a version tag for production pipelines to prevent a library change from silently affecting all 18 services at once.

---

## 5. What is a webhook, and how is it used in a CI/CD pipeline?

A webhook is an HTTP callback — instead of Jenkins polling GitHub every 5 minutes asking "did anything change?", GitHub pushes a notification to Jenkins the moment something happens. It's the difference between checking your email every 5 minutes and getting a notification when a new email arrives.

**How it works:**

1. You register a URL with GitHub (the Jenkins webhook endpoint)
2. When a push, PR open, or merge event happens, GitHub sends a POST request to that URL with a JSON payload describing the event
3. Jenkins receives the payload, identifies which repo/branch triggered it, and starts the relevant pipeline

**Setup:**

In GitHub: Repo → Settings → Webhooks → Add webhook
- Payload URL: `http://jenkins.company.internal/github-webhook/`
- Content type: `application/json`
- Events: "Just the push event" or "Let me select individual events" (PRs, pushes, etc.)

In Jenkins (Multibranch Pipeline or GitHub plugin), check "Build when a change is pushed to GitHub."

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

Jenkins parses this, matches `our-org/my-app` to a Jenkins job, and triggers the build for branch `feature/my-feature` at commit `abc123def456`.

**Webhook vs polling — the real difference:**

| | Webhook | Polling |
|---|---|---|
| Trigger speed | Instant | Up to 5 min delay |
| Jenkins load | Zero until event | Constant HTTP requests to GitHub |
| GitHub API rate limit | Not affected | Can exhaust rate limits at scale |
| Requires Jenkins reachable from internet | Yes | No |

**When we use polling instead of webhooks:**

Jenkins is inside a private VPC with no public ingress. GitHub.com can't reach it. We use polling (`pollSCM('H/2 * * * *')`) with a 2-minute interval. For enterprise GitHub (self-hosted in the same VPC), we always use webhooks — the network is private, but GitHub can reach Jenkins directly.

**Real scenario:** We once had all 18 pipelines fail to trigger for 3 hours. Root cause: the Jenkins webhook endpoint returned 500 because a Jenkins plugin update broke the GitHub plugin. GitHub retried the webhook 3 times, then stopped. We never got notified because the failure was silent from GitHub's side. Fix: configured GitHub webhook delivery alerts and added a polling fallback (`pollSCM('H/15 * * * *')`) for production pipelines as a safety net.
