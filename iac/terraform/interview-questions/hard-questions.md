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

---

## 3. How do you structure Terraform modules for enterprise-scale projects?

At enterprise scale, the question isn't "how do I write Terraform" — it's "how do I structure it so 20 teams can work independently without breaking each other, and so a new environment can be created in minutes."

**The two anti-patterns we see and fix:**

**Anti-pattern 1: One giant `main.tf` for everything.**

```
terraform/
└── main.tf    ← 3,000 lines. Everything in one file. One state file.
                  One team member's change blocks all others.
                  `terraform plan` takes 8 minutes.
                  A destroy here destroys everything.
```

**Anti-pattern 2: Copy-paste per environment.**

```
terraform/
├── prod/main.tf      ← 500 lines
├── staging/main.tf   ← 498 lines (almost identical, slightly different)
└── dev/main.tf       ← 502 lines (same, but with a fix that never got merged to prod)
```

Three copies of the same code diverge immediately. Prod gets a security fix that staging never gets.

**The structure we use:**

```
terraform/
├── modules/                          # Reusable building blocks
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── eks/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── rds/
│   └── microservice/                 # K8s + ECR + IAM for one service
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── environments/                     # Per-environment root modules
│   ├── dev/
│   │   ├── main.tf                  # Calls modules with dev vars
│   │   ├── terraform.tfvars         # Dev-specific values
│   │   └── backend.tf               # Dev S3 state bucket
│   ├── staging/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       ├── terraform.tfvars
│       └── backend.tf
│
└── services/                        # Per-service infrastructure
    ├── order-service/
    │   ├── main.tf                  # Calls the microservice module
    │   ├── variables.tf
    │   └── backend.tf
    └── payment-service/
        ├── main.tf
        └── backend.tf
```

**The `microservice` module — one module for all services:**

```hcl
# modules/microservice/main.tf
variable "service_name" { type = string }
variable "environment"  { type = string }
variable "image_tag"    { type = string }
variable "min_replicas" { type = number; default = 2 }
variable "max_replicas" { type = number; default = 10 }

# ECR repository for this service
resource "aws_ecr_repository" "service" {
  name                 = "${var.service_name}-${var.environment}"
  image_tag_mutability = "IMMUTABLE"
}

# IAM role for the service (IRSA)
resource "aws_iam_role" "service" {
  name = "${var.service_name}-${var.environment}"
  assume_role_policy = data.aws_iam_policy_document.irsa_trust.json
}

# Output: what the calling environment needs
output "ecr_repository_url" { value = aws_ecr_repository.service.repository_url }
output "service_role_arn"   { value = aws_iam_role.service.arn }
```

A new service's entire infrastructure is:

```hcl
# services/new-service/main.tf
module "new_service" {
  source = "../../modules/microservice"

  service_name = "inventory-service"
  environment  = var.environment
  image_tag    = var.image_tag
  min_replicas = var.environment == "prod" ? 3 : 1
}
```

New service: 10 lines of HCL, gets ECR + IAM + Helm values + everything automatically.

**Separate state per layer — critical for blast radius control:**

```
State file 1: environments/prod/     ← VPC, EKS, RDS (shared infra)
State file 2: services/order-service/ ← Order service resources
State file 3: services/payment-service/ ← Payment service resources
```

A developer working on the payment service can't accidentally affect the VPC or the order service. Each state file is independent. `terraform destroy` on `services/payment-service` only destroys payment service resources.

**Module versioning — pin in production, float in dev:**

```hcl
# In production environments — pinned version
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "= 20.8.4"   # Exact version — no surprises
}

# In dev environments — allow minor updates
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"   # Any 20.x, including patch updates
}
```

This prevents a module update from breaking production while allowing dev to test new versions first.

**Real scenario:** We inherited a monorepo Terraform setup where 6 teams shared one state file for an account with 80+ resources. Every `terraform plan` took 9 minutes (refreshing all 80 resources). A junior engineer's mistake in their service's `.tf` block could trigger an unexpected change to the shared RDS instance. We refactored to the layered structure over 6 weeks: separate state per service, separate state for shared infra. `terraform plan` for one service now takes 45 seconds. A mistake in one team's code cannot affect another team's resources.

---

## 4. How do you manage multi-environment (dev/stage/prod) Terraform architecture?

Multi-environment Terraform is the most common architecture challenge. The goal: same code, different configurations, complete isolation between environments.

**The wrong approach — copy-paste directories:**

