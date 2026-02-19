# Terraform — Hard Questions

---

## 1. What problem does IaC solve, and when can it become dangerous?

This question separates engineers who use IaC from engineers who understand it. The "when can it become dangerous" part is what most people miss.

**What IaC solves:**

**1. Reproducibility.** Without IaC, infrastructure is a snowflake — manually configured, impossible to exactly reproduce. When the "prod server" needs to be replaced, nobody knows its full configuration. With Terraform, `terraform apply` on the same code in a different account produces identical infrastructure.

**2. Drift prevention.** In a manually managed environment, someone always "just makes a quick change in the console." Over time, staging and prod diverge. IaC with a CI pipeline that runs `terraform plan` on every PR means drift is caught before it ships — if the console change isn't in the code, the next apply will revert it.

**3. Audit trail.** Every infrastructure change is a Git commit — who made it, what changed, when, and why (commit message + PR review). "Who opened port 3306 to 0.0.0.0/0?" has a clear answer in Git history.

**4. Velocity.** Creating a new environment (staging clone for a new team) used to take a week of manual work. With Terraform modules, it's `terraform workspace new team-b && terraform apply -var-file=team-b.tfvars`. Done in 15 minutes.

**5. Disaster recovery.** If a region fails and you need to spin up infrastructure in another region, IaC is the only way to do it in hours rather than days.

---

**When IaC becomes dangerous:**

**1. `terraform destroy` in the wrong directory.**

Terraform operates on whatever state file the backend points to. If an engineer runs `terraform destroy` in a directory configured to point at production, all production infrastructure is gone.

```bash
# This looks innocent — but if backend.tf points to prod state:
terraform destroy     # Destroys all prod infrastructure
```

Mitigations:
- `prevent_destroy = true` on critical resources (RDS, EKS) — they'll fail even if destroy runs
- Separate AWS accounts for staging and prod (not just separate workspaces)
- Require manual confirmation: `terraform destroy -target` + plan review before any destroy

**2. A plan that looks like a small change but has massive blast radius.**

Changing a security group name in Terraform seems trivial. But security groups can't be renamed in-place — it's a destroy + recreate. If 10 EC2 instances reference that security group, they momentarily lose their network rules during replacement.

```bash
# Plan shows:
-/+ aws_security_group.app (forces replacement)
# What it doesn't show: every EC2 in the ASG briefly loses its SG
```

The plan output describes the resource, not all side effects. An engineer who doesn't know this merges it thinking it's safe.

**3. State file corruption or loss.**

If the state file is corrupted or deleted and there's no backup, Terraform doesn't know what it manages. Every `terraform apply` tries to create everything from scratch — on top of existing infrastructure. You get duplicate resources, naming conflicts, and resource orphans.

Mitigations:
- Remote state in S3 with versioning enabled — roll back to a previous state version
- Never manually edit the state file — use `terraform state mv`, `terraform state rm`

**4. Modules as a single point of failure.**

A shared module used by 10 teams sounds efficient. One breaking change to the module — a removed variable, a renamed output — breaks all 10 teams' pipelines simultaneously.

Mitigation: Version pin modules. Teams reference `module "eks" { source = "terraform-aws-modules/eks/aws"; version = "~> 20.0" }` — not `version = "latest"`. Module updates are explicit, reviewed, and rolled out deliberately.

**5. IaC as a false sense of security.**

A team adopts Terraform but still makes manual changes "just this once." The state file diverges from reality. `terraform plan` starts showing phantom changes. Engineers start using `ignore_changes` to suppress the noise. After 6 months, the IaC describes a different infrastructure than what exists. You get the complexity of IaC without the safety properties.

**The real answer interviewers want:** IaC is powerful because it makes infrastructure reproducible, auditable, and reviewable. It's dangerous when engineers don't understand what Terraform is about to do (plan review is not optional), when critical resources aren't protected against accidental destruction, and when the discipline of "all changes go through code" breaks down.

