# Monitoring Logging — Scenario-Based Questions

---

## 1. You get paged at 2 AM — high latency alerts firing across multiple services. Walk me through your response.

**"Across multiple services" is the key phrase.** Single-service latency is usually a bug in that service. Multi-service latency simultaneously means shared infrastructure is failing — the database, the message queue, the network, or a dependency that everything calls.

**First 2 minutes — narrow the blast radius, not the root cause.**

```bash
# Are these services all in the same namespace? Same node? Same AZ?
kubectl get pods -A -o wide | grep -E "high-latency-service-1|high-latency-service-2"
# If all pods are on the same node → node-level problem (disk, network, noisy neighbor)
# If spread across nodes → VPC/network-level or shared dependency

# Check Grafana — which services, since when
# GROUP BY service — is it all services or specific ones?
```

**Look for a common dependency:**

```promql
# Does the spike correlate with DB latency?
histogram_quantile(0.99, rate(db_query_duration_seconds_bucket[5m]))

# Does it correlate with external API calls?
histogram_quantile(0.99, rate(http_client_request_duration_seconds_bucket[5m]))
```

If multiple services call the same PostgreSQL instance and PostgreSQL latency spiked at the same time the alerts fired — that's your root cause. Not any of the services themselves.

**Check shared infrastructure:**

```bash
# Is the database healthy?
aws rds describe-db-instances --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,SecondaryAvailabilityZone]'

# Is ElastiCache healthy?
aws elasticache describe-cache-clusters

# Check CloudWatch for RDS CPU, connections, freeable memory
aws cloudwatch get-metric-statistics --namespace AWS/RDS --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=prod-postgres \
  --start-time $(date -u -d '15 minutes ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 --statistics Average
```

**Check for a recent deployment:**

```bash
# Was anything deployed in the last 30 minutes?
kubectl rollout history deployment -A | grep -v "No rollout"
git log --all --oneline --since="30 minutes ago"   # In the manifest repo
```

If a shared library or a central service (auth, discovery, messaging) was deployed 5 minutes before the latency spike — that's your starting point.

**Communicate within 5 minutes of being paged:**

> "Investigating high latency on [services]. All affected services share [database/cache/common dependency]. Checking shared infrastructure. Next update in 10 minutes."

Don't wait until you have the answer. The on-call engineer's first job is to reduce stakeholder anxiety, not solve the problem in silence.

**Resolution pattern:** Most multi-service latency spikes resolve to: (1) database CPU/connection saturation, (2) ElastiCache evictions or connection storm, (3) a shared service (auth, discovery) degraded, (4) DNS resolution timeouts affecting all pods, (5) a network path issue in a specific AZ. Start with the shared dependency that all affected services call.

---

## 2. Error rates are spiking, but CPU and memory look completely normal. What's your diagnostic approach?

CPU and memory are compute metrics. Error rates are application-correctness metrics. Normal CPU/memory + high error rates means the application isn't overloaded — it's returning errors for a different reason.

**Step 1 — What type of errors?**

```bash
# What HTTP status codes are being returned?
kubectl logs -n prod -l app=order-service --tail=100 | grep -E '"status":|"level":"ERROR"'

# In Grafana — break down by error type
sum by (status_code) (rate(http_requests_total{service="order-service", status=~"[45].."}[5m]))
```

- `5xx` errors → application/infrastructure problem (your code or what it depends on)
- `4xx` errors → bad requests coming in (client error, or your validation logic changed)
- `4xx` with `499` specifically → clients are timing out before you respond

**Step 2 — Are errors isolated to specific pods?**

```promql
# Error rate per pod
sum by (pod) (rate(http_requests_total{status=~"5..", service="order-service"}[5m]))
/
sum by (pod) (rate(http_requests_total{service="order-service"}[5m]))
```

If one pod has 100% error rate and others have 0% → that pod is broken. Kill it, it'll be replaced. We've had this happen with corrupted local caches and botched config file mounts.

