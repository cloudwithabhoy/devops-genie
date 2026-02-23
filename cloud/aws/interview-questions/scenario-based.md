# AWS — Scenario-Based Questions

---

## 1. One AWS region is failing while other regions are healthy — how do you respond and what do you do?

A single-region failure means either: (1) your service is single-region and users everywhere are affected, or (2) you're multi-region and one region's traffic needs to shift to others. The response differs dramatically.

**First — confirm it's actually AWS infrastructure, not your application.**

```bash
# Check AWS Service Health Dashboard — is AWS reporting an outage?
# https://status.aws.amazon.com/ or
aws health describe-events --region us-east-1 \
  --filter '{"regions":["ap-south-1"],"eventStatusCodes":["open"]}'

# Check specific services in the affected region
aws ec2 describe-availability-zones --region ap-south-1
# If this times out or returns errors — region infrastructure issue

# Check if your EKS nodes are reachable
kubectl get nodes --context=prod-ap-south-1
```

If AWS confirms an outage — this is not something you fix. This is something you route around.

---

**If you're single-region (no DR setup):**

Your options are limited and painful. This is the moment you wish you had multi-region.

**Short-term:**
- Display a maintenance page (serve from S3 in a different region or CloudFront, which has global distribution)
- Communicate clearly to users — give realistic ETA from AWS status page

```bash
# Redirect all traffic to a static maintenance page via Route53
aws route53 change-resource-record-sets --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.yourapp.com",
        "Type": "A",
        "AliasTarget": {
          "DNSName": "maintenance.s3-website-us-east-1.amazonaws.com",
          "EvaluateTargetHealth": false,
          "HostedZoneId": "Z3AQBSTGFYJSTF"
        }
      }
    }]
  }'
```

**What to do while the region is down:**
- Start provisioning infrastructure in an alternate region from your Terraform code — this is the moment multi-region Terraform modules pay off
- Restore the latest database backup in the new region
- Update DNS to point at the new region once ready

The RTO (Recovery Time Objective) for a cold start in a new region is typically 2–4 hours. If your business can't tolerate that, you needed an active-passive DR setup already running.

---

**If you're multi-region with active-passive DR:**

This is the scenario you've prepared for.

**Step 1 — Validate the secondary region is healthy:**

```bash
# Check secondary region resources
kubectl get nodes --context=prod-us-east-1   # Secondary region
kubectl get pods -n prod --context=prod-us-east-1

# Is the DR database up-to-date?
aws rds describe-db-instances \
  --region us-east-1 \
  --query 'DBInstances[*].[DBInstanceIdentifier,StatusInfos]'
```

**Step 2 — Promote the read replica in the secondary region:**

RDS read replicas are the most common passive DR setup. Promoting makes it writable:

```bash
aws rds promote-read-replica \
  --db-instance-identifier prod-postgres-us-east-1-replica \
  --region us-east-1
# Takes 5–10 minutes — this is your RTO floor
```

**Step 3 — Fail over DNS to the secondary region:**

```bash
# Route53 health checks can do this automatically (if configured)
# Manual failover:
aws route53 change-resource-record-sets --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.yourapp.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "<secondary-region-alb-ip>"}]
      }
    }]
  }'
```

**Step 4 — Update application config to point at secondary region resources:**

The secondary region cluster needs to know about the promoted database, the regional secrets, and any regional endpoints. If you've used Helm values files per region, this is a config swap + ArgoCD sync.

```bash
# Sync the secondary region's ArgoCD to the DR values
argocd app sync order-service --grpc-web --server argocd.us-east-1.internal
```

**What this should cost you:** If your TTL was 300 seconds, DNS propagation takes 5 minutes. RDS promotion takes 10 minutes. Application startup takes 2 minutes. Total RTO: ~20 minutes. Users experience 20 minutes of degraded service (maintenance page during failover).

**After the primary region recovers:**

Don't failback immediately. AWS regions don't instantly recover from all issues — sometimes there's a partial recovery that fails again. Wait until AWS marks the event resolved and monitor for 30 minutes before failing back.

