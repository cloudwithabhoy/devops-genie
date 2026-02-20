# CI/CD — Hard Questions

---

## 1. End-to-end: Write a Dockerfile, Kubernetes manifest, and CI pipeline for a Python service.

This is the "show me you can connect all the pieces" question. Here's the full production-ready flow.

**The Dockerfile:**

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY src/ ./src/

RUN adduser --disabled-password --gecos "" appuser
USER appuser

EXPOSE 8080
ENTRYPOINT ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

**The Kubernetes manifests (Helm values file — what ArgoCD applies):**

```yaml
# k8s-manifests/apps/order-service/values-prod.yaml
replicaCount: 3

image:
  repository: 123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "abc123"       # CI pipeline updates this on every deploy

resources:
  requests:
    cpu: "200m"
    memory: "256Mi"
  limits:
    memory: "512Mi"   # No CPU limit — avoid CFS throttling

readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 30
  failureThreshold: 3

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1

lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]   # Drain requests before termination

hpa:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

**The Helm chart template (Deployment):**

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Release.Name }}
  template:
    metadata:
      labels:
        app: {{ .Release.Name }}
    spec:
      containers:
        - name: {{ .Release.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 8080
          resources: {{- toYaml .Values.resources | nindent 12 }}
          readinessProbe: {{- toYaml .Values.readinessProbe | nindent 12 }}
          livenessProbe: {{- toYaml .Values.livenessProbe | nindent 12 }}
          lifecycle: {{- toYaml .Values.lifecycle | nindent 12 }}
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: password
      terminationGracePeriodSeconds: 30
```

**The CI pipeline (Jenkinsfile):**

```groovy
@Library('shared-lib@v2.4.1') _

pipeline {
  agent any

  environment {
    ECR_REPO    = '123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service'
    IMAGE_TAG   = "${GIT_COMMIT}"
    MANIFEST_REPO = 'https://github.com/our-org/k8s-manifests.git'
  }

  stages {
    stage('Test') {
      steps {
        sh 'pytest tests/ --cov=src --cov-report=xml'
      }
      post {
        always {
          publishCoverage adapters: [coberturaAdapter('coverage.xml')]
        }
      }
    }

    stage('Build Image') {
      steps {
        sh 'docker build -t $ECR_REPO:$IMAGE_TAG .'
      }
    }

    stage('Scan') {
      steps {
        sh '''
          trivy image \
            --exit-code 1 \
            --severity CRITICAL,HIGH \
            --ignore-unfixed \
            $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Push to ECR') {
      steps {
        sh '''
          aws ecr get-login-password --region ap-south-1 | \
            docker login --username AWS --password-stdin 123456789.dkr.ecr.ap-south-1.amazonaws.com
          docker push $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Update Manifest') {
      when {
        branch 'main'    // Only update prod manifest on main branch
      }
      steps {
        withCredentials([gitUsernamePassword(credentialsId: 'github-token', gitToolName: 'Default')]) {
          sh '''
            git clone $MANIFEST_REPO
            yq -i '.image.tag = "'$IMAGE_TAG'"' k8s-manifests/apps/order-service/values-prod.yaml
            cd k8s-manifests
            git config user.email "jenkins@ci.internal"
            git config user.name "Jenkins CI"
            git add . && git commit -m "ci: update order-service to $IMAGE_TAG [skip ci]"
            git push
          '''
        }
      }
    }
  }

  post {
    success {
      slackSend channel: '#deployments', color: 'good',
        message: "order-service deployed: ${env.IMAGE_TAG} | ${env.BUILD_URL}"
    }
    failure {
      slackSend channel: '#deployments', color: 'danger',
        message: "order-service build FAILED: ${env.BRANCH_NAME} | ${env.BUILD_URL}"
    }
    always {
      cleanWs()
    }
  }
}
```

**What ArgoCD does after the pipeline pushes to the manifest repo:**

