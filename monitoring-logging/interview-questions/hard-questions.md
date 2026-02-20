# Monitoring Logging — Hard Questions

---

## 1. What is the difference between a CPU bottleneck and an I/O bottleneck, and how do you diagnose each?

This question tests whether you understand what processes are actually doing when they're "slow." The symptoms look similar on the surface — high latency, slow throughput — but the root cause and fix are completely different.

**The mental model:**

A process does two types of work:
- **CPU work** — executing instructions: calculations, string processing, JSON parsing, crypto operations
- **I/O wait** — waiting for something external: disk reads, network calls, database queries

A CPU bottleneck means the process is compute-constrained — not enough CPU cycles. An I/O bottleneck means the process is waiting — CPU is idle but threads are blocked waiting for data.

---

**CPU bottleneck — characteristics and diagnosis:**

```
Symptoms:
- CPU usage is high (70–100% sustained)
- Latency is proportional to request complexity
- Scaling horizontally (more pods/instances) reduces latency
- Performance doesn't improve if you give the pod more memory
```

```bash
# On the host — see which processes are CPU-intensive
top -o %CPU

# Inside a container — see CPU usage
kubectl top pods -n prod

# Prometheus — CPU saturation
rate(container_cpu_usage_seconds_total{pod=~"my-app.*"}[5m])
```

**The hidden CPU bottleneck — throttling:**

Average CPU can look low (15%) but the pod can still be CPU-throttled if it needs short bursts. CFS throttling limits burst CPU usage per 100ms scheduling period.

```promql
# Percentage of CPU scheduling periods being throttled
rate(container_cpu_cfs_throttled_periods_total{pod=~"my-app.*"}[5m])
/
rate(container_cpu_cfs_periods_total{pod=~"my-app.*"}[5m])
```

If this is above 25%, the pod is CPU-throttled even though average usage looks low. Fix: remove the CPU limit (keep CPU request for scheduling priority, remove the limit to allow bursts).

**What causes CPU bottlenecks:**
- JSON serialization/deserialization at scale (common in Python)
- Cryptographic operations (TLS handshakes, JWT validation, bcrypt)
- GC pressure (Java/Go objects being allocated and collected at high rate)
- Inefficient algorithm (O(n²) instead of O(n log n))

**Fix:** More CPUs, more replicas (horizontal scaling), optimize the algorithm, reduce allocation rate to ease GC.

---

**I/O bottleneck — characteristics and diagnosis:**

```
Symptoms:
- CPU usage is LOW (5–20%) but latency is high
- Threads are blocked in "waiting" state, not "running"
- Adding more CPUs/replicas doesn't help much
- Latency correlates with disk or network activity
```

**Type 1: Disk I/O wait**

```bash
# CPU iowait — percentage of time CPU is idle waiting for I/O
# If this is high, processes are stuck waiting on disk
iostat -x 1    # Extended disk stats — look at %await and %util

# Prometheus
rate(node_disk_io_time_seconds_total{device="nvme0n1"}[5m])
rate(node_cpu_seconds_total{mode="iowait"}[5m])
```

High iowait + high disk read/write time = disk I/O bottleneck. Common causes: log-heavy applications writing to local disk, applications doing full-table scans, databases with poorly tuned buffer pools.

