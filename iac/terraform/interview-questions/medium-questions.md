# IaC — Medium Questions

---

## 1. Explain the Terraform lifecycle, terraform refresh, and terraform import.

> **Also asked as:** "Write a terraform script for VPC architecture for production."

For a production VPC, you shouldn't just create a VPC block. You need a multi-AZ setup with public and private subnets, NAT Gateways for egress, and proper tagging for EKS/ALB integration.

**Production-grade VPC using Terraform:**

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"

  # Spread across 3 AZs for High Availability
  azs             = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
  
  # Private subnets for Apps/DBs (No direct internet access)
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  
  # Public subnets for LBs/NAT Gateways
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  # Enable NAT Gateway for private instances to reach the internet (updates/patches)
  enable_nat_gateway = true
  single_nat_gateway = false  # One per AZ for redundancy
  one_nat_gateway_per_az = true

  enable_vpn_gateway = false

  # Required for EKS / ALB integration
  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }

  tags = {
    Environment = "production"
    Owner       = "devops-team"
  }
}
```

---

## 2. Explain the Terraform lifecycle, terraform refresh, and terraform import.

> **Also asked as:** "What is Terraform import and when do we use it?"

These three are often asked together because they're all about how Terraform manages the relationship between your code, your state file, and your actual infrastructure. Let me explain them the way I think about them in production.

---

### Terraform Lifecycle

Every `terraform apply` runs through the same lifecycle internally. Understanding it tells you why Terraform does what it does.

```
Write code (.tf files)
       ↓
terraform init       → Downloads providers, initializes backend (S3, etc.)
       ↓
terraform validate   → Syntax check, type check
       ↓
terraform plan       → Compare .tf files vs state file → generate diff (execution plan)
       ↓
terraform apply      → Execute the plan → call provider APIs → update state file
       ↓
