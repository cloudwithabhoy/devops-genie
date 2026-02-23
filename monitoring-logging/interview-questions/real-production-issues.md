# Monitoring Logging — Real Production Issues

---

## 1. CPU is low, memory is stable, but latency is high — what metrics do you look at next and why?

Low CPU and stable memory means the app isn't working hard — it's **waiting** on something. The bottleneck is external to the pod's compute. Here's the exact sequence I follow.

**1. CPU throttling — check this before anything else, it's the most common hidden cause.**

Average CPU being low doesn't mean the pod isn't throttled. Linux CFS (Completely Fair Scheduler) limits burst CPU usage per 100ms scheduling period. A pod with a 200m CPU limit can average 15% but still be throttled 60% of the time — whenever it needs a burst (handling a request, GC, connection setup), the kernel pauses it.

```promql
# Prometheus — percentage of CPU periods being throttled
rate(container_cpu_cfs_throttled_periods_total{pod=~"my-app.*"}[5m])
/
rate(container_cpu_cfs_periods_total{pod=~"my-app.*"}[5m])
```

If this is above 25%, CPU throttling is your latency cause — even though `kubectl top pods` shows low CPU.

**Real scenario:** Our Node.js API showed 18% average CPU usage but p99 latency was 800ms. Prometheus showed 70% of CPU scheduling periods were throttled. The app needed short bursts to handle each request, but the 200m limit was capping those bursts. We removed the CPU limit (kept the request at 200m for scheduling). Latency dropped to 120ms immediately. CPU dashboard still showed "18%" — the average hadn't changed, but the bursts were no longer being cut off.

**2. Network latency to dependencies.**

The app executes fast but waits for downstream responses — database queries, external APIs, cache lookups. CPU is idle during the network wait.

```bash
# Distributed traces (Jaeger/Tempo) — find slow spans
# Filter by high-latency traces and look at which span is the bottleneck

# Or quick check from inside the pod
kubectl exec <pod> -- curl -w "%{time_total}s\n" -s -o /dev/null http://postgres-service:5432
kubectl exec <pod> -- curl -w "%{time_total}s\n" -s -o /dev/null http://redis-service:6379
```

```promql
# Prometheus — p99 latency per downstream service
histogram_quantile(0.99, rate(http_client_request_duration_seconds_bucket{service="my-app"}[5m]))
```

**3. Connection pool exhaustion.**

App has 10 database connections in the pool but 50 concurrent requests arrive. 40 threads block waiting for a free connection — CPU idle, memory stable, but every request waits. This doesn't show in CPU or memory metrics at all.

```promql
# HikariCP (Java) pool wait time — if this is non-zero, you have pool exhaustion
hikaricp_connections_pending{pool="HikariPool-1"}
hikaricp_connections_acquire_seconds_sum / hikaricp_connections_acquire_seconds_count
```

For Python (SQLAlchemy), Go (pgxpool): check the pool's queue_depth or wait_time metric if exposed.

**Real scenario:** A microservice had CPU at 12%, memory at 40%, but p99 latency was 3 seconds. Traces showed the slow span was always on the database call — not the query execution time (5ms), but the time waiting to acquire a connection (2.9 seconds). The pool was configured with `max_connections=5` — fine for staging traffic, completely insufficient for production. Increased to 20 connections. Latency dropped to 80ms.

**4. Disk I/O — relevant for services writing logs, using local storage, or running databases.**

```promql
# Node-level disk I/O wait
rate(node_disk_io_time_seconds_total{device="nvme0n1"}[5m])

# CPU iowait — percentage of time CPU is idle waiting for I/O
rate(node_cpu_seconds_total{mode="iowait"}[5m])
```

If `iowait` is high, threads are blocked waiting for disk reads/writes. App looks idle but is actually stuck in I/O.

**5. DNS resolution time.**

With `ndots:5` in Kubernetes, every outbound call (database, external API) triggers up to 5 DNS lookups before the actual query. Under load, CoreDNS gets overwhelmed. Each DNS lookup that should take 1ms starts taking 100ms.

```bash
kubectl exec <pod> -- time nslookup postgres-service.prod.svc.cluster.local
# Should be < 5ms. If it's 100ms+, CoreDNS is your bottleneck.
```

```promql
# CoreDNS request latency
histogram_quantile(0.99, rate(coredns_dns_request_duration_seconds_bucket[5m]))
```

**6. GC pauses (JVM, Go, Node.js).**

Garbage collection pauses all threads briefly. A 200ms GC pause every 10 seconds means 2% of requests are slow — CPU average looks normal, but p99 latency spikes periodically.

