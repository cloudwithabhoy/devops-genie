# Terraform — Basic Questions

---

## 1. Write Terraform configuration to create an S3 bucket.

A "write an S3 bucket in Terraform" question is actually testing whether you know production-grade config, not just the minimum valid HCL.

**What most people write (valid but incomplete):**

```hcl
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
}
```

This creates a bucket, but it's public by default in older provider versions, has no encryption, no versioning, and will cause `terraform destroy` to fail if it has objects. Not production-ready.

**What we actually deploy:**

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}

# variables.tf
variable "region" {
  type    = string
  default = "ap-south-1"
}

variable "environment" {
  type = string
}

variable "bucket_name" {
  type = string
}
```

```hcl
# main.tf

# The S3 bucket
resource "aws_s3_bucket" "artifacts" {
  bucket = "${var.bucket_name}-${var.environment}"

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

# Block all public access — S3 buckets are private by default in AWS provider v4+
# but this makes it explicit and prevents future accidental exposure
resource "aws_s3_bucket_public_access_block" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Enable versioning — allows recovery of accidentally deleted/overwritten objects
resource "aws_s3_bucket_versioning" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Server-side encryption at rest
resource "aws_s3_bucket_server_side_encryption_configuration" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"    # Use KMS for audit trail of key usage
    }
    bucket_key_enabled = true      # Reduces KMS API call costs by ~99%
  }
}

# Lifecycle rules — automatically transition old versions to cheaper storage
resource "aws_s3_bucket_lifecycle_configuration" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id

  rule {
    id     = "expire-old-versions"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"   # Move to Infrequent Access after 30 days
    }

    noncurrent_version_expiration {
      noncurrent_days = 90    # Delete old versions after 90 days
    }
  }
}

# outputs.tf
output "bucket_name" {
  value = aws_s3_bucket.artifacts.bucket
}

output "bucket_arn" {
  value = aws_s3_bucket.artifacts.arn
}
```

**Run it:**

```bash
terraform init
terraform plan -var="environment=prod" -var="bucket_name=my-app-artifacts"
terraform apply -var="environment=prod" -var="bucket_name=my-app-artifacts"
```

**Why each block matters:**

- **`public_access_block`** — We had an S3 bucket exposed publicly for 3 days before it was caught in a security audit. Now this is mandatory on every bucket we create
- **`versioning`** — Accidentally deleted 2 weeks of build artifacts from an S3 bucket without versioning. Everything was gone. Now every bucket has versioning on
- **`server_side_encryption`** — Required by our SOC2 controls. `bucket_key_enabled = true` cut our KMS costs by 95%
- **`lifecycle_configuration`** — Without this, old versions accumulate forever. A versioned bucket with daily writes and no lifecycle rules will cost 10x more than expected after a year

---

## 2. What is the Terraform workflow, and what does each command do?

```
Write code (.tf files)
       ↓
terraform init       → Download providers + modules, configure backend
       ↓
terraform validate   → Syntax check (no API calls)
       ↓
terraform plan       → Compare code vs state file → show what will change
       ↓
terraform apply      → Execute the plan, update real infrastructure
       ↓
terraform destroy    → Remove all resources managed by this config
```

**`terraform init`**
Downloads the provider plugins (AWS, Azure, etc.) and modules referenced in your config. Must be run once when starting, or whenever you add a new provider or module. Also configures the backend (S3 state bucket, DynamoDB lock table).

**`terraform validate`**
Checks HCL syntax and type correctness. Fast — no AWS API calls. We run this as the first CI step because it catches obvious mistakes in seconds.

**`terraform plan`**
The most important command. Calls AWS APIs to compare current state with your code. Shows exactly what will be created, modified, or destroyed. We treat the plan output like a code review — any `-/+` (destroy and recreate) gets a human review before it can merge.

```bash
# Save the plan to a file — apply from file to guarantee exactly what was reviewed
terraform plan -out=tfplan
terraform apply tfplan
```

**`terraform apply`**
Executes the plan. Calls AWS/GCP/Azure APIs to create/modify/delete resources. Updates the state file after each resource operation. If it fails halfway through, the state file reflects what completed — the next `apply` resumes from where it stopped.

**`terraform destroy`**
Destroys all resources in the state file. We never run this in production without first checking: does `prevent_destroy = true` protect critical resources? Is the state file pointing at the right environment? We've had an incident where someone ran `terraform destroy` in a directory they thought was staging but was prod. Now we add `prevent_destroy = true` to RDS and EKS resources.

> **Also asked as:** "What does the terraform init command do?" — covered above (`terraform init` downloads provider plugins, initialises modules, and configures the backend).

---

## 3. What's the difference between Terraform and CloudFormation?

Both are Infrastructure as Code (IaC) tools — they let you define infrastructure in code and manage its lifecycle. The difference is in scope, language, state management, and ecosystem. This is not a "one is better" answer — it's a "when to use which" answer.

**The fundamental difference: Terraform is cloud-agnostic. CloudFormation is AWS-native.**

Terraform uses providers to manage resources across AWS, Azure, GCP, Kubernetes, Datadog, PagerDuty, GitHub — anything with an API. CloudFormation manages only AWS resources.

```hcl
# Terraform — one tool to manage AWS + Datadog + PagerDuty
resource "aws_instance" "app" {
  ami           = "ami-0abc123"
  instance_type = "t3.micro"
}