```
terraform/dev/main.tf     # Copy 1
terraform/staging/main.tf  # Copy 2 (already different)
terraform/prod/main.tf     # Copy 3 (most different — "fixes" that never went back)
```

By month 3, these are three completely different codebases. A security fix in prod never gets to dev.

**The right approach — shared modules + per-environment variables.**

Same module code, different variable values per environment:

```hcl
# environments/prod/main.tf
module "eks" {
  source = "../../modules/eks"

  cluster_name    = "prod-cluster"
  instance_types  = ["m5.xlarge"]
  min_size        = 3
  max_size        = 10
  desired_size    = 5
}

module "rds" {
  source = "../../modules/rds"

  identifier      = "prod-postgres"
  instance_class  = "db.r5.xlarge"
  multi_az        = true          # Prod only
  deletion_protection = true      # Prod only
}
```

```hcl
# environments/staging/main.tf — same modules, different values
module "eks" {
  source = "../../modules/eks"

  cluster_name    = "staging-cluster"
  instance_types  = ["t3.medium"]   # Cheaper
  min_size        = 1
  max_size        = 3
  desired_size    = 2
}

module "rds" {
  source = "../../modules/rds"

  identifier      = "staging-postgres"
  instance_class  = "db.t3.medium"  # Cheaper
  multi_az        = false           # Staging doesn't need HA
  deletion_protection = false
}
```

**Separate backends per environment — non-negotiable:**

```hcl
# environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state-prod"
    key            = "prod/main/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-lock-prod"
    encrypt        = true
  }
}

# environments/staging/backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state-staging"
    key            = "staging/main/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-lock-staging"
    encrypt        = true
  }
}
```

Separate state buckets mean a misfire in staging can never corrupt prod state. Separate DynamoDB tables mean staging and prod locks are independent.

**Separate AWS accounts per environment (strongest isolation):**

Beyond separate state files, using separate AWS accounts per environment means:
- An IAM misconfiguration in dev can't affect prod
- Dev engineers can have admin access in dev without any prod access
- Cost Explorer shows exact cost per environment without tagging

```hcl
# providers.tf — use different AWS profiles per environment
provider "aws" {
  region  = var.region
  profile = "${var.environment}-admin"   # Assumes different named profiles
  # OR use assume_role for cross-account access from CI
}
```

**CI/CD pipeline for Terraform multi-environment:**

```groovy
// Jenkins pipeline — different stages for different envs
pipeline {
  stages {
    stage('Plan Dev') {
      steps {
        dir('environments/dev') {
          sh 'terraform plan -out=tfplan'
        }
      }
    }

    stage('Apply Dev') {
      when { branch 'develop' }
      steps {
        dir('environments/dev') {
          sh 'terraform apply tfplan'
        }
      }
    }

    stage('Plan Staging') {
      steps {
        dir('environments/staging') {
          sh 'terraform plan -out=tfplan'
        }
      }
    }

    stage('Apply Staging') {
      when { branch 'main' }
      steps {
        dir('environments/staging') {
          sh 'terraform apply tfplan'
        }
      }
    }

    stage('Plan Prod') {
      when { branch 'main' }
      steps {
        dir('environments/prod') {
          sh 'terraform plan -out=tfplan'
        }
      }
    }

    stage('Apply Prod') {
      when { branch 'main' }
      input {
        message "Review the prod plan and approve to apply"
        ok "Apply to Production"
      }
      steps {
        dir('environments/prod') {
          sh 'terraform apply tfplan'
        }
      }
    }
  }
}
```

The `input` block is critical — prod applies require a human to review the plan and click Approve in Jenkins. No code path can automatically apply to prod.

**The `count` trick for optional environment features:**

```hcl
# modules/rds/main.tf
variable "enable_multi_az" { type = bool; default = false }
variable "enable_deletion_protection" { type = bool; default = false }

resource "aws_db_instance" "main" {
  multi_az            = var.enable_multi_az
  deletion_protection = var.enable_deletion_protection
}
```

Prod calls with `enable_multi_az = true`. Dev calls with default `false`. Same module, behavior controlled by variables.

**Real scenario:** A team had `dev/main.tf`, `staging/main.tf`, `prod/main.tf` — three 400-line files. When we audited them, prod had 12 security fixes that never got to dev (dev was running with old insecure defaults). We refactored to the module pattern in a day. Now a security fix to the `rds` module applies to dev, staging, and prod in sequence automatically. Dev and staging are guaranteed to test the exact configuration that hits prod.

---