**Fix:** Move logs to a log aggregator (don't write to disk), use SSDs instead of spinning disks, add database read replicas, tune buffer pool size to cache hot data.

**Type 2: Network I/O (most common in microservices)**

```
App is waiting on:
- Database query response
- External API call
- Redis cache lookup
- Another service's HTTP response
```

```bash
# Check network latency to dependency from inside the pod
kubectl exec <pod> -n prod -- curl -w "%{time_total}s\n" -s -o /dev/null http://postgres:5432

# Distributed trace — which span is taking the most time?
# Open Jaeger/Tempo, find a slow trace, look at which span is the bottleneck
```

```promql
# Per-dependency latency (if you instrument outbound calls)
histogram_quantile(0.99, rate(http_client_request_duration_seconds_bucket{
  service="my-app", target="postgres-service"
}[5m]))
```

**Type 3: Connection pool exhaustion (looks like I/O bottleneck)**

10 connection pool slots, 50 concurrent requests. 40 threads block waiting for a free connection. CPU is idle, memory is stable, latency is high. The bottleneck isn't network speed — it's queue depth.

```promql
# HikariCP (Java) — pending threads waiting for a DB connection
hikaricp_connections_pending{pool="HikariPool-1"}
hikaricp_connections_acquire_seconds_sum / hikaricp_connections_acquire_seconds_count
```

If pending > 0 or acquire time is high, increase pool size or optimize queries to reduce connection hold time.

---

**The decision tree:**

```
High latency, investigate:
├── CPU usage high (>70%)? → CPU bottleneck
│   └── CPU average low but throttled? → Remove CPU limit
│
└── CPU usage low (<30%)?
    ├── Disk iowait high? → Disk I/O bottleneck
    │   └── Move logs off disk, tune buffer pools
    ├── Network latency high to dependency? → Network I/O bottleneck
    │   └── Check traces for slow spans
    └── Connection pool pending > 0? → Pool exhaustion
        └── Increase pool size, optimize query duration
```

**Real scenario:** A Java service showed 18% CPU, 40% memory, but p99 latency was 2.8 seconds. Classic "CPU and memory look fine, latency is high." Traces showed the slow span was on the database call — but the query execution time in PostgreSQL was 8ms. The slow part was `time_to_acquire_connection`: 2.7 seconds. The pool had `max_connections=5` — set when the service handled 10 req/sec in staging. Production load was 200 req/sec. 195 threads were queuing for 5 connections. Increasing pool to 30 dropped latency to 90ms immediately.

---

## 2. Why do distributed systems fail when CPU and memory look completely normal?

This is a conceptual question that separates engineers who've operated distributed systems from those who've only read about them.

**The core insight:** CPU and memory are compute-plane metrics. Distributed systems fail for reasons that never show up in compute metrics.

**1. Network partition or intermittent packet loss.**

Two services can't reach each other. CPU is idle (waiting), memory is stable (no data being processed). The only signal is: timeouts in logs, request latency spikes, connection refused errors.

CPU usage of 5% tells you nothing about whether the network path between pod A and pod B has 40% packet loss.

**2. Downstream service degradation (not failure).**

Your service is fine. The database is "up" but taking 3 seconds per query instead of 5ms. Your service threads are waiting on the DB response — CPU is idle, memory is stable. But every user request takes 3+ seconds.

```promql
# What you need — per-dependency latency
histogram_quantile(0.99, rate(db_query_duration_seconds_bucket{service="postgres"}[5m]))

# What you're probably looking at — doesn't tell you this
container_cpu_usage_seconds_total
container_memory_usage_bytes
```

**3. Thread pool / connection pool exhaustion.**

All pool slots are occupied. New requests queue indefinitely. CPU: idle (threads are waiting, not computing). Memory: stable. But every new request is stuck in a queue.

**4. DNS resolution failures.**

`ndots:5` in Kubernetes means every outbound call triggers 5 DNS lookups before resolving. If CoreDNS is overwhelmed (common under load), DNS takes 500ms per lookup. Your app makes 4 outbound calls per request = 2 seconds of pure DNS overhead. CPU: idle (waiting on DNS). Memory: unaffected.

```bash
# Check CoreDNS latency
kubectl exec <pod> -- time nslookup postgres.prod.svc.cluster.local
# Should be <5ms. If it's 200ms+, CoreDNS is your problem
```

**5. Distributed deadlock / cascading timeouts.**

Service A calls Service B. Service B is slow → A's timeout fires → A returns an error. But A's thread holds a database connection while waiting for B to timeout. 50 concurrent requests × 30 second timeout = 50 DB connections held for 30 seconds each. Pool exhausts. New requests to A start failing immediately — not because of anything in A's code, but because of B's slowness and A's timeout configuration.

CPU: A is mostly idle. Memory: stable. But A is completely unable to serve new requests.

**6. Clock skew causing JWT/certificate validation failures.**

Two services have clocks 5 minutes apart. Service A generates a JWT with `iat: now`. Service B validates it: `iat` is 5 minutes in the future → rejects as invalid. 100% of requests fail. CPU on both services: minimal. Memory: stable.

```bash
# Check time sync status on nodes
kubectl debug -it <node> --image=busybox -- date
# Should match within a few seconds across nodes
```

**7. Ephemeral storage exhaustion on a node.**

A service writes excessive logs or temp files. The node's `/var/lib/docker` partition fills up. Docker can't write new container layers. New containers fail to start. Existing containers start failing writes. CPU: normal. Memory: normal. Disk: 100%.

```promql
# Node disk usage — not container CPU/memory
(node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"})
/
node_filesystem_size_bytes{mountpoint="/"}
```

**The engineering discipline that comes from this:**

Stop treating CPU and memory dashboards as the definition of "healthy." Build observability for the actual failure modes: per-dependency latency, connection pool metrics, DNS resolution time, disk usage on nodes, error rates broken down by downstream service. CPU and memory tell you about your application's compute. They don't tell you about the network, dependencies, or infrastructure around it.

---

## 3. When should you avoid horizontal autoscaling, and when is vertical scaling the right answer?

Autoscaling sounds universally good — scale out when load increases, scale in when it drops. But it's not always the right answer. Knowing when NOT to autoscale, and when to scale vertically instead, is the more sophisticated answer.

**When horizontal autoscaling (HPA) works well:**

- Stateless services where each replica is independent
- Traffic is variable (spiky during business hours, low overnight)
- Request processing is CPU or memory-bound and scales linearly with replicas
- Startup time is fast (new pods ready in < 30 seconds)

**When horizontal autoscaling creates problems:**

**1. Stateful services where data locality matters.**

If your service has per-instance in-memory state (local caches, connection pools to specific databases, session state), adding replicas doesn't distribute load evenly — some replicas have warm caches, others don't. New replicas start cold, see high latency, HPA adds more cold replicas, cache hit rate drops further. Classic autoscaling death spiral.

**2. Slow startup time — scale-out too slow to help.**

Java Spring Boot services can take 45–90 seconds to start. If a traffic spike hits and HPA triggers scale-out, new pods won't be ready for 90 seconds. By then, the spike might be over — or existing pods have already crashed under load. Autoscaling is a lagging indicator solution.

Fix for slow startup: Vertical Pod Autoscaler (VPA) or pre-provisioned buffer replicas (`minReplicas` higher than base load).

**3. HPA based on CPU can be wrong metric for I/O-bound services.**

A service that's bottlenecked on database connections won't have high CPU. HPA configured on CPU won't scale it out when it needs it. Requests timeout not because of CPU saturation, but because the connection pool is exhausted.

```yaml
# HPA on CPU — wrong for I/O-bound services
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        averageUtilization: 70

# Better — custom metric on actual saturation signal
metrics:
  - type: External
    external:
      metric:
        name: db_connections_pending
      target:
        value: "5"    # Scale when more than 5 threads waiting for DB connections
```

**4. Base cost: minimum replicas must cover worst-case failure.**

If you run 3 replicas with `maxUnavailable: 1`, and one node fails, 2 replicas share 100% of traffic. If 2 replicas can't handle full traffic, you need `minReplicas: 3` at all times — which means you're paying for 3 regardless of load. The "save money by scaling to 1 at night" idea only works if 1 replica can safely absorb any traffic burst during the scale-out window.

**When vertical scaling is the right answer:**

**Single-threaded or poorly multi-threaded applications.** A process that can only use 1 CPU core no matter how many replicas you run (Python GIL for CPU-bound work, single-threaded event loop with blocking calls). More replicas means more processes, but each process is still bottlenecked on 1 core. Vertical scaling (bigger instance for the process) may be more cost-effective than running 10 small replicas.

**Memory-intensive workloads with large working sets.** Machine learning inference services loading a 4GB model into memory. You can't split the model across replicas — each replica needs the full model. Horizontal scaling means paying for 4GB × N replicas. A single large instance with enough memory + more goroutines/threads might be more efficient.

**Legacy applications not designed for horizontal scale.** Applications with assumptions of being singletons (file locking, single-node cron jobs, applications that assume they're the only writer to a resource). Horizontal scaling breaks these. Vertical scaling (more RAM, more CPU on one instance) extends their life without refactoring.

**VPA (Vertical Pod Autoscaler) for right-sizing:**

We use VPA in "recommendation" mode (not automatic) to right-size resources:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Off"    # Recommendation only — don't auto-apply
```

VPA watches actual CPU and memory usage over time and recommends resource requests/limits. We review recommendations monthly and apply them manually. This prevents pods from being over-provisioned (wasting money) or under-provisioned (getting OOMKilled or throttled).

**Real scenario:** We had an HPA configured on CPU for our PDF generation service. PDF generation is CPU-intensive but each job takes 30 seconds. A burst of 50 PDF requests triggered HPA scale-out. New pods spun up in 90 seconds — by then, the burst was handled by the original pods (barely). The new pods sat idle for 10 minutes, then HPA scaled back in. The autoscaling added no benefit and cost $12 in unnecessary pod-minutes. We replaced HPA with a static `replicas: 5` (based on VPA recommendations for peak load) and a queue-depth-based HPA using the SQS queue length as the metric. Now scale-out happens based on actual queue depth, not lagging CPU metrics.

---

## 4. How do you define SLI, SLO, and SLA in production, and how do you calculate error budget?

SLI/SLO/SLA is the reliability engineering framework that turns "the system should be reliable" into a measurable, actionable target.

**SLI (Service Level Indicator) — the actual measurement.**

An SLI is a metric that measures one aspect of service quality. The most useful SLIs follow the RED method:

- **Request rate** — how many requests per second
- **Error rate** — what fraction of requests fail
- **Duration** — how long requests take (latency)

```promql
# SLI: proportion of successful HTTP requests (success rate)
sum(rate(http_requests_total{service="checkout", status!~"5.."}[5m]))
/
sum(rate(http_requests_total{service="checkout"}[5m]))

# SLI: proportion of requests completing within 500ms
sum(rate(http_request_duration_seconds_bucket{service="checkout", le="0.5"}[5m]))
/
sum(rate(http_request_duration_seconds_count{service="checkout"}[5m]))
```

Good SLIs are: user-facing (measures what users experience), measurable (not "team feels good"), proportional (fraction, not absolute count).

**SLO (Service Level Objective) — the target.**

An SLO is the target value for an SLI, measured over a time window. It's the internal commitment the engineering team makes.

```
SLO: checkout service success rate ≥ 99.9% over a rolling 30-day window
SLO: 95% of checkout requests complete within 500ms over a rolling 30-day window
```

The SLO is your internal bar. It should be set below 100% — targeting 100% availability means every incident is a breach, and the team burns out constantly firefighting.

**SLA (Service Level Agreement) — the external promise.**

An SLA is a contract with customers. It's usually set lower than the SLO to give engineering headroom:

```
SLA: 99.5% availability per month (if breached, customers get credits)
SLO: 99.9% availability (internal target — gives 4x headroom above SLA)
```

If your SLO is 99.9% and your SLA is 99.5%, you have a 0.4% buffer. Even if you miss your SLO by a little, you don't breach the customer SLA.

**Error budget — the math:**

Error budget = the amount of downtime/errors allowed before the SLO is breached.

```
SLO: 99.9% success rate over 30 days
Total requests in 30 days: 10,000,000

Allowed failures = 10,000,000 × (1 - 0.999) = 10,000 failures

If you've had 7,000 failures so far this month → 3,000 failures remaining in the budget
```

For availability:

```
SLO: 99.9% uptime over 30 days
Total minutes in 30 days: 30 × 24 × 60 = 43,200 minutes

Error budget = 43,200 × (1 - 0.999) = 43.2 minutes of downtime allowed per month
```

99.9% SLO = ~43 minutes of downtime per month. 99.99% SLO = ~4.3 minutes per month. 100% SLO = zero tolerance for any downtime — not realistic.

**How we track error budget in Prometheus:**

```promql
# Remaining error budget as a percentage
(
  sum(rate(http_requests_total{service="checkout", status!~"5.."}[30d]))
  /
  sum(rate(http_requests_total{service="checkout"}[30d]))
  - 0.999    # SLO threshold
)
/ (1 - 0.999)   # Maximum allowed error rate
* 100
```

If this returns 60%, you have 60% of your error budget remaining for the month.

**Error budget policies — what to do with the budget:**

```
Error budget remaining > 50%:  Normal velocity. Ship features, take risks.
Error budget remaining 25–50%: Caution. Risky deployments need extra review.
Error budget remaining 0–25%:  Feature freeze. Focus on reliability work.
Error budget exhausted (0%):   Hard feature freeze. All hands on reliability.
                                No new deployments until budget resets.
```

This is the key SRE insight: error budget converts "reliability vs velocity" from a negotiation into math. When budget is healthy, engineering can deploy frequently. When budget is low, the numbers justify slowing down — no argument needed with product.

**Real scenario:** Our payment service had an SLO of 99.95%. In month 3, two incidents consumed 80% of the error budget by day 15. Under the error budget policy, we declared a feature freeze for the payment team for the remaining 15 days. The team did only reliability work: fixed root causes, added runbooks, improved alerting. The month ended with 0 additional incidents. Following month: full budget available, team resumed normal feature velocity. The budget policy made the decision objective — not a manager overruling an engineer, but a shared metric.

---

## 5. How do you design monitoring and alerting for a microservices platform?

The wrong approach: add a Prometheus alert for every metric you have. The result: hundreds of alerts, 90% of which are noise. Engineers start ignoring pages. The right approach: alert on symptoms, not causes, and design for actionability.

**The monitoring stack we use:**

```
Metrics:  Prometheus (scrape) → Thanos (long-term storage, HA) → Grafana (dashboards)
Logs:     Fluent Bit (collect) → Loki → Grafana (query)
Traces:   OpenTelemetry SDK → Tempo → Grafana (trace view)
Alerts:   Alertmanager → PagerDuty (Sev-1/2) → Slack (Sev-3/4)
```

**What to instrument in each service:**

Every service exposes a `/metrics` endpoint with these RED metrics:

```python
# Python — using prometheus_client
from prometheus_client import Counter, Histogram

REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status_code', 'service']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint', 'service'],
    buckets=[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5]
)
```

Every service has a standard Grafana dashboard (generated from a template):
- Request rate (req/s)
- Error rate (%)
- p50, p95, p99 latency
- Pod count and restarts
- CPU and memory usage

**Alerting philosophy — symptoms over causes:**

```
Wrong alert: "CPU is above 70%"
  → Fires constantly, CPU high isn't always a user problem

