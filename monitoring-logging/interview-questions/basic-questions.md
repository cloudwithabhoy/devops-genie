# Monitoring Logging — Basic Questions

---

## 1. What is the configuration file of Prometheus?

Prometheus's main configuration file is **`prometheus.yml`**. It defines what to scrape, how often, and where to send alerts.

**Default location:**

```bash
/etc/prometheus/prometheus.yml    # Installed via package manager
# or wherever you ran Prometheus from, e.g.:
/opt/prometheus/prometheus.yml
```

**Structure of `prometheus.yml`:**

```yaml
global:
  scrape_interval:     15s    # How often to scrape targets (default: 1 minute)
  evaluation_interval: 15s    # How often to evaluate alerting rules

# Alertmanager configuration — where to send alerts
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load alerting rule files
rule_files:
  - "alert_rules.yml"
  - "recording_rules.yml"

# What to scrape (the core config)
scrape_configs:
  # Prometheus scrapes itself
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  # Scrape a Node Exporter running on the same host
  - job_name: "node"
    static_configs:
      - targets: ["localhost:9100"]
        labels:
          environment: "prod"
          server: "web-01"

  # Scrape multiple servers
  - job_name: "app-servers"
    static_configs:
      - targets:
          - "10.0.1.10:9100"
          - "10.0.1.11:9100"
          - "10.0.1.12:9100"

  # Service discovery — scrape all EC2 instances with a specific tag
  - job_name: "ec2-nodes"
    ec2_sd_configs:
      - region: ap-south-1
        port: 9100
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Monitoring]
        regex: "true"
        action: keep
```

**Apply config changes:**

```bash
# Prometheus hot-reloads config on SIGHUP — no restart needed
kill -HUP $(pgrep prometheus)

# Or if you started with --web.enable-lifecycle:
curl -X POST http://localhost:9090/-/reload
```

**Validate config before applying:**

```bash
promtool check config /etc/prometheus/prometheus.yml
# SUCCESS: /etc/prometheus/prometheus.yml is valid
```

---

## 2. Where do you set up alerting — in Prometheus or Grafana?

Both can send alerts, but they serve different roles:

| | Prometheus + Alertmanager | Grafana Alerts |
|---|---|---|
| Alert source | Metric thresholds defined in `.yml` rule files | Grafana query-based alert rules |
| Works without Grafana? | Yes — fully independent | No — needs Grafana running |
| Routing | Alertmanager routes by labels (team, severity) | Grafana contact points |
| Common use | Production alerting, infrastructure | Dashboard-based alerts, non-Prometheus sources |

**The standard production setup: Prometheus + Alertmanager.**

Alerting rules live in Prometheus, routing and notification logic lives in Alertmanager:

```yaml
# alert_rules.yml — loaded by prometheus.yml
groups:
  - name: infrastructure
    rules:
      - alert: HighDiskUsage
        expr: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100 > 80
        for: 5m    # Must be true for 5 minutes before firing
        labels:
          severity: warning
        annotations:
          summary: "Disk usage above 80% on {{ $labels.instance }}"
          description: "Disk usage is {{ $value | printf \"%.1f\" }}% on {{ $labels.instance }}"

      - alert: HighDiskUsageCritical
        expr: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100 > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "CRITICAL: Disk almost full on {{ $labels.instance }}"
```

Prometheus evaluates these rules every `evaluation_interval`. When the condition is true for the `for` duration, Prometheus fires the alert to Alertmanager.

**Alertmanager handles routing and deduplication:**

```yaml
# alertmanager.yml
route:
  receiver: "default"
  group_by: ["alertname", "instance"]
  routes:
    - match:
        severity: critical
      receiver: "pagerduty-critical"
    - match:
        severity: warning
      receiver: "slack-warnings"

receivers:
  - name: "default"
    slack_configs:
      - api_url: "https://hooks.slack.com/services/T00/B00/xxx"
        channel: "#alerts"

  - name: "pagerduty-critical"
    pagerduty_configs:
      - service_key: "<pagerduty-key>"
```

**Grafana Alerts** are good for alerting on data from multiple sources (a single alert combining Prometheus metrics + Loki logs + MySQL query). For pure Prometheus alerting, Alertmanager is more powerful and doesn't require Grafana to be running.

---

## 3. How do you configure CPU alerts in Prometheus?

To configure CPU alerts, you need a precise PromQL expression and an alerting rule file.

**Step 1: The PromQL expression**

For host-level (EC2/VM) CPU usage using `node_exporter`, you calculate CPU usage by subtracting the `idle` time from 100%:
```promql
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

For Kubernetes pod limit utilization, you compare actual usage to the configured limit:
```promql
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod) 
  / 
sum(kube_pod_container_resource_limits{resource="cpu"}) by (pod) * 100
```

**Step 2: Create the Alert Rule**

Create an alerting rule definition in a YAML file (e.g., `cpu-alerts.yml`). The `for: 5m` prevents the alert from firing during short, normal CPU bursts.

```yaml
groups:
  - name: cpu-alerts
    rules:
      - alert: HostHighCpuLoad
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host high CPU load (instance {{ $labels.instance }})"
          description: "CPU load is > 80% for 5 minutes. Current value: {{ $value | printf \"%.2f\" }}%"
```

**Step 3: Load the rule in prometheus.yml**

Tell Prometheus to evaluate this rule file:

```yaml
rule_files:
  - "cpu-alerts.yml"
```

Finally, reload Prometheus configuration (`curl -X POST http://localhost:9090/-/reload`) and verify the alert appears under the "Alerts" tab in the Prometheus UI.
