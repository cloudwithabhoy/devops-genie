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
