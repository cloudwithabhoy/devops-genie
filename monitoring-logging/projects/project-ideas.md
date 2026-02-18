# Monitoring & Logging Project Ideas

> Hands-on projects to build real observability skills.

## Beginner

- **Prometheus + Grafana Stack** — Deploy Prometheus and Grafana with Docker Compose; scrape node-exporter metrics; build a CPU/memory dashboard.
- **Application Metrics** — Instrument a simple web app with a Prometheus client library and expose a `/metrics` endpoint.

## Intermediate

<!-- Add intermediate project ideas here -->

- **Alertmanager Setup** — Configure Alertmanager to send Slack notifications when CPU > 80% for 5 minutes.
- **Loki Log Aggregation** — Set up Loki + Promtail + Grafana to collect and visualise application logs from Docker containers.
- **SLO Dashboard** — Define SLIs for a sample API and build a Grafana dashboard showing error budget burn rate.

## Advanced

<!-- Add advanced project ideas here -->

- **Full Observability Stack on Kubernetes** — Deploy the kube-prometheus-stack Helm chart; add custom dashboards and alerts for a demo app.
- **Distributed Tracing** — Instrument a multi-service app with OpenTelemetry, send traces to Jaeger or Tempo, and visualise in Grafana.
- **Log-Based Alerting** — Use Loki alerting rules to fire on error log patterns and route notifications via Alertmanager.

## Project Template

For each project, document:
1. **Goal** — What you're monitoring and why
2. **Stack** — Tools and versions
3. **Setup** — Docker Compose or Helm values
4. **Dashboards** — Screenshot or JSON export
5. **Alert Rules** — YAML definitions
