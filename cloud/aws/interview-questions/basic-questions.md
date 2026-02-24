# AWS — Basic Questions

---

## 1. What is AWS CloudTrail and what are its use cases?

CloudTrail is AWS's audit logging service. Every API call made in your AWS account — through the console, CLI, SDK, or any AWS service — is recorded as an event in CloudTrail. It answers the question: **"Who did what, when, from where, and on which resource?"**

**What it captures:**

```json
{
  "eventTime": "2024-01-15T14:32:01Z",
  "eventName": "DeleteBucket",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "alice"
  },
  "sourceIPAddress": "203.0.113.45",
  "requestParameters": {
    "bucketName": "prod-data-backup"
  },
  "responseElements": null,
  "awsRegion": "ap-south-1"
}
```

Every management action — creating an EC2 instance, modifying a security group, assuming an IAM role, deleting an S3 bucket — generates an event like this.

**Use cases:**

**1. Security incident investigation.**

"Someone deleted our production RDS database at 3 AM. Who was it?"

```bash
# Query CloudTrail for DeleteDBInstance events in the last 24 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteDBInstance \
  --start-time 2024-01-15T00:00:00Z
# Returns: who called it, from which IP, using which IAM role
```

Without CloudTrail, you have no answer. With CloudTrail, you have a full audit trail.

**2. Compliance and audit requirements.**

PCI-DSS, SOC 2, ISO 27001, HIPAA all require audit logs of who accessed what. CloudTrail provides this for every AWS action. Auditors ask "show me who had access to production data in the last 6 months" — CloudTrail answers it.

**3. Detecting unauthorized access or anomalous behavior.**

Route CloudTrail → CloudWatch Logs → Metric Filters + Alarms:

```hcl
# Alert when root account is used (should never happen in production)
resource "aws_cloudwatch_log_metric_filter" "root_login" {
  name           = "root-account-usage"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  pattern        = "{ $.userIdentity.type = \"Root\" }"

  metric_transformation {
    name      = "RootAccountUsage"
    namespace = "SecurityAlerts"
    value     = "1"
  }
}
```

We also alert on: IAM policy changes, security group modifications, CloudTrail itself being disabled, S3 bucket policy changes on sensitive buckets.

**4. Drift detection alongside Terraform.**

When someone changes AWS infrastructure manually (outside Terraform), CloudTrail records it. We pipe CloudTrail to a Lambda that detects changes to resources Terraform manages and sends a Slack alert: "Security group sg-0abc123 was modified manually by alice at 2:30 PM."

**5. Forensics after a breach.**

"Was any data exfiltrated?" → CloudTrail + S3 access logs shows every `GetObject` call on the S3 bucket, who made it, and the object key.

**CloudTrail setup — what we configure:**

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "prod-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true    # IAM, Route53 (global services)
  is_multi_region_trail         = true    # Capture events in ALL regions
  enable_log_file_validation    = true    # Detect if log files are tampered with

  event_selector {
    read_write_type   = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::prod-data-bucket/"]  # Log S3 data events (GetObject, PutObject)
    }
  }
}
```

`is_multi_region_trail` is critical — attackers often create resources in unusual regions to avoid detection. A single-region trail misses them.

**Real scenario:** We had an AWS access key leaked in a public GitHub repo. CloudTrail showed the key was used 47 times in 2 hours from an IP in Eastern Europe — creating EC2 instances and launching a cryptocurrency mining operation. CloudTrail told us: exactly which instances were created, in which regions, at what times. We terminated all instances, revoked the key, and had a full forensic timeline within 20 minutes. Without CloudTrail, we'd have had no idea what was done with the compromised credentials.

---

## 2. What is AWS SQS and what are its use cases?

SQS (Simple Queue Service) is a managed message queue. It decouples components of a distributed system — a producer puts messages into the queue, a consumer reads and processes them. The producer and consumer don't need to know about each other, don't need to be running at the same time, and don't need to handle failures in the other component.

**The core model:**

```
[Producer]  →  [SQS Queue]  →  [Consumer]
  (writes)    (stores msgs)    (reads + processes)

