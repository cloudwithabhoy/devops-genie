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

> **Also asked as:** "Terraform plan shows destroy + recreate for a critical DB — how do you prevent downtime?" — covered above (`terraform state mv` to rename without replace, `lifecycle { ignore_changes }` for drift, `create_before_destroy` for safe replacement, `prevent_destroy` as hard stop).

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

---

## 3. What do you do when your Terraform state file becomes too large?

A state file grows large when all infrastructure is managed in a single root module — hundreds of resources, all in one `terraform.tfstate`. The problems: slow `terraform plan` (reads every resource from AWS), high blast radius on any mistake, and difficult team collaboration.

**Signs it's too large:**
- `terraform plan` takes 10+ minutes
- State file is >10MB
- One team's change triggers plan for another team's resources
- Accidental destroy risk is high (one `terraform destroy` kills everything)

**Solution 1: Split into smaller state files (stacks by domain).**

```
Before (monolith):
  terraform/
    main.tf   ← 500 resources, one state file

After (split by domain):
  terraform/
    networking/    ← VPC, subnets, route tables
    eks/           ← EKS cluster, node groups, Karpenter
    rds/           ← RDS instances, parameter groups
    iam/           ← IAM roles, policies
    monitoring/    ← CloudWatch, Grafana, Prometheus
```

Each directory has its own `terraform.tfstate` in S3. Teams own their slice and only plan/apply changes to their domain.

**Cross-stack references with `terraform_remote_state`.**

```hcl
# eks/main.tf — read VPC outputs from the networking state
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "ap-south-1"
  }
}

resource "aws_eks_cluster" "main" {
  vpc_config {
    subnet_ids = data.terraform_remote_state.networking.outputs.private_subnet_ids
  }
}
```

**Solution 2: Move resources to a new state without destroying them.**

```bash
# Move a resource from the monolith state to the new networking state:
terraform state mv \
  -state=terraform.tfstate \
  -state-out=networking/terraform.tfstate \
  aws_vpc.main \
  aws_vpc.main

# Verify it moved:
terraform -chdir=networking state list | grep aws_vpc
```

The resource stays in AWS — only its state tracking moves. No downtime.

**Solution 3: Use `moved` blocks (Terraform 1.1+) for safe refactoring.**

```hcl
# In the new networking module, declare where resources moved from:
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}
```

**Real scenario:** Our monolith state had 380 resources. `terraform plan` took 18 minutes and scanned every resource in our account. We split into 6 stacks over 2 sprints using `terraform state mv`. After the split, each stack's plan takes 2–3 minutes. Teams work independently without merge conflicts on the state file. Most importantly — a mistake in the monitoring stack can't accidentally destroy the EKS cluster anymore.

---

## 4. Your Terraform apply succeeded, but some resources are not behaving as expected — how do you debug?

`terraform apply` exiting 0 means Terraform successfully made the API calls and the state file was updated. It does not mean the resources are working correctly. Many issues only appear after the resource is running.

**Step 1 — Check what Terraform actually applied.**

```bash
# Show the current state of the resource — exactly what attributes Terraform recorded
terraform show -json | jq '.values.root_module.resources[] | select(.address == "aws_lb.main")'

# Or for a specific resource
terraform state show aws_lb.main
# Shows all attributes: DNS name, ARN, subnets, security groups, access logs config, etc.
```

Compare this against what you expected. If an attribute is wrong (wrong subnet, wrong port), you've found the issue.

**Step 2 — Compare state vs actual AWS resource.**

Terraform state records what it *applied*, but the actual resource might be different if:
- The apply partially failed and was retried
- AWS modified the resource after creation (auto-scaling, version upgrades)
- Another Terraform run or manual change modified it

```bash
# Force Terraform to re-read current state from AWS
terraform plan -refresh-only
# If this shows differences → drift between state and actual resource
# Apply the refresh to update state without changing infrastructure
terraform apply -refresh-only
```

**Step 3 — Check the AWS resource directly.**

Terraform is a layer on top of AWS APIs. If something is wrong, verify at the AWS level:

```bash
# Example: ALB not routing correctly
aws elbv2 describe-load-balancers --names my-alb
aws elbv2 describe-listeners --load-balancer-arn <alb-arn>
aws elbv2 describe-target-groups --load-balancer-arn <alb-arn>
aws elbv2 describe-target-health --target-group-arn <tg-arn>

# Example: EKS node group not joining the cluster
aws eks describe-nodegroup --cluster-name prod --nodegroup-name general
# Look for: "status": "ACTIVE" vs "DEGRADED"
# "health.issues" array shows the exact problem
```