terraform destroy    → Same as apply but deletes everything
```

**The key thing most people miss:** `terraform plan` compares your code against the **state file**, not against actual infrastructure. If something changes in AWS outside of Terraform (someone clicks in the console), `plan` won't know unless you first run `terraform refresh`.

---

**Resource lifecycle meta-arguments** — these control how Terraform handles create, update, and destroy:

```hcl
resource "aws_s3_bucket" "data" {
  bucket = "my-prod-data"

  lifecycle {
    create_before_destroy = true   # Create the replacement first, then destroy old
    prevent_destroy       = true   # Refuse to destroy this resource — terraform destroy fails
    ignore_changes        = [tags] # Don't show drift if tags change outside Terraform
  }
}
```

**`create_before_destroy`** — We use this for resources that can't have downtime during replacement (like TLS certificates, security groups). Without it, Terraform destroys the old resource first, then creates the new one. There's a gap where the resource doesn't exist.

**`prevent_destroy`** — We put this on RDS instances and S3 buckets with production data. Anyone who runs `terraform destroy` gets a hard error. This has saved us twice from accidental destroys during staging cleanup scripts that accidentally pointed at prod.

**`ignore_changes`** — We use this for tags that our AWS tag enforcement Lambda adds automatically. Without `ignore_changes = [tags]`, `terraform plan` always shows drift because the Lambda keeps adding tags that aren't in our Terraform code.

---

### terraform refresh

`terraform refresh` queries the actual cloud APIs and updates the state file to match current reality — without making any changes to real infrastructure.

```bash
terraform refresh
```

**When you need it:** Someone went into the AWS console and changed something manually — modified a security group, changed an instance type, added a tag. Your state file now reflects the old config, not what actually exists. `terraform plan` will show phantom changes until you refresh.

**Real scenario:** A developer manually added an inbound rule to a security group during an incident ("just let me fix it now, I'll add it to Terraform later"). They forgot. A week later, `terraform plan` showed that rule as "to be removed" because the state didn't know about it. Running `terraform refresh` updated the state to include that rule, and `terraform plan` stopped showing it as a change. We then added it properly to the `.tf` file.

**In Terraform 0.15+**, `terraform refresh` is being superseded. `terraform plan -refresh-only` is the recommended way:
```bash
terraform plan -refresh-only    # Shows what would change in state only, doesn't change infra
terraform apply -refresh-only   # Updates state file to match actual infra
```

**Important:** Don't run `terraform refresh` blindly before every apply in production. If someone deleted a resource manually that Terraform doesn't know about, refresh will note it's gone, and the next `apply` will try to recreate it. Make sure you understand what changed before refreshing.

---

### terraform import

`terraform import` is for bringing existing infrastructure — resources you created manually or that existed before Terraform — under Terraform management.

```bash
# Syntax: terraform import <resource_type.resource_name> <cloud_resource_id>
terraform import aws_s3_bucket.my_bucket my-existing-bucket-name
terraform import aws_instance.web i-0abcdef1234567890
terraform import aws_security_group.app sg-0abc123def456789
```

**What it does:** It adds the resource to the Terraform state file — but it does NOT generate the `.tf` code. You have to write the resource block yourself and make it match what already exists.

**The real workflow for importing:**

1. Write the resource block in your `.tf` file (best guess at the config)
2. Run `terraform import`
3. Run `terraform plan` — it will show a diff between your config and actual state
4. Edit your `.tf` file to make `terraform plan` show "No changes"
5. Commit

**Real scenario we did at work:** We inherited an AWS account that had 3 VPCs, 12 security groups, and an RDS instance — all created manually by the previous team. We needed to bring them under Terraform control.

The import process took a full day. The hardest part wasn't running the import commands — it was writing `.tf` blocks that exactly match the existing resources. Things like: does this security group have the exact CIDR blocks? Does this RDS instance have the exact parameter group settings? Every mismatch shows up as a diff in `terraform plan`, and you have to decide: do I update the code to match reality, or do I update reality to match the code?

```bash
# We imported resources one by one
terraform import aws_vpc.main vpc-0abc12345678
terraform import aws_db_instance.prod arn:aws:rds:ap-south-1:123456789:db:prod-postgres
```

**Terraform 1.5+ has `import` blocks** (generated import):
```hcl
import {
  to = aws_s3_bucket.my_bucket
  id = "my-existing-bucket-name"
}
```

And `terraform plan -generate-config-out=generated.tf` will auto-generate the resource block for you. This is a massive time saver — instead of writing the `.tf` block manually, Terraform generates it from the actual resource. We've started using this for all new imports.

**Key insight for interviews:** `import` is not magic. It puts the resource in state, but if your code doesn't match reality, your next `terraform apply` will try to "fix" the resource to match your code — which might mean modifying or even replacing a production resource. Always run `terraform plan` and review carefully after every import.

---

## 2. How do you create and manage Kubernetes clusters using Terraform?

Terraform treats the EKS cluster as infrastructure code — the same way you'd define a VPC or an S3 bucket. Cluster creation, node group config, add-ons, and IAM are all version-controlled, repeatable, and reviewed via PRs.

**Folder structure we use:**

```
terraform/
├── modules/
│   └── eks/                  # Reusable EKS module used across envs
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── staging/
│   │   ├── main.tf           # Calls the eks module with staging vars
│   │   ├── terraform.tfvars
│   │   └── backend.tf        # S3 state for staging
│   └── prod/
│       ├── main.tf
│       ├── terraform.tfvars
│       └── backend.tf        # S3 state for prod
└── versions.tf
```

**Core resources we define:**

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name               = "${var.cluster_name}-vpc"
  cidr               = "10.0.0.0/16"
  azs                = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
  private_subnets    = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets     = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  enable_nat_gateway = true
  single_nat_gateway = false   # One NAT per AZ for HA

  # Required tags for EKS to discover subnets for load balancers
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = "1.30"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  enable_irsa = true   # IRSA — lets pods assume IAM roles via ServiceAccounts

  eks_managed_node_groups = {
    general = {
      instance_types = ["m5.large"]
      min_size       = 2
      max_size       = 10
      desired_size   = 3

      labels = {
        role = "general"
      }
    }
  }

  cluster_addons = {
    coredns    = { most_recent = true }
    kube-proxy = { most_recent = true }
    vpc-cni    = { most_recent = true }
  }
}
```

**The subnet tags (`kubernetes.io/role/internal-elb`) are critical** — without them, the AWS Load Balancer Controller can't find subnets to deploy ALBs/NLBs into. We missed this once and spent 45 minutes debugging why internal load balancers failed to provision.

**Configuring kubectl after cluster creation:**

```hcl
output "configure_kubectl" {
  value = "aws eks update-kubeconfig --region ${var.region} --name ${module.eks.cluster_name}"
}
```

After `terraform apply`, the output tells you exactly the command to configure kubectl. No manual steps.

**Managing cluster add-ons via Terraform Helm provider:**

```hcl
resource "helm_release" "aws_load_balancer_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"

  set {
    name  = "clusterName"
    value = module.eks.cluster_name
  }
  set {
    name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
    value = aws_iam_role.alb_controller.arn
  }

  depends_on = [module.eks]
}
```

**Cluster lifecycle operations:**

```bash
# Create cluster
terraform init && terraform plan -out=tfplan && terraform apply tfplan

# Upgrade cluster version — change cluster_version = "1.31" in variables
terraform plan   # Shows the upgrade plan
terraform apply  # Triggers managed rolling upgrade on EKS

# Scale node group — change desired_size in tfvars
terraform apply  # Updates ASG desired count

# Destroy cluster (always requires explicit confirmation)
terraform destroy -target=module.eks
```