Producer can write 10,000 messages/second
Consumer processes 50 messages/second
Queue absorbs the difference — consumer catches up at its own pace
```

**Queue types:**

- **Standard queue** — at-least-once delivery, best-effort ordering. A message might be delivered twice (idempotent consumer required). Maximum throughput.
- **FIFO queue** — exactly-once delivery, strict order. Lower throughput (3,000 msg/sec). Use when order matters.

**Use cases:**

**1. Decoupling microservices — most common use case.**

Without SQS: Order service calls Payment service synchronously. If Payment is slow, Order is slow. If Payment is down, Order fails.

With SQS: Order service puts a message in the queue and returns 200 immediately. Payment service reads from the queue and processes at its own pace. Payment downtime → messages accumulate in queue, processed when service recovers.

```python
# Order service — producer
import boto3
sqs = boto3.client('sqs')
sqs.send_message(
    QueueUrl='https://sqs.ap-south-1.amazonaws.com/123456789/orders-queue',
    MessageBody=json.dumps({'order_id': 'ORD-123', 'amount': 450.00})
)
# Returns immediately — doesn't wait for payment processing
```

**2. Load leveling — absorb traffic spikes.**

A flash sale at noon creates 50,000 order requests in 10 seconds. Your backend processes 500 orders/second. Without a queue: backend overwhelmed → errors → lost orders. With SQS: 50,000 messages in queue → backend processes them at 500/sec → all orders fulfilled in 100 seconds. No user sees an error.

**3. Async processing of time-consuming tasks.**

Image upload → SQS → Lambda consumer → resize image, generate thumbnails → save to S3. User gets "upload successful" immediately. Thumbnail generation happens asynchronously. If thumbnail generation fails, the message goes back to the queue and retries.

**4. Dead Letter Queue — handling failed messages.**

```hcl
resource "aws_sqs_queue" "orders" {
  name                       = "orders-queue"
  visibility_timeout_seconds = 30

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.orders_dlq.arn
    maxReceiveCount     = 3   # After 3 failed attempts → move to DLQ
  })
}

resource "aws_sqs_queue" "orders_dlq" {
  name = "orders-dlq"
  # Messages here need investigation — why did they fail 3 times?
  # Alert on DLQ depth > 0
}
```

Messages that fail processing 3 times go to the Dead Letter Queue. We alert on DLQ depth and investigate why those messages keep failing — often reveals a bug in the consumer.

**5. Fan-out with SNS → SQS.**

One event (new user registered) needs to trigger multiple systems (send welcome email, create CRM record, provision resources):

```
SNS Topic "user-registered"
    → SQS queue → Lambda (send welcome email)
    → SQS queue → Lambda (create CRM record)
    → SQS queue → Lambda (provision user workspace)
```

Each queue is independent. If the CRM Lambda fails, the email Lambda still runs. No single point of failure.

**SQS vs SNS vs EventBridge:**

| | SQS | SNS | EventBridge |
|---|---|---|---|
| Model | Pull (consumer polls) | Push (fan-out) | Event routing |
| Retention | Up to 14 days | No storage | Limited |
| Use case | Work queue, load leveling | Notify multiple subscribers | Complex event routing |

**Real scenario:** Our report generation endpoint was causing timeouts — generating a PDF report took 45 seconds. Users were getting 504 Gateway Timeout. We moved it to SQS: user clicks "Generate Report" → message in SQS → endpoint returns 202 "Report queued" → Lambda consumer generates PDF → uploads to S3 → sends email with download link. Response time went from 45 seconds (or timeout) to 50ms. User experience dramatically improved, and Lambda retries handle transient failures automatically.

---

## 3. What are the dependent resources needed for an EC2 instance to have a publicly accessible IP?

To make an EC2 instance reachable from the internet, you need a chain of resources — each one depends on the previous. This is the minimal AWS networking stack:

```
VPC
 └── Internet Gateway (attached to VPC)
      └── Public Subnet (auto-assign public IP enabled)
           └── Route Table (0.0.0.0/0 → IGW)
                └── Security Group (inbound port 22/80/443 allowed)
                     └── EC2 Instance (launched in public subnet)
