# Monitoring Logging — Medium Questions

---

## 1. What is the difference between metrics, logs, and traces — and when do you start with each?

These are the three pillars of observability. Most people can define them; the real answer is knowing which one to reach for first in different scenarios.

**Metrics — numbers over time.**

Metrics are numeric measurements sampled at intervals: CPU usage, request count, error rate, latency percentiles, memory usage. They're aggregated and efficient to store.

```promql
# Prometheus — request rate, error rate, latency (RED metrics)
rate(http_requests_total{service="order-service"}[5m])
rate(http_requests_total{service="order-service", status=~"5.."}[5m])
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

**Logs — text events with timestamps.**

Logs record what happened: "user 123 placed order 456", "database connection failed", "processed 1000 records in 5.2s". They're verbose and expensive to store at scale.

```bash
# Structured log (JSON) — queryable in Loki/CloudWatch
{"timestamp": "2024-01-15T14:32:01Z", "level": "ERROR",
 "service": "order-service", "user_id": "123",
 "message": "payment processor timeout", "duration_ms": 30000}
```

**Traces — the path of a request across services.**

A trace records the journey of one specific request through your entire system — how long each service took, which downstream calls happened, where time was spent. Each hop is a "span."

```
Trace ID: abc123
├── API Gateway         5ms
├── order-service       150ms
│   ├── auth check      12ms (external call)
│   ├── inventory check 8ms
│   └── DB: insert_order 120ms  ← bottleneck
└── notification-service 25ms
```

---

**When to start with each:**

**Start with metrics** when:
- An alert fires — metrics tell you *what* is wrong at a glance (error rate spike, latency up, CPU high)
- You're doing capacity planning or trend analysis
- You need to know the scope of a problem (is it affecting 1% or 80% of requests?)

```
Alert fires: p99 latency > 2s on order-service
→ Open Grafana → check metrics dashboard
→ See: latency spike started 4 minutes ago, correlates with a deployment
→ Scope: all users affected
→ Metrics answered "what" and "when"
```

**Start with logs** when:
- You know *what* is broken (metrics) but need to know *why* for a specific error
- You need to investigate a specific user-reported issue ("my order ID 456 failed")
- You're looking for error messages, stack traces, or context around a specific event

```
Metrics show: 3% of order requests returning 500
→ Query Loki: level=ERROR, service=order-service, last 10 minutes
→ See: "NullPointerException at PaymentProcessor.java:145"
→ Logs answered "why" and "where in code"
```

**Start with traces** when:
- Latency is high but the cause isn't obvious from per-service metrics — the problem might be a specific downstream call
- You're debugging a distributed request that involves 3+ services
- You want to understand where time is spent across service boundaries

```
Latency high for checkout flow — but all individual service metrics look fine
→ Open Jaeger/Tempo → find a high-latency trace
→ See: payment-service call to fraud-check API takes 1800ms
→ fraud-check API is the bottleneck — metrics for individual services missed it
→ Traces answered "which part of the distributed flow is slow"
```

**In practice — the investigation sequence:**

```
Alert fires
    ↓
Metrics: what is broken, how bad is it, since when?
    ↓
Logs: why is it failing, what error, which user/request?
    ↓
Traces: which service, which span, which downstream call?
```

We don't start with traces for every alert — trace query is expensive. Metrics and logs often resolve the issue first. Traces are the "last mile" tool when the problem is in the interaction between services, not within one service.

**Real scenario:** Alert fired — checkout conversion dropped by 8%. Metrics showed payment service error rate was 0% (no errors). Logs showed payment service was completing successfully. Traces showed: every checkout request was making 4 sequential calls to the inventory service (N+1 query pattern introduced in the last deploy). Each call was fast (30ms), but 4 × 30ms = 120ms added to every checkout. Users were abandoning at the longer load time without errors being thrown. We'd have never found this from metrics or logs alone — it required traces to see the N+1 pattern across service calls.
