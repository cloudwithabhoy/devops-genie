# Jenkins — Real Production Issues

---

## 1. Jenkins pipeline runs but the build does not trigger — what could be the reasons?

"Pipeline runs" means Jenkins executed something, but the actual build step — compiling, testing, building the image — never started. This is different from a build that fails. Here's how I debug it systematically.

**1. Webhook delivered but Jenkins didn't match the branch.**

The most common cause. The webhook fires, Jenkins receives it, but the Multibranch Pipeline doesn't see a matching branch to build.

Check: Jenkins → Pipeline job → Scan Multibranch Pipeline Log. It shows which branches were discovered and why some were skipped.

Common causes:
- Branch filter regex doesn't match your branch name (`feature/my-feature` vs expected `feature-my-feature`)
- The Jenkinsfile doesn't exist on that branch (Jenkins scans for it, can't find it, skips the branch)
- Branch was just created and the scan hasn't run yet — trigger a manual scan or reduce the scan interval

**2. Webhook not reaching Jenkins.**

GitHub shows the webhook as "delivered" with a green tick, but Jenkins never received it.

Check GitHub: Repo → Settings → Webhooks → Recent Deliveries. Look at the response code. If it's `302`, `401`, `500`, Jenkins received it but rejected or errored. If there's no delivery at all, the URL is wrong or Jenkins is unreachable.

We had this after rotating a Jenkins reverse proxy SSL cert. GitHub started getting SSL errors on the webhook endpoint. The webhook showed "failed" in GitHub delivery logs. Fix: updated the cert, re-tested the webhook delivery manually.

**3. Jenkins CSRF protection blocking the webhook.**

Jenkins has CSRF protection that can reject POST requests from GitHub. The fix is to configure the GitHub plugin properly or add an exception.

Check: Jenkins → Manage Jenkins → Security → CSRF Protection. If "Crumb Issuer" is on, webhooks need to include a crumb token. The GitHub plugin handles this automatically if configured correctly — but a misconfigured plugin causes silently dropped webhooks.

**4. Pipeline was triggered but skipped due to `when {}` condition.**

```groovy
stage('Deploy to Prod') {
  when {
    branch 'main'
  }
  steps { ... }
}
```

The pipeline ran, but the stage was skipped because the `when` condition evaluated to false. The build shows as "Success" but nothing deployed. This is correct behavior — but it looks like the build "didn't trigger" from the outside.

Check the build log — skipped stages show as light grey in Blue Ocean or have "Stage skipped due to when condition" in the console output.

**5. Build throttling or executor starvation.**

Jenkins has a limited number of executors. If all executors are busy, new builds queue but don't start. From the outside, it looks like the build isn't triggering.

Check: Jenkins home page → Build Queue. If builds are stacking up, you need more executors or more agents.

We had a night where a scheduled nightly job consumed all 4 agents, and 12 PR builds queued for 2 hours without starting. Fix: added dedicated agents for PR builds and reserved the nightly job to a specific agent label.

**6. `skipDefaultCheckout` or pipeline agent misconfiguration.**

```groovy
pipeline {
  agent none    // No global agent
  stages {
    stage('Build') {
      // No agent defined here either — this stage has nowhere to run
      steps {
        sh 'docker build .'
      }
    }
  }
}
```

If `agent none` is set at the pipeline level and a stage doesn't have its own `agent` block, that stage silently does nothing.

**My debug checklist:**
1. GitHub webhook delivery log — was it delivered? What was the response?
2. Jenkins Multibranch scan log — was the branch discovered?
3. Build queue — is the build queued and waiting for an executor?
4. Build console log — did it run and skip via `when {}` condition?
5. Agent availability — are there idle agents with the right labels?

---

## 2. A Jenkins job works in Dev but fails in Production — how do you troubleshoot?

> **Also asked as:** "A job works in Dev but fails in Production — how do you troubleshoot?"
> **Also asked as:** "How do you find errors in the pipelines?"

The root cause of "works in dev, fails in prod" is almost always one of: environment variables, credentials, network access, resource limits, or a dependency that exists on the dev agent but not the prod agent.

