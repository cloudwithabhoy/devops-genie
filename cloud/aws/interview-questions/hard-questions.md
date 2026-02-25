# AWS — Hard Questions

---

## 1. How would you design a highly available 3-tier architecture across multiple AZs?

A 3-tier architecture separates concerns into presentation (frontend), application (backend), and data (database) layers. Designing it for HA means no single AZ failure takes down any tier.

**The architecture:**

```
                    Route53 (latency/failover routing)
                           |
                    CloudFront (CDN)
                           |
              ┌────────────────────────┐
              │   Public Subnets       │
              │   ALB (multi-AZ)       │
              │   AZ-1a | AZ-1b | AZ-1c│
              └────────────────────────┘
                           |
              ┌────────────────────────┐
              │   Private Subnets      │
              │   EKS / EC2 App Layer  │
              │   AZ-1a | AZ-1b | AZ-1c│
              └────────────────────────┘
                           |
              ┌────────────────────────┐
              │   Private Subnets      │
              │   RDS Multi-AZ         │
              │   ElastiCache Cluster  │
              └────────────────────────┘
```

**Tier 1 — Presentation layer (HA):**

- Static frontend (React/Next.js) on S3 + CloudFront. S3 is already multi-AZ by design. CloudFront has 400+ edge locations — no single point of failure
- CloudFront origin failover: primary origin (S3) + fallback origin (another S3 bucket in a different region) — if primary returns 5xx, CloudFront automatically retries against the fallback

**Tier 2 — Application layer (HA):**

```hcl
module "eks" {
  source = "terraform-aws-modules/eks/aws"

  eks_managed_node_groups = {
    general = {
      # Spread nodes across all 3 AZs
      subnet_ids   = module.vpc.private_subnets   # [1a, 1b, 1c]
      min_size     = 3    # At minimum 1 per AZ
      max_size     = 9
      desired_size = 3
    }
  }
}
```

- ALB is inherently multi-AZ — it distributes traffic across targets in all AZs
- EKS pods deployed with anti-affinity rules to spread across AZs:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: topology.kubernetes.io/zone
          labelSelector:
            matchLabels:
              app: order-service
```

- HPA ensures min replicas > 1 so an AZ failure doesn't take the service to zero
- PodDisruptionBudget: `minAvailable: 2` so no maintenance window can take all pods down

**Tier 3 — Data layer (HA):**

```hcl
# RDS Multi-AZ — automatic failover to standby replica in another AZ
resource "aws_db_instance" "prod" {
  multi_az               = true    # Synchronous standby in different AZ
  db_subnet_group_name   = aws_db_subnet_group.prod.name   # Subnets in 3 AZs
  deletion_protection    = true
}

# ElastiCache Redis cluster mode — shards across AZs
resource "aws_elasticache_replication_group" "redis" {
  num_cache_clusters         = 3    # 1 primary + 2 replicas across AZs
  automatic_failover_enabled = true
  multi_az_enabled           = true
  subnet_group_name          = aws_elasticache_subnet_group.redis.name
}
```

RDS Multi-AZ failover time: ~60–120 seconds (automatic, DNS-based). If this is too slow, use RDS Proxy — it maintains a connection pool and handles the failover transparently to the application (failover time drops to ~5 seconds).

**What makes HA actually work — the things people miss:**

1. **Health checks matter more than redundancy.** ALB only routes to healthy targets. EKS only sends traffic to pods passing readiness probes. If your health check endpoint is `/health` but you changed it to `/api/health`, you'll get 0 healthy targets in the ALB, even though your app is running
2. **Data layer is the hardest tier.** Compute (EC2, EKS) fails over in seconds. RDS failover is 60–120 seconds. Stateful data that's written to only one AZ is the last mile problem in every HA design
3. **Cross-AZ data transfer costs.** Spreading across AZs isn't free. Every byte that crosses an AZ boundary costs $0.01/GB each way. ElastiCache nodes in different AZs than the app pods = cross-AZ traffic on every cache read. Design read replicas in the same AZ as the pods that read from them

**Real scenario:** Our first "multi-AZ" deployment was actually single-AZ in disguise — we had 3 EC2 instances all in `ap-south-1a`. The ALB was multi-AZ but all targets were in one AZ. When `ap-south-1a` had a brief outage, the ALB had no healthy targets. 22-minute outage. After that: min 1 instance per AZ enforced by ASG AZ balancing, RDS Multi-AZ enabled, ElastiCache cluster mode. Next AZ issue: 0 downtime.

> **Also asked as:** "Explain how you would design a highly available EC2 architecture across multiple AZs for a web application." — covered above (ALB + ASG across ≥2 AZs, RDS Multi-AZ, ElastiCache Multi-AZ, stateless app tier).

---

## 2. How do you implement cross-account access securely in AWS?

Cross-account access is needed when you have separate AWS accounts (prod, staging, dev, shared-services) and one account needs to access resources in another. The secure way is IAM role assumption — never share credentials between accounts.

**The mechanism: `sts:AssumeRole`**

Account A (the "trusting" account) creates a role with a trust policy that allows Account B's identity to assume it. Account B's identity calls `sts:AssumeRole` and gets temporary credentials.

```
Account B (dev)          Account A (prod)
  IAM User / Role  →  sts:AssumeRole  →  Role in Account A
                                          (with limited permissions)
                    ←  Temporary credentials (15min – 12h)
