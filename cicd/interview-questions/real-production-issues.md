# CI/CD — Real Production Issues

---

## 1. A critical bug is found in production — what is your approach?

The wrong answer: "I create a hotfix branch and deploy it." That's a step, not an approach.

The right answer covers: assess → communicate → fix → deploy → verify → prevent.

**Step 1 — Assess severity and impact (first 2 minutes).**

Before touching any code, I need to know: how many users are affected? Is it a data issue (worse) or a display issue (less bad)? Is it getting worse over time?

Check: error rate in Grafana, number of users hitting the error (from logs or analytics), whether the issue is isolated to a specific endpoint or global.

**Step 2 — Communicate immediately.**

Post in the incident channel: "Critical bug in production affecting [X]. Investigating. Next update in 10 minutes." Don't wait until you have the fix. Stakeholders need to know now.

**Step 3 — Decide: rollback or hotfix.**

- If the bug was introduced by the last deployment → **rollback first, fix later.** `kubectl rollout undo` takes 30 seconds. You can investigate the root cause without users experiencing the bug.
- If rolling back would cause a different problem (schema migration already ran, downstream services adapted to new API) → **hotfix**.

We default to rollback unless there's a specific reason not to. Investigating a bug while users are down is the wrong order of operations.

**Step 4 — Hotfix flow (if rollback isn't appropriate).**

```bash
# Branch from main (production), NOT from develop
git checkout main
git checkout -b hotfix/payment-null-pointer

# Make the fix, commit
git add . && git commit -m "fix: handle null userId in payment processor"

# PR to main — expedited review (1 reviewer instead of 2)
# Once merged, same CI/CD pipeline runs
# ArgoCD deploys to production
```

Critical rule: **the hotfix goes through the CI/CD pipeline**. No direct kubectl edits, no bypassing tests. Even in an emergency, running tests takes 3 minutes and has caught bugs in the hotfix itself twice.

After deploying to main, cherry-pick the fix back to `develop` so it doesn't get overwritten by the next regular release:

```bash
git checkout develop
git cherry-pick <hotfix-commit-sha>
```

**Step 5 — Verify.**

Don't declare victory when the deploy completes. Watch:
- Error rate in Grafana (should drop to baseline)
- The specific endpoint that was broken (hit it manually or via smoke test)
- Any secondary effects — fixing one thing sometimes breaks another

**Step 6 — Post-incident review within 48 hours.**

Not a blame session. Questions:
- Why didn't tests catch this?
- Why didn't staging catch this?
- What monitoring should have alerted us earlier?
- How do we prevent this class of bug?

**Real scenario:** A null pointer in our payments service started throwing 500s for ~2% of users. The bug was introduced 3 deploys ago — not the latest one. Rolling back 3 releases would have reverted 2 weeks of features. We did a hotfix: 1-line null check, PR reviewed in 5 minutes, deployed in 8 minutes. Total user impact: 23 minutes from alert to resolution. The postmortem found we had no test coverage for null userId (edge case that only appeared when a user's session expired mid-transaction). We added the test case and it's been in the test suite since.

---

## 2. A deployment succeeded but users get 502 errors — what do you check first, and in what order?

502 means the gateway received an invalid or no response from the upstream. The deployment succeeded — K8s is happy, ArgoCD shows synced — but users aren't.

My order of checks, from the closest layer to the user inward:

**1. Ingress controller logs — first stop, most common cause.**

During a rolling update, NGINX still has old pod IPs in its upstream list. Those pods are terminating — NGINX sends a request, gets a connection reset, returns 502. This is the #1 cause of deploy-related 502s.

```bash
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=100 | grep 502
# Look for: "upstream prematurely closed connection" or "connect() failed"
```

If you see `upstream prematurely closed connection while reading response header from upstream` — this is the terminating pod race condition. Fix: add a `preStop` hook to old pods to keep them alive briefly while the ingress updates its upstream list:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]
```

This single fix eliminated 90% of our deploy-time 502s.

**2. Pod readiness — are new pods actually ready?**

```bash
kubectl get pods -n prod
```

If new pods show `0/1 Ready` (Running but not Ready), the readiness probe is failing. The Service has no endpoints, the ingress backend has no targets → 502.

Common readiness probe failures after a deploy:
- New code changed the health endpoint path (`/health` → `/api/health`) but the probe wasn't updated
- App takes longer to initialize than `initialDelaySeconds` allows — traffic arrives before the app is ready
- New version depends on a config or secret that doesn't exist yet

**3. Service endpoints — zero backends means guaranteed 502.**

```bash
kubectl get endpoints <service-name> -n prod
```

If the endpoints list is empty, the Service selector doesn't match any pod labels. Happens when someone renames a label in the Deployment but forgets to update the Service. The deployment succeeds because the Deployment itself is valid — but the Service is now pointing at nothing.

**4. Resource limits — OOMKill loop.**

New version uses slightly more memory. Pod starts, handles a request, hits the memory limit, gets OOMKilled, restarts. During restart the pod is unavailable → 502 for that request slot.

```bash
kubectl describe pod <pod-name> -n prod | grep -A5 "Last State"
# Look for: "Reason: OOMKilled"
```

The pod might show Running at any moment (fast restart), but `kubectl describe` shows the OOMKill in Last State.

**5. App initialization time vs probe timing.**

New Spring Boot service, `initialDelaySeconds: 10`, but the app takes 45 seconds to load context, connect to DB, warm caches. The probe passes at 10 seconds (HTTP server accepts connections), traffic arrives, app returns 500 (not ready) which the ingress translates to 502.

Check: `kubectl logs <new-pod> -n prod` — is the app still printing initialization logs when the first requests arrive?

**My 30-second triage:**
```bash
kubectl get pods -n prod                          # Are new pods Ready?
kubectl get endpoints <svc> -n prod               # Any backends?
kubectl logs -n ingress-nginx <ingress-pod> | grep 502  # Upstream errors?
kubectl describe pod <new-pod> -n prod | grep OOM  # Memory kills?
```

**Pattern to remember:** 502s that appear exactly during rollouts and vanish 2–3 minutes later = lifecycle coordination issue. Fix with `preStop` hooks, correct `initialDelaySeconds`, and `maxUnavailable: 0`.

---

## 3. A service works in staging but fails in production — how do you narrow the difference without guessing?

The answer is systematic diffing, not intuition. "It worked in staging" tells you the code is probably fine — the difference is in the environment.

**1. Diff the manifests first.**

```bash
kubectl get deploy <name> -n staging -o yaml > staging.yaml
kubectl get deploy <name> -n prod -o yaml > prod.yaml
diff staging.yaml prod.yaml
```

Look for: different image tags, different env vars, different resource limits, different ConfigMaps mounted. We once spent 3 hours debugging a prod failure. Root cause: staging had `LOG_LEVEL=debug` which happened to initialize a module that `LOG_LEVEL=warn` (prod) skipped. One env var, completely different behavior.

**2. Diff ConfigMaps and Secrets (values, not just existence).**

```bash
kubectl get cm app-config -n staging -o yaml
kubectl get cm app-config -n prod -o yaml
```

Staging often points to mock services (mock payment gateway, test SMTP). Prod points to real services that require real authentication. A new microservice dependency added in staging might not have been whitelisted in prod's config.

**3. Diff NetworkPolicies.**

Staging typically has no NetworkPolicies — everything can talk to everything. Prod has default-deny. A new service dependency works in staging and silently times out in prod because nobody added an allow rule.

```bash
kubectl get networkpolicy -n prod
# Is there a rule allowing traffic from this service to its new dependency?
```

**4. Diff RBAC.**

If the service calls the Kubernetes API (service discovery, reading ConfigMaps), staging might have a permissive ServiceAccount. Prod has a scoped one. The call gets a `403 Forbidden` in prod, works fine in staging.

```bash
kubectl auth can-i get configmaps --as=system:serviceaccount:prod:my-app-sa -n prod
```

**5. Diff infrastructure.**

Different K8s versions between staging and prod? A deprecated API that works in 1.28 might behave differently in 1.27. Different CNI versions? Different ingress controller versions?

```bash
kubectl version --short --context=staging
kubectl version --short --context=prod
```

**6. Diff traffic scale.**

Staging gets 10 req/sec from synthetic tests. Prod gets 10,000. The app works fine at low load but hits connection pool limits, thread exhaustion, or third-party API rate limits at production scale.

This one can't be caught by config diffing — you need load testing in staging with production-level traffic patterns.

**My go-to first command:**

```bash
kubectl get events -n prod --sort-by=.lastTimestamp | head -20
```

Events tell you what Kubernetes is complaining about — failed mounts, image pull errors, probe failures — things the app may not log at all.

**Real scenario:** New service deployed. Worked perfectly in staging for 2 weeks. Failed in prod immediately — connection refused errors to a downstream API. Diff of the NetworkPolicies found prod had a `deny-all` egress rule that staging didn't. The new service called an internal API on port 8443. That port wasn't in the allow list. Zero errors in the app's own logs — just connection timeouts. Added one line to the NetworkPolicy, deployed, fixed.

---

## 4. A rollout caused a partial outage — how do you decide between rollback, roll-forward, or hotfix?

Three options, different use cases. The wrong answer is picking one without reasoning about the situation.

**The decision framework:**

```
Is the outage severe (payments down, data at risk, >30% users affected)?
  → ROLLBACK NOW. Don't investigate while users are suffering.

Is the outage minor and root cause obvious?
  → Is it a config issue fixable in < 5 minutes?
     → ROLL-FORWARD (update config, redeploy)
  → Is it a 1–2 line code fix and rollback has side effects?
     → HOTFIX (goes through full pipeline)
  → Not sure of root cause?
     → ROLLBACK first, then investigate
```

**Rollback — default choice for severe incidents:**

```bash
kubectl rollout undo deployment/<name> -n prod
# Takes 30–60 seconds to complete

# Verify it worked
kubectl rollout status deployment/<name> -n prod
kubectl get endpoints <svc> -n prod
```

When to choose rollback:
- Outage is severe and every minute costs users/revenue
- Root cause is unclear — you can't fix what you don't understand yet
- Previous version was stable
- No risky side effects from reverting (no DB migration, no downstream API contract change)

**Real scenario:** We deployed a new auth service version. Within 4 minutes, every service that depended on it started returning 401s. Error rate went from 0.1% to 45%. The auth pods were Running and Ready — but the new code used a different JWT signing key than what other services expected. We didn't know the root cause yet. Rollback: `kubectl rollout undo deployment/auth-service`. 45 seconds later error rate dropped to 0.2%. Then we investigated. If we'd tried to debug while 45% of requests were failing, we'd have made worse decisions under pressure.

**Roll-forward — when rollback would cause its own problems:**

When to choose roll-forward:
- A database migration already ran — rolling back would leave the schema in an inconsistent state
- Downstream services already adapted to the new API — reverting breaks them instead
- Root cause is clear and it's a config mistake fixable immediately

```bash
# Example: wrong Redis endpoint in ConfigMap pointed to staging
kubectl edit configmap app-config -n prod  # Fix the endpoint
kubectl rollout restart deployment/<name> -n prod  # Pick up new config
```

Takes 3–5 minutes total. Faster than a full rollback + re-deploy cycle.

**Hotfix — when the fix is small and rollback is unsafe:**

When to choose hotfix:
- A 1–2 line code fix you're confident about
- Rollback would revert 2 weeks of other features that are working fine
- DB migration makes rollback destructive

```bash
git checkout main
git checkout -b hotfix/fix-jwt-key-rotation
# fix the code
git push && open PR → expedited review → merge → CI/CD pipeline runs
```

Critical rule: **hotfix goes through the pipeline**. No `kubectl edit` in prod, no bypassing tests. We've caught bugs in the hotfix itself during the 3-minute test run twice.

**Non-negotiable regardless of which option you choose:**

Post in the incident channel within the first 2 minutes:
> "Partial outage on [service] since [time]. Caused by deployment [version]. Rollback in progress. Next update in 10 minutes."

Don't speculate on root cause until you know. Don't go silent. The worst outcome isn't picking the wrong option — it's 5 engineers silently debugging while stakeholders are in the dark.

---

## 5. CI/CD pipeline succeeded, but the deployment failed in production — how do you handle it?

"Pipeline succeeded" means CI passed — tests, scan, build, image push all green. "Deployment failed" means the application isn't actually running correctly in production. These are two different things, and confusing them is dangerous.

**First: understand what "deployment failed" actually means.**

The pipeline's last step is usually updating the image tag in the manifest repo (for GitOps) or running `kubectl apply` / `helm upgrade`. "Succeeded" in the pipeline could mean the manifest was updated — but ArgoCD hasn't synced yet, or the rollout failed after the pipeline finished.

```bash
# Check what ArgoCD actually did with the manifest update
argocd app get order-service --grpc-web
# Status: Degraded? OutOfSync? Progressing?

# Check the actual rollout status in the cluster
kubectl rollout status deployment/order-service -n prod
# Waiting for deployment — this is where real failures appear

# Check pod status
kubectl get pods -n prod -l app=order-service
# CrashLoopBackOff / OOMKilled / ErrImagePull — tells you why
```

**Common reasons the pipeline passes but production breaks:**

**1. Smoke test not in the pipeline (most common).**

The pipeline marks the build successful after pushing to ECR and updating the manifest. But nobody checks if the new pods actually started correctly. A smoke test should be the final pipeline stage:

```groovy
stage('Verify Deployment') {
  steps {
    script {
      // Wait for rollout to complete
      sh 'kubectl rollout status deployment/order-service -n prod --timeout=5m'

      // Hit the health endpoint of the new pods
      def response = sh(
        script: 'curl -s -o /dev/null -w "%{http_code}" https://api.prod.com/health',
        returnStdout: true
      ).trim()

      if (response != '200') {
        error("Smoke test failed: health endpoint returned ${response}")
        // Trigger rollback automatically
        sh 'kubectl rollout undo deployment/order-service -n prod'
      }
    }
  }
}
```

**2. GitOps lag — pipeline finished before ArgoCD synced.**

In a GitOps setup (ArgoCD), the pipeline updates the manifest repo and exits. ArgoCD polls the repo every 3 minutes and then applies. The pipeline is done 3 minutes before the actual deployment even starts.

```groovy
stage('Wait for ArgoCD sync') {
  steps {
    sh '''
      argocd app wait order-service \
        --grpc-web \
        --server argocd.internal.company.com \
        --timeout 300 \
        --health
    '''
    // argocd app wait blocks until the app is Healthy or times out
  }
}
```

`argocd app wait --health` blocks the pipeline stage until ArgoCD reports the app as `Healthy`. If the new pods crash loop, ArgoCD reports `Degraded`, `argocd app wait` fails, and the pipeline marks the stage failed.

**3. Readiness probe failure — pods run but never become Ready.**

The deployment shows `Running` in the pod list but the rollout doesn't complete. New pods aren't passing readiness probes, so the Service doesn't route traffic to them, and the rolling update stalls with old pods still running.

```bash
kubectl describe pod <new-pod> -n prod | grep -A 20 "Events:"
# "Readiness probe failed: HTTP probe failed with statuscode: 404"
# → Health endpoint path changed in new version
# "Readiness probe failed: dial tcp: connection refused"
# → App not listening on the expected port
```

**4. OOMKilled — new version has higher memory usage.**

New pod starts, passes readiness probe, gets traffic, handles a few requests with increased memory usage, hits limit, OOMKilled, restarts. The pipeline sees the rollout complete (pods reached Ready once), marks success. But pods are in a restart loop.

```bash
kubectl describe pod <pod> -n prod | grep OOM
# "Last State: Terminated Reason: OOMKilled"
```

**Rollback procedure when you catch post-pipeline failure:**

```bash
# Immediate rollback via kubectl
kubectl rollout undo deployment/order-service -n prod

# In GitOps — revert the manifest commit so ArgoCD re-syncs to old state
git revert <manifest-commit-sha>
git push    # ArgoCD picks up the revert and rolls back

# Verify rollback completed
kubectl rollout status deployment/order-service -n prod
kubectl get endpoints order-service -n prod   # Should show healthy backends
```

**What we added after our first "pipeline succeeded, prod broken" incident:**

1. `argocd app wait` in the final pipeline stage — pipeline only succeeds when ArgoCD confirms healthy
2. Smoke test hitting the real production endpoint post-deploy
3. Automatic rollback triggered if smoke test fails
4. PagerDuty alert if rollout takes > 10 minutes (rolling updates should complete in < 3 minutes)

The pipeline went from "build + push + manifest update" to "build + push + manifest update + wait for healthy + smoke test." Added 3 minutes to pipeline time. Caught 3 production failures automatically in the first month — none reached users.

---

## 6. Your build pipeline failed at the deployment stage — how would you debug it?

The deployment stage is the last stage — by this point, build, test, and image push have all passed. Failure here is narrower than a general pipeline failure, but it still has several distinct root causes.

**Step 1 — Read the exact error message before doing anything else.**

```bash
# For Jenkins — Console Output
# For GitHub Actions — failed step's output in the Actions tab
# For GitLab CI — job log

# Common deployment stage errors and what they mean:
# "Error from server (Forbidden): deployments is forbidden"
#    → CI service account doesn't have RBAC permission to create/update Deployments

# "ImagePullBackOff" after kubectl apply
#    → Image tag doesn't exist in registry, or registry auth failed

# "error: unable to recognize 'manifests/deploy.yaml': no matches for kind"
#    → Kubernetes API version mismatch — manifest uses deprecated/removed API

# "argocd app wait: timed out waiting for app to become healthy"
#    → Deployment in cluster is failing (pods not ready) — separate from pipeline

# "helm upgrade failed: another operation is in progress"
#    → Concurrent Helm release lock (from a stuck previous run)
```

**Step 2 — Identify which layer failed.**

```
CI pipeline layer:  kubectl/helm command returned non-zero exit code
                    → Check the exact command output in the pipeline log

Cluster layer:      kubectl returned success but deployment is degraded
                    → Check ArgoCD app status / kubectl rollout status

Application layer:  Pods start but crash or fail readiness probes
                    → kubectl describe pod, kubectl logs
```

**Step 3 — Debug the most common causes.**

**RBAC / permission error:**

```bash
# Reproduce locally — run the exact same kubectl/helm command
# with the CI service account's kubeconfig
kubectl auth can-i create deployments \
  --as=system:serviceaccount:ci:jenkins-sa -n prod
# "no" → missing ClusterRole or RoleBinding

# Fix: check the CI service account's RBAC
kubectl get rolebinding -n prod | grep jenkins
kubectl describe clusterrolebinding jenkins-deploy
```

**Image doesn't exist in registry:**

```bash
# Check the pipeline — did the push stage succeed?
# Did the image tag in the manifest match what was pushed?

# Often the bug: pipeline uses $GIT_COMMIT but manifest update used wrong variable
grep "image:" k8s-manifests/apps/order-service/values-prod.yaml
# image.tag: ""  ← empty — variable substitution silently failed

# Test image pull manually
docker pull 123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service:abc123
# Or from inside the cluster:
kubectl run test --image=123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service:abc123 \
  --restart=Never -n prod
kubectl describe pod test -n prod | grep -A5 Events
```

**Helm lock stuck (concurrent release):**

```bash
helm list -n prod | grep order-service
# STATUS: pending-upgrade  ← previous Helm operation didn't complete

# Release the lock
helm rollback order-service 0 -n prod   # 0 = previous release
# OR
kubectl delete secret -n prod -l "status=pending-upgrade,name=order-service"
```

**Kubernetes API version removed:**

```bash
# Check what API versions the cluster supports
kubectl api-versions | grep apps

# Manifest uses apps/v1beta1 but cluster is K8s 1.25+ (removed it)
kubectl explain deployment --api-version=apps/v1
# Use apps/v1 instead
```

**Step 4 — Check cluster-side events, not just pipeline logs.**

A deployment stage can fail because the pipeline command succeeded but the cluster rejected the result:

```bash
kubectl get events -n prod --sort-by=.lastTimestamp | tail -20
# "FailedCreate: Error creating: pods 'order-service-xxxx' is forbidden:
#  exceeded quota: pods, requested: pods=1, used: pods=10, limited: pods=10"
# → ResourceQuota in the namespace is exhausted — no quota for new pod

kubectl describe resourcequota -n prod
```

**Step 5 — Reproduce outside the pipeline.**

If the error is hard to read in CI output, run the deployment command manually from your machine using the same credentials:

```bash
# Extract what the pipeline runs
kubectl apply -f manifests/ --dry-run=server
helm upgrade --install order-service ./charts/order-service \
  -f values-prod.yaml --dry-run
```

`--dry-run=server` sends the manifest to the API server for validation without applying it. It catches API version issues, schema validation errors, and RBAC problems — without touching the live deployment.

**Real scenario:** A deployment stage started failing after a Kubernetes upgrade from 1.24 to 1.25. The pipeline error was cryptic: `no matches for kind "PodSecurityPolicy" in version "policy/v1beta1"`. PodSecurityPolicy was removed in 1.25. Our Helm chart had a PSP template. The fix was to remove the PSP template and replace it with Pod Security Admission labels on the namespace. The pipeline error came from `helm upgrade` failing validation. Reproducing it with `helm upgrade --dry-run` locally made the error readable immediately.

---

## 7. What's your approach to handle failures in automated test or build stages?

Test and build failures are where pipelines spend most of their failing time. The question is: how do you make failures actionable — not just "the pipeline is red."

**Principle: fail fast, fail clearly.**

The goal is that a developer who gets a failed pipeline notification can understand what broke and why within 30 seconds, without opening the full log.

**For test failures:**

```groovy
// JUnit result publishing — turns test failures into structured reports
// Jenkins: failures show as individual test cases in the UI, not buried in log
stage('Test') {
  steps {
    sh 'pytest tests/ --junitxml=test-results/results.xml -v'
  }
  post {
    always {
      junit 'test-results/results.xml'
      // Failed tests appear in "Test Results" tab with:
      // - Which test failed
      // - What was expected vs actual
      // - Full traceback
    }
    failure {
      // Attach the test report to the Slack notification
      slackSend channel: '#ci-alerts',
        message: "Tests failed in ${env.JOB_NAME} — ${env.BUILD_URL}testReport"
    }
  }
}
```

For GitHub Actions, publish test results with a summary in the PR:

```yaml
- name: Publish test results
  uses: EnricoMi/publish-unit-test-result-action@v2
  if: always()   # Run even if tests failed
  with:
    files: test-results/*.xml
# Creates a check on the PR that shows which tests failed
```

**For flaky tests — the silent killer of CI reliability:**

A flaky test fails intermittently — passes 80% of runs, fails 20% with no code change. This destroys trust in the pipeline. Engineers start re-running builds and ignoring failures.

```groovy
// Retry flaky tests before failing the build — but track flakiness
stage('Test') {
  steps {
    retry(2) {
      sh 'pytest tests/ --junitxml=results.xml'
    }
  }
}
// If a test passes on retry — it's flaky. Track it.
// We use a Slack alert when retries occur: "Tests passed after retry — possible flakiness in test X"
```

Better: use pytest-rerunfailures to retry at the individual test level:

```bash
pytest tests/ --reruns 2 --reruns-delay 1 --junitxml=results.xml
# Retries each failing test 2 times before marking it as failed
# Flaky tests that pass on retry are marked in the JUnit XML for tracking
```

We maintain a "flaky test register" — tests that fail more than 3 times in 30 days without a code change are quarantined (moved to a separate non-blocking test suite) until fixed. This keeps the main pipeline trustworthy.

**For build failures (Docker build, compilation, dependency install):**

```bash
# Make build errors immediately visible
docker build --progress=plain -t my-app:$GIT_COMMIT .
# --progress=plain shows each layer's output — you see which step failed and why
# Without this: Docker's default output hides the failing command's error
```

For dependency failures:

```bash
# Cache dependencies to avoid slow reinstalls — but also cache the error
# If pip install fails: is it a new transitive dependency that broke?
pip install -r requirements.txt --verbose 2>&1 | tail -50
# See the actual package installation error, not just "exit code 1"
```

**Notification strategy — alert the right person, not everyone:**

```groovy
post {
  failure {
    // Notify the person who triggered the build — not the whole channel
    emailext (
      to: "${env.BUILD_USER_EMAIL}",
      subject: "Build failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
      body: """
        Stage: ${env.FAILED_STAGE}
        Branch: ${env.BRANCH_NAME}
        Commit: ${env.GIT_COMMIT}
        Log: ${env.BUILD_URL}console
      """
    )
    // Notify the team channel only for main/production pipelines
    script {
      if (env.BRANCH_NAME == 'main') {
        slackSend channel: '#deployments', color: 'danger',
          message: "MAIN branch build failed: ${env.JOB_NAME} | ${env.BUILD_URL}"
      }
    }
  }
}
```

**Blocking vs non-blocking failures:**

Not all failures should block the pipeline:

```groovy
stage('SAST') {
  steps {
    // Critical CVEs: block the build
    sh 'trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed $IMAGE'
  }
}
stage('Code Coverage') {
  steps {
    // Coverage report: informational, don't block for low coverage on old code
    sh 'pytest --cov=app --cov-report=xml || true'  // Never fails the build
    // Separately: fail if coverage drops below threshold on new code only
    sh 'coverage report --fail-under=80 --include="app/new_feature*"'
  }
}
```

**Real scenario:** Our pipeline had an integration test that called a real third-party API (SMS provider). It worked 95% of the time. The 5% failures were API timeouts — not our code. Engineers were re-running builds daily. We moved that test to a separate nightly pipeline (not on every PR), replaced it with a mocked version for the PR pipeline, and added a weekly report of nightly flakiness. PR pipeline reliability went from 93% to 99.4%. Engineers stopped ignoring red builds.