```

**1. VPC** — the isolated network your resources live in.

**2. Internet Gateway** — attaches to the VPC, enables communication between the VPC and the internet. Without it, nothing in your VPC can reach or be reached from the internet.

**3. Public Subnet** — a subnet with "Auto-assign public IPv4 address" enabled. Instances launched here get a public IP automatically.

**4. Route Table** — must have a route `0.0.0.0/0 → igw-xxxxxxxx`. Without this route, traffic can't leave the subnet to reach the internet even if IGW exists.

**5. Security Group** — stateful firewall. Must allow the ports you need (22 for SSH, 80/443 for web).

**In Terraform:**

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true   # Auto-assign public IP to instances
  availability_zone       = "ap-south-1a"
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id   # All traffic → internet
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "ec2" {
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # SSH from anywhere (restrict in production)
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]   # Allow all outbound
  }
}
```

Miss any one of these and the instance won't be publicly reachable — a common cause of "EC2 launched but can't SSH in" issues.

---

## 4. What is the command to connect to an EC2 instance, and what authentication is used?

You connect to EC2 instances using **SSH**. EC2 uses **key-pair authentication** (asymmetric cryptography) — not username/password. You use a `.pem` private key file, not a password.

**The SSH command:**

```bash
ssh -i "my-ec2-key.pem" ec2-user@<public-ip>

# For Ubuntu AMIs:
ssh -i "my-ec2-key.pem" ubuntu@<public-ip>

# For Amazon Linux 2:
ssh -i "my-ec2-key.pem" ec2-user@<public-ip>

# For Debian AMIs:
ssh -i "my-ec2-key.pem" admin@<public-ip>

# Example with actual IP:
ssh -i "~/keys/prod-ap-south-1.pem" ubuntu@13.235.67.89
```

**How the authentication works:**

1. When you launch an EC2 instance, you specify a **key pair**
2. AWS places the **public key** in `~/.ssh/authorized_keys` on the instance
3. You keep the **private key** (`.pem` file) locally
4. When you SSH, your local key proves you own the corresponding private key — no password needed

**Required: correct permissions on the .pem file:**

```bash
# SSH refuses to use the key if permissions are too open
chmod 400 my-ec2-key.pem   # Read-only for owner, nothing for others

# If you skip this:
# WARNING: UNPROTECTED PRIVATE KEY FILE!
# Permissions 0644 for 'my-ec2-key.pem' are too open.
# Bad permissions: ignore key
```

**Connect to a private instance (via bastion host / jump server):**

```bash
# Two-hop SSH — bastion is in public subnet, target is in private subnet
ssh -i key.pem -J ubuntu@<bastion-public-ip> ubuntu@<private-instance-ip>

# Or use ProxyJump in ~/.ssh/config for convenience:
Host private-instance
  HostName 10.0.11.50
  User ubuntu
  IdentityFile ~/keys/prod.pem
  ProxyJump ubuntu@13.235.67.89
```

---

## 5. How do you implement an Internet Gateway and connect it to a Route Table?

**Step 1: Create and attach the Internet Gateway to the VPC.**

```bash
# CLI:
aws ec2 create-internet-gateway
# Returns: { "InternetGateway": { "InternetGatewayId": "igw-0abc123" } }

aws ec2 attach-internet-gateway \
  --internet-gateway-id igw-0abc123 \
  --vpc-id vpc-0xyz789
```

**Step 2: Create a route table (or use an existing one) and add the IGW route.**