```

**Setting up cross-account access (Terraform):**

```hcl
# In Account A (prod) — create a role that Account B can assume
resource "aws_iam_role" "cross_account_reader" {
  name = "cross-account-read-role"

  # Trust policy — who can assume this role
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = {
        AWS = "arn:aws:iam::ACCOUNT_B_ID:role/ci-deploy-role"   # Specific role, not entire account
      }
      Action    = "sts:AssumeRole"
      Condition = {
        StringEquals = {
          "sts:ExternalId" = "prod-read-access-external-id"    # Extra verification token
        }
      }
    }]
  })
}

# Attach permissions to the role — least privilege
resource "aws_iam_role_policy" "cross_account_reader_policy" {
  role = aws_iam_role.cross_account_reader.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:ListBucket"]
      Resource = [
        "arn:aws:s3:::prod-artifacts",
        "arn:aws:s3:::prod-artifacts/*"
      ]
    }]
  })
}
```

```hcl
# In Account B (dev/CI) — allow the CI role to assume the prod role
resource "aws_iam_role_policy" "assume_prod_reader" {
  role = aws_iam_role.ci_deploy.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "sts:AssumeRole"
      Resource = "arn:aws:iam::ACCOUNT_A_ID:role/cross-account-read-role"
    }]
  })
}
```

**Using the cross-account role:**

```bash
# In CI pipeline — assume the prod account role
CREDS=$(aws sts assume-role \
  --role-arn "arn:aws:iam::ACCOUNT_A_ID:role/cross-account-read-role" \
  --role-session-name "ci-pipeline-deploy" \
  --external-id "prod-read-access-external-id" \
  --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
  --output text)

export AWS_ACCESS_KEY_ID=$(echo $CREDS | awk '{print $1}')
export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | awk '{print $2}')
export AWS_SESSION_TOKEN=$(echo $CREDS | awk '{print $3}')

# Now all subsequent AWS CLI calls use the prod account role
aws s3 ls s3://prod-artifacts/
```

**The ExternalId condition — why it matters:**

Without `ExternalId`, if Account C's identity somehow gets permission to assume Account A's role, they can access Account A's resources. `ExternalId` is a shared secret between Account A and Account B — Account C doesn't know it. This prevents the "confused deputy" attack where a third party tricks a service into acting on their behalf.

**Multi-account pattern we use (AWS Organizations):**

```
Management Account (billing, Organizations)
├── Security Account (CloudTrail, Security Hub, GuardDuty)
├── Shared Services Account (ECR, Artifactory, Jenkins)
├── Prod Account (production workloads)
├── Staging Account (pre-prod)
└── Dev Account (development, sandboxes)
```

- CI/CD in Shared Services assumes roles in Prod/Staging/Dev to deploy
- Security Account's GuardDuty monitors all accounts, findings aggregated centrally
- ECR in Shared Services is accessible from Prod/Staging/Dev via cross-account repository policies
- No long-lived credentials ever leave their origin account

**ECR cross-account access (repository policy):**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowProdAccountPull",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::PROD_ACCOUNT_ID:root"
    },
    "Action": [
      "ecr:GetDownloadUrlForLayer",
      "ecr:BatchGetImage",
      "ecr:BatchCheckLayerAvailability"
    ]
  }]
}
```

EKS nodes in the prod account can pull images from the shared-services ECR without any credential sharing — IAM role assumption handles it transparently.

