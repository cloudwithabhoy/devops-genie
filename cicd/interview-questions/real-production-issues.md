# CI/CD — Real Production Issues

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

## 2. A critical bug is found in production — what is your approach?

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

## 3. A deployment succeeded but users get 502 errors — what do you check first, and in what order?

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

## 4. A service works in staging but fails in production — how do you narrow the difference without guessing?

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

## 5. A rollout caused a partial outage — how do you decide between rollback, roll-forward, or hotfix?

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