When failing back: you'll need to sync data from the promoted secondary back to the primary (it wrote data during the outage). This is a careful DB replication setup — don't failback without confirming data consistency.

**The real lesson:** Single-region production is a risk decision, not a technical one. The conversation before this incident should have been: "What's our RTO/RPO requirement? What does 2 hours of downtime cost? Is the cost of multi-region DR less than the cost of that downtime?" If the answer was yes, this incident was inevitable. If the answer was no, this is an acceptable incident and the maintenance page is the right response.

**Real scenario:** We had a single-region deployment in `ap-south-1` for a B2B platform. AWS had a control-plane issue in `ap-south-1` that lasted 3 hours. Our EKS control plane was unavailable — pods kept running but we couldn't deploy, scale, or access logs via kubectl. The pods themselves were serving traffic because the data plane was unaffected. We kept serving traffic from the frozen state but couldn't do any deployments. After this, we moved to a multi-region EKS setup with Route53 latency-based routing. The next `ap-south-1` incident lasted 40 minutes — we failed over to `ap-southeast-1` in 8 minutes and users saw only a brief service degradation.

---

## 2. Design a DR strategy with RTO/RPO considerations for a production workload.

This question is about trade-offs. Every DR strategy involves a cost (money + complexity) vs a guarantee (how fast you recover, how much data you lose). The right answer starts with business requirements, not technology choices.

**Define RTO and RPO first — everything flows from these:**

- **RTO (Recovery Time Objective)** — how long the business can tolerate being down. "We need to be back within 30 minutes" is an RTO of 30 minutes
- **RPO (Recovery Point Objective)** — how much data loss the business can tolerate. "We can lose at most 5 minutes of transactions" is an RPO of 5 minutes

The tighter the RTO/RPO, the more expensive the DR solution. The conversation before any architecture decision should be: "What does 1 hour of downtime cost the business?" If the answer is $50,000, spending $10,000/month on active-active DR is justified. If the answer is $500, a simple backup-and-restore strategy is proportionate.

**The four DR strategies (in order of cost and recovery speed):**

```
Strategy          | RTO           | RPO       | Cost multiplier
Backup & Restore  | Hours         | Hours     | 1x (cheapest)
Pilot Light       | 10–30 min     | Minutes   | 1.5–2x
Warm Standby      | 2–10 min      | Seconds   | 2–3x
Active-Active     | Near-zero     | Near-zero | 3–5x (most expensive)
```

**Strategy 1: Backup & Restore (RTO: 2–4 hours, RPO: 1–24 hours)**

Everything is backed up to S3. When disaster strikes, you provision new infrastructure from Terraform and restore from backup. Lowest cost, highest recovery time.

```hcl
# RDS automated backups — RPO is determined by backup frequency
resource "aws_db_instance" "prod" {
  backup_retention_period = 7          # Keep 7 days of backups
  backup_window           = "03:00-04:00"  # Daily backup at 3 AM
  copy_tags_to_snapshot   = true
}

# Cross-region backup copy for true DR
resource "aws_db_instance_automated_backups_replication" "cross_region" {
  source_db_instance_arn = aws_db_instance.prod.arn
  retention_period       = 7
}
```

Terraform stores infrastructure-as-code. In a disaster, `terraform apply` in a new region recreates all infrastructure in ~30 minutes. Restore RDS from the latest snapshot (~20 minutes). Total RTO: ~1 hour.

**Strategy 2: Pilot Light (RTO: 10–30 min, RPO: minutes)**

Core components run in the DR region at minimal scale (1 instance, 1 replica). When disaster strikes, scale up and redirect traffic.

```hcl
# DR region RDS — read replica (continuous replication)
resource "aws_db_instance" "dr_replica" {
  provider                = aws.dr_region   # us-east-1
  replicate_source_db     = aws_db_instance.prod.arn
  instance_class          = "db.t3.medium"  # Small instance — just keeping replication alive
  publicly_accessible     = false
  skip_final_snapshot     = false
}

# EKS in DR region — minimal node group (0 or 1 nodes)
# On activation: scale up node group, promote RDS replica, redirect DNS
```

