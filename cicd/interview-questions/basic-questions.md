# CI/CD — Basic Interview Questions

---

## 1. What are artifacts in CI/CD?

> **Also asked as:** "What are artifacts?"

**Artifacts** are the files generated during the build process of a CI/CD pipeline. They are the deployable units of software.

**Common Examples:**
- **Compiled Binaries:** `.exe`, `.bin`, `.o` (for C++/Go).
- **Package Files:** `.jar`, `.war` (for Java), `.whl`, `.tar.gz` (for Python).
- **Docker Images:** A packaged snapshot of the application and its environment.
- **Documentation/Reports:** Test results, code coverage reports, or security scan summaries.

**Why are they important?**
- **Consistency:** You build the artifact once and deploy the *exact same file* to Dev, Staging, and Production. This eliminates "built from a different commit" errors.
- **Traceability:** You can link a specific artifact back to the Git commit and the Jenkins build that created it.
- **Efficiency:** You don't need to rebuild from source code on every server; you just pull the stored artifact.

**Where are they stored?**
We store them in **Artifact Repositories** like:
- **Nexus / JFrog Artifactory:** For JARs, Python packages, etc.
- **Docker Hub / Amazon ECR:** For container images.
- **S3 Buckets:** For static assets or zip files.

---

## 2. What is the difference between Git and GitHub?

> **Also asked as:** "Difference between Git and GitHub?"

- **Git** is a **version control system** (the tool). It is a command-line tool that tracks changes to your code locally on your machine.
- **GitHub** is a **hosting service** for Git repositories (the platform). It provides a web interface, collaboration tools (Pull Requests), and cloud storage for your Git projects.

*Analogy:* Git is like the engine of a car; GitHub is like the garage where you store it and share it with others.