**Step 1: Compare the environments side by side.**

```bash
# On the dev Jenkins agent:
printenv | sort > /tmp/dev-env.txt

# On the prod Jenkins agent:
printenv | sort > /tmp/prod-env.txt

diff /tmp/dev-env.txt /tmp/prod-env.txt
# Shows every env variable that differs between environments
```

Common differences: `JAVA_HOME`, `PATH`, `AWS_REGION`, `KUBECONFIG`, tool versions (`docker --version`, `kubectl version`, `helm version`).

**Step 2: Check which agent/node the prod job ran on.**

```
Jenkins UI → Failed Build → Console Output → first line:
"Running on prod-agent-03 in /var/jenkins/workspace/..."
```

SSH to `prod-agent-03` and check:
```bash
# Does the tool exist?
which docker && docker --version
which kubectl && kubectl version --client
which helm && helm version

# Does the user running Jenkins have permission to use it?
sudo -u jenkins docker info
```

**Step 3: Compare credentials and secrets.**

If the failure message includes "Access Denied", "Authentication failed", or "403":

```groovy
// In the failed pipeline, check which credential ID is being used
withCredentials([string(credentialsId: 'aws-prod-key', variable: 'AWS_KEY')]) {
  // Is 'aws-prod-key' actually configured in Production Jenkins?
  // Jenkins Manage Credentials → search for the ID
}
```

Go to Manage Jenkins → Credentials. Verify the credential ID exists in the production Jenkins. In dev, the credential might be a test key with wider permissions; in prod, it might be scoped down and missing a specific permission.

**Step 4: Check network access from the prod agent.**

```bash
# From the prod Jenkins agent — can it reach the target?
curl -v https://ecr.ap-south-1.amazonaws.com
curl -v https://kubernetes-api-endpoint:6443
nc -zv internal-nexus.company.com 8081
```

Prod agents are often in a more restricted subnet with tighter security groups. If the job calls an internal service that dev can reach but prod cannot (missing security group rule), it fails silently as a timeout.

**Step 5: Check resource limits on the prod agent.**

```bash
# Is the prod agent running out of disk space?
df -h /var/jenkins

# Is there a Docker image cache filling the disk?
docker system df
docker system prune -f   # Clean up if disk is full

# Memory?
free -h
```

**Common specific causes and their fixes:**

| Symptom | Likely cause | Fix |
|---|---|---|
| "command not found" | Tool not installed on prod agent | Install tool or use Docker agent |
| "Permission denied" | Wrong user or missing sudo | Fix agent user permissions |
| "No such credential" | Credential ID missing in prod Jenkins | Add credential to prod Jenkins |
| Connection timeout | Security group blocks prod agent → target | Add firewall rule |
| "Docker login failed" | ECR token expired or wrong region | Fix aws ecr get-login-password region |

---

## 3. A stage is failing in a Jenkins pipeline due to a credentials issue — how do you fix it?

> Also asked as: "A Jenkins stage fails due to credential issues — how do you fix it?"

**Step 1: Read the error message carefully.**

The console output tells you exactly what failed:

```
hudson.AbortException: No such credential found: 'ecr-prod-credentials'
# → Credential ID doesn't exist in Jenkins

Error: credentialsId=ecr-prod-credentials not found
# → Same — credential not configured

ERROR: Permission denied — IAM role does not have ecr:GetAuthorizationToken
# → Credential exists but the IAM role/key lacks the required permission

The supplied credentials are invalid
# → Credential exists, has permission, but the value is wrong (expired key, wrong password)
```

**Step 2: Verify the credential exists in Jenkins.**

```
Jenkins UI → Manage Jenkins → Credentials → (Global) or (folder-level)
Search for the credential ID used in the Jenkinsfile
```

If it doesn't exist:
1. Go to Credentials → Add Credentials
2. Choose the correct type (Username/Password, Secret Text, AWS credentials, SSH key)
3. Set the ID exactly as referenced in the Jenkinsfile
4. Save

**Step 3: Verify the Jenkinsfile references the correct ID.**

