# Monitoring & Logging Cheatsheet

> Quick-reference queries and concepts for observability tools.

## Core Commands / Concepts

### Prometheus — PromQL Examples
```promql
# CPU usage per instance
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# HTTP request rate
rate(http_requests_total[5m])

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
```

### Grafana — Common Actions
```
Dashboard → Add panel → Select data source → Write PromQL / LogQL
Alerting → Alert rules → New alert rule → Set condition and notification
Explore → Select Loki → Run LogQL query
```

### Loki — LogQL Examples
```logql
# Filter logs by label
{app="nginx"} |= "error"

# Count errors per minute
count_over_time({app="nginx"} |= "error" [1m])

# Parse log fields
{app="app"} | json | status_code >= 500
```

### Linux Log Commands
```bash
journalctl -u nginx -f                   # follow service logs
journalctl --since "1 hour ago"
tail -f /var/log/syslog
grep -i error /var/log/app.log
```

## Key Metrics to Monitor

| Category | Metrics |
|----------|---------|
| System | CPU, memory, disk I/O, network |
| Application | Request rate, error rate, latency (RED) |
| Database | Query time, connections, cache hit ratio |
| Kubernetes | Pod restarts, pending pods, node pressure |

## Common Patterns

<!-- Add alerting rules, dashboard templates, runbook patterns here -->
