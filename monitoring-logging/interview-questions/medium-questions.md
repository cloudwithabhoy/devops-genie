# Monitoring Logging — Medium Questions

---

## 1. What is the difference between metrics, logs, and traces — and when do you start with each?

> **Also asked as:** "Can you explain logs, metrics, and traces?"
> **Also asked as:** "Explain the monitoring tools you used and difference in metrices, logs and traces ?"

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

> **Also asked as:** "I want to configure alerting — disk usage of a server reaches 80%, it should alert SMS. Where do I do this configuration?" — covered above (Node Exporter → Prometheus alert rule → Alertmanager → AWS SNS / Twilio).

---

## 3. How do you create a monitoring dashboard for your organisation?

> **Also asked as:** "How do you create a monitoring dashboard?"

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

---

## 4. How do you implement Kubernetes cluster-level monitoring using Prometheus?

Kubernetes cluster monitoring has two layers: **infrastructure metrics** (nodes, pods, containers) and **workload metrics** (application-level RED signals). Prometheus covers both with the right exporters and scrape configuration.

**The standard stack: kube-prometheus-stack.**

The easiest production setup is the `kube-prometheus-stack` Helm chart, which bundles Prometheus, Alertmanager, Grafana, Node Exporter, kube-state-metrics, and pre-built dashboards:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values prometheus-values.yaml
```

**What gets installed automatically:**

```
kube-prometheus-stack
├── prometheus-operator        ← manages Prometheus config via CRDs
├── prometheus                 ← metrics storage and query engine
├── alertmanager               ← alert routing and deduplication
├── grafana                    ← dashboards and visualisation
├── node-exporter (DaemonSet)  ← host metrics from every node
└── kube-state-metrics         ← K8s object state (pod counts, deployment status)
```

**Key metrics each component provides:**

```
Node Exporter (per node):
  node_cpu_seconds_total          → CPU usage per core per mode
  node_memory_MemAvailable_bytes  → available memory
  node_filesystem_avail_bytes     → disk space per mount
  node_network_receive_bytes_total → network throughput

kube-state-metrics (K8s objects):
  kube_pod_status_phase           → how many pods in Running/Pending/Failed state
  kube_deployment_status_replicas_available → available vs desired replicas
  kube_pod_container_resource_limits → configured CPU/memory limits
  kube_node_status_condition      → node Ready/NotReady/MemoryPressure

cAdvisor (container-level, built into kubelet):
  container_cpu_usage_seconds_total
  container_memory_working_set_bytes
  container_fs_usage_bytes
```

**Custom prometheus-values.yaml for production:**

```yaml
# prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d              # Keep 30 days of metrics
    retentionSize: "50GB"       # Or until 50GB is reached (whichever first)
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 100Gi   # Adjust based on retention and cluster size

    # Scrape all ServiceMonitors across all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 10Gi

grafana:
  adminPassword: "change-me-use-secret"
  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: default
          orgId: 1
          folder: "Kubernetes"
          type: file
          disableDeletion: false
          options:
            path: /var/lib/grafana/dashboards/default
```

**Scraping application metrics with ServiceMonitor.**

Applications expose metrics on `/metrics` (Prometheus format). A `ServiceMonitor` tells Prometheus where to scrape:

```yaml
# serviceMonitor for your application
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: order-service
  namespace: prod
  labels:
    release: kube-prometheus-stack   # Must match Prometheus' serviceMonitorSelector
spec:
  selector:
    matchLabels:
      app: order-service
  endpoints:
    - port: metrics      # The named port in your Service exposing /metrics
      interval: 30s
      path: /metrics
```

For pods without a Service (batch jobs, etc.), use `PodMonitor`:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: batch-worker
  namespace: prod
spec:
  selector:
    matchLabels:
      app: batch-worker
  podMetricsEndpoints:
    - port: metrics
      interval: 60s
```

**Essential cluster-level alerts.**