**Real scenario:** Before Terraform, we provisioned two EKS clusters manually. Staging and prod had subtle differences — different CIDR ranges, different node types, different add-on versions. When we hit a bug in staging that we couldn't reproduce in prod, we spent hours diffing the two environments manually. After migrating to Terraform with a shared module and per-environment `tfvars`, staging and prod are structurally identical. The only differences are in the variables file — instance types, replica counts, cluster name.

---

## 3. Why do we store the Terraform state file in S3?

This is a foundational question — the answer explains what the state file is, why local state breaks down in teams, and what S3 + DynamoDB gives you.

**What the state file is:**

Terraform's state file (`terraform.tfstate`) is the mapping between your `.tf` resource blocks and the actual cloud resources they represent. When you run `terraform plan`, Terraform reads this file to know what currently exists — not by calling AWS APIs for every resource (that would be too slow and would require broad read permissions across your account).

Without state, Terraform doesn't know that `aws_s3_bucket.artifacts` corresponds to the bucket `my-app-prod-artifacts`. It would try to create a new one every time.

**Why local state breaks in teams:**

- **Only works on your machine** — if you run `terraform apply` and it creates an RDS instance, your colleague's machine has no idea that instance exists. Their next `apply` will try to create another one
- **No locking** — two engineers run `terraform apply` at the same time. Both reads/writes the local file simultaneously → corrupted state → Terraform no longer knows what exists
- **Lost on disk failure** — local state file on a laptop gets wiped. Terraform has no record of what it manages. You have to `terraform import` everything back manually

**Remote state in S3 solves all three:**

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"
    key            = "prod/eks-cluster/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true              # State at rest is encrypted

    dynamodb_table = "terraform-state-lock"   # DynamoDB for distributed locking
  }
}
```

**S3 gives you:**
- **Centralized storage** — every team member reads/writes the same state file. No divergence
- **Versioning** — S3 versioning keeps history of every state file version. If state corruption happens, roll back to a previous version
- **Encryption** — state files often contain sensitive values (database passwords, private IPs). `encrypt = true` encrypts at rest with SSE-KMS

**DynamoDB gives you state locking:**

When you run `terraform apply`, Terraform writes a lock entry to DynamoDB. If another engineer runs `apply` at the same time, they get:

```
Error: Error acquiring the state lock
Error message: ConditionalCheckFailedException
Lock Info:
  ID: 6f3d9c2a-1234-...
  Who: abhoy@workstation
  Created: 2024-01-15 14:32:01
```

They can't proceed until the first apply finishes and releases the lock. Without this, concurrent applies corrupt the state file.

**The DynamoDB table we create:**

```hcl
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**Real incident without remote state:** A junior engineer ran `terraform apply` from their local machine (using local state) after a colleague had already applied from a different machine. The local state was 3 days out of date. Terraform didn't know about 4 security groups created in the meantime. It treated them as "not managed" and on the next destroy — nuked them. Three services lost their security groups simultaneously. 22-minute partial outage. After this, we made remote state non-negotiable in our onboarding checklist.

> **Also asked as:** "What is the .tfstate file and where is it stored?" — covered above (maps resource blocks to real cloud resources; stored in S3 + DynamoDB lock for team use).

---

## 4. What is `terraform taint`, and when would you use it vs terraform import?

These two commands are opposite operations — `taint` forces Terraform to destroy and recreate an existing resource on the next apply; `import` brings an existing resource that Terraform doesn't know about into state management.

**`terraform taint` — mark a resource for forced replacement:**

```bash
# Mark the resource for destruction + recreation on next apply
terraform taint aws_instance.web_server

# Verify — plan will show -/+ (destroy + create)
terraform plan
# Output: aws_instance.web_server must be replaced
```

After tainting, the next `terraform apply` will destroy the instance and create a new one, even if nothing in the `.tf` code changed.

**`terraform taint` was soft-deprecated in Terraform 1.0+.** The modern equivalent is `terraform apply -replace`:

```bash
# Preferred modern way — replaces the resource in the same apply
terraform apply -replace="aws_instance.web_server"
```

**When you'd use taint / `-replace`:**

- An EC2 instance has gotten into a bad state (corrupted OS, failed cloud-init, botched manual configuration) and you want a clean replacement without changing any Terraform code
- A resource's internal state is inconsistent with what Terraform thinks — recreation is simpler than debugging
- You want to force rotation of a resource that has no configuration change (e.g., rotate an EKS node by tainting an EC2 instance in the node group)

```bash
# Real use case: force-replace an EKS node that's NotReady and not self-healing
terraform apply -replace="aws_instance.eks_node_1"
```