Failover procedure:
1. Promote RDS read replica to primary (~5 minutes)
2. Scale EKS node group from 1 to desired count (~5 minutes)
3. Update Route53 to point at DR region ALB (~1 minute + TTL)

**Strategy 3: Warm Standby (RTO: 2–5 min, RPO: seconds)**

DR region runs at reduced but functional capacity. Can serve traffic immediately at reduced scale, then scale up.

```hcl
# DR region EKS — smaller but running
eks_managed_node_groups = {
  dr_general = {
    instance_types = ["m5.large"]
    min_size = 2         # Reduced vs prod (prod runs 6)
    max_size = 10        # Can scale up when needed
    desired_size = 2
  }
}

# Route53 health check — automatic failover
resource "aws_route53_health_check" "primary" {
  fqdn              = "api.primary-region.internal"
  port              = 443
  type              = "HTTPS"
  failure_threshold = "3"
  request_interval  = "10"
}

resource "aws_route53_record" "api" {
  set_identifier = "primary"
  failover_routing_policy {
    type = "PRIMARY"
  }
  health_check_id = aws_route53_health_check.primary.id
  ...
}

resource "aws_route53_record" "api_dr" {
  set_identifier = "secondary"
  failover_routing_policy {
    type = "SECONDARY"
  }
  ...
}
```

When the Route53 health check fails against the primary region, DNS automatically switches to the DR region. No human action required. RTO = Route53 health check failure time + TTL (60–120 seconds total).

**Strategy 4: Active-Active (RTO: near-zero, RPO: near-zero)**

Both regions serve traffic simultaneously. Route53 uses latency-based or weighted routing. Failure of one region: Route53 automatically routes all traffic to the other.

```hcl
resource "aws_route53_record" "api_ap_south" {
  latency_routing_policy { region = "ap-south-1" }
  set_identifier = "ap-south-1"
  ...
}

resource "aws_route53_record" "api_us_east" {
  latency_routing_policy { region = "us-east-1" }
  set_identifier = "us-east-1"
  ...
}
```

The hard problem in active-active: **database writes from both regions**. You need a globally distributed database: Amazon Aurora Global Database (sub-second replication, < 1 second RPO, < 1 minute RTO for failover) or DynamoDB Global Tables (multi-region active-active writes).

```hcl
resource "aws_rds_global_cluster" "prod" {
  global_cluster_identifier = "prod-global-cluster"
  engine                    = "aurora-postgresql"
  engine_version            = "15.4"
}
```

**What we chose and why:**

For our B2B SaaS, the business case was: 1 hour of downtime costs ~$30K in SLA penalties. We chose Warm Standby (Strategy 3). Cost: ~$3K/month extra for the DR region. ROI: one averted outage per quarter pays for 12 months of DR cost. Active-active was considered and rejected — Aurora Global Database added $4K/month and the architecture complexity (handling write conflicts, dual-region traffic routing) wasn't proportionate for our traffic patterns.

---

## 3. You need to store important documents that must never be accidentally deleted. How would you use S3 versioning and object lock?

This question checks whether you understand S3's data protection features at a practical level — not just that versioning exists, but how you configure it and what the operational implications are.

**The problem without versioning:**

```bash
# Without versioning — this is permanent and unrecoverable
aws s3 rm s3://prod-documents/contracts/client-A-agreement.pdf

# Also permanent: an application bug that overwrites the file with garbage
aws s3 cp /tmp/corrupted.pdf s3://prod-documents/contracts/client-A-agreement.pdf
# Original is gone — no way to get it back
```

**Enabling versioning:**

```hcl
resource "aws_s3_bucket_versioning" "documents" {
  bucket = aws_s3_bucket.documents.id

  versioning_configuration {
    status = "Enabled"
    # Once enabled, every PUT creates a new version
    # DELETE adds a "delete marker" — object appears deleted but all versions retained
    # MfaDelete = "Enabled" requires MFA to permanently delete versions
  }
}
```

Once versioning is enabled:
- Every `PutObject` creates a new version with a unique `VersionId`
- `DeleteObject` without a `VersionId` adds a delete marker — the object looks deleted but all previous versions are preserved
- To permanently delete, you must `DeleteObject` with the specific `VersionId`

