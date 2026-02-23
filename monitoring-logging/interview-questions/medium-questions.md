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

---

## 2. How do you configure an alert when a server's disk usage reaches 80% and send an SMS notification?

This requires three components working together: **Node Exporter** (collect disk metrics) → **Prometheus** (evaluate the threshold) → **Alertmanager** (route and send SMS).

**Step 1: Ensure Node Exporter is running on the target server.**

Node Exporter exposes Linux system metrics (disk, CPU, memory) on port 9100. Prometheus scrapes it.

```bash
# Install and start Node Exporter on the target server
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xvf node_exporter-*.tar.gz
sudo cp node_exporter-*/node_exporter /usr/local/bin/
sudo systemctl start node_exporter
# Exposes metrics at http://<server-ip>:9100/metrics
```

**Step 2: Write the alerting rule in Prometheus.**

```yaml
# /etc/prometheus/alert_rules.yml
groups:
  - name: disk-alerts
    rules:
      - alert: DiskUsageHigh
        # Disk used percentage across all non-tmpfs mountpoints
        expr: |
          (
            node_filesystem_size_bytes{fstype!~"tmpfs|overlay"} -
            node_filesystem_free_bytes{fstype!~"tmpfs|overlay"}
          ) / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"} * 100 > 80
        for: 5m    # Must stay above 80% for 5 consecutive minutes before alerting
        labels:
          severity: warning
          team: infrastructure
        annotations:
          summary: "High disk usage on {{ $labels.instance }}"
          description: "Disk {{ $labels.mountpoint }} on {{ $labels.instance }} is {{ $value | printf \"%.1f\" }}% full."
```

Load this rule file in `prometheus.yml`:

```yaml
rule_files:
  - "alert_rules.yml"
```

Reload Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

**Step 3: Configure Alertmanager to send SMS.**

Alertmanager sends SMS via an external provider. Common options:
- **AWS SNS** (Simple Notification Service) — supports SMS natively
- **Twilio** — direct SMS API
- **PagerDuty** — routes to SMS/call based on escalation policy

**Using AWS SNS for SMS:**

```yaml
# /etc/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: "default"
  routes:
    - match:
        severity: warning
        team: infrastructure
      receiver: "sms-via-sns"

receivers:
  - name: "default"
    slack_configs:
      - api_url: "https://hooks.slack.com/..."
        channel: "#alerts"

  - name: "sms-via-sns"
    sns_configs:
      - api_url: "https://sns.ap-south-1.amazonaws.com"
        topic_arn: "arn:aws:sns:ap-south-1:123456:disk-alerts-sms"
        sigv4:
          region: ap-south-1
          access_key: "<aws-access-key>"
          secret_key: "<aws-secret-key>"
        message: |
          ALERT: {{ .GroupLabels.alertname }}
          {{ range .Alerts }}
          Server: {{ .Labels.instance }}
          Disk: {{ .Annotations.description }}
          {{ end }}
```

**AWS SNS setup for SMS delivery:**

```bash
# Create SNS topic
aws sns create-topic --name disk-alerts-sms

# Subscribe a phone number to the topic
aws sns subscribe \
  --topic-arn arn:aws:sns:ap-south-1:123456:disk-alerts-sms \
  --protocol sms \
  --notification-endpoint "+919876543210"

# Confirm the subscription (SNS sends a confirmation SMS)
```

**Using Twilio (alternative — more reliable for production SMS):**

Alertmanager doesn't have a native Twilio receiver, but you can use a **webhook** receiver that calls a small Lambda or webhook service that forwards to Twilio:

```yaml
receivers:
  - name: "sms-twilio"
    webhook_configs:
      - url: "https://your-lambda-or-service/alert-webhook"
        send_resolved: true
```

The webhook service receives the Alertmanager JSON payload and calls the Twilio API to send SMS.

**The full alert flow:**

```
Node Exporter (server:9100/metrics)
    ↓ scrape every 15s
Prometheus evaluates: disk > 80% for 5 minutes?
    ↓ if yes → sends alert to Alertmanager
Alertmanager deduplicates, groups, and routes
    ↓ matches "sms-via-sns" route
AWS SNS topic
    ↓ delivers to subscribed phone numbers
SMS received: "ALERT: DiskUsageHigh — Server: web-01, Disk /dev/xvda1 is 83.2% full"
```

