# Security — Real Production Issues

---

## 1. SSL/TLS certificate expired in production — how do you respond and prevent it from happening again?

> **Also asked as:** "You get a call that the site shows 'Your connection is not private' — what do you do?" · "Certificate expired on production — walk me through your response" · "How do you prevent SSL expiry in a Kubernetes cluster?" · "cert-manager is not renewing certificates automatically — how do you debug it?"

SSL expiry is an embarrassing, entirely preventable incident. The site works at 11:59 PM, and at midnight it's down — hard. Browsers refuse to load it, APIs return TLS handshake errors, and mobile apps start crashing. Here's how to handle it fast and make it impossible to happen again.

**Immediate response — confirm it's really expiry.**

```bash
# From your terminal — check the cert expiry on the live site
echo | openssl s_client -servername your-domain.com -connect your-domain.com:443 2>/dev/null \
  | openssl x509 -noout -dates
# notBefore=Jan  1 00:00:00 2024 GMT
# notAfter=Jan  1 00:00:00 2025 GMT    ← if this is in the past → cert is expired

# One-liner with days remaining
echo | openssl s_client -servername your-domain.com -connect your-domain.com:443 2>/dev/null \
  | openssl x509 -noout -checkend 0
# "Certificate will expire" or "Certificate will not expire"

# Check the exact error code
curl -v https://your-domain.com 2>&1 | grep -i "expire\|SSL\|TLS"
# curl: (60) SSL certificate problem: certificate has expired
```

**Emergency fix — renew the certificate.**

The path depends on how your cert was issued:

**Option A: Let's Encrypt (Certbot)**

```bash
# If certbot is installed on the server:
sudo certbot renew --force-renewal
sudo systemctl reload nginx   # or apache2

# If behind a load balancer / no port 80 access:
sudo certbot certonly --manual --preferred-challenges dns \
  -d your-domain.com \
  -d *.your-domain.com
# Add the TXT record to DNS, then complete the challenge
```

**Option B: AWS Certificate Manager (ACM)**

```bash
# ACM certs don't expire if auto-renewal is enabled and validation is in place
# Check why auto-renewal failed:
aws acm describe-certificate \
  --certificate-arn arn:aws:acm:ap-south-1:123456789:certificate/abc-123 \
  --query 'Certificate.{Status:Status,RenewalStatus:RenewalSummary.RenewalStatus,Reason:RenewalSummary.RenewalStatusReason}'

# Status: EXPIRED or RenewalStatus: FAILED
# Most common reason: DNS validation record was deleted from Route 53
# Fix: re-add the CNAME validation record and wait for ACM to re-validate (usually < 30 mins)

# For an already-expired cert: request a new one and re-attach to ALB/CloudFront
aws acm request-certificate \
  --domain-name your-domain.com \
  --validation-method DNS \
  --subject-alternative-names "*.your-domain.com"
```

**Option C: Kubernetes + cert-manager**

```bash
# Check if cert-manager is managing the certificate
kubectl get certificate -n <namespace>
# NAME          READY   SECRET            AGE
# my-tls-cert   False   my-tls-secret     367d   ← Not Ready after 367 days → problem

kubectl describe certificate my-tls-cert -n <namespace>
# Events:
# Failed to create Order: 429 Too Many Requests (rate limited by Let's Encrypt)
# ACME account not found (ACME Issuer misconfigured)
# DNS challenge failed (DNS provider credentials expired)

# Force an immediate renewal attempt
kubectl delete certificaterequest -n <namespace> -l cert-manager.io/certificate-name=my-tls-cert
# cert-manager will create a new CertificateRequest automatically

# Check the renewal challenge
kubectl get challenge -n <namespace>
kubectl describe challenge <challenge-name> -n <namespace>
# If challenge is stuck → DNS/HTTP validation is failing
```

**Emergency workaround while waiting for renewal:**

```bash
# If you can't renew in time and need the site up NOW:
# Option 1: Generate a self-signed cert (users still get warning but API traffic works)
openssl req -x509 -nodes -days 7 \
  -newkey rsa:2048 \
  -keyout /tmp/emergency.key \
  -out /tmp/emergency.crt \
  -subj "/CN=your-domain.com"

# Load into Kubernetes secret
kubectl create secret tls my-tls-secret \
  --cert=/tmp/emergency.crt \
  --key=/tmp/emergency.key \
  -n <namespace> \
  --dry-run=client -o yaml | kubectl apply -f -

# Option 2: Temporarily disable HTTPS redirect (only if tolerable)
# Revert Ingress annotation: nginx.ingress.kubernetes.io/ssl-redirect: "false"
```