**`terraform import` — bring existing infrastructure into Terraform management:**

`import` is the opposite direction. The resource already exists in AWS; Terraform doesn't know about it. Import adds it to the state file.

```bash
# Bring an existing S3 bucket under Terraform management
terraform import aws_s3_bucket.legacy my-existing-bucket-name

# Bring an existing EC2 instance
terraform import aws_instance.prod i-0abc123def456789

# Bring an existing RDS instance
terraform import aws_db_instance.prod arn:aws:rds:ap-south-1:123456789:db:prod-postgres
```

**When you'd use import:**
- Inheriting infrastructure created manually (before Terraform was introduced)
- A resource was created outside Terraform (someone clicked in the console) and you want to manage it going forward
- Recovering after state file loss — re-importing resources so Terraform knows they exist

**The critical difference:**

| | `terraform taint` / `-replace` | `terraform import` |
|---|---|---|
| **Direction** | Terraform destroys existing + creates new | Adds existing to state (no infra change) |
| **Infrastructure change?** | Yes — resource is destroyed and recreated | No — existing resource is just tracked |
| **Use case** | Fix bad resource state | Bring unmanaged resource under management |
| **Risk** | Data loss if stateful resource (RDS, EBS) | Low — doesn't touch infrastructure |

**Real scenario for import:** We inherited an AWS account with 6 manually created security groups, 2 VPCs, and an RDS cluster — all from a previous vendor. None of it was in Terraform. We had to `terraform import` each resource one by one, then write matching `.tf` blocks until `terraform plan` showed "No changes." Took two days. The hardest part was writing the `.tf` to exactly match what existed — any attribute mismatch shows as a diff and needs reconciliation.

**Key insight:** After `terraform import`, always run `terraform plan` immediately. If the plan shows changes, your `.tf` doesn't match reality yet. Do NOT run `apply` until the plan is clean — otherwise Terraform will "fix" the resource to match your code, which might involve destroying or modifying production infrastructure.

---

## 5. What is the difference between `count` and `for_each` in Terraform?

Both create multiple instances of a resource from a single block. The choice between them determines how Terraform identifies resources in state — and getting it wrong causes unnecessary destroys.

**`count` — create N identical resources by index.**

```hcl
resource "aws_iam_user" "developers" {
  count = 3
  name  = "developer-${count.index}"
}

# Creates:
# aws_iam_user.developers[0]  → "developer-0"
# aws_iam_user.developers[1]  → "developer-1"
# aws_iam_user.developers[2]  → "developer-2"
```

References: `aws_iam_user.developers[0].arn`, `count.index` inside the block.

**`for_each` — create one resource per map entry or set item.**

```hcl
resource "aws_iam_user" "developers" {
  for_each = toset(["alice", "bob", "charlie"])
  name     = each.key
}

# Creates:
# aws_iam_user.developers["alice"]   → "alice"
# aws_iam_user.developers["bob"]     → "bob"
# aws_iam_user.developers["charlie"] → "charlie"
```

References: `aws_iam_user.developers["alice"].arn`, `each.key` / `each.value` inside the block.

**The critical difference — what happens when you remove an item:**

With `count`, resources are identified by index. Remove the middle item → Terraform destroys and recreates everything after it:

```hcl
# Before
count = 3  # [0]="dev-0", [1]="dev-1", [2]="dev-2"

# Remove index 1 (dev-1)
count = 2  # Terraform sees: [0]="dev-0", [1]="dev-2"
           # It thinks: [1] changed from "dev-1" to "dev-2" → DESTROY + CREATE
           # And [2] was removed → DESTROY dev-2
           # Result: dev-1 destroyed, dev-2 destroyed and recreated as [1]
           # Even though you only wanted to remove dev-1
```

With `for_each`, resources are identified by key. Remove an item → only that specific item is destroyed:

```hcl
# Before
for_each = toset(["alice", "bob", "charlie"])

# Remove "bob"
for_each = toset(["alice", "charlie"])
# Terraform sees: "bob" no longer in set → DESTROY aws_iam_user.developers["bob"]
# alice and charlie: unchanged → no action
# Exactly what you intended
```

**When to use `count`:**

- Creating N identical resources where the index doesn't matter for identity
- Simple toggles: `count = var.enable_feature ? 1 : 0`

```hcl
# Toggle a resource on/off based on a variable
resource "aws_cloudwatch_log_group" "app" {
  count = var.enable_logging ? 1 : 0
  name  = "/app/${var.service_name}"
}

# Reference it
arn = length(aws_cloudwatch_log_group.app) > 0 ? aws_cloudwatch_log_group.app[0].arn : ""
```

**When to use `for_each`:**