Right alert: "Error rate is above 1% for 5 minutes"
  → Always means users are experiencing failures

Wrong alert: "Pod restart count increased"
  → A single restart might be normal (OOMKill on low-traffic pod)

Right alert: "Pod is restarting more than 5 times in 10 minutes"
  → Users are definitely affected (CrashLoopBackOff)
```

**Our alert tiers:**

```yaml
# Sev-1: Pages on-call immediately (PagerDuty)
- alert: ServiceDown
  expr: sum(up{job="order-service"}) == 0
  for: 1m
  labels:
    severity: critical

- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m])) > 0.05
  for: 5m
  labels:
    severity: critical

# Sev-2: Pages on-call, can wait until acknowledged (< 30 min)
- alert: ElevatedErrorRate
  expr: |
    sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m])) > 0.01
  for: 10m
  labels:
    severity: warning

- alert: HighLatency
  expr: |
    histogram_quantile(0.99,
      rate(http_request_duration_seconds_bucket[5m])
    ) > 2
  for: 10m
  labels:
    severity: warning

# Sev-3: Slack notification, no page
- alert: ErrorBudgetBurning
  expr: error_budget_remaining < 0.25
  labels:
    severity: info
```

**Multi-window, multi-burn-rate alerts (Google SRE book approach):**

Standard alerts have a lag — error rate must be elevated for `for: 5m` before alerting. If your error budget burns fast (a critical outage), you want to alert faster. Multi-burn-rate alerts detect both fast burns and slow burns:

```yaml
# Fast burn: 14x normal burn rate for 1 hour → alert immediately
- alert: ErrorBudgetFastBurn
  expr: |
    (
      job:slo_errors:ratio1h{service="checkout"} > 14 * (1 - 0.999)
      and
      job:slo_errors:ratio5m{service="checkout"} > 14 * (1 - 0.999)
    )
  for: 2m

