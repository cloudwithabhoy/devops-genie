# Security — Medium Questions

---

## 1. Which security tools or plugins have you used for image scanning?

The answer isn't just a list of tool names — it's which tool you use at which stage of the pipeline and why.

**Trivy (primary — used in CI):**

Open-source, fast, comprehensive. Scans OS packages, language dependencies (pip, npm, go.sum, Gemfile.lock), config files, and Kubernetes manifests. We run it in Jenkins after every `docker build`.

```bash
trivy image \
  --exit-code 1 \
  --severity CRITICAL,HIGH \
  --ignore-unfixed \
  --format sarif \
  --output trivy-results.sarif \
  $ECR_REPO:$IMAGE_TAG
```

`--format sarif` outputs results in SARIF format which GitHub Security tab can ingest natively — scan results appear directly in the PR as code scanning alerts.

**ECR Enhanced Scanning with Snyk (registry-level — continuous):**

ECR's enhanced scanning uses Snyk under the hood. Runs automatically on push and continuously re-scans images when new CVEs are published. We get EventBridge events → SNS → Slack when a critical CVE appears in any deployed image.

```hcl
resource "aws_ecr_registry_scanning_configuration" "main" {
  scan_type = "ENHANCED"
  rule {
    scan_frequency = "CONTINUOUS_SCAN"
    repository_filter {
      filter      = "*"
      filter_type = "WILDCARD"
    }
  }
}
```

**Hadolint (Dockerfile linting — in CI):**

Catches Dockerfile best-practice violations before the image is even built.

```bash
hadolint Dockerfile
# DL3008: Pin versions in apt get install
# DL3018: Pin versions in apk add
# DL3025: Use COPY instead of ADD for files/folders
```

We run this as the first stage — fast, zero cost, catches real issues.

**Checkov (IaC scanning — in CI):**

Scans Terraform, Kubernetes YAML, Dockerfiles, and Helm charts for misconfigurations.

```bash
checkov -d . --framework dockerfile    # Scan Dockerfiles
checkov -d terraform/ --framework terraform   # Scan Terraform
```

Caught things like: S3 buckets without encryption, security groups with 0.0.0.0/0 inbound, Lambda functions without VPC config.

**OWASP Dependency-Check (for Java/Maven projects):**

Scans Java dependencies (JAR files) against the NVD CVE database. More thorough than Trivy for Java-specific vulnerabilities.

```bash
dependency-check --project my-app --scan . --format JSON --out reports/
```

We use this only for Java services — Trivy covers Python/Node/Go adequately.

**SonarQube (SAST — static application security testing):**

Not an image scanner but part of our security pipeline. Scans source code for SQL injection, XSS, hardcoded credentials, insecure deserialization. Runs in parallel with the image build stage.

**Real scenario:** Trivy was passing a Node.js image cleanly for months. A new CVE was published against `lodash` — a transitive dependency we didn't even know we had. ECR's continuous scan flagged it the day the CVE was published. Without registry-level scanning, we'd have found it only on the next rebuild (which might have been weeks away). We rebuilt with `lodash` pinned to the patched version and redeployed within 4 hours of the CVE being published.

---

## 2. How do you establish secure connections with databases?

"Secure connection" means: encrypted in transit, authenticated with least-privilege credentials, and not accessible from the public internet. All three.

**Network-level isolation:**

RDS lives in private subnets. No public endpoint. The security group allows inbound connections only from the EKS node security group (or pod CIDR with VPC CNI) on the specific database port:

```hcl
resource "aws_security_group_rule" "rds_from_eks" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = aws_security_group.rds.id
  source_security_group_id = aws_security_group.eks_nodes.id
}
```

Nothing else reaches the database — not the internet, not other VPCs (unless explicitly peered), not other security groups.

**Credentials management — AWS Secrets Manager:**

Database credentials (username, password) are stored in Secrets Manager. The application reads them at startup via the AWS SDK:

```python
import boto3, json

def get_db_credentials():
    client = boto3.client('secretsmanager', region_name='ap-south-1')
    secret = client.get_secret_value(SecretId='prod/myapp/db')
    return json.loads(secret['SecretString'])

creds = get_db_credentials()
conn = psycopg2.connect(
    host=creds['host'],
    database=creds['dbname'],
    user=creds['username'],
    password=creds['password'],
    sslmode='require'     # Enforce TLS
)
```

