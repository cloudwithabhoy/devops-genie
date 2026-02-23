# Terraform — Scenario-Based Questions

---

## 1. You deleted all your Terraform config files and ran `terraform apply` — what happens and how do you recover?

This scenario tests understanding of how Terraform's state file and configuration files relate to each other — specifically, that the state file is authoritative about what exists, and configuration is what you want to exist.

**What happens when you run `terraform apply` with no config files:**

Terraform compares the desired state (your `.tf` files) against the recorded state (the `.tfstate` file). With no `.tf` files, the desired state is empty — you want nothing. The recorded state still has all your resources. The diff: "these resources exist in state but are not in configuration — they must be destroyed."

```bash
# After deleting all .tf files
terraform apply

# Output:
# Plan: 0 to add, 0 to change, 47 to destroy.
#
# Terraform will perform the following actions:
#   # aws_vpc.main will be destroyed
#   # aws_eks_cluster.prod will be destroyed
#   # aws_rds_instance.prod will be destroyed
#   ...
```

If you confirm this apply, Terraform destroys all 47 resources in the state file. **This is one of the most catastrophic mistakes in infrastructure automation** — it's irreversible for stateful resources like RDS, and even stateless resources like EKS clusters take 30–60 minutes to recreate.

**Why this matters: the state file is the source of truth about what Terraform manages.**

The `.tf` files describe desired state. The `.tfstate` file records what Terraform knows it created. When they diverge, Terraform reconciles by making reality match the config. No config = reality should be empty.

**How to recover (if you haven't confirmed the destroy):**

If you ran `terraform plan` and saw the destroy plan but haven't applied yet — you're safe. Restore your config:

```bash
# Option 1: Restore from git
git log --oneline     # Find the last good commit
git checkout HEAD -- .   # Restore all .tf files
terraform plan        # Verify: "No changes" or only expected changes

# Option 2: Restore from CI/CD artifact
# Most pipelines archive the terraform plan and config as artifacts
# Retrieve and restore from the artifact store
```

**If you already ran the apply — recovery options:**

The answer to "can you recover?" depends on what was destroyed:

**For stateless resources (EC2 instances, security groups, IAM roles, EKS nodes):**

Restore the config files from git and `terraform apply` again — Terraform recreates them from scratch. The resource IDs will be different (new instance IDs, etc.) but the infrastructure is equivalent.

**For stateful resources (RDS, EFS, S3 buckets with data):**

These were actually deleted. Data may be gone.

```bash
# Check if RDS has a final snapshot (depends on your config)
aws rds describe-db-snapshots \
  --query 'DBSnapshots[?DBInstanceIdentifier==`prod-postgres`]'

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-postgres-restored \
  --db-snapshot-identifier <snapshot-id>
```

If `skip_final_snapshot = false` in your Terraform RDS config, AWS takes a final snapshot before deletion. If `skip_final_snapshot = true` (a common mistake in dev environments that gets copied to prod) — data is gone.

**For S3 buckets:**

S3 buckets with versioning enabled retain all object versions even after the bucket is "deleted" by Terraform if you have `force_destroy = false`. If `force_destroy = true` — bucket and all contents are deleted.

**How to prevent this:**

**1. Remote state in S3 — always, for any production infra.**

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

Remote state is separate from your config files. Even if you delete all `.tf` files, the state in S3 is untouched and can be examined before you apply anything.

**2. `prevent_destroy` lifecycle on critical resources.**

```hcl
resource "aws_db_instance" "prod" {
  ...
  lifecycle {
    prevent_destroy = true
    # Terraform will error rather than destroy this resource
    # Error: "Instance cannot be destroyed" — must remove this block manually first
  }
}
```

`prevent_destroy = true` makes Terraform refuse to destroy that resource. Even if your config disappears and reappears without this block, Terraform will error before destroying — forcing a human decision.

**3. State file backup.**

```hcl
# S3 backend with versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration { status = "Enabled" }
}
```

Every `terraform apply` that changes state creates a new version in S3. If the state itself gets corrupted, you can restore the previous state version.

**4. Git for config, always.**

Every `.tf` file should be in version control. The delete-all-configs scenario starts with a human mistake — git prevents accidental deletions from being permanent.

**Real scenario:** A contractor on our team was cleaning up a deprecated test environment. They ran `rm -rf terraform/` from the wrong directory — deleting the production Terraform config. They then ran `terraform init && terraform plan` out of habit before realizing the plan showed destroying 60+ resources. They stopped before `apply`. We restored the config from git in 2 minutes (`git checkout HEAD -- terraform/`). The state file in S3 was untouched. `terraform plan` after restore showed zero changes — infra was intact. The lesson: remote state saved us. If the state had been local (`.terraform/terraform.tfstate`), it would have been deleted too, and recovery would have required manually importing 60+ resources.

---

## 2. How do you use Terraform workspaces to manage multiple environments in a single AWS account?

Terraform workspaces allow a single set of configuration files to manage multiple, isolated state files. The practical question is: when are workspaces the right pattern, and when does a separate directory-per-environment work better?

**What workspaces are:**

```bash
# Default workspace always exists
terraform workspace list
# * default

# Create environments as workspaces
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

terraform workspace list
#   default
#   dev
# * prod    (current — asterisk shows active)

terraform workspace show
# prod
```

Each workspace has its own state file in the backend. The same configuration runs against each workspace, producing isolated infrastructure.

**The key: `terraform.workspace` variable.**

Workspaces are only useful if your configuration differentiates behavior by workspace:

```hcl
locals {
  env = terraform.workspace   # "dev", "staging", "prod"

  # Environment-specific sizing
  instance_config = {
    dev = {
      instance_type    = "t3.small"
      min_size         = 1
      max_size         = 2
      rds_class        = "db.t3.micro"
      multi_az         = false
    }
    staging = {
      instance_type    = "m5.large"
      min_size         = 2
      max_size         = 4
      rds_class        = "db.r6g.large"
      multi_az         = false
    }
    prod = {
      instance_type    = "m5.xlarge"
      min_size         = 4
      max_size         = 20
      rds_class        = "db.r6g.2xlarge"
      multi_az         = true
    }
  }
}

resource "aws_db_instance" "app" {
  instance_class        = local.instance_config[local.env].rds_class
  multi_az              = local.instance_config[local.env].multi_az
  identifier            = "app-${local.env}"   # app-dev, app-staging, app-prod
  ...
}
```

**Resource naming — every resource gets the workspace as a suffix/prefix.**

```hcl
resource "aws_s3_bucket" "app_data" {
  bucket = "mycompany-app-data-${terraform.workspace}"
  # dev:     mycompany-app-data-dev
  # staging: mycompany-app-data-staging
  # prod:    mycompany-app-data-prod
}

resource "aws_eks_cluster" "main" {
  name = "eks-${terraform.workspace}"
}
```

Without the workspace suffix, all environments would collide on the same resource names.

**S3 backend with workspace support:**

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "app/terraform.tfstate"
    # With workspaces, the actual key becomes:
    # app/env:/dev/terraform.tfstate
    # app/env:/staging/terraform.tfstate
    # app/env:/prod/terraform.tfstate
    region         = "ap-south-1"
    dynamodb_table = "terraform-state-lock"
  }
}
```

Terraform automatically prefixes the state key with `env:/<workspace>/` — each workspace's state is isolated.

**Multi-team usage — separate workspaces per team using the same AWS account:**

```bash
# Team A creates their workspace
terraform workspace new team-a-dev
terraform workspace new team-a-prod

