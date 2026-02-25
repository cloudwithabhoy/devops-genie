# Jenkins — Medium Questions

---

## 1. What is the difference between Declarative and Scripted Jenkins pipelines?

> **Also asked as:** "Write a rough pipeline script for microservices architecture."

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

## 2. How do you define and trigger Jenkins pipelines?

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

## 3. What are Jenkins Shared Libraries? How are they structured and used in Jenkinsfiles?

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

## 4. How do you secure secrets in Jenkins pipelines?

The wrong answer: "I store them in environment variables in the Jenkinsfile." The right answer covers: where secrets live, how pipelines access them, and what never appears in logs.

**Option 1: Jenkins Credentials Store (built-in — minimum viable).**

Jenkins has a built-in encrypted credentials store. Credentials are stored encrypted on disk, referenced by ID in Jenkinsfiles, and masked in console output.

```groovy
pipeline {
  environment {
    // Username + password credential — injected as env vars
    DB_CREDS = credentials('prod-db-credentials')
    // Creates: DB_CREDS_USR (username) and DB_CREDS_PSW (password)
    // DB_CREDS_PSW is automatically masked in logs

    // Single string (API key, token)
    SONAR_TOKEN = credentials('sonarqube-token')
  }
  stages {
    stage('Deploy') {
      steps {
        withCredentials([string(credentialsId: 'ecr-password', variable: 'ECR_PASS')]) {
          sh 'echo $ECR_PASS | docker login --username AWS --password-stdin $ECR_REPO'
          // ECR_PASS is masked in the console output — appears as ****
        }
      }
    }
  }
}
```

**The built-in store has limitations:**
- Credentials are tied to one Jenkins instance — if Jenkins is lost, credentials are lost
- No audit log of who accessed which credential
- No automatic rotation
- Secret values visible to Jenkins admins via the API

**Option 2: AWS Secrets Manager + Jenkins IAM role (what we use).**

Jenkins runs on EC2 with an IAM instance profile (or on EKS with IRSA). The IAM role has permission to call Secrets Manager. Secrets are fetched at pipeline runtime — not stored in Jenkins at all.

```groovy
stage('Deploy') {
  steps {
    script {
      // Fetch secret at runtime from AWS Secrets Manager
      def dbSecret = sh(
        script: '''aws secretsmanager get-secret-value \
          --secret-id prod/order-service/db \
          --query SecretString \
          --output text''',
        returnStdout: true
      ).trim()

      def creds = readJSON(text: dbSecret)

      // Use the secret — never echo it
      sh """
        export DB_HOST=${creds.host}
        export DB_PASS=${creds.password}
        python deploy.py
      """
    }
  }
}
```

**Option 3: HashiCorp Vault (enterprise-grade).**

Vault provides dynamic secrets — credentials generated on-demand with short TTLs, automatically revoked after the pipeline completes.

```groovy
withVault(configuration: [vaultUrl: 'https://vault.company.internal'],
          vaultSecrets: [[path: 'secret/prod/db', secretValues: [
            [envVar: 'DB_PASSWORD', vaultKey: 'password']
          ]]]) {
  sh 'python deploy.py'    // DB_PASSWORD available, masked in logs
}
```

Vault advantages:
- Dynamic credentials: every pipeline run gets a unique short-lived DB credential, automatically revoked after 1 hour
- Full audit log: who accessed which secret, when, from which Jenkins job
- Lease management: if a pipeline fails halfway, Vault revokes the credential automatically

**What we never do:**

```groovy
// NEVER — hardcoded secret in Jenkinsfile (in Git = exposed forever)
environment {
  DB_PASSWORD = "super-secret-prod-password"
}

// NEVER — secret echoed in build steps
sh "echo DB password is $DB_PASSWORD"   // Appears in console log

// NEVER — secret in docker build ARG
sh "docker build --build-arg DB_PASS=$DB_PASSWORD ."
// ARG values are stored in docker history — visible via docker image inspect
```

**Real incident:** A developer ran `echo $AWS_SECRET_ACCESS_KEY` inside a Jenkins pipeline step to debug a credential issue.

> **Also asked as:** "How do you store secrets in Jenkins?" — covered above (Jenkins Credentials Store, `credentials()` binding, `withCredentials`, masking in logs).