**Step 4 — Check CloudTrail for what actually happened during apply.**

```bash
# What API calls did Terraform make during the apply?
aws cloudtrail lookup-events \
  --start-time "2024-01-15T14:00:00Z" \
  --end-time "2024-01-15T14:30:00Z" \
  --lookup-attributes AttributeKey=Username,AttributeValue=<ci-role-name> \
  | jq '.Events[] | {time: .EventTime, api: .EventName, resource: .Resources}'
```

This shows the exact sequence of API calls, responses, and any errors that AWS returned — even if Terraform considered the apply successful.

**Step 5 — Enable Terraform detailed logging.**

```bash
# Set log level to TRACE — extremely verbose, shows every API call and response
export TF_LOG=TRACE
export TF_LOG_PATH=/tmp/terraform-debug.log
terraform apply

# Check the log for errors
grep -i "error\|failed\|denied" /tmp/terraform-debug.log

# The log shows:
# - Which API calls Terraform made
# - The full request and response (including AWS error codes)
# - How Terraform interpreted the response
```

**Step 6 — Common "apply succeeded but resource broken" scenarios.**

**Scenario: Security Group applied but traffic still blocked.**

```bash
# Terraform applied the SG with correct rules, but it's attached to the wrong interface
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --query 'Reservations[*].Instances[*].SecurityGroups'
# Check: is the new SG in the list? Or is there a conflicting SG still attached?
```

**Scenario: IAM role applied but Lambda can't assume it.**

```bash
# Check trust policy on the role
aws iam get-role --role-name my-lambda-role \
  --query 'Role.AssumeRolePolicyDocument'

# Terraform might have applied the role but with wrong trust policy format
# JSON whitespace or ordering issues can cause trust policy rejection
```

**Scenario: RDS parameter group applied but DB still using old parameters.**

```bash
aws rds describe-db-instances --db-instance-identifier prod-postgres \
  --query 'DBInstances[*].{PendingModified:PendingModifiedValues,DBParameterGroups:DBParameterGroups}'

# "PendingModifiedValues" — the parameter is applied but requires a reboot
# "DBParameterGroups.ParameterApplyStatus" = "pending-reboot" → needs reboot
```

Some RDS parameter changes require a reboot to take effect. Terraform applies the parameter group change but doesn't reboot — the database still uses old parameters until rebooted.

**Step 7 — Use `terraform plan` again after debugging.**

After investigating, run a new plan to see if Terraform's desired state now matches what you found:

```bash
terraform plan
# "No changes" → state matches AWS reality, problem is post-configuration (app-level)
# Shows changes → state diverged from AWS, terraform will reconcile on next apply
```

**Real scenario:** After a Terraform apply that added EKS node group autoscaling, pods were still not scheduling on new nodes. `terraform state show aws_eks_node_group.general` showed the right configuration. `aws eks describe-nodegroup` showed `"scalingConfig": {"minSize": 2, "maxSize": 10}` — correct. But pods weren't scheduling. The actual issue: the new node IAM role was applied, but the `aws-auth` ConfigMap in Kubernetes wasn't updated to trust the new role ARN. Terraform applied the AWS-side resources correctly, but the K8s-side ConfigMap update was in a separate module that hadn't been run yet. Running `terraform apply` on the EKS auth module resolved it. Lesson: Terraform apply per-module succeeded, but the full apply across modules was incomplete.

---

## 5. A production Terraform deployment failed halfway — some resources were created and others weren't. How do you recover?

A partial apply is more dangerous than a clean failure. Some resources exist, some don't, and the state file may or may not reflect reality accurately. The recovery approach depends on which resources were created and what depends on what.

**Step 1 — Understand what Terraform recorded in state.**

```bash
# What does Terraform think it created?
terraform state list

# Compare against the resources in your .tf files
# Any resource in .tf but not in state → was not created or failed silently
# Any resource in state but not in .tf → may have been orphaned
```

**Step 2 — Check AWS directly for resources that may or may not exist.**

Don't trust only the state file. Verify against AWS:

```bash
# Example: EKS cluster — did it actually get created?
aws eks describe-cluster --name prod-cluster 2>&1
# "ResourceNotFoundException" → cluster doesn't exist (even if state has it)
# Cluster details returned → it exists

# Example: RDS instance — did it get created?
aws rds describe-db-instances --db-instance-identifier prod-postgres 2>&1
```

