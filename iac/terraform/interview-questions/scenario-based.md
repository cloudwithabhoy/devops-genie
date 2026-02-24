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

---

## 3. How would you provision infrastructure across 10 AWS regions simultaneously?

Provisioning the same infrastructure in 10 regions requires provider aliasing or a loop-based approach. The right choice depends on whether regions are truly identical or have region-specific config.

**Approach 1: Provider aliases (small number of regions).**

```hcl
# providers.tf
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

# main.tf — explicitly reference each alias
module "vpc_us" {
  source    = "./modules/vpc"
  providers = { aws = aws.us_east_1 }
  cidr      = "10.0.0.0/16"
}

module "vpc_eu" {
  source    = "./modules/vpc"
  providers = { aws = aws.eu_west_1 }
  cidr      = "10.1.0.0/16"
}
```

This works but becomes repetitive across 10 regions.

**Approach 2: `for_each` with dynamic providers (Terraform 1.6+).**

```hcl
# variables.tf
variable "regions" {
  default = [
    "us-east-1", "us-west-2", "eu-west-1", "eu-central-1",
    "ap-southeast-1", "ap-south-1", "ap-northeast-1",
    "ca-central-1", "sa-east-1", "af-south-1"
  ]
}

# Use a wrapper module per region (still needs alias per provider in older Terraform)
# With Terraform 1.6+ provider iteration:
provider "aws" {
  for_each = toset(var.regions)
  alias    = each.key
  region   = each.key
}
```

**Approach 3: Separate Terraform workspaces + CI matrix (most practical for 10 regions).**

```yaml
# GitHub Actions matrix — runs terraform apply in parallel for each region
strategy:
  matrix:
    region: [us-east-1, us-west-2, eu-west-1, eu-central-1, ap-southeast-1,
             ap-south-1, ap-northeast-1, ca-central-1, sa-east-1, af-south-1]

steps:
  - name: Terraform Apply
    run: |
      terraform init -backend-config="region=${{ matrix.region }}"
      terraform workspace select ${{ matrix.region }} || terraform workspace new ${{ matrix.region }}
      terraform apply -auto-approve -var="region=${{ matrix.region }}"
    env:
      AWS_DEFAULT_REGION: ${{ matrix.region }}
```

Each region runs in parallel — 10 `terraform apply` jobs run simultaneously across matrix jobs, completing in the time it takes for one apply.

**State management for multi-region:**

```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "regions/${var.region}/terraform.tfstate"
    region = "us-east-1"   # State bucket is centralised in one region
    dynamodb_table = "terraform-locks"
  }
}
```

Each region gets its own state file path. The DynamoDB lock table is centralised.

**Real scenario:** We had to deploy a WAF + CloudFront + Route 53 health check stack to 6 regions for latency-based routing. We used the CI matrix approach — all 6 regions applied in parallel, total time 4 minutes instead of 24 minutes sequential.

---

## 4. How do you run Terraform safely in CI/CD pipelines?

Running Terraform in CI/CD means automation executes infrastructure changes — no human reviewing each apply. Safety means: no accidental destroys, plan always reviewed before apply, prod requires approval, and secrets are never in CI logs.

**The core pattern: plan on PR, apply on merge, require approval for prod.**

```
PR opened    → terraform plan (read-only, fails fast on errors)
PR merged    → terraform apply to dev/staging (automatic)
Prod release → terraform plan + manual approval → terraform apply to prod
```

**Step 1 — Use OIDC for credential-free authentication (GitHub Actions + AWS).**

Never store AWS access keys in CI secrets. Use OIDC federation — GitHub Actions gets a temporary AWS role via trust policy:

```hcl
# Terraform — create the trust relationship
resource "aws_iam_role" "github_actions_terraform" {
  name = "github-actions-terraform"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Federated = aws_iam_openid_connect_provider.github.arn }
      Action    = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          # Only allow runs from your specific repo and main branch
          "token.actions.githubusercontent.com:sub" = "repo:myorg/myrepo:ref:refs/heads/main"
        }
      }
    }]
  })
}
```

```yaml
# GitHub Actions — no static credentials needed
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/github-actions-terraform
    aws-region: ap-south-1
    # Token expires after 1 hour — no long-lived credentials
```

**Step 2 — Plan on every PR, save the plan as an artifact.**