**Restoring a previous version:**

```bash
# List all versions of a specific object
aws s3api list-object-versions \
  --bucket prod-documents \
  --prefix contracts/client-A-agreement.pdf

# Restore: copy a previous version to become the current version
aws s3api copy-object \
  --bucket prod-documents \
  --copy-source "prod-documents/contracts/client-A-agreement.pdf?versionId=abc123XYZ" \
  --key contracts/client-A-agreement.pdf

# Or: delete the delete marker to "undelete" an accidentally deleted file
aws s3api delete-object \
  --bucket prod-documents \
  --key contracts/client-A-agreement.pdf \
  --version-id <delete-marker-version-id>
```

**S3 Object Lock — for compliance and immutability:**

Versioning protects against accidental overwrites but a user with `DeleteObject` permissions can still permanently delete versions. Object Lock prevents deletion by anyone — including root — for a defined period.

Two lock modes:

- **Governance mode** — users with `s3:BypassGovernanceRetention` permission can delete. Protects against accidents, not malicious admins.
- **Compliance mode** — no one can delete or modify the object until retention expires, not even root. Used for regulatory requirements (SEBI, HIPAA, SEC 17a-4).

```hcl
resource "aws_s3_bucket_object_lock_configuration" "documents" {
  bucket = aws_s3_bucket.documents.id

  rule {
    default_retention {
      mode = "COMPLIANCE"   # or GOVERNANCE
      days = 2555           # 7 years retention for compliance
    }
  }
}
```

Per-object lock at upload time:

```python
import boto3
from datetime import datetime, timedelta

s3 = boto3.client('s3')

# Upload with explicit retention — object cannot be deleted for 7 years
s3.put_object(
    Bucket='prod-documents',
    Key='contracts/client-A-agreement.pdf',
    Body=file_content,
    ObjectLockMode='COMPLIANCE',
    ObjectLockRetainUntilDate=datetime.now() + timedelta(days=2555)
)
```

**Lifecycle policies — managing version costs:**

Versioning keeps every version indefinitely, which increases storage costs over time. Lifecycle policies clean up old versions automatically:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "documents" {
  bucket = aws_s3_bucket.documents.id

  rule {
    id     = "expire-old-versions"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"   # Move old versions to cheaper storage after 30 days
    }

    noncurrent_version_expiration {
      noncurrent_days = 365   # Delete versions older than 1 year
    }

    expiration {
      expired_object_delete_marker = true   # Clean up delete markers automatically
    }
  }
}
```

**Real scenario:** A client required 7-year document retention for regulatory compliance. We enabled versioning + Object Lock in COMPLIANCE mode with a 7-year retention period. Six months later, a developer accidentally deleted a contract folder (ran `aws s3 rm --recursive`). The delete added delete markers, but all underlying versions were locked. We deleted the delete markers and every document was restored in 5 minutes. Without Object Lock, those documents would have been permanently gone — and we would have had a regulatory audit failure.

---

## 4. When would you use EBS vs EFS, and what are the trade-offs?

EBS and EFS both provide persistent storage for AWS workloads but they're fundamentally different — using the wrong one causes performance problems, cost problems, or correctness problems.

**EBS (Elastic Block Store) — dedicated block storage, one instance at a time.**

EBS is a virtual hard drive attached to a single EC2 instance (or EKS node). Think of it like an SSD physically installed in a server — only that one server can read/write it (with Multi-Attach as an exception).

```hcl
resource "aws_ebs_volume" "data" {
  availability_zone = "ap-south-1a"
  size              = 100    # GB
  type              = "gp3"  # General Purpose SSD

  # gp3: 3,000 IOPS baseline, 125 MB/s throughput by default
  # Tunable up to 16,000 IOPS and 1,000 MB/s without extra cost vs gp2
  iops              = 3000
  throughput        = 125
}