```bash
# Create route table
aws ec2 create-route-table --vpc-id vpc-0xyz789
# Returns: { "RouteTable": { "RouteTableId": "rtb-0def456" } }

# Add route: all internet-bound traffic goes through IGW
aws ec2 create-route \
  --route-table-id rtb-0def456 \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id igw-0abc123

# Associate route table with the public subnet
aws ec2 associate-route-table \
  --route-table-id rtb-0def456 \
  --subnet-id subnet-0ghi789
```

**In Terraform (the standard way):**

```hcl
# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = { Name = "prod-igw" }
}

# Route Table for public subnets
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"           # All outbound traffic
    gateway_id = aws_internet_gateway.main.id   # → Internet Gateway
  }

  tags = { Name = "public-rt" }
}

# Associate with each public subnet
resource "aws_route_table_association" "public_a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "public_b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}
```

**What happens without the route table association:**

The IGW exists, the route table has the IGW route, but if the route table isn't associated with the subnet — the subnet still uses the **main route table** (which has no IGW route). Instances in that subnet can't reach the internet. This is a very common misconfiguration — the IGW is attached to the VPC, but traffic still can't flow because the route table association is missing.

---

## 6. What is Route 53 and can you purchase a domain in it?

**Route 53** is AWS's managed DNS (Domain Name System) service. It translates human-readable domain names (`api.myapp.com`) into IP addresses or AWS resource endpoints (ALB DNS names, CloudFront distributions, S3 websites).

**What it does:**

```
User types: https://api.myapp.com
    ↓
Route 53 DNS query: "what is the IP for api.myapp.com?"
    ↓
Route 53 returns: "203.0.113.45" (or ALB DNS name)
    ↓
User's browser connects to 203.0.113.45
```

**Record types you work with daily:**

| Record type | Purpose | Example |
|---|---|---|
| A | Domain → IPv4 address | `api.myapp.com → 13.235.67.89` |
| CNAME | Domain → another domain | `www.myapp.com → myapp.com` |
| Alias | Domain → AWS resource (no extra lookup) | `myapp.com → alb-1234.ap-south-1.elb.amazonaws.com` |
| MX | Mail server routing | `myapp.com → mail.myapp.com` |
| TXT | Verification records, SPF | Domain ownership proof |

**Alias records are Route 53-specific** — they behave like CNAME but work at the zone apex (`myapp.com` rather than `sub.myapp.com`). Use Alias for ALBs, CloudFront, S3 websites, and other AWS resources — it's free (no DNS query charge) and auto-updates when the ALB IP changes.

**Yes — you can register/purchase a domain directly in Route 53:**

```
Route 53 Console → Registered domains → Register domain
→ Search for "myapp.com"
→ If available: add to cart, provide registrant details, pay
→ AWS registers the domain and automatically creates a hosted zone
```

Route 53 supports most TLDs (`.com`, `.in`, `.io`, `.dev`, etc.). Pricing varies by TLD — `.com` is typically $12–15/year. Route 53 also supports transferring existing domains from other registrars.

**Routing policies — Route 53 does more than basic DNS:**

- **Simple** — one record, one destination
- **Weighted** — split traffic between two endpoints (e.g., 90% → v1, 10% → v2 for canary)
- **Latency-based** — route users to the closest AWS region
- **Failover** — primary + secondary, switch automatically on health check failure
- **Geolocation** — route Indian users to `ap-south-1`, US users to `us-east-1`

---

## 7. What is NACL in AWS, and what is the difference between a Security Group and a NACL?

**NACL (Network Access Control List)** is a stateless firewall that operates at the **subnet level** in a VPC. It controls traffic entering and leaving a subnet by evaluating rules in numbered order (lowest number first).

**Security Group** is a stateful firewall that operates at the **instance (ENI) level**. It controls traffic to and from individual EC2 instances, Lambda functions, RDS instances, etc.

**Key differences:**

| | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) | Subnet |
| State | **Stateful** — return traffic allowed automatically | **Stateless** — must explicitly allow inbound AND outbound |
| Rules | Allow rules only | Allow AND Deny rules |
| Rule evaluation | All rules evaluated together | Rules evaluated in order (lowest number wins) |
| Default | Deny all inbound, allow all outbound | Allow all inbound and outbound |
| Scope | Applies to specific instances | Applies to all instances in the subnet |