---

## 2. How does Terraform state work, and what happens when it drifts?

State is the mechanism that lets Terraform know what it has already created. Understanding it deeply explains why most Terraform production issues happen.

**How state works:**

When you run `terraform apply`, Terraform:
1. Reads your `.tf` code (desired state)
2. Reads the state file (recorded state — what Terraform last knew about)
3. Calls provider APIs (AWS) to check current actual state
4. Plans the diff: recorded state → desired state
5. Applies changes, then updates the state file

```
.tf code           state file          AWS
(desired)    ←→   (recorded)    ←→   (actual)
```

The state file records every resource Terraform manages — its type, ID, all attributes, and dependencies. Example state entry for an S3 bucket:

```json
{
  "type": "aws_s3_bucket",
  "name": "artifacts",
  "instances": [{
    "attributes": {
      "bucket": "my-app-prod-artifacts",
      "arn": "arn:aws:s3:::my-app-prod-artifacts",
      "region": "ap-south-1",
      "versioning": [{"enabled": true}],
      ...
    }
  }]
}
```

**Terraform uses the state file as a cache.** It doesn't call AWS to describe every resource on every `plan` — it reads state. This is fast but means the state can go stale.

---

**What is drift?**

Drift is when actual infrastructure (AWS) differs from recorded state (state file), which differs from desired state (.tf code). Drift has two directions:

**1. Infrastructure drifted away from state** — someone made a manual change in AWS.

```
.tf code: security_group allows port 443
state file: security_group allows port 443
AWS actual: security_group allows port 443 AND port 3306 (someone added this manually)
```

Now `terraform plan` shows port 3306 "to be removed" even though nobody put it in the code. This is drift — your infrastructure has something extra that Terraform will fight to remove.

**2. State file diverged from both code and AWS** — rare, but catastrophic.

Happens when state is manually edited, state file is corrupted, or someone ran `terraform state rm` incorrectly. Terraform thinks a resource doesn't exist → tries to create it → fails because it already exists.

---

**Detecting drift:**

```bash
# Refresh the state file to match current AWS reality
terraform plan -refresh-only
# Output: shows what changed in AWS that isn't in state

# Apply the refresh — updates state file, no infra changes
terraform apply -refresh-only
```

We run this weekly in production as a drift detection job:

```groovy
// Jenkins scheduled job — runs Sunday at 2 AM
stage('Drift Detection') {
  sh '''
    terraform plan -refresh-only -out=drift-plan
    terraform show -json drift-plan | python3 check-drift.py
  '''
}
```

If the plan shows changes beyond "No changes", it pages the team — someone made a manual change that isn't in code.

---

**Handling drift:**

**Option A: Accept the drift into code.** Someone manually added a useful security group rule. Add it to `.tf`, run `terraform plan -refresh-only`, then `apply`. Now code matches reality.

**Option B: Revert the drift.** The manual change was a mistake. Run `terraform apply` — Terraform removes the manual change and restores the code-defined state.

**Option C: `ignore_changes` for expected drift.** Some AWS services auto-modify attributes (minor version auto-upgrades, tags added by AWS Config). You can't control this, so tell Terraform to ignore it:

```hcl
resource "aws_db_instance" "prod" {
  lifecycle {
    ignore_changes = [engine_version]   # AWS manages minor version — don't fight it
  }
}
```

**Real scenario:** Our Terraform state showed our production RDS was at `postgres 14.9`. AWS automatically upgraded it to `14.12` (minor version auto-upgrade we hadn't disabled). Every `terraform plan` showed: `~ engine_version = "14.9" -> "14.12"`. Engineers were alarmed — "is Terraform about to downgrade our database?" No — but the drift noise was causing alert fatigue. We added `ignore_changes = [engine_version]` and the noise stopped. The key insight: not all drift is a problem. Know which drift you own vs which drift AWS manages.