- Creating resources from a list/map where each has a meaningful identifier
- When you might add or remove items (to avoid destroying/recreating unrelated resources)
- When resources have different configurations

```hcl
# Map with different values per resource
variable "s3_buckets" {
  default = {
    "artifacts" = { versioning = true,  encryption = true  }
    "logs"      = { versioning = false, encryption = true  }
    "backups"   = { versioning = true,  encryption = true  }
  }
}

resource "aws_s3_bucket" "buckets" {
  for_each = var.s3_buckets
  bucket   = "${each.key}-${var.environment}"
}

resource "aws_s3_bucket_versioning" "buckets" {
  for_each = { for k, v in var.s3_buckets : k => v if v.versioning }
  bucket   = aws_s3_bucket.buckets[each.key].id
  versioning_configuration {
    status = "Enabled"
  }
}
```

**Real scenario where `count` caused pain:**

We had `count = length(var.availability_zones)` for subnets. When we changed the AZ list from `["ap-south-1a", "ap-south-1b", "ap-south-1c"]` to `["ap-south-1b", "ap-south-1c", "ap-south-1a"]` (reordered), Terraform wanted to destroy and recreate all 3 subnets — even though the same 3 AZs were being used. The plan showed `-/+` for all subnets, which would have destroyed the EKS node groups running in them.

Migrated to `for_each = toset(var.availability_zones)`. Now reordering the list causes zero changes in the plan. Resources are identified by AZ name, not position.

**Rule of thumb:** Default to `for_each`. Use `count` only for simple on/off toggles or when creating genuinely identical, interchangeable resources.

---

## 6. How do you structure modules and code in Terraform?

Terraform modules are reusable, self-contained packages of configuration. A well-structured module layout makes the codebase scalable, avoids duplication, and lets teams work on different environments independently.

**Standard module structure:**

```
terraform/
├── modules/                      # Reusable modules (shared building blocks)
│   ├── eks/
│   │   ├── main.tf               # Resource definitions
│   │   ├── variables.tf          # Input variables
│   │   ├── outputs.tf            # Output values
│   │   └── versions.tf           # Required providers + versions
│   ├── rds/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── vpc/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── environments/                 # Environment-specific root configurations
│   ├── dev/
│   │   ├── main.tf               # Calls modules with dev-specific values
│   │   ├── variables.tf
│   │   ├── terraform.tfvars      # Dev variable values
│   │   └── backend.tf            # Remote state config for dev
│   ├── staging/
│   │   ├── main.tf
│   │   └── backend.tf
│   └── prod/
│       ├── main.tf
│       └── backend.tf
```

**Example: calling a module from an environment:**

```hcl
# environments/prod/main.tf

module "vpc" {
  source = "../../modules/vpc"

  cidr_block         = "10.0.0.0/16"
  availability_zones = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
  environment        = "prod"
}

module "eks" {
  source = "../../modules/eks"

  cluster_name    = "prod-cluster"
  vpc_id          = module.vpc.vpc_id           # Output from vpc module
  subnet_ids      = module.vpc.private_subnets  # Output from vpc module
  instance_type   = "m5.xlarge"
  desired_nodes   = 6
  environment     = "prod"
}

module "rds" {
  source = "../../modules/rds"

  identifier     = "prod-postgres"
  instance_class = "db.r6g.2xlarge"
  vpc_id         = module.vpc.vpc_id
  subnet_ids     = module.vpc.private_subnets
  multi_az       = true
}
```

**What goes in each file:**

```hcl
# variables.tf — define inputs with types and descriptions
variable "instance_type" {
  description = "EC2 instance type for EKS nodes"
  type        = string
  default     = "m5.large"
}

variable "desired_nodes" {
  description = "Desired number of worker nodes"
  type        = number
}

# outputs.tf — expose values other modules/environments can use
output "cluster_endpoint" {
  description = "EKS cluster API server endpoint"
  value       = aws_eks_cluster.main.endpoint
}

output "cluster_name" {
  value = aws_eks_cluster.main.name
}

# versions.tf — pin provider versions for reproducibility
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

**Benefits of this structure:**

- **DRY (Don't Repeat Yourself)** — VPC module written once, used in dev/staging/prod
- **Isolation** — running `terraform apply` in `environments/dev/` can only affect dev infrastructure
- **Testable** — modules can be tested independently with tools like Terratest
- **Versioned** — modules can be published to a Terraform registry and pinned to versions

```hcl
# Using a versioned module from the Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.2"    # Pinned version — reproducible builds
  ...
}

---

## 7. What is the difference between Terraform and CloudFormation?

Both are infrastructure-as-code tools, but they have different philosophies and trade-offs. Knowing when to choose which is the key part of this answer.