## 5. You want faster Terraform runs in large projects — what optimisations would you apply?

Slow Terraform plans and applies are a productivity killer. A 15-minute plan for every PR means engineers stop running plan locally and miss issues. Here are the concrete optimisations, ordered from easiest to most impactful.

**Problem 1: `terraform plan` refreshes every resource on every run.**

By default, Terraform calls the AWS API to verify the state of every resource before computing the plan. With 300 resources, that's 300+ API calls — slow and subject to throttling.

**Fix: Use `-refresh=false` when you know state is current.**

```bash
# For PR review plans — skip the refresh (state is fresh from last CI apply)
terraform plan -refresh=false

# When you DO want to catch drift:
terraform plan -refresh-only   # Separate drift detection run
```

In CI, the state is just-applied and current. Refreshing 300 resources for every PR plan adds 5–10 minutes with no benefit. We run `-refresh=false` for PR plans and full refresh for the nightly drift detection job.

**Problem 2: Single large state file — plan scans everything.**

```
Before: One state file, 300 resources, 18-minute plan
After:  6 state files, 50 resources each, 3-minute plan per stack
```

Splitting state by domain (networking, EKS, RDS, services) means a change to the `order-service` stack only plans 40 resources, not 300. Covered in detail in real-production Q3.

**Problem 3: Default parallelism is too conservative (or too aggressive).**

Terraform creates/destroys resources in parallel up to a default of 10 concurrent operations. Too low = slow. Too high = AWS API throttling.

```bash
# Increase parallelism for faster applies (if not hitting API throttling)
terraform apply -parallelism=20

# Decrease if you see ThrottlingException errors from AWS
terraform apply -parallelism=5
```

For large environments with diverse resource types (IAM, EC2, EKS), higher parallelism works well because different AWS services have independent rate limits. For environments heavy on a single service (many EC2 instances), lower parallelism avoids throttling.

**Problem 4: Module downloads on every `terraform init`.**

```bash
# Default init downloads all modules from the registry every time
terraform init   # Downloads modules from Terraform Registry

# Cache the module downloads in CI
- uses: actions/cache@v4
  with:
    path: |
      .terraform/modules
      .terraform/providers
    key: terraform-${{ hashFiles('.terraform.lock.hcl') }}
    restore-keys: terraform-
```

Module and provider downloads can add 2–5 minutes to `terraform init`. Cache them keyed on the lock file — they only re-download when `required_providers` or module versions change.

**Problem 5: Provider initialisation is slow at scale.**

If you use many providers (AWS, Kubernetes, Helm, Datadog, PagerDuty), init downloads all of them. Each provider binary is 50–300MB.

```bash
# Check what you're downloading
cat .terraform.lock.hcl | grep "provider\|version"

# Split providers across stacks — an EKS stack doesn't need the PagerDuty provider
# networking stack: aws provider only
# eks stack: aws + kubernetes + helm providers
# monitoring stack: aws + datadog + pagerduty providers
```

Using the CI cache (above) eliminates repeated downloads after the first run.

**Problem 6: `for_each` and `count` with large sets.**

```hcl
# Slow: for_each over 50 IAM users
resource "aws_iam_user" "devs" {
  for_each = toset(var.developers)  # 50 users → 50 separate AWS API calls
}
```

Each element in `for_each` generates a separate resource in state and a separate API call during plan. For large sets, this is inherently slow.

**Fix: Use AWS-native bulk resources where available.**

```hcl
# Instead of 50 individual users, use a group with a membership list
resource "aws_iam_group" "developers" {
  name = "developers"
}

resource "aws_iam_group_membership" "developers" {
  name  = "developers-membership"
  group = aws_iam_group.developers.name
  users = var.developers  # One API call, not 50
}
```

**Problem 7: Using `terraform show` on a large state.**

```bash
# Slow: terraform show renders all 300 resources as HCL
terraform show

# Fast: query only what you need
terraform state show aws_eks_cluster.main    # Single resource
terraform output                             # Just outputs
terraform state list | grep order_service   # Find specific resources
```

**Problem 8: Provider version constraints too broad.**

```hcl
# Bad: allows Terraform to search for the latest matching version on every init
provider "aws" {
  version = ">= 5.0"
}

# Good: pin to an exact or narrow range — init resolves instantly from lock file
provider "aws" {
  version = "~> 5.31"
}
```

Broad version constraints cause the Terraform registry to be queried for the latest matching version. Pinned versions resolve from `.terraform.lock.hcl` with no network call.