# Team B creates theirs
terraform workspace new team-b-dev
terraform workspace new team-b-prod

# Each team switches to their workspace before applying
terraform workspace select team-a-dev
terraform apply -var="team=team-a"
```

Combined with IAM policies, each team can be limited to creating resources with their team prefix:

```json
{
  "Condition": {
    "StringLike": {
      "ec2:ResourceTag/Team": "team-a"
    }
  }
}
```

**When workspaces are the right pattern:**

- Environments are structurally identical (same resources, just different sizes)
- A single team manages all environments
- Number of environments is predictable and small (dev/staging/prod)
- You want to share modules without duplicating directory structure

**When workspaces are the WRONG pattern (use separate directories instead):**

```
Use separate directories when:

1. Environments have significantly different architecture
   (prod has EKS + RDS + ElastiCache; dev has a single EC2 with Docker)

2. Different teams own different environments
   (another team's `terraform apply` on the wrong workspace could destroy prod)

3. Blast radius isolation is critical
   (a bug in .tf code + accidental workspace = wrong environment modified)
```

```
Directory-per-environment pattern (more isolation):
terraform/
├── modules/          # Shared modules
│   ├── eks/
│   └── rds/
├── dev/
│   ├── main.tf      # Uses modules, dev-specific values
│   └── backend.tf   # Key: "dev/terraform.tfstate"
├── staging/
│   ├── main.tf
│   └── backend.tf   # Key: "staging/terraform.tfstate"
└── prod/
    ├── main.tf
    └── backend.tf   # Key: "prod/terraform.tfstate"
```

Separate directories: `terraform apply` in the `prod/` directory can only affect production — you cannot accidentally apply it to dev. Each directory has its own state file. The cost: code is slightly duplicated (each directory references modules, but the variable values are explicit per environment).

**Workspace pitfalls:**

```bash
# Most dangerous: running apply in the wrong workspace
terraform workspace select prod
terraform apply    # Intended for dev — now running against prod

# Always verify before apply:
terraform workspace show
# prod
# → stop, switch workspace if wrong
```

Some teams add a guard:

```hcl
locals {
  # Fail loudly if workspace is unknown
  valid_workspaces = ["dev", "staging", "prod"]
  _ = contains(local.valid_workspaces, terraform.workspace) ? null : tobool("Invalid workspace: ${terraform.workspace}")
}
```

**Real scenario:** Our team started with separate directories per environment (dev/, staging/, prod/) and managed all three manually. After 6 months, the directories had drifted — staging had resources dev didn't, prod had a different VPC CIDR than staging. We migrated to a workspace-based approach: one set of .tf files with `local.env` lookups for all size/configuration differences. The benefit: a change to the EKS node group configuration is applied to dev first, then staging, then prod — same code guaranteed. The risk we mitigated: we added a CI/CD policy that only allows `terraform apply` to prod through a pull request + manual approval step. No direct CLI applies to the prod workspace without PR review.
