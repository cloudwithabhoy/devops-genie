# IaC — Medium Questions

---

## 1. Explain the Terraform lifecycle, terraform refresh, and terraform import.

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
