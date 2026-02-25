# Jenkins — Scenario-Based Questions

---

## 1. Which type of Jenkins pipeline do you use and why?

We use **Declarative pipelines** for all production pipelines. The answer isn't "Declarative is better" — it's understanding why it fits the team.

**Declarative is what we chose because:**

- **It's reviewable.** Every engineer on the team can read and understand a Declarative Jenkinsfile without knowing Groovy. When a pipeline fails at 2 AM, the on-call engineer can read the file and understand exactly what each stage does. With Scripted pipelines, you often end up with nested Groovy loops and closures that only the original author understands.

- **It has guardrails.** Declarative syntax validates the pipeline structure before it runs. If you have a syntax error in a `stage` block, Jenkins tells you immediately. Scripted pipelines fail at runtime — halfway through a build.

- **`post {}` blocks are clean.** Declarative has built-in `post { success {} failure {} always {} }` at both the pipeline and stage level. In Scripted, you have to wrap everything in try/catch/finally manually. We had a Scripted pipeline where a developer forgot the `finally` block — docker-compose containers leaked on the agent for weeks.

- **Shared Library compatibility.** When we moved to Shared Libraries, Declarative made it easy to call library functions as steps. The pipeline structure remained readable; complex logic moved to the library.

**When Scripted makes sense:**

Scripted is Groovy — fully programmable. If you need dynamic stage generation (e.g., loop over 15 microservices and create a stage per service), Declarative can't do it cleanly. We have one legacy pipeline that generates stages dynamically from a config file — that one stays Scripted.

```groovy
// Scripted — dynamic stages
def services = ['auth', 'orders', 'payments', 'notifications']
node {
  stage('Build All') {
    def parallelStages = [:]
    services.each { svc ->
      parallelStages[svc] = {
        sh "docker build -t ecr//${svc}:${GIT_COMMIT} ./${svc}"
      }
    }
    parallel parallelStages
  }
}
```

**Real scenario:** We inherited a Scripted monolith pipeline from a previous team — 600 lines of Groovy, multiple nested closures, no comments. When the original author left, nobody could safely modify it. Every change was trial-and-error. We rewrote it in Declarative over a weekend. New pipeline: 80 lines. Every engineer can modify it confidently.

---

## 2. How do you deploy a new Jenkins instance with zero downtime?

> Also asked as: "How do you perform a zero-downtime Jenkins deployment or upgrade?"

The goal is to replace or upgrade Jenkins without any gap in the ability to run builds. The strategy depends on whether you're upgrading an existing Jenkins or migrating to a new one.

**Approach 1: Blue/Green Jenkins deployment (new instance in parallel).**

```
Step 1: Spin up a new Jenkins instance (Green) alongside the existing one (Blue)
Step 2: Restore config from backup or replicate via Configuration as Code
Step 3: Validate Green — test pipelines run correctly
Step 4: Switch ingress/load balancer to point at Green
Step 5: Decommission Blue after a stabilization period
```

**Step 1: Deploy Jenkins with Helm on Kubernetes (infrastructure as code — reproducible).**

```bash
# Add Jenkins Helm chart
helm repo add jenkins https://charts.jenkins.io
helm repo update

# Install new Jenkins (Green) with your values file
helm install jenkins-green jenkins/jenkins \
  --namespace jenkins \
  --values jenkins-values.yaml \
  --set controller.tag=2.440.3-lts
```

**Step 2: Jenkins Configuration as Code (JCasC) — the key to zero-downtime upgrades.**

JCasC stores all Jenkins configuration (plugins, credentials, job configs, security settings) in a YAML file in Git. When a new Jenkins starts, it reads this file and configures itself automatically.

```yaml
# jenkins-casc.yaml (stored in Git)
jenkins:
  systemMessage: "Production Jenkins — managed by JCasC"
  numExecutors: 0
  agentProtocols:
    - "JNLP4-connect"
  securityRealm:
    ldap:
      server: "ldap://corp-ldap:389"
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            permissions: ["Overall/Administer"]
            assignments: ["devops-team"]

credentials:
  system:
    domainCredentials:
      - credentials:
          - aws:
              accessKeyId: "${AWS_ACCESS_KEY_ID}"
              secretAccessKey: "${AWS_SECRET_ACCESS_KEY}"
              id: "aws-prod"

jobs:
  - script: |
      multibranchPipelineJob('my-app') {
        branchSources {
          github {
            id('my-app-github')
            repoOwner('myorg')
            repository('my-app')
          }
        }
      }
```

Without JCasC: migration means manually recreating 50+ jobs and all credentials in the new Jenkins — impossible without downtime. With JCasC: new Jenkins reads the YAML, configures itself in under 2 minutes, fully identical to the old one.

**Step 3: Drain in-progress builds on the old Jenkins before switching.**

```groovy
// Mark old Jenkins as "quiet mode" — accept no new builds, let running ones finish
// Jenkins UI → Manage Jenkins → Quiet Down
// Or via CLI:
java -jar jenkins-cli.jar -s http://jenkins-blue:8080 quiet-down
```

Wait for running builds to complete (or set a timeout), then switch the ingress.

**Step 4: Switch traffic to Green Jenkins.**

```yaml
# Update Kubernetes ingress to point at Green service
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins
spec:
  rules:
  - host: jenkins.mycompany.com
    http:
      paths:
      - backend:
          service:
            name: jenkins-green   # Was jenkins-blue
            port:
              number: 8080
```

Apply this change — it takes effect in seconds. All new browser sessions and webhook deliveries go to Green. In-flight builds on Blue complete normally.

**Approach 2: In-place upgrade with a maintenance window (simpler).**

For minor Jenkins upgrades where Blue/Green is overkill:

```bash
# Back up Jenkins home
tar -czvf jenkins-home-backup-$(date +%Y%m%d).tar.gz /var/lib/jenkins/

# Update the Helm values file with new tag
# controller.tag: 2.440.3-lts → 2.452.1-lts

# Helm upgrade — Kubernetes rolling update handles the pod replacement
helm upgrade jenkins jenkins/jenkins \
  --namespace jenkins \
  --values jenkins-values.yaml \
  --set controller.tag=2.452.1-lts

# Jenkins pod is replaced — downtime is the pod restart time (~60-90 seconds)
# This is not zero-downtime but is acceptable for minor upgrades
```

**The real key to zero-downtime:** Have everything in code. JCasC + Helm + Git = any Jenkins instance is reproducible in under 5 minutes. When that's true, the question of "downtime" changes from "how do we avoid losing config" to "how do we handle the brief switch window" — which is just a load balancer update.