ArgoCD detects the `image.tag` change in `values-prod.yaml` (via Git polling or webhook). It runs `helm template` with the new values, diffs against the current cluster state, and applies the change. New pods roll out with the new image. Old pods are terminated after `preStop: sleep 10` to drain in-flight requests. ArgoCD marks the app as `Synced` when the rollout completes.

**Why the CI pipeline doesn't apply to the cluster directly:** If CI credentials are compromised, an attacker can push bad images — but they can only push to ECR and commit to Git. They cannot run arbitrary `kubectl apply` against the cluster. ArgoCD's Git-based deployment model is a security boundary that direct `kubectl apply` in CI can't provide.

---

## 2. What's your approach to secrets management in pipelines — Jenkins, GitHub Actions, GitLab?

The principle is the same across all three tools: secrets never live in pipeline code, never appear in logs, and ideally never touch the CI system at all. The implementation differs by platform.

**Jenkins:**

The built-in Credentials Store is the baseline. Credentials are encrypted at rest, referenced by ID, and masked in console output:

```groovy
pipeline {
  environment {
    // withCredentials injects and masks automatically
    DB_CREDS = credentials('prod-db-creds')   // Creates DB_CREDS_USR + DB_CREDS_PSW
  }
  stages {
    stage('Deploy') {
      steps {
        withCredentials([string(credentialsId: 'stripe-api-key', variable: 'STRIPE_KEY')]) {
          sh './deploy.sh'   // STRIPE_KEY is masked as **** in console output
        }
      }
    }
  }
}
```

For production: Jenkins runs with an IAM instance profile (on EC2) or IRSA (on EKS). Secrets live in AWS Secrets Manager — not in Jenkins at all:

```groovy
script {
  def secret = sh(
    script: "aws secretsmanager get-secret-value --secret-id prod/stripe-key --query SecretString --output text",
    returnStdout: true
  ).trim()
  // Use secret inline — never print it
}
```

**GitHub Actions:**

Repository and environment secrets. Secrets are masked automatically in logs:

```yaml
jobs:
  deploy:
    environment: production    # Uses "production" environment secrets
    steps:
      - name: Deploy
        env:
          STRIPE_KEY: ${{ secrets.STRIPE_API_KEY }}
          DB_URL: ${{ secrets.DATABASE_URL }}
        run: ./deploy.sh
```

For AWS access — OIDC, not long-lived credentials:

```yaml
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
    aws-region: ap-south-1
# GitHub exchanges an OIDC token for a short-lived AWS token (15-min TTL)
# No AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY stored anywhere
```

Trust policy on the IAM role restricts which repo and branch can assume it:

```json
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:our-org/order-service:ref:refs/heads/main"
    }
  }
}
```

Only the `main` branch of `order-service` can assume this role. A fork or a feature branch cannot.

**GitLab CI:**

CI/CD Variables in GitLab Settings → CI/CD → Variables. Mark them as `Protected` (only available on protected branches) and `Masked` (hidden in logs):

```yaml
deploy-prod:
  stage: deploy
  script:
    - echo "$STRIPE_KEY" | docker login ...   # STRIPE_KEY is a GitLab CI variable
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

For AWS: GitLab OpenID Connect to AWS (same OIDC pattern as GitHub Actions):

```yaml
deploy:
  id_tokens:
    AWS_TOKEN:
      aud: https://gitlab.com
  script:
    - >
      aws sts assume-role-with-web-identity
      --role-arn $AWS_ROLE_ARN
      --web-identity-token $AWS_TOKEN
      --role-session-name gitlab-ci
```

**HashiCorp Vault — works with all three.**

Vault generates dynamic credentials that expire after the pipeline run:

```yaml
# GitHub Actions with Vault
- uses: hashicorp/vault-action@v2
  with:
    url: https://vault.company.internal
    method: jwt          # GitHub OIDC token authenticates to Vault
    role: github-deploy
    secrets: |
      secret/prod/stripe key | STRIPE_KEY ;
      secret/prod/db password | DB_PASSWORD