**Step 3 — Check logs for the actual error.**

```bash
kubectl logs -n prod -l app=order-service --tail=500 | grep -E "ERROR|Exception|panic"
```

Nine times out of ten, the log tells you exactly what's happening:
- `connection refused to redis:6379` → cache is down
- `NullPointerException at PaymentProcessor.java:145` → code bug
- `context deadline exceeded` → timeout to a downstream service
- `certificate has expired` → TLS cert expiry

**Step 4 — Check if a dependency is returning errors.**

```bash
# From inside the pod — is the downstream service reachable?
kubectl exec <pod> -n prod -- curl -s http://payment-service/health
kubectl exec <pod> -n prod -- curl -s http://redis:6379/ping
```

**Step 5 — Check for a recent deployment or config change.**

```bash
kubectl rollout history deployment/order-service -n prod
# Did something deploy 10 minutes before the error spike?
```

If yes → check what changed in that deployment. The error is almost certainly in the new code or in a config the new code reads differently.

**The answer interviewers want:** Not "I check metrics." The answer is "I look at the error type to understand the category, then I look at logs to get the specific error message. CPU and memory are irrelevant for this class of problem — the app isn't overloaded, it's returning errors. The logs will tell me why within 2 minutes."

---

## 3. A critical service is flapping — repeatedly going healthy then unhealthy. How do you stabilize it?

Flapping means the service passes health checks, starts receiving traffic, then fails, gets removed from the load balancer, recovers, gets traffic again, and the cycle repeats. Every user who hits it during the unhealthy phase gets an error.

**Immediate action — break the cycle:**

```bash
# Scale down to 0 pods temporarily to stop user-facing errors
kubectl scale deployment <service> --replicas=0 -n prod

# Communicate: "Taken <service> offline. Users will see [fallback behavior].
# Investigating root cause."
```

Taking it offline is better than letting flapping continue — at least the failure mode is predictable.

**Diagnose why it's flapping:**

```bash
# See restart count and last termination reason
kubectl describe pod <pod> -n prod | grep -A 10 "Last State"
# "OOMKilled" → memory limit too low
# "Error" with exit code 1 → application crash
# "Completed" unexpectedly → process exits cleanly (signal handling issue)

# Check the logs right before the crash
kubectl logs <pod> -n prod --previous
```

**Common flapping causes:**

**OOMKill loop:** Pod starts, handles traffic, memory usage grows, hits the limit, OOMKilled, restarts with empty memory. Handles traffic again, hits limit again. Forever.

```bash
kubectl describe pod <pod> | grep -A5 "Last State"
# "Reason: OOMKilled"
```

Fix: Raise memory limit (quick fix), or find the memory leak and fix the root cause.

**Liveness probe too aggressive:** App does heavy initialization (loading a 200MB model, establishing 10 DB connections). Liveness probe fires too early, decides the app is dead, kills the pod. Pod restarts, same thing.

```bash
kubectl describe pod <pod> | grep "Liveness"
# Liveness: http-get http://:8080/health delay=15s timeout=1s period=10s
# If app takes 45s to init, delay=15s means probe fires before app is ready
```

Fix: Increase `initialDelaySeconds` or `failureThreshold`.

**Resource contention on the node:** Something else on the same node is consuming all CPU/memory. Your pod gets evicted or throttled.

```bash
kubectl get events -n prod | grep Evicted
kubectl describe node <node> | grep -A 20 "Conditions:"
```

Fix: Pod affinity/anti-affinity rules to spread pods across nodes, or add resource requests so scheduler places pods on nodes with available resources.

**Restore service after identifying root cause:**

```bash
# Fix the root cause (raise memory limit, fix probe timing, etc.)
kubectl edit deployment <service> -n prod    # Or update via Helm/GitOps

# Scale back up
kubectl scale deployment <service> --replicas=3 -n prod

# Watch the rollout
kubectl rollout status deployment/<service> -n prod
kubectl get pods -n prod -w
```

