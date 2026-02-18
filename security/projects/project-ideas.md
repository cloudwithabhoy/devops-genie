# Security Project Ideas

> Hands-on projects to build real DevSecOps skills.

## Beginner

- **Secret Scanning** — Run gitleaks or truffleHog on a sample repo and document any findings.
- **Container Image Scan** — Use Trivy to scan a public Docker image and categorise CVEs by severity.

## Intermediate

<!-- Add intermediate project ideas here -->

- **Security Pipeline** — Add SAST (e.g., Semgrep) and dependency scanning (e.g., Dependabot, Snyk) to a GitHub Actions workflow.
- **SSH Hardening Lab** — Harden a fresh Linux VM following CIS SSH benchmark recommendations.
- **Vault Integration** — Set up HashiCorp Vault, store database credentials, and fetch them dynamically from an application.

## Advanced

<!-- Add advanced project ideas here -->

- **Zero-Trust Kubernetes** — Implement network policies, OPA Gatekeeper policies, and Pod Security Standards in a cluster.
- **Supply Chain Security** — Sign container images with Cosign and verify signatures in a pipeline using Sigstore.
- **Compliance Audit** — Run a CIS benchmark scan with Lynis or OpenSCAP and remediate findings.

## Project Template

For each project, document:
1. **Goal** — Security objective and threat model
2. **Tools** — Scanner, policy engine, secrets manager
3. **Steps** — Implementation guide
4. **Findings** — Results or policy output
5. **Remediation** — How issues were fixed