**Testing the alert without waiting for real disk usage:**

```bash
# Temporarily lower the threshold to trigger in your lab
# Or test Alertmanager directly:
curl -X POST http://localhost:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {"alertname": "DiskUsageHigh", "instance": "web-01:9100", "severity": "warning"},
    "annotations": {"description": "Disk / on web-01:9100 is 81.0% full."}
  }]'
# This fires a test alert and should trigger the SMS
```

**Real scenario:** Alert fired — checkout conversion dropped by 8%.

---

## 3. How do you create a monitoring dashboard for your organisation?

Building a monitoring dashboard isn't just adding graphs — it's asking "what questions does each team need answered?" and structuring dashboards accordingly. A dashboard no one looks at is wasted effort.

**Step 1: Define the audience and questions before touching Grafana.**

Different stakeholders need different views:

| Audience | Questions they need answered |
|---|---|
| On-call engineer | Is anything broken right now? What's the error rate? Which service? |
| Engineering manager | What's our reliability this week? SLO burn rate? Recent deployments? |
| Business / Product | How many users active? Orders per minute? Conversion rate? |

**Step 2: Start with the Four Golden Signals (Google SRE approach).**

Every service dashboard should show these four metrics:

```
1. Latency       — how long requests take (p50, p95, p99)
2. Traffic       — how many requests/second
3. Errors        — error rate (5xx / total requests × 100)
4. Saturation    — how "full" is the service (CPU %, memory %, queue depth)
```

In Grafana + Prometheus:

```promql
# Traffic — requests per second
rate(http_requests_total{service="order-service"}[5m])

# Error rate — percentage of 5xx
rate(http_requests_total{service="order-service", status=~"5.."}[5m])
  /
rate(http_requests_total{service="order-service"}[5m]) * 100

# Latency — 95th percentile response time
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{service="order-service"}[5m]))

# Saturation — CPU utilisation
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Step 3: Layer dashboards — overview → service → infrastructure.**

```
Layer 1: Platform Overview dashboard
  → All services: error rate, latency, traffic — one row per service
  → Instantly see which service has elevated error rate
  → Link to individual service dashboard

Layer 2: Per-service dashboard (e.g., Order Service)
  → Four golden signals for this service
  → Downstream dependencies (DB query time, Redis cache hit rate)
  → Recent deployments annotated on the timeline

Layer 3: Infrastructure dashboard
  → Node CPU, memory, disk, network per EKS node
  → Pod count, restarts, OOMKills
  → RDS connections, replication lag, slow query count
```

**Step 4: Add deployment annotations.**

Every deployment should annotate the dashboard timeline so engineers can see "the error rate spiked right after this deployment":

```bash
# From your CI/CD pipeline, post an annotation to Grafana after each deployment
curl -X POST http://grafana.internal/api/annotations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -d '{
    "time": '"$(date +%s000)"',
    "tags": ["deployment", "order-service"],
    "text": "Deployed order-service v2.4.1 (commit: '"$GIT_COMMIT"')"
  }'
```

**Step 5: Set alert thresholds based on SLOs, not gut feel.**

```
SLO: 99.5% of requests succeed (0.5% error budget)
Alert at: error rate > 1% for 5 minutes (consuming budget twice as fast as allowed)

SLO: p99 latency < 500ms
Alert at: p99 > 800ms for 5 minutes
```

Alerts should be based on user impact, not raw metrics. An alert for "CPU > 70%" with no user impact creates alert fatigue. An alert for "SLO burn rate exceeds 2x" is always meaningful.

**Step 6: Grafana practical setup.**

```yaml
# grafana-dashboard-configmap.yaml — provision dashboards as code
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  order-service.json: |
    {
      "title": "Order Service",
      "panels": [...]
    }
```

Store dashboards as JSON in Git. Use `grafana-dashboard-provisioner` to load them on startup. This way, dashboards are version-controlled and recreated automatically if Grafana is redeployed. Metrics showed payment service error rate was 0% (no errors). Logs showed payment service was completing successfully. Traces showed: every checkout request was making 4 sequential calls to the inventory service (N+1 query pattern introduced in the last deploy). Each call was fast (30ms), but 4 × 30ms = 120ms added to every checkout. Users were abandoning at the longer load time without errors being thrown. We'd have never found this from metrics or logs alone — it required traces to see the N+1 pattern across service calls.