```promql
# JVM GC pause duration
rate(jvm_gc_pause_seconds_sum[5m])
```

In Grafana, if you see latency spikes that correlate with GC pause events, you either need to tune GC settings or reduce allocation rate.

**The mental model:** "Low CPU + high latency" means the app is **blocked on something**, not computing. Always ask: what are the threads waiting for? It's one of: network (dependency latency), I/O (disk), locks (connection pool, mutex), or GC. Metrics tell you which.

---

## 2. Logs look fine, metrics look fine, but customers report errors — what signal do you trust next?

Your observability has a blind spot. The user is always right. Here's how I find what's missing.

**1. Per-pod metrics, not aggregate.**

If 10 pods serve traffic and 1 is broken, the aggregate error rate shows 90% healthy — below your alert threshold. But 10% of users hitting that pod see 100% errors.

```promql
# Error rate per pod, not averaged across the deployment
sum by (pod) (rate(http_requests_total{status=~"5.."}[5m]))
/
sum by (pod) (rate(http_requests_total[5m]))
```

**Real scenario:** We had 8 healthy pods and 1 pod with a corrupted local cache. Aggregate error rate was 0.4% — below the 1% alert threshold. But for users hitting that one pod, every request failed. Found it by switching from the service-level Grafana dashboard to a per-pod breakdown. The broken pod was immediately obvious.

**2. Client-side 499s — the user gave up before the server responded.**

If the client closes the connection before the server responds (slow response, timeout), the server may not log an error at all — or logs a 499 (NGINX client disconnect). The user experienced a timeout, but your app metrics show zero errors because no response was sent.

```bash
# Check NGINX ingress access logs for 499s
kubectl logs -n ingress-nginx <ingress-pod> | grep " 499 "
```

We found 300+ 499s per minute during peak hours — clients timing out after 30 seconds. Server-side: zero errors. The latency was the problem, not errors.

**3. CDN, WAF, or load balancer level.**

Errors may happen before traffic reaches your cluster. The CDN serves a cached error page, the WAF blocks requests matching a new rule, the ALB health check fails and stops routing to some targets.

None of this appears in pod-level logs. Check:

```bash
# ALB target health
aws elbv2 describe-target-health --target-group-arn <arn>

# WAF blocked requests
aws wafv2 get-sampled-requests --web-acl-arn <arn> --rule-metric-name <rule>

# CloudFront error rates (5xx from origin vs from cache)
# Check CloudFront metrics in CloudWatch — separate from your app metrics
```

**4. External synthetic monitoring — hit the real URL from outside.**

Your internal health checks hit the Service directly, bypassing DNS, CDN, load balancer, and ingress. A synthetic monitor hits `https://api.yourapp.com` — the same path a real user takes.

```bash
# Quick external check from your laptop
curl -v https://api.yourapp.com/health

# Automated: Prometheus Blackbox Exporter
# Pings the public URL every 30s, alerts if it fails
```

We had a 22-minute outage where all internal dashboards were green. The ingress NetworkPolicy had been updated and blocked user traffic — but our internal health checks went directly to the Service, bypassing the ingress entirely. An external synthetic check would have caught it in 30 seconds.

**5. Downstream dependency returning degraded data (not errors).**

Your service returns 200 with correct-looking responses. But it's calling an upstream service that's returning stale or wrong data. No errors, correct latency — but users see incorrect data.

Check distributed traces for the actual response bodies from dependencies, or add validation metrics: `orders_created_total` dropping while `checkout_requests_total` stays stable → checkout is completing but orders aren't being created.

**The meta-lesson:** After every "logs fine, metrics fine, users unhappy" incident, I add the missing observability signal. Aggregate metrics → per-pod metrics. Internal health checks → external synthetic monitoring. Pod-level metrics → CDN/LB-level metrics. Over time, the blind spots shrink.

---

## 3. Grafana dashboard is responding slowly — how do you troubleshoot it?

Slow Grafana dashboards have three possible root causes: expensive queries hitting Prometheus/datasource, Grafana's own resource constraints, or datasource performance degradation. You eliminate them in order.

**Step 1: Identify which panel is slow using the browser.**

Open the dashboard. Open browser DevTools → Network tab. Reload the dashboard. Look for slow requests — each panel fires its own query. The slowest request points to the expensive panel.

In Grafana itself:

```
Dashboard → Query Inspector (click the panel → Inspect → Query)
→ Shows: query sent to Prometheus, response time, data points returned
```