The application IAM role (via IRSA in EKS) has permission to call `secretsmanager:GetSecretValue` on only this specific secret ARN — not all secrets. The credentials never appear in environment variables, logs, or the Docker image.

**TLS for in-transit encryption:**

RDS supports TLS connections. We enforce `sslmode='require'` (Postgres) or `ssl=True` (MySQL). The RDS CA certificate is available from AWS and can be downloaded:

```bash
curl -o rds-ca.pem https://truststore.pki.rds.amazonaws.com/ap-south-1/ap-south-1-bundle.pem
```

For applications that need certificate verification (not just encryption):
```python
conn = psycopg2.connect(..., sslmode='verify-full', sslrootcert='rds-ca.pem')
```

**Developer access (no RDS public endpoint):**

Developers connect via AWS SSM Session Manager port forwarding — no bastion, no open inbound port, short-lived access:

```bash
aws ssm start-session \
  --target i-0abc123def456 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["prod-db.abc123.ap-south-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["15432"]}'

psql -h localhost -p 15432 -U app_user -d mydb
```

The SSM session is audited — every developer access is logged in CloudTrail.

**Automatic credential rotation:**

Secrets Manager can rotate RDS credentials automatically. We use the AWS-provided rotation Lambda for RDS:

```hcl
resource "aws_secretsmanager_secret_rotation" "db" {
  secret_id           = aws_secretsmanager_secret.db.id
  rotation_lambda_arn = "arn:aws:lambda:ap-south-1:123456789:function:SecretsManagerRDSPostgreSQLRotationSingleUser"
  rotation_rules {
    automatically_after_days = 30
  }
}
```

Every 30 days, Secrets Manager rotates the password in RDS and updates the secret — zero manual intervention, zero downtime (rotation creates a new password before invalidating the old one).

**Real scenario:** Before this setup, database credentials were in a `.env` file that developers shared over Slack. The file contained prod database credentials. Three separate developers had copies on their local machines. One developer's laptop was stolen. We had to rotate all database credentials immediately, update every service, and restart all pods — 4 hours of work across the team at 11 PM. After moving to Secrets Manager + IRSA, no developer has the database password. The application gets it at runtime, and it rotates automatically.

---

## 3. How do you authenticate to EKS clusters and securely manage secrets?

Two separate concerns that often get answered together superficially. Here's the full answer for each.

**Authenticating to EKS:**

EKS uses IAM for authentication and Kubernetes RBAC for authorization. The flow:

1. `aws eks update-kubeconfig` generates a kubeconfig that uses `aws eks get-token` as the credential provider
2. When kubectl makes a request, it calls `aws eks get-token` to get a short-lived token (15-minute expiry)
3. EKS validates the token against IAM, identifies the IAM principal (user or role)
4. EKS maps the IAM principal to a Kubernetes RBAC user/group via the `aws-auth` ConfigMap
5. RBAC evaluates whether that user/group has permission for the requested operation

```bash
# Developer authenticates via AWS SSO
aws sso login --profile prod-readonly

# Configure kubectl for EKS
aws eks update-kubeconfig --region ap-south-1 --name prod-cluster --profile prod-readonly
# kubectl now uses their SSO role for auth
```

The `aws-auth` ConfigMap maps IAM roles to K8s RBAC:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789:role/developer-role
      username: developer
      groups:
        - dev-readonly      # Custom RBAC group with get/list/watch only
    - rolearn: arn:aws:iam::123456789:role/ci-deploy-role
      username: ci-system
      groups:
        - deployers         # RBAC group with create/update on deployments only
```

No long-lived kubeconfigs stored on disk. No shared credentials. Every access is traced to a specific IAM principal in CloudTrail.

**Managing secrets in EKS — External Secrets Operator:**

We use External Secrets Operator (ESO) syncing from AWS Secrets Manager. The secret lives in AWS, ESO creates a Kubernetes Secret from it, pods mount the K8s Secret.

```yaml
# The ExternalSecret tells ESO what to fetch and where to put it
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h         # Re-sync every hour
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-credentials      # Name of the K8s Secret to create/update
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: prod/myapp/db
        property: password