```yaml
# PrometheusRule — alert rules managed as K8s objects
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cluster-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: node-health
      rules:
        - alert: NodeNotReady
          expr: kube_node_status_condition{condition="Ready",status="true"} == 0
          for: 5m
          labels: { severity: critical }
          annotations:
            summary: "Node {{ $labels.node }} is not ready"

        - alert: NodeMemoryPressure
          expr: kube_node_status_condition{condition="MemoryPressure",status="true"} == 1
          for: 2m
          labels: { severity: warning }

        - alert: NodeDiskPressure
          expr: kube_node_status_condition{condition="DiskPressure",status="true"} == 1
          for: 2m
          labels: { severity: warning }

    - name: workload-health
      rules:
        - alert: DeploymentReplicasMismatch
          expr: |
            kube_deployment_status_replicas_available
            < kube_deployment_spec_replicas
          for: 10m
          labels: { severity: warning }
          annotations:
            summary: "Deployment {{ $labels.deployment }} has fewer replicas than desired"

        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total[1h]) > 5
          for: 5m
          labels: { severity: warning }
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"

        - alert: PodOOMKilled
          expr: |
            kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
          for: 0m
          labels: { severity: warning }
```

**Grafana dashboards — pre-built for K8s.**

kube-prometheus-stack includes the Kubernetes Mixin dashboards automatically. Access them at:
- `Kubernetes / Nodes` — per-node CPU, memory, disk, network
- `Kubernetes / Pods` — per-pod resource usage and restarts
- `Kubernetes / Workloads` — deployments, DaemonSets, StatefulSets health

**Scaling Prometheus for large clusters:**

For clusters with 100+ nodes and 1000+ pods, single Prometheus can struggle:

```yaml
# Option 1: Thanos sidecar — long-term storage + global queries across multiple Prometheus
prometheus:
  prometheusSpec:
    thanos:
      objectStorageConfig:
        key: thanos.yaml
        name: thanos-objstore-config
      # Thanos sidecar uploads metrics blocks to S3 for long-term retention

# Option 2: Prometheus Operator with sharding
prometheus:
  prometheusSpec:
    shards: 3   # Split scraping across 3 Prometheus instances
```

**Real scenario:** We set up kube-prometheus-stack on an EKS cluster with 40 nodes and 200+ pods. Initial setup took 20 minutes. Within an hour, Prometheus had scraped all nodes and pods. The first alert that fired: `PodCrashLooping` for a payment-service pod that was crashing due to a misconfigured environment variable. Without monitoring, we'd have found out from a user complaint. With monitoring, we fixed it in 8 minutes of being alerted — before any user noticed.

---

## 5. How can you integrate Prometheus with Alertmanager and Slack for real-time notifications?

The Prometheus → Alertmanager → Slack pipeline sends a Slack message when a metric breaches a threshold. Each component has a specific role: Prometheus evaluates, Alertmanager routes and deduplicates, Slack delivers.

**The pipeline:**

```
Prometheus (evaluates alert rules every 15s)
    ↓ alert fires if condition true for `for:` duration
Alertmanager (groups, deduplicates, routes)
    ↓ matches route → sends to Slack receiver
Slack channel (formatted notification)
```

**Step 1 — Create a Slack incoming webhook.**

In Slack:
1. Go to `api.slack.com/apps` → Create New App → From scratch
2. Enable **Incoming Webhooks** → Add New Webhook to Workspace
3. Choose a channel → Copy the webhook URL: `https://hooks.slack.com/services/T.../B.../xxx`

Store the webhook URL as a Kubernetes secret:

```bash
kubectl create secret generic alertmanager-slack-webhook \
  --from-literal=webhook-url="https://hooks.slack.com/services/T.../B.../xxx" \
  -n monitoring
```

**Step 2 — Configure Alertmanager routing.**

```yaml
# alertmanager.yml (or AlertmanagerConfig CRD in kube-prometheus-stack)
global:
  resolve_timeout: 5m

route:
  receiver: "default-slack"        # Default: all alerts go to Slack
  group_by: ["alertname", "namespace"]  # Group related alerts together
  group_wait: 30s                  # Wait 30s to collect related alerts before sending
  group_interval: 5m               # Wait 5m before sending new alerts in same group
  repeat_interval: 4h              # Resend unresolved alert every 4 hours

  routes:
    # Critical alerts → immediate Slack + PagerDuty
    - match:
        severity: critical
      receiver: "critical-alerts"
      continue: true               # Also evaluate following routes

    # Warning alerts → Slack only
    - match:
        severity: warning
      receiver: "warning-slack"

    # Database alerts → separate channel
    - match:
        team: database
      receiver: "db-slack"

receivers:
  - name: "default-slack"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/T.../B.../xxx"
        channel: "#alerts"
        send_resolved: true        # Send "resolved" message when alert clears
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: '{{ template "slack.default.title" . }}'
        text: '{{ range .Alerts }}*Alert:* {{ .Annotations.summary }}\n*Severity:* {{ .Labels.severity }}\n*Description:* {{ .Annotations.description }}\n{{ end }}'

  - name: "critical-alerts"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/T.../B.../xxx"
        channel: "#critical-alerts"
        send_resolved: true
    pagerduty_configs:
      - routing_key: <pagerduty-integration-key>
        description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'

  - name: "warning-slack"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/T.../B.../xxx"
        channel: "#alerts-warning"
        send_resolved: true

  - name: "db-slack"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/T.../B.../xxx"
        channel: "#database-alerts"

inhibit_rules:
  # If a critical alert is firing, suppress the corresponding warning
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ["alertname", "namespace"]
```