resource "aws_volume_attachment" "data" {
  device_name = "/dev/xvdf"
  volume_id   = aws_ebs_volume.data.id
  instance_id = aws_instance.app.id
}
```

**EFS (Elastic File System) — shared NFS filesystem, many instances simultaneously.**

EFS is a managed NFS filesystem. Multiple EC2 instances, EKS pods, Lambda functions, and Fargate tasks can mount the same EFS filesystem simultaneously. Think of it like a NAS drive on the network.

```hcl
resource "aws_efs_file_system" "shared" {
  performance_mode = "generalPurpose"
  throughput_mode  = "bursting"
  encrypted        = true

  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"   # Move infrequently accessed files to cheaper IA storage
  }
}

resource "aws_efs_mount_target" "az_a" {
  file_system_id  = aws_efs_file_system.shared.id
  subnet_id       = aws_subnet.private_a.id
  security_groups = [aws_security_group.efs.id]
  # Creates an NFS endpoint in this AZ — low-latency access for instances in ap-south-1a
}
```

**Decision framework:**

| Requirement | Choose |
|---|---|
| Single instance needs fast, local-equivalent storage (database, OS disk) | EBS |
| Multiple instances/pods need to share the same files | EFS |
| Kubernetes persistent volume for stateful pod (only one pod reads/writes) | EBS (ReadWriteOnce) |
| Kubernetes shared volume (multiple pods read/write same data) | EFS (ReadWriteMany) |
| Need maximum IOPS (database-grade: 64,000+ IOPS) | EBS io2 Block Express |
| Need shared storage for a CMS, media server, or ML training data | EFS |
| Cost-sensitive, infrequent access | EFS with IA tier |

**EBS: when it's the right choice.**

```bash
# RDS uses EBS under the hood — block storage for database files
# When you size an RDS instance's storage, you're sizing its EBS volume

# EKS stateful apps: Kafka, Elasticsearch, PostgreSQL on Kubernetes
# These need ReadWriteOnce (one pod owns the disk)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: kafka-data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: gp3
  resources:
    requests:
      storage: 200Gi
```

**EFS: when it's the right choice.**

```yaml
# EKS: multiple pods need to read the same ML model files
# Model is 10GB — don't want to copy it to each pod's local storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ml-models
spec:
  accessModes: ["ReadWriteMany"]    # Multiple pods mount simultaneously
  storageClassName: efs-sc
  resources:
    requests:
      storage: 50Gi
```

EFS in Kubernetes requires the EFS CSI driver. EBS CSI driver is the default for most Kubernetes workloads.

**The cost comparison:**

- EBS gp3: ~$0.08/GB/month. You pay for provisioned capacity even if unused.
- EFS Standard: ~$0.30/GB/month for data actually stored. EFS IA: ~$0.025/GB/month.

EBS is cheaper per GB for high-utilization volumes. EFS is cost-effective when data is sparse or access is infrequent (IA tier).

**What you cannot do with EBS that you can with EFS:**

When a pod or instance using EBS is rescheduled to a different AZ, the EBS volume can't follow — EBS volumes are AZ-locked. EFS spans all AZs in a region — the pod can land anywhere and still mount the same filesystem.

**Real scenario:** We ran a media processing service where 8 worker pods each needed to read the same set of source video files and write outputs to a shared location. We initially used EBS with ReadWriteOnce — each pod had its own volume, so we had to copy the source files 8 times. After switching to EFS, one copy of the source files was mounted by all 8 pods simultaneously. Storage cost dropped from 8x the data size to 1x. More importantly, new pods added during peak processing could immediately access all files without a copy step.

---

## 5. You need to migrate 50TB of data from on-premises to AWS. How do you approach it?

Data migration at this scale has three concerns: **time** (how long will it take), **cost** (data transfer fees, service costs), **consistency** (ensuring the migrated data matches the source). The right approach depends on the answer to: "How long does the business have, and what's the network bandwidth available?"

**First: calculate whether network transfer is feasible.**

```
50 TB = 50,000 GB = 400,000 Gb (gigabits)

Dedicated 1 Gbps connection:
  400,000 Gb ÷ 1 Gbps = 400,000 seconds ÷ 3,600 = ~111 hours (~4.6 days)
  With 80% efficiency (protocol overhead): ~5.8 days