resource "datadog_monitor" "cpu_alert" {
  name    = "High CPU on app server"
  type    = "metric alert"
  query   = "avg(last_5m):avg:aws.ec2.cpuutilization{name:app} > 80"
}

resource "pagerduty_service" "app" {
  name = "App Service"
  escalation_policy = pagerduty_escalation_policy.default.id
}
```

CloudFormation can't manage Datadog or PagerDuty. You'd need a separate tool or custom resources.

**Language:**

```yaml
# CloudFormation — YAML/JSON (declarative but verbose)
Resources:
  AppInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0abc123
      InstanceType: t3.micro
      SecurityGroupIds:
        - !Ref AppSecurityGroup
      Tags:
        - Key: Name
          Value: app-server
```

```hcl
# Terraform — HCL (declarative, purpose-built, more readable)
resource "aws_instance" "app" {
  ami                    = "ami-0abc123"
  instance_type          = "t3.micro"
  vpc_security_group_ids = [aws_security_group.app.id]

  tags = {
    Name = "app-server"
  }
}
```

HCL was designed specifically for infrastructure — it has loops, conditionals, modules, and dynamic blocks that make complex configurations manageable. CloudFormation's YAML relies on intrinsic functions (`!Ref`, `!Sub`, `!If`, `Fn::Select`) that become unreadable at scale.

**State management — the biggest operational difference:**

| | Terraform | CloudFormation |
|---|---|---|
| State | You manage it (S3 + DynamoDB for locking) | AWS manages it (automatic, invisible) |
| State corruption risk | Possible if misconfigured | Very rare |
| State visibility | `terraform state list` — you can inspect and manipulate | Hidden — you see stack events only |
| Import existing resources | `terraform import` | `aws cloudformation import` (limited) |

Terraform's state is both its strength and its operational burden. You must secure the state file (it contains sensitive outputs), configure locking (DynamoDB), and handle state drift. CloudFormation handles all of this automatically.

```hcl
# Terraform backend — you manage this
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/infrastructure.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-locks"   # Prevents concurrent applies
    encrypt        = true
  }
}
```

**Drift detection:**

CloudFormation has built-in drift detection — it tells you if someone changed a resource manually:

```bash
aws cloudformation detect-stack-drift --stack-name prod-stack
aws cloudformation describe-stack-resource-drifts --stack-name prod-stack
```

Terraform detects drift every time you run `terraform plan` — it compares real infrastructure (via API calls) with the state file. If someone changed a security group manually, the plan shows it.

**Rollback behaviour:**

CloudFormation automatically rolls back on failure — if creating resource 5 out of 10 fails, it deletes resources 1–4 and returns to the previous state. This is safe but slow (cleanup can take 20 minutes).

Terraform does not auto-rollback. If resource 5 fails, resources 1–4 stay created. The state file reflects what was actually created. Running `terraform apply` again retries from where it stopped. You're responsible for cleanup if you want to undo.

**Module ecosystem:**

| | Terraform | CloudFormation |
|---|---|---|
| Public modules | Terraform Registry (large ecosystem) | AWS Solutions Library (limited) |
| Module quality | Community-maintained — verify before using | AWS-maintained — generally reliable |
| Example | `terraform-aws-modules/vpc/aws` | AWS QuickStart templates |

**When to use each:**

| Scenario | Choose |
|---|---|
| Multi-cloud (AWS + Azure + GCP) | Terraform |
| AWS-only shop with simpler infrastructure | Either works |
| Team with no IaC experience, wants managed state | CloudFormation (lower ops burden) |
| Complex infrastructure with modules, loops, dynamic config | Terraform (HCL is more expressive) |
| Need to manage non-AWS resources (Datadog, GitHub, K8s) | Terraform |
| AWS native services deeply integrated (SAM for Lambda, CDK) | CloudFormation / CDK |
| Compliance requirement for automatic rollback on failure | CloudFormation |

**AWS CDK — the modern CloudFormation experience:**

AWS CDK (Cloud Development Kit) generates CloudFormation under the hood but lets you write infrastructure in Python, TypeScript, or Java. It brings real programming language constructs (loops, classes, inheritance) to CloudFormation:

```typescript
// CDK (TypeScript) — compiles to CloudFormation YAML
const vpc = new ec2.Vpc(this, 'AppVpc', { maxAzs: 3 });
const cluster = new ecs.Cluster(this, 'AppCluster', { vpc });
```

CDK is worth considering if you're AWS-only and your team prefers a real programming language over HCL. But the output is still CloudFormation — with its state management model, rollback behavior, and AWS-only scope.

**What we use and why:**

We use Terraform for everything — AWS infrastructure, Kubernetes resources (via the `kubernetes` provider), Datadog monitors, PagerDuty services, and GitHub repository settings. One tool, one workflow (`plan → review → apply`), one state management pattern. We evaluated CDK and CloudFormation — CDK was appealing for its programming language support, but we'd still need Terraform for Datadog and GitHub. Running two IaC tools doubles the operational complexity (two pipelines, two state management strategies, two sets of expertise). Standardizing on Terraform was the pragmatic choice.

**Real scenario:** We inherited a project that used CloudFormation for VPC/EC2 and Terraform for EKS/Kubernetes. Any change that involved both layers (adding a subnet AND deploying a service to it) required two separate PRs, two separate applies, and careful ordering. We migrated everything to Terraform over 3 weeks — VPC moved first, then EC2, then Route53. Each migration was a `terraform import` of existing resources. After migration: one PR, one pipeline, one apply. Deployment time for infrastructure changes dropped from 45 minutes (two sequential pipeline runs) to 15 minutes (one run).

---

## 4. What is state management in Terraform?

> **Also asked as:** "Where you will keep your stateful file ?"

State management is how Terraform keeps track of the real-world infrastructure it has provisioned and maps it to your configuration files.

When you run `terraform apply`, Terraform creates an `aws_instance` in AWS. Terraform needs a way to remember that "the EC2 instance `i-0abcdef123` in AWS belongs to the resource block `aws_instance.app` in my code." It stores this mapping in the **state file** (`terraform.tfstate`), which is a large JSON document.

**Why state is critical:**
1. **Mapping Configuration to Reality:** If you change your code to add a tag, Terraform checks the state to know exactly which AWS resource to update.
2. **Detecting Drift:** When you run `terraform plan`, Terraform compares what's in the state file with what's actually running in the cloud. If an engineer manually deleted a security group that is in the state file, Terraform knows it's missing and will propose recreating it.
3. **Dependency Resolution:** The state file holds the output attributes of resources (like the auto-generated AMI ID or IP address) so other resources can depend on them.
4. **Destroying Resources:** When you remove a block of code, Terraform uses the state file to find the corresponding real-world resource and destroy it.

**Production State Management:**
In production, you **never** store the state file locally on your laptop. If your hard drive crashes, Terraform completely loses track of the infrastructure. Instead, you use a **Remote Backend**:

1. **Remote Storage (e.g., AWS S3):** The state file is stored centrally in an S3 bucket so the entire team and the CI/CD pipeline share the single source of truth.
2. **State Locking (e.g., DynamoDB):** If two developers run `terraform apply` at the exact same time, it can corrupt the state or cause race conditions. A DynamoDB table is used to "lock" the state. The first apply acquires the lock, and the second apply is rejected until the first finishes.

---

## 5. Explain terraform structure ?

> **Also asked as:** "Explain terraform structure ?"

Terraform doesn't enforce a specific directory structure. You could put 10,000 lines of HCL into one giant `main.tf` file, and Terraform would parse it perfectly. However, for human readability and maintainability, the industry relies on a standard modular structure.

**The Standard Root Module Structure:**
Every Terraform project should at minimum be split into these files:

```text
my-project/
├── main.tf        # The actual resources being created (EC2, S3, VPC)
├── variables.tf   # Input variables (declarations, types, and defaults)
├── outputs.tf     # What information to print out after an apply (e.g., ALB URL)
├── providers.tf   # Provider configurations (AWS region, versions)
├── backend.tf     # Where to store the remote state file (S3 + DynamoDB)
└── terraform.tfvars # (Optional) Environment-specific values passed locally
```

**Why split them up?**
If I want to know exactly what this Terraform project *requires* to run, I look at `variables.tf`. If I want to know what it *gives back*, I look at `outputs.tf`. I don't have to scroll through 800 lines of `main.tf` to find what I need.

**The Enterprise Module Structure:**
When managing complex infrastructure, we never put everything in the "root" folder. We break infrastructure down into reusable **Modules** (like functions in programming).

```text
infrastructure/
├── environments/
│   ├── dev/
│   │   └── main.tf        # Calls the generic modules, passing in "dev" variables
│   └── prod/
│       └── main.tf        # Calls the generic modules, passing in "prod" variables
│
└── modules/               # Reusable blocks of code
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── database/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    └── web-app/
```

In this structure, the definitions in `environments/prod/main.tf` are incredibly short. They just call the modules:

```hcl
module "production_vpc" {
  source     = "../../modules/vpc"
  cidr_block = "10.0.0.0/16"
  env_name   = "prod"
}
```
This guarantees that `dev` and `prod` share the exact same architectural components, differing only by the variable inputs.