If a query returns 100,000+ data points with sub-second time ranges, that's the bottleneck.

**Step 2: Check the Prometheus query performance.**

Grafana is slow because Prometheus is slow evaluating the query, or returning too much data.

```bash
# Open Prometheus directly and test the query
# http://prometheus:9090/graph → paste the query → run it
# Check how long the query takes

# Common slow query patterns:
# 1. High cardinality — too many label combinations
rate(http_requests_total[5m])   # If http_requests_total has 10,000 time series → slow

# Fix: add label filters to reduce cardinality
rate(http_requests_total{service="order-service", env="prod"}[5m])

# 2. Range too wide
increase(http_errors_total[30d])  # 30-day range = Prometheus reads 30 days of chunks → very slow

# Fix: use recording rules for long-range queries
# precompute the result and store it as a new metric
```

**Step 3: Add recording rules for expensive queries.**

Recording rules precompute query results at scrape time so Grafana reads the result instantly instead of computing it on demand.

```yaml
# prometheus-recording-rules.yml
groups:
  - name: dashboard_precompute
    interval: 1m    # Recompute every minute
    rules:
      # Precompute error rate per service
      - record: job:http_error_rate:rate5m
        expr: |
          rate(http_requests_total{status=~"5.."}[5m])
          /
          rate(http_requests_total[5m])

      # Precompute p99 latency per service
      - record: job:http_request_duration_p99:rate5m
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

In Grafana, replace the original slow query with the recording rule metric:

```promql
# Before (computed at query time — slow):
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# After (pre-recorded — instant):
job:http_request_duration_p99:rate5m
```

**Step 4: Check Grafana's own resource usage.**

```bash
# Check Grafana pod resources (if running in Kubernetes)
kubectl top pod grafana-0 -n monitoring
# CPU: 950m (near limit), Memory: 1.8Gi

# Grafana is CPU-throttled or OOM — increase limits
kubectl edit deployment grafana -n monitoring
# resources:
#   requests: { cpu: "500m", memory: "512Mi" }
#   limits:   { cpu: "2000m", memory: "2Gi" }   ← increase these

# Check Grafana's own logs for slow query warnings
kubectl logs grafana-0 -n monitoring | grep -i "slow\|timeout\|warn"
```

**Step 5: Reduce dashboard complexity.**

```
Common dashboard anti-patterns that cause slowness:

1. Too many panels on one dashboard (>20 panels all querying simultaneously)
   Fix: split into multiple dashboards, use variables to filter scope

2. Auto-refresh set to 5s on a heavy dashboard
   Fix: set refresh to 1m or 5m — users rarely need sub-minute refresh

3. Time range set to "Last 30 days" by default
   Fix: default to "Last 1 hour" — use longer ranges only when investigating

4. No use of $__interval variable (fixed intervals = Prometheus returns more data than Grafana can render)
   Fix: use $__interval in rate() functions so Grafana adapts to the selected time range
```

```promql
# Bad — fixed 5m interval regardless of dashboard time range
rate(http_requests_total[5m])

# Good — interval adapts to dashboard zoom level
rate(http_requests_total[$__interval])
```

**Step 6: Check the datasource performance directly.**

```bash
# Test Prometheus response time directly (bypassing Grafana)
curl -s -w "\nTime: %{time_total}s\n" \
  "http://prometheus:9090/api/v1/query?query=up"

# If Prometheus itself is slow: check its resources
kubectl top pod prometheus-0 -n monitoring

# Prometheus may need more memory (it caches chunks in RAM)
# Or the TSDB may need compaction:
# Prometheus UI → /tsdb-status → check cardinality

# High cardinality = millions of unique time series = slow queries + high memory
```

**Diagnostic sequence:**

```
1. Browser DevTools → identify slowest panel/query
2. Run that query directly in Prometheus UI → measure response time
3. If Prometheus is slow → add recording rules or reduce cardinality
4. If Prometheus is fast but Grafana is slow → check Grafana resources
5. Reduce dashboard panels, increase refresh interval, use $__interval
```

**Real scenario:** Our main engineering dashboard had 28 panels and was taking 45 seconds to load. Query Inspector showed 3 panels were each running histogram_quantile over 30d ranges with no label filters — each query took 12–15 seconds in Prometheus. We added recording rules for the three expensive queries and added `env="prod"` label filters. Dashboard load time dropped to 3 seconds. We also split the dashboard into "Operations" (real-time, 1-hour default range) and "Weekly Review" (7-day default range) so engineers always opened the fast one first.