**Step 3 — Define alert rules in Prometheus.**

```yaml
# PrometheusRule CRD (kube-prometheus-stack)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: application-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: application
      rules:
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{status=~"5.."}[5m])
            /
            rate(http_requests_total[5m]) * 100 > 5
          for: 5m
          labels:
            severity: critical
            team: application
          annotations:
            summary: "High error rate on {{ $labels.service }}"
            description: "Error rate is {{ $value | printf \"%.1f\" }}% on {{ $labels.service }} (threshold: 5%)"
            runbook: "https://wiki.company.com/runbooks/high-error-rate"

        - alert: HighLatency
          expr: |
            histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High p95 latency on {{ $labels.service }}"
            description: "p95 latency is {{ $value | printf \"%.2f\" }}s on {{ $labels.service }}"
```

**Step 4 — Apply Alertmanager config via kube-prometheus-stack (Kubernetes secret).**

```yaml
# When using kube-prometheus-stack, provide config via values.yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: default-slack
      group_by: [alertname, namespace]
      routes: [...]
    receivers:
      - name: default-slack
        slack_configs:
          - api_url:
              # Reference the Kubernetes secret we created
              secretKeyRef:
                name: alertmanager-slack-webhook
                key: webhook-url
            channel: "#alerts"
            send_resolved: true
```

**Step 5 — Test the integration without waiting for a real alert.**

```bash
# Send a test alert directly to Alertmanager
curl -X POST http://alertmanager.monitoring.svc.cluster.local:9093/api/v1/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {
      "alertname": "TestAlert",
      "severity": "warning",
      "service": "test-service"
    },
    "annotations": {
      "summary": "This is a test alert",
      "description": "Testing Alertmanager → Slack pipeline"
    },
    "generatorURL": "http://prometheus.monitoring.svc.cluster.local:9090"
  }]'

# Check Alertmanager UI to confirm it received the alert:
# kubectl port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 -n monitoring
# Open: http://localhost:9093
```

**What a good Slack alert looks like:**

```
🔴 [FIRING] HighErrorRate — prod/order-service

Alert: High error rate on order-service
Severity: critical
Description: Error rate is 8.3% on order-service (threshold: 5%)
Runbook: https://wiki.company.com/runbooks/high-error-rate

Started: 14:32 UTC (5 minutes ago)
Labels: namespace=prod, service=order-service
```

Include the runbook link in every alert — on-call engineers know exactly where to look.

**Common Alertmanager problems and fixes:**

```bash
# Alert firing in Prometheus but not appearing in Slack?
# 1. Check Alertmanager received it:
curl http://localhost:9093/api/v1/alerts

# 2. Check Alertmanager routing:
# Alertmanager UI → Status → Routing → shows which receiver will handle the alert

# 3. Check Alertmanager logs for Slack API errors:
kubectl logs -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0

# Common error: "context deadline exceeded" → Slack webhook URL wrong or rotated
# Common error: "no receivers defined" → route doesn't match alert labels
```

**Real scenario:** We had Prometheus alerts firing but nobody being notified — the alerts were in Prometheus UI but engineers only saw them during weekly reviews. After setting up the Alertmanager → Slack pipeline: the first week, 12 alerts fired that the team had never seen before. 3 were noise (tuned away). 9 were real issues: 2 memory leaks, 1 disk filling on a logging node, 4 intermittent connection pool exhaustions, 2 pods with high restart counts. All resolved within hours of being alerted — instead of discovered days later during user escalations. Mean Time To Detect (MTTD) dropped from 4 hours to 8 minutes.

---

## 6. What monitoring tools are set up in your project? Have you configured alerts and what common pod errors have you faced?

