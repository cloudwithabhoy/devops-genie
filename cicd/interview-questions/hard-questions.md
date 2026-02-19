# CI/CD — Hard Questions

---

## 1. Write a parallel Jenkins pipeline (Jenkinsfile).

Most people write a sequential pipeline and think they're done. A parallel pipeline is about running independent stages simultaneously to cut total build time — and getting the error handling right so one failure doesn't silently let the other branches continue.

**The scenario:** We had a pipeline where tests, security scan, and lint were all sequential. Total build time was 18 minutes. Running them in parallel cut it to 7 minutes.

**Parallel stages — the real Jenkinsfile:**

```groovy
pipeline {
  agent any

  environment {
    ECR_REPO    = '123456789.dkr.ecr.ap-south-1.amazonaws.com/my-app'
    IMAGE_TAG   = "${GIT_COMMIT}"
    SONAR_TOKEN = credentials('sonar-token')
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    // Run tests, lint, and security scan in parallel
    stage('Validate') {
      parallel {
        stage('Unit Tests') {
          steps {
            sh 'pytest tests/unit --junitxml=test-results/unit.xml --cov=app --cov-report=xml'
          }
          post {
            always {
              junit 'test-results/unit.xml'
            }
          }
        }

        stage('Integration Tests') {
          steps {
            // Spin up deps via docker-compose, run tests, tear down
            sh '''
              docker-compose -f docker-compose.test.yml up -d
              pytest tests/integration --junitxml=test-results/integration.xml
              docker-compose -f docker-compose.test.yml down
            '''
          }
          post {
            always {
              junit 'test-results/integration.xml'
              sh 'docker-compose -f docker-compose.test.yml down --volumes || true'
            }
          }
        }

        stage('Lint & SAST') {
          steps {
            sh 'flake8 app/ --max-line-length=120'
            sh 'bandit -r app/ -ll'   // Python security linter
            sh '''
              sonar-scanner \
                -Dsonar.projectKey=my-app \
                -Dsonar.sources=app \
                -Dsonar.login=$SONAR_TOKEN
            '''
          }
        }
      }
    }

    stage('Build Image') {
      steps {
        sh 'docker build -t $ECR_REPO:$IMAGE_TAG .'
      }
    }

    // Run image scans in parallel — trivy + checkov (IaC scan)
    stage('Security Scan') {
      parallel {
        stage('Trivy - Image') {
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

        stage('Checkov - IaC') {
          steps {
            sh 'checkov -d terraform/ --framework terraform --soft-fail'
          }
        }
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
      steps {
        withCredentials([gitUsernamePassword(credentialsId: 'github-token', gitToolName: 'Default')]) {
          sh '''
            git clone https://github.com/our-org/k8s-manifests.git
            yq -i '.image.tag = "'$IMAGE_TAG'"' k8s-manifests/apps/my-app/values-staging.yaml
            cd k8s-manifests
            git config user.email "jenkins@ci.internal"
            git config user.name "Jenkins"
            git add . && git commit -m "ci: update my-app image to $IMAGE_TAG [skip ci]"
            git push
          '''
        }
      }
    }

  }

  post {
    success {
      slackSend channel: '#deployments', color: 'good',
        message: "Build SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER} | Image: ${env.IMAGE_TAG}"
    }
    failure {
      slackSend channel: '#deployments', color: 'danger',
        message: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER} | Branch: ${env.BRANCH_NAME}"
    }
    always {
      cleanWs()   // Clean workspace after every build — prevents disk bloat on the agent
    }
  }
}
```

**Key things that separate this from a basic answer:**

- `[skip ci]` in the manifest commit message — without this, pushing to the manifest repo triggers another pipeline build. Infinite loop. Got hit by this in production — 47 pipeline runs before someone noticed.

- `cleanWs()` in the `post { always {} }` block — Docker build layers from previous runs accumulate on the agent. We had a Jenkins agent go to 98% disk after 3 weeks. `cleanWs()` prevents it.

- `--ignore-unfixed` on Trivy — without this, Trivy fails on CVEs that have no fix available yet, blocking every build. You want to fail on fixable critical CVEs only.

- The `post { always {} }` inside the integration test stage tears down `docker-compose` even if tests fail. Without this, containers from failed test runs pile up on the agent and port conflicts start breaking future builds.

**What the interviewer is really testing:** Do you know `parallel {}` syntax? Yes, but more importantly — do you know the operational pain points that come with it? Infinite loops from manifest commits, disk bloat, and orphaned containers are the real interview-differentiating answers.

---

## 2. How do you design and reuse Jenkins Shared Libraries across multiple pipelines?