# Slow burn: 2x normal burn rate for 6 hours → alert but not urgent
- alert: ErrorBudgetSlowBurn
  expr: |
    job:slo_errors:ratio6h{service="checkout"} > 2 * (1 - 0.999)
  for: 60m
```

**The synthetic monitor — the most important alert we have:**

All internal metrics can be green while users experience failures (NetworkPolicy blocking ingress, CDN misconfiguration, DNS issue). Prometheus Blackbox Exporter hits the real public URL every 30 seconds:

```yaml
# Blackbox exporter job
- job_name: blackbox_https
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://api.yourapp.com/health
        - https://api.yourapp.com/checkout/health
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: blackbox-exporter:9115

# Alert on external probe failure
- alert: ExternalEndpointDown
  expr: probe_success{job="blackbox_https"} == 0
  for: 2m
  labels:
    severity: critical
```

This is the last line of defense — if all pod metrics are green but `probe_success` is 0, something in the network path is broken.

**Real scenario:** We started with 47 Prometheus alerts across 18 services. Average of 8 alert pages per day. Engineers were burned out. We ran a 2-month alert audit: for each alert, did it fire in the last 2 months? Was it actionable? Was it a symptom or a cause? Result: deleted 31 alerts, modified 12 to be symptom-based with better thresholds, kept 4. After the audit: 1–2 alert pages per day, all actionable, response quality improved dramatically. Alert fatigue is as dangerous as missing alerts — both result in the real problems being ignored.

---

## 6. What is the difference between reactive and proactive reliability engineering?

Most teams do reactive reliability: something breaks, they fix it. Proactive reliability is engineering the system so failures are less likely and recovery is faster when they do happen.

**Reactive reliability — responding to what's already broken:**

```
Pattern:
  Incident happens → alert fires → on-call engineer wakes up
  → diagnoses → fixes → postmortem → maybe adds a runbook
  → repeat next month with same class of issue