**Prevention — make certificate expiry impossible.**

**cert-manager in Kubernetes (the right way):**

```yaml
# ClusterIssuer using Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: devops@your-company.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
      # Or DNS-01 for wildcard certs:
      - dns01:
          route53:
            region: ap-south-1
            hostedZoneID: ZYXWVUTSRQPONMLK
```

```yaml
# Certificate resource — cert-manager renews 30 days before expiry automatically
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-tls-cert
  namespace: production
spec:
  secretName: my-tls-secret
  duration: 2160h      # 90 days (Let's Encrypt default)
  renewBefore: 720h    # Start renewing 30 days before expiry
  dnsNames:
    - your-domain.com
    - www.your-domain.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

```yaml
# Ingress — automatically uses the certificate
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - your-domain.com
      secretName: my-tls-secret
  rules:
    - host: your-domain.com
      ...
```

**Monitor certificate expiry — alert at 30 days, page at 7 days:**

```yaml
# Prometheus alert rules (works with cert-manager metrics or blackbox exporter)
groups:
  - name: ssl_expiry
    rules:
      - alert: SSLCertExpiringSoon
        expr: |
          (certmanager_certificate_expiration_timestamp_seconds
          - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Certificate {{ $labels.name }} expires in {{ $value | humanizeDuration }}"

      - alert: SSLCertExpiringCritical
        expr: |
          (certmanager_certificate_expiration_timestamp_seconds
          - time()) / 86400 < 7
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "URGENT: Certificate {{ $labels.name }} expires in {{ $value | humanizeDuration }}"
```

```bash
# Blackbox exporter — monitor SSL expiry on any external endpoint
# prometheus.yml scrape config:
- job_name: ssl_expiry
  metrics_path: /probe
  params:
    module: [http_2xx]
  static_configs:
    - targets:
        - https://your-domain.com
        - https://api.your-domain.com
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: blackbox-exporter:9115

# Alert:
probe_ssl_earliest_cert_expiry - time() < 86400 * 14  # Alert if < 14 days
```

**Debugging cert-manager renewal failures:**

```bash
# Full cert-manager debug flow
kubectl get certificate,certificaterequest,order,challenge -n <namespace>

# Certificate stuck "False"?
kubectl describe certificate <name> -n <namespace>
# → Look at Events for the error

# CertificateRequest pending?
kubectl describe certificaterequest <name> -n <namespace>
# → Check if ACME order was created

# Order pending?
kubectl describe order <name> -n <namespace>
# → Check if challenge was created and what its state is

# Challenge failing?
kubectl describe challenge <name> -n <namespace>
# Common failure reasons:
# "Error presenting challenge: error setting TXT record"
#    → DNS provider credentials (Route53 IAM role) are wrong or expired
# "Waiting for DNS record to be created"
#    → DNS propagation delay — wait 5-10 minutes
# "HTTP-01 challenge failed: 404"
#    → Ingress path /.well-known/acme-challenge/ not routing to cert-manager solver
#    → Check that the ClusterIssuer uses the correct ingress class

# cert-manager controller logs
kubectl logs -n cert-manager deployment/cert-manager --tail=100 | grep -i "error\|fail\|renew"
```

**Real scenario:** At 8 AM on a Monday, 3 enterprise clients called simultaneously — our API was returning `TLS handshake failed`. Certificate for `api.our-company.com` had expired at midnight. We'd been running cert-manager for 6 months without issues. The root cause: 2 weeks earlier, we'd migrated from nginx ingress to ALB ingress controller. The old ClusterIssuer used `http01.ingress.class: nginx`. With the nginx ingress gone, cert-manager couldn't serve the ACME HTTP-01 challenge. The renewal attempt 30 days before expiry silently failed — the certificate just sat there "Ready: True" with the old cert, no alert. Immediate fix: manually created a new certificate using DNS-01 validation (Route53), loaded it into the secret — service recovered in 8 minutes. Permanent fix: switched all Certificates to DNS-01 validation (more reliable, works regardless of ingress class), added Prometheus alerts for cert expiry at 30 days and 7 days, and added a weekly `kubectl get certificates -A` to our ops runbook. SSL expiry hasn't happened since.

---

## 2. Secret or credentials leaked — container logs exposed sensitive data. What do you do?

> **Also asked as:** "A developer accidentally logged a JWT token or DB password — how do you respond?" · "Sensitive data found in container logs — incident response steps?" · "How do you prevent secret leakage in Kubernetes?"

A leaked secret in logs is a P1 incident. The secret must be treated as fully compromised from the moment it appeared in logs — because logs are often shipped to external systems (Elasticsearch, Datadog, Splunk) and retain data for months.

**Immediate response — first 15 minutes.**

```bash
# Step 1: Identify the scope of the leak
# When did it first appear?
kubectl logs <pod> -n <namespace> | grep -i "password\|secret\|token\|key\|auth" | head -20
# Note the earliest timestamp

# Step 2: Is it still being logged?
kubectl logs <pod> -n <namespace> --tail=50 | grep -i "password\|token"
# If yes → fix the app immediately, redeploy

# Step 3: Where does this log go?
# Check your logging pipeline: Fluentd/Fluent Bit → Elasticsearch/Datadog/CloudWatch
# The secret is now in ALL downstream log stores
```

**Rotate the compromised credential immediately — before doing anything else.**

```bash
# Database password → rotate immediately
# For RDS:
aws rds modify-db-instance \
  --db-instance-identifier prod-db \
  --master-user-password "$(openssl rand -base64 32)"

# AWS Access Key → deactivate old, create new
aws iam update-access-key \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive \
  --user-name service-user
aws iam create-access-key --user-name service-user

# JWT signing secret → rotate and force re-login for all users
# API key → invalidate via the provider's dashboard

# Kubernetes Secret → update immediately
kubectl create secret generic db-credentials \
  --from-literal=password="$(openssl rand -base64 32)" \
  -n production \
  --dry-run=client -o yaml | kubectl apply -f -

# Rollout the pods to pick up the new secret
kubectl rollout restart deployment/<app> -n production
```

**Remove the secret from logs in downstream systems.**

```bash
# CloudWatch: delete the log events containing the secret
aws logs delete-log-group --log-group-name /eks/prod/my-app
# (nuclear option — deletes all logs for that group)

# More targeted: use filter-log-events + delete-log-stream for affected streams
aws logs describe-log-streams \
  --log-group-name /eks/prod/my-app \
  --order-by LastEventTime \
  --descending

# For Elasticsearch/OpenSearch: delete affected documents
# POST /logs-2024.01.15/_delete_by_query
# { "query": { "match": { "message": "db_password" } } }
```

**Prevention — never let secrets reach logs.**

```python
# Application-level: use a custom log filter (Python example)
import logging
import re

class SecretFilter(logging.Filter):
    PATTERNS = [
        r'password["\s:=]+\S+',
        r'token["\s:=]+\S+',
        r'Authorization["\s:=]+\S+',
        r'AKIA[0-9A-Z]{16}',   # AWS Access Key pattern
    ]

    def filter(self, record):
        for pattern in self.PATTERNS:
            record.msg = re.sub(pattern, '[REDACTED]', str(record.msg), flags=re.IGNORECASE)
        return True
```

```yaml
# Fluent Bit: redact secrets before they reach the log store
[FILTER]
    Name    modify
    Match   *
    Condition Key_value_matches log (?i)(password|token|secret|key)\s*[=:]\s*\S+
    # Replace with redacted value using grep/sed filter
```

```yaml
# Kubernetes: never put secret values in env var names that get logged by frameworks
# Wrong — Spring Boot logs all env vars on startup:
env:
  - name: DB_PASSWORD
    value: "mysecret"    # Logged in plain text on startup!

# Right — use secretKeyRef, and ensure your app doesn't log env vars:
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

**Real scenario:** A Node.js service was logging the full request object for debugging — a developer added `console.log('Request received:', req)`. The `req.headers` included `Authorization: Bearer eyJhbGci...` (a JWT token). 6 hours of traffic logs contained thousands of valid JWT tokens. They were shipped to our Elasticsearch cluster and retained for 30 days. Response: rotated the JWT signing secret (invalidated all existing tokens — forced all users to log in again), deleted the Elasticsearch index for that 6-hour window, redeployed the service with the logging line removed. Permanent fix: added Falco rules to alert on logs containing known secret patterns, added `DEBUG` log level gate so debug logging only activates with an explicit flag (not in prod), and added pre-commit hook that scans code for common logging patterns near sensitive fields.

---