Shared Libraries are about discipline, not just syntax. The question is: how do you design them so they're actually reusable — not just "shared" in name but specific to one team's quirks.

**The design principles we follow:**

**1. Functions take config maps, not positional args.**

```groovy
// Bad — brittle, order-dependent
def call(String repo, String tag, String region) { ... }

// Good — explicit, extensible, self-documenting
def call(Map config = [:]) {
  def repo    = config.repo    ?: error('repo is required')
  def tag     = config.tag     ?: env.GIT_COMMIT
  def region  = config.region  ?: 'ap-south-1'
  ...
}
```

When you add a new optional parameter, existing callers don't break — they just don't pass the new key and the default applies.

**2. One function, one responsibility.**

```
vars/
├── buildDockerImage.groovy     ← build only
├── pushToECR.groovy            ← push only
├── runTrivy.groovy             ← scan only
├── updateHelmValues.groovy     ← manifest update only
├── notifySlack.groovy          ← notification only
└── deployPipeline.groovy       ← orchestrates the above (opinionated full pipeline)
```

Teams that want the full opinionated flow call `deployPipeline(...)`. Teams with custom needs call individual functions and compose their own flow. Both cases use the same underlying logic.

**3. Version pin in production, float in development.**

```groovy
// Development pipelines — get latest library changes immediately
@Library('shared-lib@main') _

// Production pipelines — explicit version, controlled updates
@Library('shared-lib@v2.4.1') _
```

This prevents a library change from silently breaking all 18 production pipelines at once. We treat library version bumps like dependency upgrades — reviewed, tested in staging pipelines first, then rolled out to prod.

**The full deployPipeline.groovy — what we actually ship:**

```groovy
// vars/deployPipeline.groovy
def call(Map config) {
  def app       = config.app       ?: error('app name required')
  def ecrRepo   = config.ecrRepo   ?: error('ECR repo required')
  def manifests = config.manifests ?: 'k8s-manifests'
  def valuesFile = config.valuesFile ?: "apps/${app}/values-staging.yaml"

  pipeline {
    agent any
    environment {
      IMAGE_TAG = "${GIT_COMMIT}"
    }
    stages {
      stage('Test') {
        steps {
          script { runTests(config.testCmd ?: 'pytest tests/') }
        }
      }
      stage('Build & Scan') {
        parallel {
          stage('Docker Build') {
            steps {
              script { buildDockerImage(repo: ecrRepo, tag: env.IMAGE_TAG) }
            }
          }
          stage('Lint') {
            steps {
              script { runLint(config.lintCmd ?: 'flake8 app/') }
            }
          }
        }
      }
      stage('Security Scan') {
        steps {
          script { runTrivy(image: "${ecrRepo}:${env.IMAGE_TAG}") }
        }
      }
      stage('Push') {
        steps {
          script { pushToECR(repo: ecrRepo, tag: env.IMAGE_TAG) }
        }
      }
      stage('Update Manifest') {
        steps {
          script {
            updateHelmValues(
              manifestsRepo: manifests,
              valuesFile: valuesFile,
              imageTag: env.IMAGE_TAG
            )
          }
        }
      }
    }
    post {
      success { notifySlack(channel: config.slackChannel ?: '#deployments', status: 'success', app: app) }
      failure { notifySlack(channel: config.slackChannel ?: '#deployments', status: 'failure', app: app) }
      always  { cleanWs() }
    }
  }
}
```

**A service's entire Jenkinsfile becomes:**

```groovy
@Library('shared-lib@v2.4.1') _

deployPipeline(
  app: 'order-service',
  ecrRepo: '123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service',
  valuesFile: 'apps/order-service/values-staging.yaml',
  slackChannel: '#orders-team'
)
```

**Real impact:** Before Shared Libraries, each of our 18 services had a ~120-line Jenkinsfile. When Trivy changed its CLI flags in a new version, we had to update 18 files. After migrating to Shared Libraries, that change was one line in `vars/runTrivy.groovy`. We rolled it out to all 18 services in one library version bump — zero individual Jenkinsfile changes required.

---

## 3. End-to-end: Write a Dockerfile, Kubernetes manifest, and Jenkins pipeline for a Python service.

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
  tag: "abc123"       # Jenkins updates this on every deploy

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

**The Jenkins pipeline:**

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

**What ArgoCD does after Jenkins pushes to the manifest repo:**

ArgoCD detects the `image.tag` change in `values-prod.yaml` (via Git polling or webhook). It runs `helm template` with the new values, diffs against the current cluster state, and applies the change. New pods roll out with the new image. Old pods are terminated after `preStop: sleep 10` to drain in-flight requests. ArgoCD marks the app as `Synced` when the rollout completes.

