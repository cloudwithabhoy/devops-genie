# Security Cheatsheet

> Quick-reference commands and concepts for DevSecOps.

## Core Commands / Concepts

### SSL/TLS
```bash
# Check certificate expiry
openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# Inspect a certificate file
openssl x509 -in cert.pem -text -noout

# Generate a self-signed certificate
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

### SSH Hardening
```bash
# Key-based auth only — edit /etc/ssh/sshd_config:
PasswordAuthentication no
PermitRootLogin no
AllowUsers deployuser

# Audit SSH keys
cat ~/.ssh/authorized_keys

# Generate a new SSH key pair
ssh-keygen -t ed25519 -C "your@email.com"
```

### Secret Scanning
```bash
# Scan with truffleHog
trufflehog git file://. --only-verified

# Scan with gitleaks
gitleaks detect --source=. --report-format=json
```

### Container Security
```bash
# Scan image with Trivy
trivy image myapp:latest

# Run as non-root
docker run --user 1000:1000 myapp
```

### Linux Hardening
```bash
# Check listening services
ss -tulnp

# Check SUID binaries
find / -perm -4000 -type f 2>/dev/null

# UFW firewall basics
ufw allow 22/tcp
ufw enable
ufw status
```

## OWASP Top 10 (2021)

<!-- Add OWASP Top 10 categories and mitigation strategies here -->

## Common Patterns

<!-- Add security pipeline patterns, least-privilege IAM patterns here -->