```

ESO uses an IAM role (via IRSA) to call Secrets Manager. The pod that needs the secret mounts the K8s Secret:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
```

**Why not plain Kubernetes Secrets:**

K8s Secrets are base64-encoded — not encrypted. Anyone with `kubectl get secret -o yaml` access can decode them. We encrypt etcd at rest (via EKS encryption configuration), but the real protection is keeping secrets out of Git and out of plain manifests entirely. ESO achieves this — the secret value never exists in Git.

**Real scenario:** Before ESO, a new developer set up their local environment by running `kubectl get secret db-credentials -o yaml` in production and decoding the base64 password to test a query. The password was now on their laptop in a terminal history file. After moving to ESO + IRSA, that same `kubectl get secret` command only returns a K8s Secret object whose value ESO manages — and the developer's RBAC role doesn't allow `get secret` on the production namespace at all.

---

## 4. What is Helm chart signing? Which tools are used?

Helm chart signing lets you cryptographically verify that a chart came from a trusted source and hasn't been tampered with since it was published. Without signing, anyone with write access to your chart registry can push a malicious chart.

**How it works:**

When you sign a chart, Helm produces two files:
- `my-app-1.4.0.tgz` — the chart package
- `my-app-1.4.0.tgz.prov` — the provenance file (SHA256 hash of the chart + PGP signature)

The provenance file proves: (1) the chart content matches the hash (integrity), and (2) the signature came from a key you trust (authenticity).

**Tools used:**

| Tool | Role |
|------|------|
| **GPG (GnuPG)** | Classic PGP key management and signing |
| **Helm** | Built-in `helm package --sign` and `helm verify` |
| **Cosign (Sigstore)** | Modern keyless signing via OIDC identity |
| **Kyverno / Connaisseur** | Enforce signature verification at cluster admission |
| **Rekor (Sigstore)** | Immutable transparency log for all signatures |

**Signing with GPG + Helm:**

```bash
# Package and sign
helm package my-app/ \
  --sign \
  --key "ci-pipeline@company.com" \
  --keyring ~/.gnupg/secring.gpg

# Produces: my-app-1.4.0.tgz + my-app-1.4.0.tgz.prov

# Verify before installing
gpg --import trusted-keys.pub
helm verify my-app-1.4.0.tgz

# Or verify at install time
helm install my-app oci://registry/charts/my-app \
  --verify --keyring ./trusted-keys.gpg
```

**Modern approach — Cosign keyless signing (what we use):**

GPG key management is operationally heavy — key generation, distribution, rotation, revocation. Cosign with Sigstore uses OIDC identity instead of a managed key. In GitHub Actions, the signing identity is the workflow's OIDC token:

```bash
# In CI (GitHub Actions) — signs using the workflow's OIDC identity, no key file needed
cosign sign --yes oci://registry/charts/my-app:1.4.0

# Verify — identity is checked against Sigstore's Rekor transparency log
cosign verify oci://registry/charts/my-app:1.4.0 \
  --certificate-identity "https://github.com/our-org/our-repo/.github/workflows/release.yaml@refs/heads/main" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com"
```

**Enforcing signature verification at the cluster level:**

Signing is useless unless you verify at deploy time. Kyverno policy blocks pods using unsigned images:

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signatures
spec:
  rules:
    - name: check-signature
      match:
        resources:
          kinds: ["Pod"]
      verifyImages:
        - imageReferences:
            - "123456789.dkr.ecr.ap-south-1.amazonaws.com/*"
          attestors:
            - entries:
                - keyless:
                    issuer: "https://token.actions.githubusercontent.com"
                    subject: "https://github.com/our-org/*"
```

With this policy, even if someone manually pushes an unsigned image to ECR, any pod using it is rejected at admission time.

**Real scenario:** We passed a SOC2 Type II audit requirement for software supply chain controls. The auditor asked: "How do you ensure no unauthorized image runs in production?" We demonstrated: every image is signed by the CI pipeline via Cosign (tied to the GitHub Actions OIDC identity), Kyverno blocks any pod using an unsigned or incorrectly-signed image, and Rekor's transparency log provides an immutable audit trail of every image ever signed by our pipelines. An image pushed manually to ECR — even by an admin — would be blocked by Kyverno at the pod level.