**Summary — fastest wins in order:**

| Optimisation | Effort | Typical time saved |
|---|---|---|
| `-refresh=false` for PR plans | Zero — just add a flag | 3–8 min per plan |
| CI cache for modules/providers | Low — add cache step | 2–5 min per init |
| Split state by domain | High — refactoring | 10–15 min per plan |
| Tune parallelism | Zero | 1–3 min per apply |
| Pin provider versions | Low | 30s–2 min per init |

**Real scenario:** Our CI plan was taking 22 minutes. Root causes: single state file with 280 resources (18 min for full refresh), no CI cache for providers (3 min for init), default parallelism. After: split to 5 stacks (each plans in 3–4 min), added provider cache (init: 20 sec), `-refresh=false` for PR plans (nightly drift job does the real refresh). Result: PR plan now runs in 4 minutes. Engineers run `terraform plan` locally before every commit — they were skipping it before because 22 minutes was too slow.

---

## 6. Multi-account Terraform setup with backend isolation

> **Also asked as:** "Multi-account Terraform setup with backend isolation"

When operating at an enterprise scale, deploying everything into a single AWS account is an anti-pattern (blast radius is too high). You need separate accounts for Dev, Staging, Prod, and Security. Terraform must be structured to accommodate this securely.

**The Architecture:**

1. **The Automation/Shared Services Account:**
This is where your CI/CD runners (like Jenkins agents or GitHub Actions runners) live. This account holds the Terraform State S3 buckets and DynamoDB lock tables for *all* other accounts.

2. **The Target Accounts (Dev, Prod, Security):**
These accounts contain the actual infrastructure. They do *not* hold their own state. Instead, they contain an IAM Role (e.g., `TerraformExecutionRole`) that trusts the Automation account.

**How Terraform is configured:**

```hcl
# backend.tf - The state always lives in the Automation Account S3 bucket
terraform {
  backend "s3" {
    bucket         = "company-automation-tf-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "company-automation-tf-locks"
  }
}

# providers.tf - Terraform assumes a role into the Target Account
provider "aws" {
  region = "us-east-1"
  assume_role {
    role_arn     = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformExecutionRole"
    session_name = "TerraformProdSession"
  }
}
```

**Why this is the industry standard:**
- **Strict Isolation:** If the Dev AWS account is compromised, the attacker cannot delete the Prod state file, because the state file lives in the strongly-secured Automation account.
- **Least Privilege:** Developers don't get long-lived AWS credentials. They push code to Git. The CI/CD runner in the Automation account runs Terraform, assumes the role into Prod, and provisions the infrastructure.

---

## 7. Handling terraform state in a remote team

> **Also asked as:** "Handling terraform state in a remote team"

Managing Terraform state when you are solo is easy. Managing it when 8 engineers across 3 time zones are deploying simultaneously requires strict tooling and process guardrails to prevent state corruption and race conditions.

**1. Remote Backend with Locking (Non-Negotiable)**
Never use local state. Use an S3 backend with DynamoDB locking (or an equivalent like Terraform Cloud). When Engineer A in London runs `terraform apply`, Terraform locks the state. If Engineer B in Tokyo runs `terraform apply` 5 seconds later, it fails instantly with a lock error, preventing concurrent state corruption.

**2. GitOps / Automation via Atlantis**
Humans should not run `terraform apply` from their laptops in a team environment. 
- Use a tool like **Atlantis** or a strict CI/CD pipeline (GitHub Actions/GitLab CI).
- When a developer opens a Pull Request, the CI runner executes `terraform plan` and posts the output directly as a comment on the PR.
- Reviewers can see exactly what will change.
- Once approved, the developer types `atlantis apply` in the PR comments. The automation applies the infra and merges the PR automatically. 
- **Result:** The `main` branch always 100% matches what is deployed in the cloud.

**3. State File Segmentation**
Do not use a single monolithic state file. If 8 developers are working in one monolithic state, every `terraform plan` will take 15 minutes, and PRs will constantly block each other with locked state files.
Split the state by logical boundaries:
- `aws-networking.tfstate`
- `kubernetes-cluster.tfstate`
- `payment-service.tfstate`
This allows the networking team to apply changes without locking out the application developers.

**4. Drift Detection**
In a distributed team, someone will inevitably make a manual change in the AWS console during an incident. Run a nightly scheduled CI job: `terraform plan -detailed-exitcode`. If it detects drift, it alerts the team's Slack channel immediately so the manual change can be retrofitted back into the code.