Post-incident: why didn't we catch this in staging? Flapping under production load suggests the resource profile or traffic pattern in staging was insufficient to trigger the issue.

---

## 4. Alerts are firing, but users aren't reporting any issues. What do you do?

This is a "false positive alert" scenario — but be careful. Sometimes the alert is real and users just haven't noticed yet. Your job is to distinguish between them quickly.

**Step 1 — Check what the alert is actually measuring.**

Read the alert rule carefully. Is it alerting on aggregate metrics or per-resource metrics?

```promql
# Alert: error_rate > 1%
# But this is aggregate — is it 1% of all pods or 1% of one pod?
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
```

If 1 pod out of 10 has a 10% error rate, the aggregate shows 1% → alert fires. But 9 out of 10 users are hitting healthy pods. Alert is real but impact is isolated.

**Step 2 — Verify from outside the cluster (external synthetic monitoring).**

Internal health checks go directly to the Service, bypassing DNS, CDN, and the ingress. A synthetic check hits the real public URL:

```bash
curl -v https://api.yourapp.com/health
# If this returns 200 — users CAN reach the service
# If this returns error — alert is real, users are affected
```

We had a 22-minute incident where all internal dashboards were green. The ingress NetworkPolicy had been updated and blocked external traffic — but our internal health checks went directly to the Service and bypassed the ingress. External synthetic check would have caught it in 30 seconds.

**Step 3 — Check what customers' error reporting would look like.**

Some user-facing errors don't generate immediate support tickets — they see a retry, the retry works, and they don't bother reporting it. Or the error affects a part of the product few users actively use at 2 AM.

**Step 4 — Lower the alert threshold or the alert is miscalibrated.**

If the alert fires at 1% error rate but the baseline noise (transient errors, retried-successfully errors) is normally 0.8%, the threshold is too low. The alert is firing on normal behavior.

```
Action: Don't silence the alert. Fix it.
- Raise the threshold to 2% if baseline is 0.8%
- Add a `for: 5m` clause — error rate must be sustained, not momentary
- Split the alert by pod so you can see which specific pod is affected
```

**Step 5 — Investigate the alert and resolve or document.**