> **Also asked as:** "Walk me through your observability stack" · "What tools do you use for monitoring K8s?" · "Tell me about alerts you've set and pod issues you've faced in production."

This is a "show your real experience" question — they want specifics, not a list of tool names.

**Our monitoring stack:**

```
Metrics:  Prometheus + kube-state-metrics + node-exporter
Dashboards: Grafana (K8s dashboards, app-level dashboards)
Alerting: Alertmanager → PagerDuty (P1/P2) + Slack (P3/P4)
Logs:     Fluent Bit → Elasticsearch → Kibana (EFK stack)
Tracing:  Jaeger (distributed tracing for microservices)
Uptime:   Blackbox Exporter (external endpoint health checks)
```

All deployed via the `kube-prometheus-stack` Helm chart — one install gives you Prometheus, Alertmanager, Grafana, node-exporter, and kube-state-metrics pre-wired together.

**Alerts we have configured:**

```yaml
# Critical alerts (PagerDuty — wake someone up)
- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Pod {{ $labels.pod }} is crash looping"

- alert: NodeNotReady
  expr: kube_node_status_condition{condition="Ready",status="true"} == 0
  for: 2m
  labels:
    severity: critical

- alert: DeploymentReplicasMismatch
  expr: kube_deployment_spec_replicas != kube_deployment_status_available_replicas
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "{{ $labels.deployment }} has {{ $value }} fewer replicas than desired"

# Warning alerts (Slack — investigate during business hours)
- alert: PodHighMemory
  expr: |
    container_memory_working_set_bytes{container!=""}
    / container_spec_memory_limit_bytes{container!=""}
    > 0.85
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Pod {{ $labels.pod }} memory > 85% of limit — OOMKill risk"

- alert: NodeDiskPressure
  expr: |
    (node_filesystem_size_bytes - node_filesystem_avail_bytes)
    / node_filesystem_size_bytes > 0.80
  for: 5m
  labels:
    severity: warning

- alert: HighErrorRate
  expr: |
    rate(http_requests_total{status=~"5.."}[5m])
    / rate(http_requests_total[5m]) > 0.05
  for: 3m
  labels:
    severity: critical
  annotations:
    summary: "Error rate > 5% on {{ $labels.service }}"
```

**Common pod errors we've faced in production:**

**1. CrashLoopBackOff:**
Most common cause in our environment: missing environment variables. A secret or ConfigMap got renamed, the pod started with `nil` values, and the app panicked on startup.
```bash
kubectl logs <pod> --previous   # Check crash reason
kubectl describe pod <pod>      # Check Events for missing secret/configmap
```

**2. OOMKilled:**
Our Java service got OOMKilled every Monday morning — the JVM wasn't respecting container memory limits (pre-Java 11 behavior). Added `-XX:MaxRAMPercentage=75 -XX:+UseContainerSupport` to JVM flags. Solved it permanently.

**3. ImagePullBackOff:**
ECR tokens expire every 12 hours. After a node came up fresh overnight, it couldn't pull images because the ECR credential helper hadn't refreshed. Fix: switched to the `amazon-ecr-credential-helper` DaemonSet which auto-refreshes tokens.

**4. Pods stuck in Pending:**
After a team pushed a new service with `requests.memory: 4Gi` copy-pasted from a heavy batch job, all pods sat Pending for 20 minutes. No node had 4Gi free. Alert: `kube_pod_status_phase{phase="Pending"} > 0` for 5 minutes triggered in Slack.

**5. Readiness probe failing → 503 on service:**
A deployment changed the health endpoint from `/health` to `/healthz` but forgot to update the readiness probe. Pods were Running but NotReady — service had 0 endpoints. Grafana showed error rate spike; Kibana logs showed the probe hitting `/health` → 404. Rollback + fix in 8 minutes.

**How we use Grafana dashboards:**
- **K8s cluster overview** (USE method: Utilization, Saturation, Errors per node)
- **Per-namespace resource consumption** (which team is consuming what)
- **Application-level dashboards** (request rate, P95 latency, error rate — RED method)
- **Business metrics dashboard** (orders per minute, payment success rate — pulled from app metrics via Prometheus custom metrics)

**One thing I'd do differently:** We set up Jaeger tracing 6 months after the project started. If I'd done it from day one, several "mystery slowdowns" that took hours to debug would have been found in minutes. Distributed tracing is not optional for microservices.