| | Terraform | CloudFormation |
|---|---|---|
| **Cloud support** | Multi-cloud (AWS, Azure, GCP, Kubernetes, Datadog…) | AWS only |
| **Language** | HCL (HashiCorp Configuration Language) | YAML or JSON |
| **State management** | Own state file (local or remote in S3 + DynamoDB) | Managed by AWS (no file to maintain) |
| **Rollback on failure** | Manual (fix code + re-apply or git revert) | Automatic stack rollback to last good state |
| **Dry run** | `terraform plan` | Change sets |
| **Drift detection** | `terraform plan` shows drift | Native drift detection feature |
| **Module ecosystem** | Large community registry (`registry.terraform.io`) | Nested stacks (more complex) |
| **Import existing infra** | `terraform import` | `aws cloudformation import` |

---

**When to use CloudFormation:**

- AWS-only shop with no plans to go multi-cloud
- You want automatic rollback on stack failure — CloudFormation rolls back the entire stack atomically if any resource fails to create
- Deep integration with AWS services: Service Catalog, Config Rules, StackSets for multi-account deployment
- You don't want to manage a state file

**When to use Terraform:**

- Multi-cloud or hybrid (AWS + Azure, or AWS + Kubernetes + Datadog)
- Team already familiar with HCL
- You want the strong module ecosystem — community modules for EKS, VPC, RDS save weeks of work
- You need to manage non-AWS resources (DNS in Cloudflare, monitoring in Datadog) in the same IaC workflow

---

**The rollback difference is important:**

```
CloudFormation failure:
  Stack update fails at resource 7 of 20
  → CloudFormation automatically reverses resources 1–6
  → Stack returns to previous working state

Terraform failure:
  Apply fails at resource 7 of 20
  → Resources 1–6 were created and are NOT rolled back
  → State file reflects partial apply
  → You must fix the code and re-run terraform apply
  → Or manually destroy the partially created resources
```

This is why CloudFormation is preferred in environments that need atomic, all-or-nothing deployments (financial services, regulated industries).

**Our team uses Terraform because:**
- We manage AWS + Kubernetes + Datadog in the same codebase
- The module ecosystem for EKS saved us weeks of configuration work
- `terraform plan` in CI gives us the exact diff before any change is applied
- We've built runbooks for failure recovery — the lack of auto-rollback hasn't been a practical issue

---

## 8. How do you manage Terraform provider versioning?

Provider versioning prevents breaking changes from automatically entering your infrastructure when a provider releases a new version.

**Declare version constraints in `versions.tf`.**

```hcl
terraform {
  required_version = ">= 1.6.0"    # Minimum Terraform CLI version

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"    # Allow 5.x but not 6.x
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
    helm = {
      source  = "hashicorp/helm"
      version = ">= 2.12, < 3.0"
    }
  }
}
```

**Version constraint operators:**

| Operator | Meaning | Example |
|---|---|---|
| `= 5.31.0` | Exact version only | Pins to one version |
| `~> 5.31` | Allow patch updates (5.31.x) | Safe for most cases |
| `~> 5.0` | Allow minor updates (5.x) | More flexible |
| `>= 5.0, < 6.0` | Range | Explicit bounds |

**Lock file — `.terraform.lock.hcl`.**

`terraform init` generates a lock file that records the exact provider version and checksum used:

```hcl
# .terraform.lock.hcl — commit this to Git
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",   # Integrity checksum
  ]
}
```

The lock file ensures every team member and every CI run uses the **exact same provider version**, even if a newer version in the `~> 5.0` range is released.

```bash
# Update providers (upgrades within constraints):
terraform init -upgrade

# Review what changed, commit the updated lock file
git diff .terraform.lock.hcl
git add .terraform.lock.hcl && git commit -m "upgrade aws provider to 5.35.0"
```

**Real scenario:** Our team didn't commit the lock file. AWS provider released 5.20.0 which changed the behaviour of `aws_security_group` resource — existing rules were now treated as external and removed on next apply. One engineer's `terraform plan` showed no changes (they had 5.19.0 in cache), another's showed destruction of all security group rules (5.20.0). We committed the lock file immediately. Provider upgrades are now a deliberate PR with a plan review, not an accidental side effect of running `terraform init`.

---

## 9. How to switch between dev and prod in AWS CLI and terraform ?

> **Also asked as:** "How to switch between dev and prod in AWS CLI and terraform ?"

In a proper DevOps setup, `dev` and `prod` live in completely separate AWS Accounts to minimize the blast radius of a mistake. Switching between them involves routing both your local AWS CLI and Terraform to the correct account and state.

