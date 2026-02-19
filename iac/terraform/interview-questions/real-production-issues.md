# Terraform — Real Production Issues

---

## 1. Your Terraform plan shows unexpected resource replacement — how do you stop it without breaking the pipeline?

Unexpected replacement is one of the most dangerous things in Terraform. A `~` (update in-place) is safe. A `-/+` (destroy and recreate) on a production RDS instance, EKS cluster, or ElastiCache is not. Here's exactly how I handle it.

**Step 1 — Understand why Terraform wants to replace it.**

The plan output tells you which attribute is forcing replacement. Look for `# forces replacement`:

```
# aws_db_instance.prod must be replaced
-/+ resource "aws_db_instance" "prod" {
      ~ identifier = "prod-db" -> "prod-postgres" # forces replacement
    }
```

Common replacement triggers:
- **Name/identifier change** — RDS, EKS, security groups can't be renamed in-place. Any rename → destroy + recreate.
- **AZ or subnet change** — Moving a resource to a different AZ requires recreation.
- **Encryption added after creation** — Adding `storage_encrypted = true` to an existing unencrypted RDS forces replacement.
- **Engine version jump** — Some RDS major version upgrades require recreation, not in-place upgrade.
- **`create_before_destroy` missing** — Without it, Terraform destroys the old resource first, creating a gap.

**Step 2 — Decide: is this replacement intentional or a mistake?**

If it's a **mistake** (someone accidentally changed an identifier in a variable):

Fix the `.tf` code to match what's currently deployed. Plan will then show "No changes" for that resource.

If it's **unavoidable** (you genuinely need to change a forcing attribute), proceed to the options below.

---

**Option A: `terraform state mv` — rename in state without touching infrastructure.**

If you renamed the resource in your `.tf` file but the actual infrastructure hasn't changed, `state mv` updates the state to match the new name — no replacement needed:

```bash
# Old name in .tf: aws_elasticache_cluster.cache
# New name in .tf: aws_elasticache_cluster.redis
terraform state mv aws_elasticache_cluster.cache aws_elasticache_cluster.redis

# Now terraform plan shows "No changes"
```

This is the most commonly needed fix. We use it whenever we refactor resource names.

---

**Option B: `lifecycle { ignore_changes = [...] }` — ignore drift on specific attributes.**

```hcl
resource "aws_db_instance" "prod" {
  engine_version = "14.9"

  lifecycle {
    ignore_changes = [engine_version]  # AWS manages minor version, Terraform ignores it
  }
}
```

Use when AWS auto-modifies an attribute (minor version updates, tags added by AWS services) and you don't want Terraform to fight it.

---

**Option C: `lifecycle { create_before_destroy = true }` — safer replacement when unavoidable.**

```hcl
resource "aws_security_group" "app" {
  lifecycle {
    create_before_destroy = true  # New SG created → references updated → old SG destroyed
  }
}
```

Without this, Terraform destroys the old security group first. Instances lose their SG for a brief moment → connectivity gap. `create_before_destroy` eliminates that window.

---

**Option D: `lifecycle { prevent_destroy = true }` — hard stop on production resources.**

```hcl
resource "aws_db_instance" "prod" {
  lifecycle {
    prevent_destroy = true  # terraform apply with -/+ will hard-fail here
  }
}
```

With this in place, any plan that includes replacement of this resource will fail immediately with:
```
Error: Instance cannot be destroyed
```

This forces a conscious human decision — someone must remove `prevent_destroy` and open a PR, which goes through review, before the resource can be replaced. We put this on every RDS instance and EKS cluster in production.

---

**Option E: `-target` to unblock the pipeline while investigating.**

```bash
# Apply everything except the risky resource
terraform apply -target=module.vpc -target=aws_iam_role.ci_deploy
# Intentionally skip aws_db_instance.prod — investigate separately
```

Use this as a temporary measure only. `-target` can leave state inconsistent if overused.

---

**How we catch this in CI before it reaches production:**

Our Jenkins pipeline runs `terraform plan` on every PR and scans the output for replacements:

```groovy
stage('Terraform Plan') {
  steps {
    script {
      sh 'terraform plan -out=tfplan'
      def planText = sh(script: 'terraform show -no-color tfplan', returnStdout: true)
      if (planText.contains('must be replaced')) {
        error("Plan contains resource replacement — manual review required before merge.")
      }
    }
  }
}
```

Any PR with a `-/+` in the plan gets a failed pipeline check. An engineer must explicitly review it. Replacements never sneak into production automatically.

**Real scenario:** A developer refactored our Terraform module and renamed `aws_elasticache_cluster.cache` to `aws_elasticache_cluster.redis`. The plan showed `-/+` — destroy the existing ElastiCache cluster and create a new one. In production. During business hours. That's a full cache flush and ~5 minutes of every database query hitting the DB directly. The pipeline's replacement check caught it before merge. Fix: `terraform state mv`. Plan went to "No changes." Zero downtime, zero cache flush. If the check hadn't existed, we'd have merged it thinking it was just a rename.

---

## 2. You accidentally ran `terraform destroy` on production — what do you do?

This is a real incident. It happens when someone's workspace is configured against prod instead of staging, or when a cleanup script runs in the wrong directory. Here's exactly how we handle it.

**The first 2 minutes — stop the bleeding:**