# Credentials injected as env vars, auto-revoked after the job
```

Vault advantages over static secrets: dynamic database credentials (unique per run, expire in 1 hour), full audit log (which pipeline accessed which secret at what time), automatic revocation if a pipeline is aborted.

**What we never do, regardless of platform:**

```yaml
# NEVER — in any CI tool
env:
  DB_PASSWORD: "super-secret-password"   # In Git forever

# NEVER — print secrets
run: echo "The password is $DB_PASSWORD"   # Appears in logs

# NEVER — pass secrets via build args
run: docker build --build-arg STRIPE_KEY=$STRIPE_KEY .
# docker history shows ARG values — anyone who pulls the image sees the secret
```

**Real incident:** A developer ran `printenv` inside a GitHub Actions step to debug an environment issue. The output contained `STRIPE_KEY=sk_live_...` in the job log — visible to every engineer with repository read access. Within 4 minutes of the log being created, we rotated the key. But this meant emergency Stripe key rotation mid-day, with brief payment failures while services picked up the new key. After this: enforced secret scanning on all workflow files (GitHub Advanced Security detects `sk_live_` patterns), and added a pre-commit hook that blocks `printenv` / `echo $SECRET_VAR` patterns in workflow files.

---

## 3. How do you ensure zero-downtime deployment through your pipeline?

Zero-downtime is not automatic — it requires coordinating the application, Kubernetes config, and the pipeline together. Each layer has a specific role.

**Layer 1: Application — must handle graceful shutdown.**

When Kubernetes terminates a pod, it sends SIGTERM. The app has `terminationGracePeriodSeconds` to finish in-flight requests before a SIGKILL. If the app exits immediately on SIGTERM, in-flight requests get a connection reset:

```python
# Python Flask/FastAPI — handle SIGTERM gracefully
import signal, sys

def handle_sigterm(*args):
    # Stop accepting new connections
    # Wait for in-flight requests to complete
    sys.exit(0)

signal.signal(signal.SIGTERM, handle_sigterm)
```

```yaml
# Pod spec
terminationGracePeriodSeconds: 30   # K8s waits up to 30s after SIGTERM before SIGKILL
```

**Layer 2: Kubernetes — prevent traffic to terminating pods.**

Kubernetes removes the pod from Service endpoints only after the readiness probe fails or the pod terminates. But the ingress controller (NGINX, etc.) might still have the old pod IP cached. The `preStop` hook delays the container shutdown long enough for the ingress to update:

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sleep", "10"]    # Wait 10s before container starts shutting down
                                   # In those 10s, ingress removes this pod from upstream
terminationGracePeriodSeconds: 30  # Must be > preStop sleep + app drain time
```

Without `preStop`, there's a race condition: pod terminates → ingress still routes → 502.

**Layer 3: Rolling update config — control the rollout pace.**

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0    # Never reduce available pods below desired count
                          # Old pod only terminates AFTER new pod passes readiness probe
    maxSurge: 1          # Allow 1 extra pod during rollout (temporary over-provisioning)
```

With `maxUnavailable: 0`: K8s creates 1 new pod → waits for it to pass readiness probe → terminates 1 old pod → repeats. At no point are you below the desired replica count.

**Layer 4: Readiness probe — only route traffic when the app is genuinely ready.**

```yaml
readinessProbe:
  httpGet:
    path: /health/ready    # Deep health check: DB connection, cache connection, etc.
    port: 8080
  initialDelaySeconds: 10  # Give app time to start before first probe
  periodSeconds: 5
  failureThreshold: 3      # 3 consecutive failures before pod is marked not-ready