**Stateful vs Stateless — the most important difference:**

```
Security Group (stateful):
  → Allow inbound port 443
  ← Return traffic (response) automatically allowed — no outbound rule needed

NACL (stateless):
  → Must explicitly allow inbound port 443
  ← Must also explicitly allow outbound ephemeral ports (1024-65535) for the response
     otherwise the response is blocked even though the request was allowed
```

**When to use each:**

- **Security Groups** — primary line of defence for every resource. Always configured.
- **NACLs** — additional layer for subnet-wide blocking. Common use: block a specific IP range or CIDR that is attacking your infrastructure across all instances in the subnet. You can't do an explicit Deny in a Security Group — you need a NACL for that.

```hcl
# NACL rule to block a specific IP (not possible with Security Groups)
resource "aws_network_acl_rule" "block_attacker" {
  network_acl_id = aws_network_acl.main.id
  rule_number    = 50          # Lower number = higher priority
  egress         = false
  protocol       = "-1"        # All traffic
  rule_action    = "deny"
  cidr_block     = "185.220.101.0/24"   # Attacker IP range
}
```

> **Also asked as:** "What's the difference between Security Groups and NACLs?" — covered above (SG = stateful, instance-level; NACL = stateless, subnet-level with explicit deny rules).

---

## 8. What is an EC2 instance store, and how is it different from an EBS volume?

**EC2 Instance Store** is temporary block storage physically attached to the host server running your EC2 instance. It's included with certain instance types (no extra cost) and provides very high I/O performance — but all data is lost when the instance stops, terminates, or the underlying hardware fails.

**EBS (Elastic Block Store)** is a persistent network-attached block storage volume. Data survives instance stops, terminations, and hardware failures. It exists independently of the instance.

**Key differences:**

| | Instance Store | EBS Volume |
|---|---|---|
| Persistence | **Ephemeral** — data lost on stop/terminate | **Persistent** — data survives independently |
| Performance | Extremely high (NVMe, locally attached) | High (up to 64,000 IOPS on io2) |
| Cost | Included with instance price | Charged per GB per month |
| Snapshots | Not supported | Snapshots to S3 supported |
| Resize | Cannot resize | Can resize without downtime |
| Reattach | Cannot detach/reattach | Can detach and attach to another instance |
| Use case | Temp files, caches, buffers, scratch space | OS disk, databases, any persistent data |

**When to use instance store:**

