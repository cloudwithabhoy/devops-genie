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