> **Also asked as:** "Your organization needs to grant cross-account access to a partner. How would you implement it?" — covered above (IAM role in your account, trust policy allowing partner's account, STS AssumeRole, no credential sharing).

---

## 3. How would you reduce AWS cost in a Kubernetes-based production workload?

Cost reduction in K8s on AWS is not about cutting corners — it's about eliminating waste. Most teams over-provision compute, pay for idle resources, and miss several AWS pricing mechanisms. Here's how we systematically reduced our bill by ~40%.

**1. Right-size node instances using VPA recommendations.**

The most common waste is over-provisioned pods. VPA in recommendation mode tells you actual CPU/memory usage vs what you requested:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: order-service-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  updatePolicy:
    updateMode: "Off"   # Recommendation only
```

We found 60% of our pods had CPU requests 3–5x their actual usage. A pod requesting 1 CPU that uses 200m is holding a node slot for 800m that nothing else can use. Reducing requests to 300m (with headroom) cut our node count by 30%.

**2. Spot instances for non-critical workloads.**

EKS managed node groups support mixed instance types with spot instances. Spot is 70–90% cheaper than on-demand:

```hcl
eks_managed_node_groups = {
  spot_workers = {
    capacity_type  = "SPOT"
    instance_types = ["m5.xlarge", "m5a.xlarge", "m4.xlarge"]  # Multiple types = higher spot availability
    min_size       = 3
    max_size       = 20

    labels = {
      "node-type" = "spot"
    }

    taints = [{
      key    = "spot"
      value  = "true"
      effect = "NO_SCHEDULE"
    }]
  }

  on_demand_core = {
    capacity_type = "ON_DEMAND"
    instance_types = ["m5.large"]
    min_size = 3
    max_size = 6
    labels = { "node-type" = "on-demand" }
  }
}
```

Stateless, fault-tolerant workloads (workers, data pipelines, batch jobs) run on spot. Stateful and critical services (payments, auth, databases) stay on on-demand. Spot interruption handling: `terminationGracePeriodSeconds: 120` + `PodDisruptionBudget` ensures pods drain gracefully when AWS reclaims a spot instance (2-minute warning).

**3. Karpenter instead of Cluster Autoscaler.**

Cluster Autoscaler provisions whole nodes (fixed instance types, 3–5 minute wait). Karpenter provisions exactly the right instance type for the pending pods, in under 60 seconds:

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot", "on-demand"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values: ["m5.large", "m5.xlarge", "m5.2xlarge", "c5.large", "c5.xlarge"]
  ttlSecondsAfterEmpty: 30    # Terminate node 30s after last pod leaves — no idle billing
```

`ttlSecondsAfterEmpty: 30` is the key cost saver. Cluster Autoscaler waits 10 minutes before scaling down. We were paying for idle nodes for 10 minutes after every scale-in event. Karpenter reclaims them in 30 seconds.

**4. VPC Endpoints to eliminate NAT Gateway costs.**

NAT Gateway charges $0.045/GB processed. EKS pods pulling from ECR, Secrets Manager, and S3 through NAT Gateway generate significant data transfer costs at scale.

VPC Endpoints route this traffic privately within AWS, bypassing NAT Gateway entirely:

```hcl
# ECR endpoints — image pulls stay private, no NAT Gateway cost
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.ap-south-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.ap-south-1.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}

# S3 Gateway endpoint — free
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.ap-south-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc.private_route_table_ids
}
```

For us: ECR + S3 endpoints saved ~$800/month in NAT Gateway data processing costs. The endpoints themselves cost $7.30/month each — net positive immediately.

**5. Savings Plans and Reserved Instances for stable workloads.**

For on-demand baseline nodes that run 24/7, Compute Savings Plans give 66% discount with 1-year commitment, 40% with no commitment (vs on-demand):

```bash
# Check your EC2 usage to find steady-state baseline
aws ce get-reservation-purchase-recommendation \
  --service "Amazon EC2" \
  --lookback-period SIXTY_DAYS
```

We committed to Compute Savings Plans for the 6 always-on `m5.large` nodes (our minimum). Everything above that uses spot. Savings Plan covers the base cost at 66% discount; spot handles burst at 70–90% discount.

**6. Delete orphaned resources — the easiest wins.**

```bash
# Unattached EBS volumes (still billed even when not attached)
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]' --output table

# Unused load balancers
aws elbv2 describe-load-balancers \
  --query 'LoadBalancers[*].[LoadBalancerArn,CreatedTime]'

# Old EBS snapshots from terminated instances
aws ec2 describe-snapshots --owner-ids self \
  --query 'Snapshots[?StartTime<=`2024-01-01`].[SnapshotId,VolumeSize,StartTime]'

# Idle NAT Gateways (in VPCs with no private subnet traffic)
aws cloudwatch get-metric-statistics --namespace AWS/NatGateway \
  --metric-name BytesOutToSource --period 86400 --statistics Sum \
  --start-time 2024-01-01T00:00:00Z --end-time 2024-01-15T00:00:00Z \
  --dimensions Name=NatGatewayId,Value=nat-0abc123
```

We ran a monthly cleanup script and found $400/month in orphaned EBS volumes from terminated nodes, unused ECR images, and old snapshots.

**Real results:** Starting point: $18,000/month AWS bill for our EKS platform. After: right-sizing (−30%), spot migration (−25% on worker nodes), Karpenter (−15% idle node time), VPC endpoints (−$800), Savings Plans (−$1,200), orphan cleanup (−$400). 12 months later: $10,500/month. 42% reduction without removing any capability.

---

## 4. How to use VPC Flow Logs to trace a dropped connection

> **Also asked as:** "How to use VPC Flow Logs to trace a dropped connection"

When an application times out connecting to a database, the network is always blamed first. VPC Flow Logs are the source of truth for whether the network actually dropped the packet or if the application ignored it.

**The Setup:**
VPC Flow Logs capture IP traffic going to and from network interfaces in your VPC. They are published to CloudWatch Logs or S3.

**The Scenario:** 
An EC2 instance (App) in Subnet A cannot connect to an RDS instance (DB) in Subnet B on port 5432.

**Step 1: Get the IPs and ENIs**
Find the private IP of the App instance (e.g., `10.0.1.50`) and the private IP of the RDS instance (e.g., `10.0.2.100`).

**Step 2: Query CloudWatch Logs Insights**
Open the log group for your VPC Flow Logs and run this query to filter for traffic between these two IPs on port 5432:

```text
fields @timestamp, srcAddr, dstAddr, srcPort, dstPort, action, status
| filter (srcAddr = '10.0.1.50' and dstAddr = '10.0.2.100' and dstPort = 5432)
| sort @timestamp desc
| limit 20
```

**Step 3: Analyze the `action` field**
There are two possible outcomes in the logs:

- **Scenario A: `REJECT`**
  If the action is `REJECT`, the network blocked it.
  *Root cause:* Look at the Security Groups and Network ACLs. 99% of the time, the App's Security Group is missing an outbound rule to the DB, or the DB's Security Group is missing an inbound rule from the App's Security Group ID.

- **Scenario B: `ACCEPT`**
  If the action is `ACCEPT`, the network delivered the packet successfully.
  *Root cause:* The network is fine. The database server is either down, actively refusing connections (e.g., PostgreSQL `pg_hba.conf` issue), or the connection pool is maxed out. Stop looking at AWS networking and start looking at the database logs.

**Why this matters in an interview:** It proves you don't just guess by clicking around the AWS console. You definitively prove where the packet died using data.

---

## 5. Architectural scenario: Read Replicas vs. ElastiCache

> **Also asked as:** "Architectural scenario: Read Replicas vs. ElastiCache"

When a database component becomes the bottleneck due to high read volume, you have two primary scaling options: RDS Read Replicas or an ElastiCache (Redis/Memcached) layer. Choosing the wrong one introduces unnecessary complexity or fails to solve the latency problem.

**When to choose RDS Read Replicas:**
- **The Problem:** CPU usage on the primary database is at 90% because of complex, heavy `SELECT` queries (e.g., generating weekly reports, BI dashboards, or complex table joins).
- **The Data Pattern:** Operations require standard SQL relationships, transaction support, and querying against real-time (or near real-time) row data.
- **The Implementation:** You spin up an RDS Read Replica. You change the application code to route "Write" operations (`INSERT`, `UPDATE`, `DELETE`) to the primary endpoint, and "Read" operations (`SELECT`) to the reader endpoint.
- **Latency Profile:** Read Replicas improve *throughput* (handling more concurrent queries), but they do not significantly improve the *latency* of an individual complex query. If a heavy JOIN takes 2 seconds to run on the primary, it will still take 2 seconds on the replica.

**When to choose ElastiCache (Redis):**
- **The Problem:** The database is overwhelmed by millions of identical, simple queries being run repeatedly (e.g., loading the top 10 products on a homepage, fetching a user's session profile).
- **The Data Pattern:** Key-value lookups where the data doesn't change every second.
- **The Implementation:** You put Redis in front of the database. The application checks Redis first (`GET user:123`). If it's a "cache hit", it returns instantly. If it's a "cache miss", it queries RDS, stores the result in Redis (`SET user:123 result EX 300`), and then returns it.
- **Latency Profile:** Redis operates in sub-millisecond territory (in-memory). It drastically improves both *throughput and latency*. A query that took 50ms in RDS now takes 0.5ms.

**The Hybrid Approach (The standard for scale):**
You use *both*. The application reads user sessions and homepage catalogs from Redis. When a user runs a complex search query that bypasses the cache, that query hits the RDS Read Replica. The RDS Primary is reserved exclusively for processing financial transactions and user updates.