```

The readiness probe must check that the app is ready to serve traffic — not just that the HTTP server started. A Spring Boot app that started the HTTP server but hasn't connected to the database yet should return 503 on `/health/ready`.

**Layer 5: The pipeline — confirm rollout before declaring success.**

```groovy
stage('Deploy') {
  steps {
    // Update manifest (for GitOps)
    sh "yq -i '.image.tag = \"${IMAGE_TAG}\"' values-prod.yaml && git push"

    // Wait for ArgoCD to apply and confirm healthy
    sh "argocd app wait order-service --health --timeout 300"
    // This blocks until all pods are Running + Ready
    // If any pod fails readiness — this fails — pipeline is red

    // Smoke test against real production traffic
    sh "curl -f https://api.prod.com/health"
  }
}
post {
  failure {
    // Auto-rollback: revert the manifest commit
    sh "git revert HEAD && git push"
  }
}
```

**Putting it together — the sequence during a rolling update:**

```
1. Pipeline updates image tag in manifest → pushes to Git
2. ArgoCD detects change → creates 1 new pod with new image
3. New pod starts → readiness probe checks /health/ready (DB, cache)
4. After 10s initial delay, probes start: 200 OK → pod is Ready
5. K8s adds new pod to Service endpoints
6. Ingress controller picks up new endpoint, starts routing ~1% of traffic to new pod
7. preStop hook fires on old pod #1 → sleeps 10s while ingress drains it
8. Old pod receives SIGTERM → gracefully finishes in-flight requests
9. Old pod terminates → K8s removes from endpoints
10. Repeat for each old pod
11. argocd app wait confirms all pods Healthy
12. Pipeline smoke test hits the real endpoint → 200 OK → pipeline succeeds
```

**What breaks zero-downtime (common mistakes):**

- `maxUnavailable: 1` without matching `maxSurge` — momentarily fewer pods than needed
- No `preStop` hook — ingress gets 502s during termination race condition
- Readiness probe that always returns 200 — new (broken) pods get traffic before they're ready
- `terminationGracePeriodSeconds` shorter than actual drain time — SIGKILL cuts requests

**Real scenario:** We had zero-downtime configured everywhere but still saw 0.1% 502s during deploys. Added nginx access log analysis: errors correlated exactly with pod termination events. Root cause: our NGINX ingress had a 30-second upstream keepalive timeout, but pod `preStop` sleep was only 5 seconds. Pods were terminating before NGINX cleared the keepalive connection. Fix: increased `preStop` sleep to 15 seconds (> NGINX keepalive). 502s during deploys dropped to zero.

---

## 4. How do you integrate security scans (SAST/DAST) as part of the CI/CD flow?

Security scans in CI/CD are about shifting security left — catching vulnerabilities before they reach production. The key is making scans fast enough not to block developers, while failing on genuinely critical issues.

**SAST (Static Application Security Testing) — scans source code.**

SAST runs on the code before it's built. It finds: SQL injection patterns, hardcoded secrets, insecure crypto, path traversal, XXE — without executing the code.

**Tools by language:**

```yaml
# Python — Bandit
- name: SAST - Bandit
  run: |
    pip install bandit
    bandit -r app/ \
      -ll \                    # Severity: low and above
      --skip B101 \            # Skip assert-used (common in tests)
      -f json \
      -o bandit-results.json
    # Exit code 1 if HIGH severity issues found

# JavaScript/TypeScript — ESLint security plugin
- name: SAST - ESLint Security
  run: npx eslint --plugin security --rule 'security/detect-sql-injection: error' src/

# Java — SpotBugs + FindSecBugs
- name: SAST - SpotBugs
  run: mvn spotbugs:check
```

**Semgrep — language-agnostic, fast, custom rules:**

```yaml
- name: SAST - Semgrep
  uses: returntocorp/semgrep-action@v1
  with:
    config: >-
      p/security-audit
      p/owasp-top-ten
      p/secrets         # Finds hardcoded secrets
  env:
    SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}
# Results appear as PR annotations in GitHub
```

**Secret scanning — always on, always blocking:**

```yaml
# Detect secrets committed to code
- name: Secret Scan - gitleaks
  uses: gitleaks/gitleaks-action@v2
  # Scans the diff in the PR — fails if any secret patterns found
  # Patterns: AWS keys, Stripe keys, private keys, JWT secrets, connection strings
```

We run gitleaks on every PR. A CRITICAL finding (actual secret pattern matched) blocks merge. This has caught 3 actual secrets in the last year — developers who accidentally committed API keys.

**SCA (Software Composition Analysis) — scans dependencies:**

```yaml
# Python — Safety / pip-audit
- name: SCA - pip-audit
  run: |
    pip install pip-audit
    pip-audit -r requirements.txt \
      --format cyclonedx-json \
      --output sbom.json
    pip-audit -r requirements.txt --fail-on-vuln CRITICAL,HIGH