```groovy
// The ID in withCredentials must exactly match what's in Jenkins Credentials store
withCredentials([
  usernamePassword(
    credentialsId: 'ecr-prod-credentials',   // ← This must exist in Jenkins
    usernameVariable: 'ECR_USER',
    passwordVariable: 'ECR_PASS'
  )
]) {
  sh 'docker login -u $ECR_USER -p $ECR_PASS $ECR_REPO'
}
```

**Step 4: Check credential scope.**

Credentials in Jenkins have scope:
- **Global** — available to all jobs
- **System** — available only to Jenkins internal use
- **Folder-level** — available only to jobs within that folder

If the credential is folder-scoped and your job is outside that folder, it won't be found even if it exists.

```
Jenkins → Credentials → Check scope column
If "Folder: my-folder" but job is at root → move credential to Global scope
```

**Step 5: Verify the credential value is still valid.**

For AWS credentials: the access key might have been rotated. For tokens: they might have expired.

```bash
# Test AWS credentials directly from the Jenkins agent
# (as the jenkins user, using the stored credential value)
aws sts get-caller-identity --profile jenkins-test
# If this fails → rotate the key in AWS IAM and update in Jenkins Credentials
```

---

## 4. Suppose the person who add his credentials in jenkins left the organization soon, then what alternative way we have ?

> **Also asked as:** "Suppose the person who add his credentials in jenkins left the organization soon, then what alternative way we have ?"

This is a classic "Bus Factor" and secure offboarding scenario. If a developer uses their *personal* AWS Access Keys, GitHub PAT, or DockerHub password inside Jenkins, the pipeline will immediately break the moment IT disables their corporate accounts upon departure.

**Immediate Fix (The Firefight):**
1. Identify exactly what pipeline broke and which credential ID it was referencing.
2. Determine what that credential was accessing (AWS, GitHub, SonarQube, etc.).
3. Generate a new, temporary (but highly scoped) credential using an active admin account.
4. Go to **Manage Jenkins → Credentials**, find the broken credential ID, and click **Update**. Replace the departed employee's secret with the new one. The pipeline will immediately start working again without requiring code changes to the `Jenkinsfile` (since the ID remained identical).

**The Architectural Fix (How to prevent this forever):**
Personal credentials should **never** be used in a CI/CD system. We must shift to service accounts and dynamic secrets.

1. **Use Machine/Service Accounts:** Instead of `john.doe@company.com` generating a GitHub token, we create a Service Account `svc-jenkins-bot@company.com`. This account is owned by the DevOps team, not an individual. If John leaves, the bot account remains active.
2. **Use GitHub Apps instead of PATs:** For GitHub specifically, we migrate Jenkins to authenticate via a GitHub App installation. This ties the access to the repository integration itself, completely removing the dependency on any human user's token.
3. **Use IAM Roles instead of Access Keys (AWS):** If Jenkins is running on AWS EC2 or EKS, we *stop storing AWS Access Keys entirely*. Instead, we attach an **IAM Instance Profile** (or IRSA in EKS) to the Jenkins worker nodes. Jenkins automatically inherits temporary, auto-rotating AWS credentials directly from the underlying infrastructure.
4. **Externalize Secrets Management:** For things like database passwords or 3rd party API keys, we integrate Jenkins with **HashiCorp Vault** or **AWS Secrets Manager**. 
   ```groovy
   // Fetching dynamically from Vault instead of Jenkins native credentials
   withVault([vaultSecrets: [[path: 'secret/production/database', secretValues: [...]]]]) {
       // ...
   }
   ```
By implementing these measures, employee turnover has zero impact on CI/CD stability.

```groovy
stage('Debug Credential') {
  steps {
    withCredentials([string(credentialsId: 'my-token', variable: 'TOKEN')]) {
      // NEVER echo the value — but you can test it works:
      sh 'curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $TOKEN" https://api.example.com/health'
      // Returns 200 → credential works. Returns 401 → credential value is wrong.
    }
  }
}
// Remove this debug stage after fixing — don't leave it in production pipeline
```
