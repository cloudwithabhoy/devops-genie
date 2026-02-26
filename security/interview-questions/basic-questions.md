# Security — Basic Questions

---

## 1. What are SAST and DAST?

> **Also asked as:** "SATS and DATS?" (Common typo in interviews)
> **Also asked as:** "How do you integrate security into CI/CD?"

Both are security testing methodologies used to find vulnerabilities, but they look at the application from different angles.

### SAST (Static Application Security Testing) — "Inside-Out"
- **Analyses:** Source code, byte code, or binaries while the application is **stationary** (not running).
- **When:** Done early in the SDLC (Shift Left).
- **Pros:** Finds vulnerabilities early (SQL injection, hardcoded secrets, insecure libraries); points to the exact line of code.
- **Cons:** High false-positive rate; cannot find runtime environment issues.
- **Tools:** SonarQube, Snyk, Checkmarx.

### DAST (Dynamic Application Security Testing) — "Outside-In"
- **Analyses:** The **running application** by simulating external attacks.
- **When:** Done later in the SDLC (Testing/Staging phase).
- **Pros:** Finds environmental issues (misconfigured headers, SSL issues, authentication flaws); low false-positive rate.
- **Cons:** Cannot point to specific lines of code; requires a fully functional running environment.
- **Tools:** OWASP ZAP, Burp Suite, Rapid7.

---

## 2. What is DevSecOps?

> **Also asked as:** "Explain the DevSecOps lifecycle."

DevSecOps is the practice of integrating security early and throughout the entire CI/CD pipeline. Instead of security being a "final check" before release, it becomes a shared responsibility of the entire team.

**Key Components:**
- **Shift Left:** Testing security as soon as code is written.
- **Automation:** Using tools (SAST, DAST, Image Scanning) that run automatically on every PR.
- **Observability:** Constant monitoring of production for security threats.