# Node.js
- name: SCA - npm audit
  run: npm audit --audit-level=high   # Fails if HIGH or CRITICAL vulns in dependencies

# Container image — Trivy (most comprehensive)
- name: SCA - Trivy Image Scan
  run: |
    trivy image \
      --exit-code 1 \
      --severity CRITICAL,HIGH \
      --ignore-unfixed \
      --format sarif \
      --output trivy.sarif \
      $IMAGE_TAG
  # --ignore-unfixed: skip CVEs with no patch available (reduces noise)
  # SARIF output: integrates with GitHub Code Scanning → appears as PR annotations
```

**DAST (Dynamic Application Security Testing) — tests running application.**

DAST runs against a deployed instance. It actually sends attack payloads and observes responses. Can only run after deployment (staging), not in the build stage.

```yaml
# OWASP ZAP — active scan against staging
dast-scan:
  stage: security
  needs: [deploy-staging]    # Runs after staging deploy
  script:
    - docker run -v $(pwd):/zap/wrk owasp/zap2docker-stable zap-api-scan.py
        -t https://staging.myapp.com/openapi.json
        -f openapi
        -r zap-report.html
        -x zap-report.xml
        -I           # Don't fail on alerts (informational run)
        -l WARN      # Report WARN and above
  artifacts:
    reports:
      junit: zap-report.xml
```

We run ZAP in "active scan" mode against staging for every release candidate. Findings are reviewed before production approval. We don't block on ZAP findings automatically — DAST has higher false positive rates than SAST, so we review them manually. Critical DAST findings (actual SQLi, actual auth bypass) become P0 bugs that block the release.

**Pipeline integration — where each type runs:**

```
PR opened:
  → gitleaks (secret scan, 5s)           BLOCK on any finding
  → Semgrep SAST (code scan, 2min)       BLOCK on HIGH+
  → pip-audit / npm audit (deps, 1min)   BLOCK on CRITICAL

After image build:
  → Trivy image scan (5min)              BLOCK on CRITICAL,HIGH unfixed