---

## 5. What is the concept of Pipeline as Code in Jenkins?

> Also asked as: "What is Pipeline as Code in Jenkins?"

Pipeline as Code means the CI/CD pipeline definition is stored in a file (`Jenkinsfile`) inside the application's Git repository — not configured through the Jenkins UI. The pipeline travels with the code, is versioned, reviewed, and audited exactly like application code.

**The problem Pipeline as Code solves:**

Before Pipeline as Code, Jenkins jobs were configured through the GUI. This created two problems:
- **No version history** — if someone changed a build step, there was no record of what changed or who changed it
- **No reproducibility** — if Jenkins went down or you needed to recreate a job, the configuration was gone or had to be manually re-entered

With a `Jenkinsfile` in the repository:
- Every pipeline change is a Git commit — you see who changed what and can revert it
- The pipeline is recreated automatically when Jenkins detects the `Jenkinsfile`
- Code review applies to pipeline changes the same as application changes
- Different branches can have different pipeline configurations (feature branch builds test only; main builds, tests, and deploys to prod)

**What a Jenkinsfile looks like:**

```groovy
// Jenkinsfile stored at root of the repository
pipeline {
  agent { label 'docker-agent' }

  environment {
    IMAGE_NAME = "myapp"
    ECR_REPO   = "123456789012.dkr.ecr.ap-south-1.amazonaws.com/myapp"
  }

  stages {
    stage('Build') {
      steps {
        sh 'docker build -t $IMAGE_NAME:$GIT_COMMIT .'
      }
    }
    stage('Test') {
      steps {
        sh 'docker run --rm $IMAGE_NAME:$GIT_COMMIT pytest tests/'
      }
    }
    stage('Push') {
      steps {
        sh '''
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO
          docker tag $IMAGE_NAME:$GIT_COMMIT $ECR_REPO:$GIT_COMMIT
          docker push $ECR_REPO:$GIT_COMMIT
        '''
      }
    }
    stage('Deploy') {
      when { branch 'main' }
      steps {
        sh "helm upgrade --install myapp ./helm --set image.tag=$GIT_COMMIT"
      }
    }
  }
}
```

**How Jenkins picks it up:**

A **Multibranch Pipeline** job scans the Git repository for `Jenkinsfile` in every branch. When a new branch is pushed, Jenkins automatically creates a new pipeline for it. When the branch is deleted, Jenkins removes the pipeline. No manual job creation needed.

---

## 6. How do you implement a manual approval gate in a Jenkins pipeline?

> Also asked as: "How do you manually trigger a Jenkins job when approval is required?"

When a deployment needs a human to approve before proceeding — typically before pushing to production — you use the `input` step.

```groovy
pipeline {
  agent any

  stages {
    stage('Build & Test') {
      steps {
        sh 'mvn clean package'
        sh 'mvn test'
      }
    }

    stage('Deploy to Staging') {
      steps {
        sh 'helm upgrade --install myapp ./helm -f values/staging.yaml'
      }
    }

    stage('Approval: Deploy to Production') {
      steps {
        // Pipeline pauses here and waits for a human to click Proceed or Abort
        input message: 'Deploy to production?',
              ok: 'Proceed',
              submitter: 'devops-leads,release-managers'
              // Only users in these groups can approve
      }
    }

    stage('Deploy to Production') {
      steps {
        sh 'helm upgrade --install myapp ./helm -f values/prod.yaml --set image.tag=$GIT_COMMIT'
      }
    }
  }
}
```

**What happens when the pipeline reaches `input`:**
- The build pauses and shows a "Proceed / Abort" button in the Jenkins UI
- Jenkins sends a notification (email, Slack — if configured)
- The pipeline holds for the configured timeout (default: forever — set a timeout)
- If no one approves within the timeout, the pipeline can abort automatically

**Adding a timeout to the approval step:**

```groovy
stage('Approval: Deploy to Production') {
  steps {
    timeout(time: 2, unit: 'HOURS') {
      input message: 'Deploy to production?',
            ok: 'Deploy Now',
            submitter: 'devops-leads'
    }
  }
}
// If no approval in 2 hours → pipeline aborts automatically
```

**Capturing the approver's identity:**