- High-performance temporary storage (ML training scratch space, video encoding intermediate files, Kafka/Elasticsearch data nodes where data is replicated across the cluster — losing one node's data is acceptable)
- Caching layers (Redis, Memcached) where the cache is cold-startable
- Instance types optimised for storage: `i3`, `i4i`, `d3` — designed for high IOPS workloads

**Never store on instance store:**
- Database files (RDS uses EBS for this reason)
- Application data that must survive a reboot
- Anything without a replica elsewhere

---

## 9. How many types of storage classes are there in S3, and what are they used for?

S3 has **7 storage classes**, each with different cost, availability, and retrieval characteristics:

| Storage Class | Use Case | Availability | Retrieval |
|---|---|---|---|
| **S3 Standard** | Frequently accessed data | 99.99% | Milliseconds |
| **S3 Intelligent-Tiering** | Unknown or changing access patterns | 99.9% | Milliseconds |
| **S3 Standard-IA** (Infrequent Access) | Accessed < once/month, must be fast | 99.9% | Milliseconds |
| **S3 One Zone-IA** | Infrequent access, single AZ only | 99.5% | Milliseconds |
| **S3 Glacier Instant Retrieval** | Archives accessed once/quarter | 99.9% | Milliseconds |
| **S3 Glacier Flexible Retrieval** | Archives, retrieval in minutes-hours | 99.99% | 1 min – 12 hrs |
| **S3 Glacier Deep Archive** | Long-term archives (7-10 years) | 99.99% | 12 – 48 hrs |

**Cost order (cheapest to most expensive per GB stored):**
Deep Archive → Glacier Flexible → Glacier Instant → One Zone-IA → Standard-IA → Intelligent-Tiering → Standard

**Practical usage:**

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    id     = "log-archival"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"    # After 30 days → cheaper infrequent-access
    }
    transition {
      days          = 90
      storage_class = "GLACIER_IR"     # After 90 days → archive
    }
    expiration {
      days = 365                        # Delete after 1 year
    }
  }
}
```

**Intelligent-Tiering** is the "set and forget" option — S3 automatically moves objects between frequent and infrequent access tiers based on actual access patterns. No retrieval fees, small monitoring fee per object. Best for data where access patterns are unpredictable.

---

## 10. What is the use of the m5.2xlarge instance type?

The instance type name follows a pattern: **family** + **generation** + **size**.

```
m  5  .  2xlarge
│  │       └── Size: 2xlarge = 8 vCPU, 32 GB RAM
│  └────────── Generation: 5th gen (Intel Xeon Platinum)
└───────────── Family: m = general purpose (balanced compute + memory)
```

**m5.2xlarge specs:**
- 8 vCPU
- 32 GB RAM
- Up to 10 Gbps network bandwidth
- EBS-optimized by default

**Use case:** General-purpose workloads that need a balance of compute and memory — web servers, application servers, small databases, dev/test environments, microservices. The `m` family is the default choice when you don't have a specific performance requirement skewing toward CPU or memory.

**EC2 instance family overview:**

| Family | Optimised for | Examples | Use case |
|---|---|---|---|
| `m` | General purpose (balanced) | m5, m6i, m7i | Web servers, app servers |
| `c` | Compute optimised (high CPU) | c5, c6i, c7i | CPU-intensive apps, batch processing |
| `r` | Memory optimised (high RAM) | r5, r6i, r7i | In-memory databases, Redis, SAP HANA |
| `i` | Storage optimised (NVMe) | i3, i4i | High IOPS databases, data warehouses |
| `p` / `g` | GPU | p4, g5 | ML training, video rendering |
| `t` | Burstable (CPU credits) | t3, t3a | Low-traffic dev/test, small websites |

**EC2 instance types count:** AWS has **700+ instance types** across all families and generations. In practice, most workloads use a small subset: `t3` (dev), `m5/m6i` (general), `c5/c6i` (compute), `r5/r6i` (memory), `i3/i4i` (storage).

---

## 11. What is CloudFront and how does it work as a CDN in AWS?

**CloudFront** is AWS's CDN (Content Delivery Network). It caches content at **edge locations** (Points of Presence) distributed globally — currently 400+ edge locations across 90+ cities. When a user requests content, CloudFront serves it from the nearest edge location instead of the origin server, reducing latency dramatically.

**How it works:**

```
User in Mumbai requests https://cdn.myapp.com/logo.png
    ↓
CloudFront edge location in Mumbai
    ↓ (cache hit) → serves from cache in <10ms
    ↓ (cache miss) → fetches from origin (S3/ALB/EC2), caches it, then serves

Next user in Mumbai requests the same file
    ↓
