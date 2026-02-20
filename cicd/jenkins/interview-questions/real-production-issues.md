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