```

Reactive is necessary — you can't eliminate all incidents. But if 80% of your engineering time is reactive, you're in a firefighting loop with no time to break it.

**Proactive reliability — engineering for resilience before things break:**

**1. Chaos engineering — deliberately break things in controlled conditions.**

```bash
# Chaos Mesh (Kubernetes) — inject failures to test system resilience
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: delay-payment-service
spec:
  action: delay
  mode: one
  selector:
    namespaces: [prod]
    labelSelectors:
      app: payment-service
  delay:
    latency: "500ms"
    jitter: "100ms"
  duration: "5m"
```

Run this during business hours (controlled), watch dashboards. Does the upstream service time out gracefully? Do circuit breakers trip? Does the retry logic work? You find out now, not at 3 AM.

**2. Game Days — practice incident response before a real incident.**

Inject a failure scenario into staging with the on-call team present. "Your database just became read-only. What do you do?" The team runs the playbook, discovers it's outdated, updates it. The next real incident where the database goes read-only: team executes confidently in 5 minutes instead of 40.

**3. SLO-driven development — reliability work is planned, not reactive.**

Every quarter, we review error budget consumption by service:

```
service          | SLO   | budget consumed this quarter
checkout         | 99.9% | 23%   → healthy
payment          | 99.95%| 87%   → reliability work needed
auth             | 99.99%| 12%   → healthy
notification     | 99.5% | 3%    → healthy
```

Payment service consumed 87% of budget → it gets 20% of next sprint's engineering capacity for reliability work: fixing root causes of past incidents, adding circuit breakers, improving runbooks. This is planned work, not reactive firefighting.

**4. Production readiness reviews — before shipping, not after.**

Before a new service goes to production, it passes a checklist:
- Does it have health check endpoints (readiness + liveness)?
- Does it expose RED metrics?
- Is there a Grafana dashboard?
- Does it have an on-call runbook?
- Has it been load tested at 2x expected production traffic?
- Does it have circuit breakers for all downstream calls?
- Are retries implemented with jitter to prevent thundering herd?

If the answer to any of these is no, the service doesn't go to production. Not as a blocker — as a conversation about accepted risk.

**5. Toil reduction — automate the manual work that's eating on-call time.**

Toil is manual, repetitive operational work that could be automated. Runbooks that say "SSH to the server and restart the process" → automate it. Manual certificate renewals → automate with cert-manager. Manual scaling → automate with HPA/Karpenter.

We track toil hours per sprint. If an on-call engineer spends > 20% of sprint time on toil, we create a reliability ticket to automate it. Reducing toil compounds: each automation frees time for the next reliability improvement.

**The maturity progression:**

```
Level 1 (Reactive):    Alert fires → fix it → hope it doesn't happen again
Level 2 (Postmortems): Fix it → write postmortem → add action items → some get done
Level 3 (SLOs):        Track error budget → planned reliability work → fewer incidents
Level 4 (Proactive):   Chaos engineering + game days + PRR + toil reduction
                        → reliability is engineered, not hoped for
```

Most teams are at Level 1 or 2. Level 3 requires organizational buy-in (product must accept that error budget controls feature velocity). Level 4 requires engineering maturity and dedicated SRE capacity. You can start Level 3 tomorrow with SLOs and Prometheus — no organizational change needed.
