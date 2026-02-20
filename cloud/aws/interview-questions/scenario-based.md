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