Even if users aren't reporting issues, a real elevated error rate needs investigation. It might be:
- A canary pod running a bad version (5% of traffic hitting errors — 5% of users aren't reporting it yet)
- A background job failing (no user impact now, but data will be stale)
- A dependency degraded but retrying successfully (will eventually stop retrying)

Never silently acknowledge an alert without either fixing the root cause or documenting why it's acceptable.

---

## 5. You fixed the incident, but the same issue keeps recurring. What do you do?

Recurring incidents mean the fix addressed the symptom, not the root cause. This question is testing whether you think in systems, not just in patches.

**Start with a proper root cause analysis (not a timeline):**

A timeline of events is not an RCA. An RCA answers: "What system property or missing safeguard allowed this to happen, and how do we change the system so it can't happen again?"

**The 5 Whys for a recurring "database connection pool exhaustion" incident:**

```
Incident: Service returns 503 errors — connection pool exhausted
Why 1: Pool exhausted — all 5 connections were in use
Why 2: 5 connections are too few for production traffic levels
Why 3: Pool was configured for staging traffic (10 req/s), not prod (200 req/s)
Why 4: No validation that staging config matches prod requirements
Why 5: We don't test pool exhaustion behavior — only functional correctness

Root cause: No production load testing exists. Staging config isn't validated against prod traffic.

Fix: Load test in staging at 2x expected prod traffic. Alert when pool pending > 0. Define pool sizing formula per service.
```

The symptom was "pool exhausted." The root cause was "no load testing and no alert on pool saturation." Fixing only the pool size means it'll exhaust again at the next traffic increase.

**For recurring incidents, look for:**

**Missing alerting:** The incident happened before an alert fired. After fixing it, add an alert that would have given you 30 minutes of warning:

```yaml
# Alert on leading indicator, not lagging symptom
- alert: DBConnectionPoolSaturation
  expr: hikaricp_connections_pending > 0
  for: 2m
  labels:
    severity: warning    # Warning at first sign — before it becomes an outage
```

**Missing automation:** If the fix is "restart the pod" — automate the detection and restart. If you're manually restarting pods every week, write a runbook that becomes a script that becomes an operator.

**Missing protection:** If the incident was "we deployed a bad config" — add CI validation for config correctness before it reaches prod.

**Missing runbook / knowledge transfer:** If the fix required tribal knowledge ("you have to also restart service B when service A gets into this state"), write it down. The next on-call engineer shouldn't spend 40 minutes rediscovering it.

**Track incident recurrence explicitly:**

In our postmortem template, we have a field: "Has this exact incident happened before?" If yes, the postmortem must include: "Why did the previous fix not prevent this?" Any incident that recurs more than twice becomes a P0 engineering task to permanently fix.

**Real scenario:** We had a recurring "Redis cluster failover causes 2-minute spike in errors" every time ElastiCache did a maintenance failover (which AWS does monthly). Same incident, same fix (retry logic in the app), same postmortem every month. After the third occurrence, we treated it as an engineering priority: implemented client-side retry with jitter, set the Redis client `retry_on_error` flag, and added a circuit breaker. The fourth maintenance window: zero user-facing errors. The fix was always possible — we just didn't prioritize it until we enforced that recurring incidents get permanent fixes, not patches.

---

## 6. Management wants a quick summary during an active incident. What do you say?

This question tests whether you can communicate under pressure — to a non-technical audience, with incomplete information, while also actually working the incident.

**The format we use — 4 sentences, under 60 seconds:**

```
1. What is broken (user impact)
2. Since when
3. What we know so far (current theory)
4. What we're doing right now + next update time
```

**Example:**

> "Checkout is failing for approximately 30% of users since 2:14 AM. Users trying to complete a purchase are seeing an error page. We believe it's related to the payment service — we deployed a new version at 2:10 AM and error rates started immediately after. We're rolling back that deployment now — should be resolved in the next 5 minutes. Next update in 10 minutes or when resolved."

**What NOT to say:**
- "We're investigating" (uninformative — of course you are)
- "It might be the database" (speculation creates panic)
- "I don't know yet" without a next step or timeline
- Technical jargon: "The HPA isn't scaling the pods fast enough due to CFS throttling" — management doesn't know what this means and it creates more questions

**During the incident, assign communication ownership:**

One engineer works the problem. A separate person handles stakeholder communication. Never have the engineer who's debugging also responding to Slack messages — context switching under pressure costs time.

We use a template in our incident Slack channel:

```
🔴 INCIDENT: [Service] checkout failures
📅 Started: 2:14 AM
👥 Impact: ~30% of checkout attempts failing
🔍 Status: Rolling back deployment from 2:10 AM
📣 Lead: @abhoy
📞 Next update: 2:30 AM or when resolved
```

This is pinned at the top of the incident thread. Stakeholders can check it without interrupting the engineer working the fix.

**After the incident — the 48-hour postmortem:**

Management will want to understand: why did this happen, what did we do to fix it, and how do we prevent it? Keep the postmortem blameless. The audience is: what system or process failed, not who failed.

The postmortem document we write:
- **Impact:** X users affected, Y minutes of downtime, Z revenue impact (if measurable)
- **Timeline:** What happened, when, in what order
- **Root cause:** The actual technical reason
- **What went well:** Response was fast, alert fired quickly, rollback worked
- **Action items:** Specific changes, each with an owner and due date

The action items matter most. A postmortem without action items is just documentation of failure. A postmortem with specific, assigned, time-bound action items is prevention.