```groovy
stage('Approval') {
  steps {
    script {
      def approver = input message: 'Approve production deployment?',
                           submitter: 'devops-leads',
                           submitterParameter: 'APPROVED_BY'
      echo "Deployment approved by: ${approver}"
      // Log to audit trail
    }
  }
}
```

---

## 7. What are agents in Jenkins and runners in GitHub Actions / GitLab CI?

> Also asked as: "What are agents and runners in CI/CD?"

All three CI/CD tools have the same concept — a machine that executes pipeline steps — but they use different terminology.

**Jenkins Agents:**

An agent is a machine (physical, VM, Docker container, or Kubernetes pod) connected to the Jenkins Controller that actually runs build steps. The Controller only orchestrates; agents do the work.

```groovy
// Declarative pipeline — agent label
pipeline {
  agent { label 'docker-agent' }   // Run on any agent with this label
  stages { ... }
}

// Kubernetes pod agent (dynamic — spins up a pod per build)
pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: docker
            image: docker:24-dind
            securityContext:
              privileged: true
      '''
    }
  }
}
```

Agents can be **static** (always-on VMs registered to Jenkins) or **dynamic** (Kubernetes pods created per build, destroyed after).

**GitHub Actions Runners:**

A runner is the equivalent of a Jenkins agent — the machine that executes a workflow job.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest      # GitHub-hosted runner (managed by GitHub)
    # OR
    runs-on: self-hosted        # Your own machine registered as a runner
```

GitHub-hosted runners are ephemeral VMs GitHub spins up and tears down per job. Self-hosted runners are machines you register with GitHub — useful when you need access to internal networks or specific hardware.

**GitLab CI Runners:**

Same concept, called a runner. Registered to a GitLab instance with a registration token.

```yaml
build:
  tags:
    - docker    # Run on a runner with the "docker" tag
    - linux
  script:
    - docker build .
```

**Comparison:**

| | Jenkins | GitHub Actions | GitLab CI |
|---|---|---|---|
| Term | Agent | Runner | Runner |
| Controller | Jenkins Controller | GitHub.com | GitLab instance |
| Managed option | No (you manage all) | GitHub-hosted runners | GitLab.com shared runners |
| Self-hosted | Yes (static + K8s dynamic) | Yes (self-hosted runners) | Yes (registered runners) |
| Config location | Agent labels in Jenkinsfile | `runs-on` in workflow YAML | `tags` in `.gitlab-ci.yml` |

---

## 8. How do you create multi-stage jobs in Jenkins?

> Also asked as: "How do you create multi-stage pipelines in Jenkins?"

Multi-stage pipelines break a build into sequential or parallel stages — Build → Test → Security Scan → Push → Deploy. Each stage has a clear name, can fail independently, and the pipeline visualization shows exactly where a failure occurred.

**Basic multi-stage pipeline:**

```groovy
pipeline {
  agent { label 'docker-agent' }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'docker build -t myapp:$GIT_COMMIT .'
      }
    }

    stage('Unit Tests') {
      steps {
        sh 'docker run --rm myapp:$GIT_COMMIT pytest tests/unit/'
      }
      post {
        always {
          junit 'test-results/*.xml'   // Publish test results regardless of pass/fail
        }
      }
    }

    stage('Security Scan') {
      steps {
        sh 'trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:$GIT_COMMIT'
      }
    }

    stage('Push to ECR') {
      steps {
        sh '''
          aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO
          docker push $ECR_REPO/myapp:$GIT_COMMIT
        '''
      }
    }

    stage('Deploy to Staging') {
      steps {
        sh 'helm upgrade --install myapp ./helm -f values/staging.yaml --set image.tag=$GIT_COMMIT'
      }
    }

    stage('Deploy to Production') {
      when { branch 'main' }   // Only run on main branch
      steps {
        input message: 'Approve production deployment?'
        sh 'helm upgrade --install myapp ./helm -f values/prod.yaml --set image.tag=$GIT_COMMIT'
      }
    }
  }

  post {
    failure {
      slackSend channel: '#builds', message: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
    success {
      slackSend channel: '#builds', message: "Build PASSED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
    }
  }
}
```

**Parallel stages — run independent stages simultaneously:**

```groovy
stage('Test + Scan in Parallel') {
  parallel {
    stage('Unit Tests') {
      steps { sh 'pytest tests/unit/' }
    }
    stage('Integration Tests') {
      steps { sh 'pytest tests/integration/' }
    }
    stage('Security Scan') {
      steps { sh 'trivy image myapp:$GIT_COMMIT' }
    }
  }
  // All three run simultaneously — total time = longest stage, not sum of all
}
```

Running Unit Tests (2 min), Integration Tests (3 min), and Scan (1 min) in parallel takes 3 minutes instead of 6. This is the single biggest pipeline optimization available. The console output was visible to all engineers with Jenkins read access — 23 people. The access key was rotated within 10 minutes of discovery, but CloudTrail showed no unauthorized usage. After this: mandatory pipeline code review that checks for `echo` + credential variable patterns, and all credentials moved from Jenkins store to Secrets Manager with the IAM instance profile approach. No human can now see the actual secret values — not even Jenkins admins.

---

## 9. How do you implement SonarQube in a Jenkins pipeline?

SonarQube is a static code analysis tool that scans your code for bugs, vulnerabilities, code smells, and test coverage. Integrating it into Jenkins ensures every build is analysed before deployment.

**Prerequisites:**
1. SonarQube server running (self-hosted or SonarCloud)
2. SonarQube Scanner plugin installed in Jenkins
3. SonarQube server configured in Jenkins (Manage Jenkins → Configure System → SonarQube servers)
4. A token generated in SonarQube (My Account → Security → Generate Token) stored as a Jenkins credential

**Step 1: Configure SonarQube in Jenkins.**

```
Manage Jenkins → Configure System → SonarQube servers
  Name: SonarQube
  Server URL: http://sonarqube.internal:9000
  Server authentication token: <select the credential storing your SonarQube token>