**Step 1: Set up AWS CLI Profiles**
In your `~/.aws/credentials` or `~/.aws/config` (if using SSO), you define profiles for each environment.
```ini
[profile dev]
sso_account_id = 111111111111
sso_role_name = DeveloperAccess

[profile prod]
sso_account_id = 999999999999
sso_role_name = ReadOnlyAccess
```

To switch context in the terminal, export the profile variable:
```bash
export AWS_PROFILE=dev
aws s3 ls # Lists Dev buckets
export AWS_PROFILE=prod
aws s3 ls # Lists Prod buckets
```

**Step 2: Tell Terraform which Profile to use**
In your Terraform `providers.tf`, configure the AWS provider to respect the local environment variable, or hardcode the profile based on a variable.

```hcl
provider "aws" {
  region  = "us-east-1"
  # This tells Terraform to use the AWS_PROFILE you exported in the terminal
}
```

**Step 3: Switch Terraform State (Workspaces vs Directories)**
You cannot let Dev and Prod share the same Terraform state file. If they do, running `apply` in Prod might delete Dev resources.

*Approach A: Workspaces (Good for identical environments)*
Workspaces allow you to use the exact same directory of `.tf` files but maintain separate state files under the hood.
```bash
# Switch to Dev
export AWS_PROFILE=dev
terraform workspace select dev
terraform apply -var-file="dev.tfvars"

# Switch to Prod
export AWS_PROFILE=prod
terraform workspace select prod
terraform apply -var-file="prod.tfvars"
```

*Approach B: Directory Separation (Enterprise Standard)*
Most teams avoid Workspaces for environments because it relies on human memory: if you forget to run `workspace select prod` but pass `prod.tfvars`, you might destroy Dev. Instead, we use directory separation:
```bash
cd environments/dev
export AWS_PROFILE=dev
terraform init   # Configured perfectly for the Dev S3 bucket
terraform apply

cd ../prod
export AWS_PROFILE=prod
terraform init   # Configured perfectly for the Prod S3 bucket
terraform apply
```
This physically separates the state configurations, making accidental cross-environment destruction nearly impossible.

---

## 9. One terraform command to detect drift

> **Also asked as:** "One terraform command to detect drift"

The most reliable, automated way to detect if your actual cloud infrastructure has drifted from your Terraform code is using the detailed exit codes flag on a plan:

```bash
terraform plan -detailed-exitcode
```

**What it does:**
Normally, `terraform plan` returns an exit code of `0` regardless of whether the plan found changes or not. This makes it useless for simple CI/CD drift detection scripts unless you parse the text output.

When you add `-detailed-exitcode`, Terraform changes its exit behavior:
- **Exit Code 0:** Success, and the plan is empty. (Zero drift).
- **Exit Code 1:** Error executing the plan (e.g., syntax error, AWS API failure).
- **Exit Code 2:** Success, but the plan contains changes. (**Drift detected!**)

**How to use it in practice:**
Set up a cron job in your CI/CD platform to run nightly:

```bash
terraform plan -detailed-exitcode
if [ $? -eq 2 ]; then
  curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"🚨 Terraform Drift Detected in Production! Someone touched the console."}' \
  <SLACK_WEBHOOK_URL>
fi
```

---

## 10. Write a Terraform script for a production VPC architecture.

> **Also asked as:** "Write Terraform code for a 3-tier VPC" · "How do you provision a production-grade VPC with Terraform?" · "Give me a rough Terraform script for VPC architecture for production."

A production VPC needs: multi-AZ, public/private/DB subnet tiers, NAT Gateways per AZ, Internet Gateway, Flow Logs, and Security Groups. Here's a complete, annotated script.

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
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/vpc/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = var.region
}
```

```hcl
# variables.tf
variable "region"      { default = "ap-south-1" }
variable "env"         { default = "production" }
variable "vpc_cidr"    { default = "10.0.0.0/16" }

variable "azs" {
  default = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
}

variable "public_subnets" {
  default = ["10.0.0.0/20", "10.0.16.0/20", "10.0.32.0/20"]
}

variable "private_subnets" {
  default = ["10.0.48.0/20", "10.0.64.0/20", "10.0.80.0/20"]
}

variable "db_subnets" {
  default = ["10.0.96.0/20", "10.0.112.0/20", "10.0.128.0/20"]
}
```

```hcl
# vpc.tf

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true   # Required for EKS, RDS endpoint resolution

  tags = {
    Name        = "${var.env}-vpc"
    Environment = var.env
  }
}

# Internet Gateway — one per VPC
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.env}-igw" }
}

# Public Subnets — one per AZ
resource "aws_subnet" "public" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.azs[count.index]

  # Instances in public subnets get a public IP automatically
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.env}-public-${var.azs[count.index]}"
    # Required tags for EKS to auto-discover subnets for ALB
    "kubernetes.io/role/elb"             = "1"
    "kubernetes.io/cluster/prod-cluster" = "shared"
  }
}