Standard business internet 100 Mbps:
  400,000 Gb ÷ 0.1 Gbps = ~46 days
  Unusable — internet is shared with production traffic

AWS Direct Connect 10 Gbps:
  ~0.6 days — viable for large migrations with existing DC connection
```

If network transfer takes more than a week (or you don't have a fast enough connection), physical transfer via AWS Snowball is faster and cheaper.

**Option 1: AWS DataSync (online transfer — up to ~10 Gbps).**

DataSync is an agent-based service that accelerates and automates data transfer. Install the DataSync agent on-premises, configure a task pointing at S3/EFS/FSx, and it handles parallel transfer, checksums, and retries.

```bash
# Create a DataSync task (via AWS CLI)
aws datasync create-task \
  --source-location-arn arn:aws:datasync:ap-south-1:123456:location/on-prem-nfs \
  --destination-location-arn arn:aws:datasync:ap-south-1:123456:location/s3-bucket \
  --name "prod-data-migration" \
  --options VerifyMode=ONLY_FILES_TRANSFERRED,OverwriteMode=ALWAYS

# Start the task
aws datasync start-task-execution --task-arn <task-arn>
```

DataSync uses multiple parallel threads and compresses data in transit. It can also do incremental syncs — run the initial transfer, then sync only changed files for the cutover.

**Option 2: AWS Snowball Edge (offline — when network is too slow).**

Snowball Edge is a physical device AWS ships to your location. You load data onto it (up to 80 TB per device), ship it back, and AWS transfers the data to S3.

```
Timeline:
  AWS ships device to you: 2–5 business days
  You load 50TB: 2–3 days (at ~250 MB/s write speed)
  Ship back + AWS import: 3–5 business days
  Total: ~10–15 days

vs. 100 Mbps internet: ~46 days
```

For 50TB with typical enterprise internet, Snowball is often 3x faster and cheaper (no egress charges, flat Snowball rental fee).

**Option 3: AWS Snowball Edge + DataSync hybrid (for large + incremental).**

1. **Initial bulk transfer** via Snowball Edge (50 TB baseline)
2. **Delta sync** via DataSync (data that changed during the Snowball transit)
3. **Cutover** — stop writes to on-premises, final DataSync sync, point application at S3

This is the standard approach for large migrations with a live source system. The Snowball handles the bulk, DataSync catches up on deltas, final cutover gap is hours not weeks.

**After transfer: verification.**

Never assume the transfer was correct. Verify checksums:

```bash
# DataSync generates a verification report automatically
aws datasync describe-task-execution --task-execution-arn <arn> \
  --query 'Result.{FilesTransferred:FilesTransferred,BytesTransferred:BytesTransferred,FilesVerified:FilesVerified}'

# Manual verification sample
aws s3api list-objects-v2 --bucket prod-data --query 'length(Contents)' --output text
# Compare count against on-premises source
```

**Migration phases for live data (zero data loss):**

```
Phase 1: Initial sync (while source is live, writes continue)
  DataSync/Snowball transfers existing 50TB
  Duration: varies by method

Phase 2: Catch-up sync
  DataSync syncs files changed since Phase 1
  Much faster — only delta

Phase 3: Cutover window (brief downtime or read-only mode)
  Stop writes to on-premises
  Final DataSync run (minutes, not hours — tiny delta)
  Validate: object counts match, checksums pass
  Switch application connection strings to S3

Phase 4: Verify and decommission
  Run application against new S3 for 1 week
  Keep on-premises as backup
  Decommission after validation period
```

**Real scenario:** We migrated 35TB of media assets from on-premises NAS to S3. Network was a 200 Mbps internet connection — estimated 16 days for a full transfer, but our production NAS was serving live traffic and we couldn't sustain 16 days of internet saturation. We used Snowball Edge for the initial 35TB bulk transfer (12 days total including transit), then DataSync for incremental sync over the last 3 days of the Snowball timeline. At cutover, DataSync had only ~400GB of delta to sync, which took 5 hours. Total cutover downtime: zero (we switched to read-only mode, ran final sync, flipped DNS in 6 hours on a Saturday night).
