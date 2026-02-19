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