```yaml
# .github/workflows/terraform.yml
jobs:
  terraform-plan:
    runs-on: ubuntu-latest
    permissions:
      id-token: write     # Required for OIDC
      contents: read
      pull-requests: write  # To post plan as PR comment

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-south-1

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.7.0"

      - name: Terraform Init
        run: terraform init
        working-directory: environments/staging

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        working-directory: environments/staging

      - name: Save plan as artifact
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ github.sha }}
          path: environments/staging/tfplan
          retention-days: 5

      - name: Post plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `\`\`\`\n${{ steps.plan.outputs.stdout }}\n\`\`\``;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
```

The plan is posted as a PR comment — reviewers see exactly what infrastructure will change before approving the merge. The saved `tfplan` artifact ensures the apply uses the exact plan that was reviewed (not a new plan that could differ).

**Step 3 — Apply using the saved plan (not a new plan).**

```yaml
  terraform-apply:
    needs: terraform-plan
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-south-1

      - name: Download saved plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan-${{ github.sha }}
          path: environments/staging

      - name: Terraform Init
        run: terraform init
        working-directory: environments/staging

      - name: Terraform Apply saved plan
        run: terraform apply -auto-approve tfplan
        working-directory: environments/staging
```

Using the saved `tfplan` guarantees the apply executes exactly what was planned and reviewed — no race conditions where infrastructure changed between plan and apply.

**Step 4 — Require manual approval for production.**

```yaml
  terraform-apply-prod:
    needs: terraform-plan-prod
    runs-on: ubuntu-latest
    environment: production    # GitHub environment with protection rules
    # In GitHub: Settings → Environments → production → Required reviewers: [lead-devops]
    # Apply only runs after a reviewer approves in the GitHub UI

    steps:
      - name: Terraform Apply to Production
        run: terraform apply -auto-approve tfplan
        working-directory: environments/prod
```

The `environment: production` setting creates a pause — the apply job waits until a designated reviewer approves in the GitHub UI. The plan output is visible to the reviewer from the PR comment before they approve.

**Step 5 — Block applies that contain resource destruction.**

```yaml
      - name: Check for resource destruction
        run: |
          PLAN_JSON=$(terraform show -json tfplan)
          DESTROYS=$(echo "$PLAN_JSON" | jq '[.resource_changes[] | select(.change.actions | contains(["delete"]))] | length')
          if [ "$DESTROYS" -gt "0" ]; then
            echo "::error::Plan contains $DESTROYS resource destructions. Manual review required."
            echo "Resources to be destroyed:"
            echo "$PLAN_JSON" | jq -r '.resource_changes[] | select(.change.actions | contains(["delete"])) | .address'
            exit 1
          fi
```

Any plan containing a `destroy` fails the CI check. An engineer must explicitly approve the destruction in a separate step — it can never happen automatically.

**Step 6 — Lock the state during applies (prevent concurrent runs).**

The S3 backend + DynamoDB locking prevents two CI jobs from applying simultaneously:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/main/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-lock-prod"   # DynamoDB table for state locking
    encrypt        = true
  }
}
```

If a CI job has the lock, a concurrent job waits or fails with:
```
Error: Error acquiring the state lock
```

Combine with GitHub Actions `concurrency` to prevent queued runs from piling up:

```yaml
concurrency:
  group: terraform-prod
  cancel-in-progress: false   # Don't cancel — let running apply finish
```

**Step 7 — Pin the Terraform version in CI.**

```yaml
- uses: hashicorp/setup-terraform@v3
  with:
    terraform_version: "1.7.0"   # Exact version — no surprises from upgrades
```

Also pin it in `versions.tf`:

```hcl
terraform {
  required_version = "= 1.7.0"   # Fails if a different version is used
}
```

If the CI and local versions differ, plans will differ — causing confusion about what "the plan" actually shows.

**Real scenario:** We had a CI pipeline that ran `terraform apply -auto-approve` directly on every merge to main — no plan review, no approval gate for prod. A developer refactored a variable name, which caused an unexpected resource replacement of an RDS instance (identifier change). The apply ran automatically and destroyed the prod database before anyone saw the plan. After rebuilding from backup (4-hour outage), we implemented: plan-as-PR-comment, saved plan artifact, destruction check (`exit 1` on any `-`), and GitHub environment approval for prod. In the next 12 months, two potential accidental destroys were caught by the destruction check. Zero unplanned infrastructure changes reached production.