**Step 3 — Run `terraform plan` to see what Terraform wants to do next.**

```bash
terraform plan
```

The plan output after a partial apply tells you exactly what's missing:
- Resources Terraform wants to create → they don't exist yet (apply failed before creating them)
- Resources Terraform wants to modify → they exist but need updates
- Resources Terraform wants to destroy → they're in state but shouldn't exist per current .tf config

**Step 4 — Fix any state inconsistencies before re-applying.**

**Case A: Resource exists in AWS but not in state (Terraform would try to re-create it).**

```bash
# Import the existing resource into state
terraform import aws_security_group.app sg-0abc1234def567890
# Now Terraform knows it exists — plan will show "update" instead of "create"
```

**Case B: Resource is in state but doesn't actually exist in AWS (Terraform shows "update" but it would fail).**

```bash
# Remove from state — Terraform will try to create it on next apply
terraform state rm aws_iam_role.broken_role
# Then re-apply — Terraform creates it fresh
```

**Case C: Resource partially created in AWS (e.g., EKS cluster in CREATING state).**

```bash
# Check if the resource is still being created
aws eks describe-cluster --name prod-cluster --query 'cluster.status'
# "CREATING" → wait, it may complete on its own
# "FAILED"   → it failed internally; may need to be deleted and recreated

# If FAILED, delete via AWS CLI then re-apply
aws eks delete-cluster --name prod-cluster
# Wait for deletion, then re-apply Terraform
```

**Step 5 — Use `-target` to apply just the failed resources.**

If you know which specific resources failed, apply only those:

```bash
# Apply only the resources that didn't get created
terraform apply \
  -target=aws_iam_role.lambda_exec \
  -target=aws_lambda_function.order_processor

# Verify those succeeded, then run a full apply
terraform apply
```

`-target` is a recovery tool — don't use it as a regular workflow. A targeted apply can create state inconsistencies if the targeted resources have dependencies on non-targeted ones.

**Step 6 — Address the root cause of the failure.**

```bash
# Check what caused the mid-apply failure
terraform plan 2>&1 | tee /tmp/plan-output.txt
grep -i "error" /tmp/plan-output.txt

# Common root causes:
# - IAM permission error: Terraform role missing permissions for a resource type
# - API throttling: AWS throttled the request (retry usually fixes it)
# - Quota exceeded: VPC limit, EIP limit, etc.
# - Dependency cycle: resource A depends on B, B depends on A
# - Invalid input: variable value not accepted by AWS API
```

For API throttling:

```bash
# Re-run the apply — throttled resources will succeed on retry
# Add parallelism setting to reduce chance of throttling on large deployments
terraform apply -parallelism=5    # Default is 10; lower = fewer concurrent API calls
```

**Step 7 — Restore state from S3 versioning if state is corrupt.**

If the state file itself is corrupted or partially written:

```bash
# List state file versions in S3
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix prod/main/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified]'

# Restore the version from before the failed apply
aws s3api copy-object \
  --copy-source "my-terraform-state/prod/main/terraform.tfstate?versionId=<good-version-id>" \
  --bucket my-terraform-state \
  --key prod/main/terraform.tfstate

# Now re-run terraform plan — state reflects pre-apply reality
terraform plan
```

**Recovery checklist:**

```
1. terraform state list                  → what does Terraform think exists?
2. Verify critical resources in AWS CLI  → do they actually exist?
3. terraform import <resource> <id>      → re-import resources present in AWS but not in state
4. terraform state rm <resource>         → remove resources that are in state but don't exist
5. terraform apply -target=<resource>    → create only the failed resources
6. terraform apply                       → full apply to verify everything is clean
7. Fix the root cause (IAM, quota, throttling)
```

**Real scenario:** A Terraform apply that provisioned a new EKS cluster failed after creating the cluster and node groups, but before creating the Kubernetes RBAC resources (aws-auth ConfigMap update). The cluster existed in AWS. The state file was partially updated — it had the EKS cluster but not the RBAC config. Re-running `terraform apply` without fixing anything: Terraform tried to re-create the EKS cluster (it was in state, then the apply failure removed it from state mid-write). Fix: ran `terraform import aws_eks_cluster.main prod-cluster` to re-add the cluster to state, then `terraform apply` which correctly saw "cluster exists, just apply the missing RBAC resources." Full recovery in 12 minutes.