If the destroy is still running, interrupt it immediately:

```bash
Ctrl+C    # Interrupt terraform destroy mid-execution
```

`terraform destroy` is sequential — it destroys resources in dependency order. Interrupting it stops further destruction. The state file will reflect what was destroyed so far and what was not. This limits the blast radius.

**Step 1 — assess what was actually destroyed.**

```bash
# See what's left in the state
terraform state list

# Compare against what should exist
git show HEAD:terraform/environments/prod/terraform.tfstate | python3 -c "
import json,sys
state = json.load(sys.stdin)
for r in state['resources']:
    print(r['type'] + '.' + r['name'])
"
```

S3 state versioning is your best friend here — you can compare the current state file against the version from 10 minutes ago to see exactly what's missing.

**Step 2 — check if resources are actually gone or just removed from state.**

`terraform destroy` removes resources from both state AND AWS. But some resources take time to fully delete. Check AWS console or CLI:

```bash
# Is the RDS instance actually gone or still deleting?
aws rds describe-db-instances --db-instance-identifier prod-postgres

# Are EC2 instances terminated or just removed from state?
aws ec2 describe-instances --filters "Name=tag:Environment,Values=prod"
```

Some AWS resources (RDS with deletion protection, S3 buckets with objects) will fail to delete even if `terraform destroy` runs. If `prevent_destroy = true` was set — those resources are still there, just not in state anymore.

**Step 3 — recover, in order of how fast you can restore service:**

**Option A: Restore state from S3 versioning (fastest).**

S3 versioning means every state write is a new version. Find the version from before the destroy:

```bash
# List state file versions
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix prod/eks-cluster/terraform.tfstate

# Restore the previous version
aws s3api copy-object \
  --copy-source my-terraform-state/prod/eks-cluster/terraform.tfstate?versionId=<VERSION_ID> \
  --bucket my-terraform-state \
  --key prod/eks-cluster/terraform.tfstate
```

Now `terraform state list` shows the resources again. `terraform plan` will show what doesn't actually exist anymore. Run `terraform apply` to recreate only the missing resources.

This is why S3 versioning on the state bucket is non-negotiable. We discovered this the hard way.

**Option B: `terraform apply` to recreate from code.**

If state restoration isn't possible, run `terraform apply` on the existing code. Terraform will recreate everything that's in the code but not in state. Problem: any stateful resources (RDS with data, EBS volumes with data) will be recreated empty. The infrastructure comes back but the data is gone unless you have backups.

```bash
terraform apply    # Recreates all destroyed resources from code
```

For RDS: after it's recreated, restore from the most recent automated snapshot:

```bash
# Find the latest automated snapshot taken before the destroy
aws rds describe-db-snapshots \
  --db-instance-identifier prod-postgres \
  --snapshot-type automated \
  --query 'DBSnapshots | sort_by(@, &SnapshotCreateTime) | [-1]'

# Restore from snapshot to the new instance
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-postgres-restored \
  --db-snapshot-identifier rds:prod-postgres-2024-01-15-03-00
```

**Option C: `terraform import` resources that survived.**

Resources that were protected (`prevent_destroy`, deletion protection) or were still deleting when you interrupted the destroy are still in AWS but no longer in state. Import them back:

```bash
terraform import aws_db_instance.prod prod-postgres
terraform import aws_s3_bucket.data my-prod-data-bucket
```

After importing, run `terraform plan`. If it shows no changes, you're recovered. If it shows changes, reconcile code vs actual state before applying.

**Communicate immediately:**

Post in the incident channel within the first minute:
> "Terraform destroy accidentally ran against prod at [time]. [X] resources affected. Assessing impact. Next update in 10 minutes."

Don't wait until you have a fix. Stakeholders need to know now.

**Prevention — what we put in place after this incident:**

```hcl
# 1. prevent_destroy on every stateful production resource
resource "aws_db_instance" "prod" {
  lifecycle {
    prevent_destroy = true    # terraform destroy fails here with an error
  }
}

# 2. RDS deletion protection at the AWS level (independent of Terraform)
resource "aws_db_instance" "prod" {
  deletion_protection = true    # Cannot be deleted via AWS API either
}
```

```bash
# 3. Workspace validation in CI — refuse to run in prod without explicit approval
if [[ $(terraform workspace show) == "prod" ]]; then
  echo "Running in PROD workspace — require manual approval"
  read -p "Type 'yes-destroy-prod' to confirm: " confirm
  [[ "$confirm" == "yes-destroy-prod" ]] || exit 1
fi
```

```hcl
# 4. Separate AWS accounts for staging vs prod (not just workspaces)
# terraform.tfvars for staging: account = "123456789 (staging account)"
# terraform.tfvars for prod:    account = "987654321 (prod account)"
# Even if you run the wrong workspace, credentials won't work in the wrong account
```

**Real scenario:** A data engineer ran `terraform destroy` in what they thought was the `staging` workspace. The terminal prompt showed `prod` — they were in a different tmux pane than they thought. `prevent_destroy` stopped the RDS instance from being deleted. The EKS node group was destroyed before they interrupted it. Recovery: `terraform apply` recreated the node group in 8 minutes. RDS was never touched. If `prevent_destroy` wasn't set on RDS, that would have been a full data restore from backup — 4+ hours of downtime.