After staging deploy:
  → OWASP ZAP DAST (15min)              REVIEW (don't auto-block)

Before prod approval:
  → Review DAST report                   HUMAN GATE
  → Check compliance dashboard           HUMAN GATE
```

**Real impact:** We added Trivy to the pipeline when log4shell (CVE-2021-44228) hit. Within 2 hours of the CVE being published, every new build of our Java services failed the Trivy scan. We rebuilt with the patched log4j version, all services updated by end of day. Without pipeline scanning, we'd have found out weeks later when a security researcher or attacker told us.

---

## 5. What strategies do you use to optimize long-running pipelines?

A pipeline over 15 minutes starts slowing down the development loop. Developers stop waiting for feedback and context-switch — then come back to a red build with no memory of the change. Pipeline speed is a developer productivity problem.

**Strategy 1: Parallelize independent stages — biggest win.**

If tests, lint, and security scan can all run independently, don't run them sequentially:

```groovy
stage('Validate') {
  parallel {
    stage('Unit Tests')      { steps { sh 'pytest tests/unit/' } }
    stage('Integration Tests') { steps { sh 'pytest tests/integration/' } }
    stage('Lint')            { steps { sh 'flake8 app/' } }
    stage('SAST')            { steps { sh 'bandit -r app/' } }
  }
}
// Sequential: 5m + 4m + 1m + 2m = 12 minutes
// Parallel:   max(5m, 4m, 1m, 2m) = 5 minutes (+ overhead)
```

In GitHub Actions, parallel jobs run automatically — each `job:` block runs on its own runner concurrently:

```yaml
jobs:
  unit-tests:      # Runs in parallel with lint and sast
    runs-on: ubuntu-latest
    steps: [...]
  lint:
    runs-on: ubuntu-latest
    steps: [...]
  sast:
    runs-on: ubuntu-latest
    steps: [...]
  build:
    needs: [unit-tests, lint, sast]   # Only starts when all three pass
    runs-on: ubuntu-latest
    steps: [...]
```

**Strategy 2: Cache dependencies — stop reinstalling the same packages.**

```yaml
# GitHub Actions — pip dependency cache
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
    restore-keys: |
      ${{ runner.os }}-pip-
# Cache key = hash of requirements.txt
# Cache hit: pip skips download, uses cached packages (30s instead of 4min)
# Cache miss (requirements.txt changed): download + cache for next run
```

```groovy
// Jenkins — Docker layer caching (build on the same agent, layers are cached)
// Or use BuildKit cache with registry backend:
sh '''
  docker buildx build \
    --cache-from type=registry,ref=$ECR_REPO:buildcache \
    --cache-to type=registry,ref=$ECR_REPO:buildcache,mode=max \
    -t $ECR_REPO:$IMAGE_TAG .
'''
// BuildKit caches individual layers in ECR
// Even on different agents, same layer = cache hit
```

**Strategy 3: Fail fast — run quick checks first.**

Order stages from fastest to slowest. A lint failure should be reported in 30 seconds, not after a 5-minute Docker build:

```
Order: lint (10s) → unit tests (1min) → build image (3min) → integration tests (5min) → push (1min)

Wrong order: build image (3min) → integration tests (5min) → lint (10s)
→ Developer waits 8 minutes to find a lint error
```

**Strategy 4: Test sharding — split tests across parallel runners.**

For test suites over 5 minutes, shard them across multiple runners:

```yaml
# GitHub Actions matrix — run tests in 4 parallel groups
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    runs-on: ubuntu-latest
    steps:
      - run: pytest tests/ --shard-id=${{ matrix.shard }} --num-shards=4
      # pytest-shard plugin splits tests deterministically
      # Each shard runs 1/4 of tests
      # Total time: test_suite_time / 4 runners
```

We went from 18-minute integration test run to 5 minutes by sharding across 4 runners.

**Strategy 5: Skip unnecessary stages for low-risk changes.**

Not every commit needs a full pipeline run:

```yaml
# Skip the slow integration tests for docs-only changes
- name: Check changed files
  id: changes
  uses: dorny/paths-filter@v3
  with:
    filters: |
      code:
        - 'src/**'
        - 'tests/**'
        - 'requirements.txt'
      docs:
        - 'docs/**'
        - '*.md'

integration-tests:
  needs: [changes]
  if: needs.changes.outputs.code == 'true'   # Only run if code changed
  runs-on: ubuntu-latest
  steps: [...]
```

Also: `[skip ci]` in commit messages to skip the pipeline entirely for manifest-only commits (to prevent infinite CI loops in GitOps setups).

**Strategy 6: Build only what changed (monorepo).**

In a monorepo with 20 services, don't build all 20 when only one changed:

```bash
# Detect which services changed since last successful build
CHANGED=$(git diff --name-only $LAST_SUCCESSFUL_COMMIT HEAD | \
  grep -oE '^services/[^/]+' | sort -u)

for service in $CHANGED; do
  echo "Building $service"
  docker build -t $ECR_REPO/$service:$GIT_COMMIT ./$service
done
```

**Real impact:** Before optimization:
- Sequential stages: 22 minutes
- No dependency caching: 4-minute pip install on every run
- No test sharding: 14-minute test suite

After:
- Parallel stages: 8 minutes wall-clock
- Pip cache: 25 seconds on cache hit
- 4-way test sharding: 4 minutes
- Total: **8 minutes** (from 22). Developer feedback loop cut by 63%.

---

## 6. How do you handle concurrent deployments from multiple branches?

Concurrent deployments become a problem when two branches deploy to the same environment simultaneously — they overwrite each other, or leave the environment in a partially deployed state.

**Problem 1: Two PRs deploy to staging at the same time.**

```
Branch A deploys order-service v1.1 to staging
Branch B deploys order-service v1.2 to staging (simultaneously)
Result: v1.1 and v1.2 race — whichever finishes last "wins"
QA tests what they think is v1.1 but it's actually partially v1.2
```

**Solution: Serialized deployments per environment.**

Only one deployment to a given environment at a time. Implement with a Kubernetes Lease (distributed lock) or a CI-level concurrency group:

```yaml
# GitHub Actions — concurrency groups
jobs:
  deploy-staging:
    concurrency:
      group: staging-deploy-${{ github.repository }}
      cancel-in-progress: true   # Cancel older pending deploy if a newer one arrives
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh staging
```

With `cancel-in-progress: true`: if branch A is deploying and branch B starts, branch A is cancelled. Branch B always wins. Last-merged branch deploys. No race condition.

```groovy
// Jenkins — throttle concurrent builds per environment
options {
  throttleJobProperty(
    throttleOption: 'project',
    maxConcurrentTotal: 1,    // Only 1 staging deploy at a time
    categories: ['staging-deploy']
  )
}
// Second deploy queues and waits for the first to finish
```

**Problem 2: Feature branch deploys to production accidentally.**

Production should only get deployments from `main`. Branch protection + pipeline conditions:

```yaml
# GitHub Actions — only deploy to prod from main
deploy-production:
  if: github.ref == 'refs/heads/main' && github.event_name == 'push'
  environment: production
  steps:
    - run: ./deploy.sh prod
```

```groovy
// Jenkins — conditional deploy
stage('Deploy to Prod') {
  when {
    branch 'main'    // Only runs when building the main branch
  }
  steps { ... }
}
```

**Problem 3: Helm release locked by concurrent deployment.**

```
Error: another operation (install/upgrade/rollback) is in progress
```

This happens when two pipelines try to `helm upgrade` the same release simultaneously. One succeeds, the other fails with a lock error.

```bash
# Detect and handle Helm lock
RELEASE_STATUS=$(helm status order-service -n prod --output json | jq -r '.info.status')

if [ "$RELEASE_STATUS" = "pending-upgrade" ]; then
  echo "Helm release locked — rolling back stuck release"
  helm rollback order-service -n prod
  sleep 5
fi

# Now proceed with the upgrade
helm upgrade order-service ./charts/order-service -f values-prod.yaml
```

Or use Helm's `--atomic` flag — it automatically rolls back on failure and releases the lock:

```bash
helm upgrade --install order-service ./charts/order-service \
  -f values-prod.yaml \
  --atomic \              # Rolls back automatically if upgrade fails
  --timeout 5m \
  --cleanup-on-fail
```

**Problem 4: Per-PR preview environments for parallel testing.**

Instead of fighting over a shared staging environment, each PR gets its own ephemeral environment:

```yaml
# GitHub Actions — deploy to a unique staging namespace per PR
deploy-preview:
  steps:
    - name: Deploy to PR namespace
      run: |
        NAMESPACE="preview-pr-${{ github.event.pull_request.number }}"
        kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
        helm upgrade --install order-service ./charts/order-service \
          -f values-staging.yaml \
          --set image.tag=${{ github.sha }} \
          --namespace $NAMESPACE
        echo "Preview URL: https://pr-${{ github.event.pull_request.number }}.staging.myapp.com"

cleanup-preview:
  if: github.event.action == 'closed'   # PR closed or merged
  steps:
    - run: kubectl delete namespace preview-pr-${{ github.event.pull_request.number }}
```

Each PR gets `preview-pr-123.staging.myapp.com`. QA can test PR 123 and PR 124 simultaneously on isolated environments. No conflicts, no "whose staging is this?" confusion.

**Real scenario:** Our team of 8 engineers was sharing one staging environment. We'd regularly see: "who deployed to staging? My tests are failing because some other service changed." Staging was a constant source of confusion. After implementing PR preview environments on Kubernetes using namespace isolation: each PR has its own environment, deploys take 90 seconds, environments are cleaned up automatically on PR close. QA throughput increased because reviewers could test PRs independently without waiting for staging to be "free."
