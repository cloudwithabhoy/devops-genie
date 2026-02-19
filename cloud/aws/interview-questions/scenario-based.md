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