**Why Jenkins doesn't apply to the cluster directly:** If Jenkins credentials are compromised, an attacker can push bad images — but they can only push to ECR and commit to Git. They cannot run arbitrary `kubectl apply` against the cluster. ArgoCD's Git-based deployment model is a security boundary Jenkins can't provide on its own.

---

## 4. How do you migrate from Jenkins to GitHub Actions?

A migration question is testing whether you can think through a real transition — what to preserve, what to rewrite, and how to avoid a big-bang cutover.

**Why teams migrate:** Jenkins requires infrastructure to run (agents, servers, plugins to manage). GitHub Actions is managed — no Jenkins server to maintain, no agents to provision (unless you need self-hosted). For teams on GitHub, Actions integrates natively: no webhook setup, no token management, repository permissions work out of the box.

**The migration approach we'd use — parallel running, service by service:**

```
Phase 1: Set up GitHub Actions foundation
Phase 2: Migrate lowest-risk pipeline (internal tool, no prod impact)
Phase 3: Migrate each service one by one, keep Jenkins as fallback
Phase 4: Decommission Jenkins after all pipelines migrated + 30-day validation
```

**Mapping Jenkins concepts to GitHub Actions:**

| Jenkins | GitHub Actions |
|---|---|
| `Jenkinsfile` | `.github/workflows/ci.yml` |
| `pipeline { agent any }` | `runs-on: ubuntu-latest` |
| `stage('Test') { steps { sh '...' } }` | `- name: Test\n  run: ...` |
| `parallel { stage('A') {...} stage('B') {...} }` | `jobs:` with multiple jobs (parallel by default) |
| `post { success {} failure {} always {} }` | `if: success()` / `if: failure()` / always runs |
| Credentials store | GitHub Secrets (`${{ secrets.MY_SECRET }}`) |
| Jenkins Shared Libraries | Composite actions / Reusable workflows |
| `when { branch 'main' }` | `if: github.ref == 'refs/heads/main'` |
| Build agents (EC2) | Self-hosted runners or `runs-on: ubuntu-latest` |

**The equivalent pipeline in GitHub Actions:**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:

env:
  ECR_REPO: 123456789.dkr.ecr.ap-south-1.amazonaws.com/order-service
  AWS_REGION: ap-south-1

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest tests/ --cov=src

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      id-token: write    # Required for OIDC authentication to AWS
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC — no long-lived keys)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-ecr-push
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and scan image
        run: |
          docker build -t $ECR_REPO:${{ github.sha }} .
          trivy image --exit-code 1 --severity CRITICAL,HIGH --ignore-unfixed \
            $ECR_REPO:${{ github.sha }}

      - name: Push to ECR
        run: docker push $ECR_REPO:${{ github.sha }}

  update-manifest:
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Update Helm values
        uses: actions/checkout@v4
        with:
          repository: our-org/k8s-manifests
          token: ${{ secrets.MANIFEST_REPO_TOKEN }}

      - name: Update image tag
        run: |
          yq -i '.image.tag = "${{ github.sha }}"' apps/order-service/values-prod.yaml
          git config user.email "github-actions@ci.internal"
          git config user.name "GitHub Actions"
          git add . && git commit -m "ci: update order-service to ${{ github.sha }} [skip ci]"
          git push
```

**Key improvements over Jenkins in this migration:**

- **OIDC for AWS authentication** — no long-lived AWS access keys stored as secrets. GitHub Actions gets a short-lived token from AWS STS using OIDC. If a key is compromised in Jenkins, an attacker has long-lived credentials. In Actions, tokens expire in 15 minutes
- **No webhook setup** — GitHub Actions triggers on native GitHub events (push, PR). No Jenkins URL to configure, no CSRF tokens
- **No Jenkins server to maintain** — `ubuntu-latest` runners are managed by GitHub. One less thing to patch and operate

**What doesn't migrate cleanly:**

- **Dynamic parallel stages** — Jenkins Scripted can loop over a list and generate stages. GitHub Actions supports matrix strategy but it's less flexible for truly dynamic cases
- **Complex shared libraries** — Jenkins Shared Libraries are Groovy. GitHub Actions reusable workflows are YAML. Complex imperative logic in Shared Libraries needs to become either composite actions or moved into shell scripts
- **Self-hosted runners** — if your Jenkins agents have special tooling (Terraform, internal tools), you need equivalent self-hosted GitHub runners. This is the main operational work in the migration

**Migration risk mitigation:** We kept Jenkins running in parallel for 6 weeks after migrating each service. Both pipelines ran on every PR. We only decommissioned Jenkins for a service after 6 weeks of clean GitHub Actions runs with no discrepancies. This gave engineers confidence and a fallback if something subtle broke.
