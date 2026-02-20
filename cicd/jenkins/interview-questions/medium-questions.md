# Jenkins — Medium Questions

---

## 1. What is the difference between Declarative and Scripted Jenkins pipelines?

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

**Real incident:** A developer ran `echo $AWS_SECRET_ACCESS_KEY` inside a Jenkins pipeline step to debug a credential issue. The console output was visible to all engineers with Jenkins read access — 23 people. The access key was rotated within 10 minutes of discovery, but CloudTrail showed no unauthorized usage. After this: mandatory pipeline code review that checks for `echo` + credential variable patterns, and all credentials moved from Jenkins store to Secrets Manager with the IAM instance profile approach. No human can now see the actual secret values — not even Jenkins admins.