```

**Step 2: Add the SonarQube scan stage to your Jenkinsfile.**

```groovy
pipeline {
  agent { label 'docker-agent' }

  environment {
    SONAR_PROJECT_KEY = "my-app"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh 'mvn clean package -DskipTests'
      }
    }

    stage('Unit Tests') {
      steps {
        sh 'mvn test'
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        // withSonarQubeEnv injects SONAR_HOST_URL and SONAR_AUTH_TOKEN automatically
        withSonarQubeEnv('SonarQube') {
          sh '''
            mvn sonar:sonar \
              -Dsonar.projectKey=$SONAR_PROJECT_KEY \
              -Dsonar.projectName="My App" \
              -Dsonar.java.binaries=target/classes \
              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        // Wait for SonarQube to finish analysis and check the result
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
          // If Quality Gate fails → pipeline aborts here, does not deploy
        }
      }
    }

    stage('Deploy') {
      when { branch 'main' }
      steps {
        sh 'helm upgrade --install myapp ./helm --set image.tag=$GIT_COMMIT'
      }
    }
  }
}
```

**For non-Maven projects (Node.js, Python) — use the sonar-scanner CLI:**

```groovy
stage('SonarQube Analysis') {
  steps {
    withSonarQubeEnv('SonarQube') {
      sh '''
        sonar-scanner \
          -Dsonar.projectKey=my-node-app \
          -Dsonar.sources=src \
          -Dsonar.exclusions=node_modules/**,coverage/** \
          -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
      '''
    }
  }
}
```

**The Quality Gate — what it checks:**

A Quality Gate is a set of conditions defined in SonarQube that code must pass:
- Coverage on new code > 80%
- No new critical/blocker bugs
- No new critical vulnerabilities
- Duplicated lines on new code < 3%

If any condition fails, `waitForQualityGate abortPipeline: true` fails the pipeline — the build never reaches the deploy stage. This enforces code quality as a mandatory step, not an optional report.

**sonar-project.properties (alternative to passing all flags in the pipeline):**

```properties
# sonar-project.properties — place at project root
sonar.projectKey=my-app
sonar.projectName=My App
sonar.sources=src
sonar.tests=tests
sonar.python.coverage.reportPaths=coverage.xml
sonar.exclusions=**/__pycache__/**,**/*.pyc
```

With this file present, the scanner picks up all settings automatically — the Jenkins stage only needs `sonar-scanner` with no extra flags.