Served from Mumbai edge cache — origin is never hit again
```

**Origins CloudFront can sit in front of:**
- S3 buckets (static websites, media files)
- ALB (dynamic application content)
- EC2 instances
- Any HTTP server (on-premises, other cloud)

```hcl
resource "aws_cloudfront_distribution" "cdn" {
  origin {
    domain_name = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id   = "S3-assets"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.oai.cloudfront_access_identity_path
    }
  }

  enabled             = true
  default_root_object = "index.html"

  default_cache_behavior {
    target_origin_id       = "S3-assets"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]

    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }

    min_ttl     = 0
    default_ttl = 86400    # Cache for 1 day
    max_ttl     = 31536000 # Max 1 year
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  viewer_certificate {
    acm_certificate_arn = aws_acm_certificate.cdn.arn
    ssl_support_method  = "sni-only"
  }
}
```

**Benefits beyond caching:**
- **DDoS protection** — AWS Shield Standard is included with CloudFront at no extra cost
- **WAF integration** — attach AWS WAF to block SQL injection, rate limit IPs
- **HTTPS everywhere** — free SSL certificate via ACM, HTTPS at the edge
- **Cost** — data transfer from CloudFront to users is cheaper than from EC2/ALB directly

---

## 12. What type of database is DynamoDB — is it relational?

**DynamoDB is not relational.** It is a **NoSQL key-value and document database** — fully managed, serverless, and built for single-digit millisecond latency at any scale.

**Relational databases (SQL):**
- Data stored in tables with rows and columns
- Fixed schema — every row has the same columns
- Relationships between tables (foreign keys, JOINs)
- ACID transactions
- Scale vertically (bigger server) or read replicas
- Examples: PostgreSQL, MySQL, Oracle, SQL Server

**DynamoDB (NoSQL):**
- Data stored as items (like documents/rows) in tables
- **Flexible schema** — each item can have different attributes
- No JOINs — data is denormalized, designed for single-table access patterns
- Primary key: Partition Key (required) + Sort Key (optional)
- Scales horizontally automatically — no manual sharding
- Millisecond latency at any scale (millions of requests/second)

```python
# DynamoDB item — no fixed schema
{
    "user_id": "usr_123",          # Partition key
    "timestamp": "2024-01-15",     # Sort key
    "name": "Alice",
    "email": "alice@example.com",
    "preferences": {               # Nested document — flexible
        "theme": "dark",
        "notifications": True
    }
}
# Another item in the same table can have completely different attributes
```

**When to use DynamoDB:**
- High-throughput, low-latency lookups (user sessions, shopping carts, leaderboards)
- Known access patterns (you design the table around how you query it)
- Serverless applications (scales to zero, pay per request)
- Global applications (DynamoDB Global Tables for multi-region active-active)

**When NOT to use DynamoDB:**
- Complex queries with multiple filters, aggregations, GROUP BY — use RDS/Aurora
- Existing relational data model with JOINs — use PostgreSQL/MySQL
- Ad-hoc analytics — use Redshift or Athena

---

## 13. What type of database is PostgreSQL?

**PostgreSQL is a relational database** — specifically an open-source, object-relational database management system (ORDBMS). It uses structured tables, SQL for queries, enforces schema, and supports ACID transactions (Atomicity, Consistency, Isolation, Durability).

**Key characteristics:**

```sql
-- Tables with defined schema (every row must conform)
CREATE TABLE orders (
    order_id    SERIAL PRIMARY KEY,
    user_id     INTEGER REFERENCES users(id),   -- Foreign key relationship
    amount      DECIMAL(10,2) NOT NULL,
    created_at  TIMESTAMP DEFAULT NOW()
);

-- JOINs across related tables
SELECT u.name, COUNT(o.order_id) AS total_orders
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.name
ORDER BY total_orders DESC;
```

**Why PostgreSQL is widely used in production:**
- **ACID compliant** — financial transactions, order processing, any data where consistency matters
- **Rich data types** — JSON/JSONB, arrays, hstore, UUID, geospatial (PostGIS)
- **Extensions** — PostGIS (geospatial), pg_vector (AI embeddings), TimescaleDB (time-series)
- **Open source** — no licensing cost
- **AWS RDS / Aurora PostgreSQL** — fully managed in AWS

**PostgreSQL vs DynamoDB — when to choose:**

| Need | Choose |
|---|---|
| Complex queries, reporting, analytics | PostgreSQL |
| Known simple lookups, massive scale | DynamoDB |
| ACID transactions across multiple tables | PostgreSQL |
| Flexible schema, JSON documents | Either (Postgres has JSONB) |
| Serverless, automatic scaling | DynamoDB |
| Existing relational data model | PostgreSQL |