# Private App Subnets — one per AZ
resource "aws_subnet" "private" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnets[count.index]
  availability_zone = var.azs[count.index]

  tags = {
    Name = "${var.env}-private-${var.azs[count.index]}"
    # Required for EKS internal load balancers
    "kubernetes.io/role/internal-elb"    = "1"
    "kubernetes.io/cluster/prod-cluster" = "shared"
  }
}

# DB Subnets — isolated, no internet route
resource "aws_subnet" "db" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.db_subnets[count.index]
  availability_zone = var.azs[count.index]

  tags = {
    Name = "${var.env}-db-${var.azs[count.index]}"
  }
}

# Elastic IPs for NAT Gateways — one per AZ
resource "aws_eip" "nat" {
  count  = length(var.azs)
  domain = "vpc"
  tags   = { Name = "${var.env}-nat-eip-${count.index + 1}" }
}

# NAT Gateways — one per AZ in public subnet
resource "aws_nat_gateway" "main" {
  count         = length(var.azs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = { Name = "${var.env}-nat-${var.azs[count.index]}" }

  depends_on = [aws_internet_gateway.main]
}
```

```hcl
# route_tables.tf

# Public route table — routes to IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${var.env}-public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private route tables — one per AZ, routes to AZ-local NAT GW
resource "aws_route_table" "private" {
  count  = length(var.azs)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = { Name = "${var.env}-private-rt-${var.azs[count.index]}" }
}

resource "aws_route_table_association" "private" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# DB subnets — no internet route (isolated)
resource "aws_route_table" "db" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.env}-db-rt" }
  # No route to 0.0.0.0/0 — DB subnets cannot reach internet
}

resource "aws_route_table_association" "db" {
  count          = length(var.azs)
  subnet_id      = aws_subnet.db[count.index].id
  route_table_id = aws_route_table.db.id
}
```

```hcl
# security_groups.tf

# ALB Security Group — internet-facing
resource "aws_security_group" "alb" {
  name   = "${var.env}-alb-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.env}-alb-sg" }
}

# App Security Group — only accepts traffic from ALB
resource "aws_security_group" "app" {
  name   = "${var.env}-app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]   # ALB SG only
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.env}-app-sg" }
}

# DB Security Group — only accepts traffic from app tier
resource "aws_security_group" "db" {
  name   = "${var.env}-db-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]   # App SG only
  }

  tags = { Name = "${var.env}-db-sg" }
}
```

```hcl
# flow_logs.tf — VPC traffic visibility

resource "aws_s3_bucket" "flow_logs" {
  bucket        = "${var.env}-vpc-flow-logs-${data.aws_caller_identity.current.account_id}"
  force_destroy = false
}

resource "aws_flow_log" "main" {
  vpc_id               = aws_vpc.main.id
  traffic_type         = "ALL"   # ACCEPT + REJECT
  log_destination_type = "s3"
  log_destination      = aws_s3_bucket.flow_logs.arn

  tags = { Name = "${var.env}-vpc-flow-log" }
}

data "aws_caller_identity" "current" {}
```

```hcl
# outputs.tf
output "vpc_id"              { value = aws_vpc.main.id }
output "public_subnet_ids"   { value = aws_subnet.public[*].id }
output "private_subnet_ids"  { value = aws_subnet.private[*].id }
output "db_subnet_ids"       { value = aws_subnet.db[*].id }
output "alb_sg_id"           { value = aws_security_group.alb.id }
output "app_sg_id"           { value = aws_security_group.app.id }
output "db_sg_id"            { value = aws_security_group.db.id }
```

**Production checklist for this VPC setup:**

```
✅ Multi-AZ: 3 AZs, 9 subnets total
✅ NAT Gateway per AZ: no cross-AZ dependency
✅ DB subnets isolated: no internet route
✅ SG chain: ALB → App → DB (no direct DB access from internet)
✅ Flow Logs: ALL traffic to S3 for audit/debugging
✅ DNS enabled: required for EKS, RDS, and service discovery
✅ EKS subnet tags: ALB controller can auto-discover subnets
✅ Remote state: S3 + DynamoDB locking
✅ Separate route table per AZ for private subnets: AZ-local NAT routing
```

**Real tip for interviews:** Walk through the `depends_on = [aws_internet_gateway.main]` on the NAT Gateway. Interviewers love seeing you understand why — the NAT Gateway lives in a public subnet, and that subnet's route to the internet goes through the IGW. If the IGW doesn't exist yet, the NAT Gateway can't function even after creation. The explicit `depends_on` makes Terraform wait for the IGW to be attached before creating the NAT Gateways.
