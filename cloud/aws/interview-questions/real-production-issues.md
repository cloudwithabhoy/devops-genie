# AWS — Real Production Issues

---

## 1. Cost spiked 40% overnight — how do you identify the exact root cause, not guesses?

A 40% overnight spike is significant enough that something specific happened — it's not gradual drift. The approach is structured elimination, starting with the highest-spend services and narrowing to the exact resource.

**Step 1 — AWS Cost Explorer: isolate the service and time.**

```
AWS Console → Cost Explorer → Daily view
→ Group by: Service
→ Date range: last 7 days (to see the spike vs baseline)
```

This immediately shows which service drove the spike. 90% of the time it's one of: EC2 (new instances, NAT Gateway data transfer), RDS (storage autoscaling, snapshot copies), S3 (egress, unexpected PUT/GET volume), or data transfer.

If Cost Explorer shows the spike is in "EC2-Other" — that's usually **NAT Gateway data transfer** or **data transfer between AZs**. This is the most common hidden cost spike and the most overlooked.

**Step 2 — Narrow to the exact resource using Cost Allocation Tags.**

Every resource in our account has tags: `Environment`, `Service`, `Team`, `Owner`. In Cost Explorer:

```
Filter by: Tag → Environment = prod
Group by: Tag → Service
```

This tells you not just which AWS service (EC2, RDS) but which application service (order-service, payment-service) drove the cost.

If you don't have cost allocation tags — add them. This is the single most impactful thing you can do for AWS cost visibility. Every Terraform resource in our stack has:

```hcl
tags = {
  Environment = var.environment
  Service     = var.service_name
  Team        = var.team
  ManagedBy   = "terraform"
}
```

**Step 3 — Check what changed the night of the spike.**

```bash
# CloudTrail — what API calls happened around the spike time?
aws cloudtrail lookup-events \
  --start-time "2024-01-15T00:00:00Z" \
  --end-time "2024-01-15T08:00:00Z" \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances

# Or check for any resource creation
aws cloudtrail lookup-events \
  --start-time "2024-01-15T00:00:00Z" \
  --end-time "2024-01-15T08:00:00Z" \
  | jq '.Events[] | {time: .EventTime, name: .EventName, user: .Username}'
```

If `RunInstances` (EC2 launch), `CreateDBInstance` (RDS), or `CreateNatGateway` events appear around the spike time — that's your culprit.

**Step 4 — Common spike root causes and how to confirm each.**

**Autoscaling event — EC2 instances stayed up.**

A traffic spike triggered scale-out. Traffic subsided, but scale-in didn't happen (cooldown too long, scale-in policy missing, or instances got stuck).

```bash
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,LaunchTime,InstanceType]' \
  --output table | sort -k3
# Look for instances launched around the spike time that are still running
```

**S3 unexpected data egress.**

Someone ran a large `aws s3 sync` to a local machine, or an application started reading large objects repeatedly, or a public-facing S3 bucket got scraped.

```bash
# S3 request metrics — check PutObject, GetObject counts
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name NumberOfObjects \
  --dimensions Name=BucketName,Value=my-bucket Name=StorageType,Value=AllStorageTypes \
  --start-time 2024-01-14T00:00:00Z \
  --end-time 2024-01-16T00:00:00Z \
  --period 3600 --statistics Average
```

Enable S3 server access logging or S3 Storage Lens for ongoing visibility.

**NAT Gateway data transfer.**

The most common surprise charge. NAT Gateway charges per GB of data processed. A new service that pulls large files, a misconfigured log shipper sending to S3 cross-AZ, or a container pulling a large base image on every startup — all generate NAT Gateway data transfer costs.

```bash
# Check NAT Gateway bytes processed
aws cloudwatch get-metric-statistics \
  --namespace AWS/NatGateway \
  --metric-name BytesOutToSource \
  --dimensions Name=NatGatewayId,Value=nat-0abc123 \
  --start-time 2024-01-14T00:00:00Z \
  --end-time 2024-01-16T00:00:00Z \
  --period 3600 --statistics Sum
```

**RDS storage autoscaling.**

RDS can auto-scale storage when it runs low. Storage increases are permanent (RDS won't shrink storage). A one-time bulk insert caused the DB to hit 80% storage → autoscaled to 2x → doubled the storage cost permanently.

```bash
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,AllocatedStorage,MaxAllocatedStorage]' \
  --output table
# Compare AllocatedStorage vs what your Terraform config specifies
```

**Lambda duration spike.**

A new Lambda function with inefficient code, or a timeout increase that allows functions to run longer.

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=my-function \
  --start-time 2024-01-14T00:00:00Z --end-time 2024-01-16T00:00:00Z \
  --period 3600 --statistics Average
```

**Step 5 — Set up proactive cost alerts so this doesn't surprise you.**

```hcl
# Terraform — budget alert at 80% of expected monthly spend
resource "aws_budgets_budget" "monthly" {
  name         = "monthly-budget-alert"
  budget_type  = "COST"
  limit_amount = "5000"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["devops@company.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["devops@company.com"]
  }
}
```

A 40% overnight spike should trigger an alert, not a morning surprise. We alert at 80% of expected spend (actual) and 100% of expected spend (forecasted). The forecasted alert catches runaway spending mid-month before it becomes a bill.

**Real scenario:** We got hit with a $3,200 unexpected charge in one night. Cost Explorer showed the spike in "EC2-Other." Tags pointed to our data-pipeline team. CloudTrail showed a `RunInstances` call at 11 PM — a data engineer had run a one-off Spark job on 40 `r5.4xlarge` instances and forgot to terminate them after it finished. The job completed in 2 hours, but the instances ran all night. After this, we added Instance Scheduler for non-production EC2 instances and mandatory `--instance-initiated-shutdown-behavior terminate` on all spot instances. Also added a CloudWatch alarm: if EC2 spend exceeds $500 in any 1-hour window, PagerDuty fires immediately.

---

## 2. ALB is healthy, but users are still getting 502 errors — how do you troubleshoot?

"ALB healthy" means the ALB itself is running and its listeners are configured. 502 means the ALB received a bad or no response from the backend targets. The ALB is fine; something between the ALB and your application isn't.

**Step 1 — Check target group health, not just ALB health.**

The ALB can be "healthy" (running, listeners active) while all its targets are unhealthy (marked as draining, returning errors, or failing health checks).

```bash
# Get your target group ARN
aws elbv2 describe-target-groups --names my-app-tg

# Check health of every registered target
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:ap-south-1:123456789:targetgroup/my-app-tg/abc123

# Output you want to see:
# "TargetHealth": {"State": "healthy"}
# Output that means 502:
# "TargetHealth": {"State": "unhealthy", "Reason": "Target.ResponseCodeMismatch"}
# "TargetHealth": {"State": "draining"}
# "TargetHealth": {"State": "unused"}  ← target registered but AZ not enabled on ALB
```

**Unhealthy reason codes and what they mean:**

| Reason | Root cause |
|---|---|
| `Target.ResponseCodeMismatch` | Health check expects 200, app returns 301/404/500 |
| `Target.Timeout` | App not responding within health check timeout |
| `Target.FailedHealthChecks` | Health check path unreachable (app not started, wrong port) |
| `Elb.InternalError` | ALB internal issue (rare) |
| `Target.DeregistrationInProgress` | Target is draining — deployment in progress |

**Step 2 — Check the health check configuration.**

The #1 cause of ALB 502: the health check path doesn't match what the app actually serves.

```bash
aws elbv2 describe-target-groups \
  --target-group-arns arn:aws:elasticloadbalancing:... \
  --query 'TargetGroups[*].{Path:HealthCheckPath,Port:HealthCheckPort,Code:Matcher}'
```

Common mismatches:
- Health check path is `/health`, app serves it at `/api/health` after a recent refactor
- Health check port is 80, app listens on 8080
- Health check expects `200` but app returns `204` on health endpoint
- App has `requireHttps` middleware that redirects port-80 health checks to 443 → ALB gets 301 → marks unhealthy

**Step 3 — Check ALB access logs for the actual error.**

ALB access logs contain the status code returned by the target, the target IP, and processing time. This is more specific than CloudWatch metrics.

```bash
# Enable access logs if not already (Terraform)
resource "aws_lb" "main" {
  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb"
    enabled = true
  }
}

# Query the logs in S3 (or use Athena for large volumes)
aws s3 cp s3://my-alb-logs/alb/AWSLogs/123456789/elasticloadbalancing/ap-south-1/2024/01/15/ . --recursive

# In the log: look for target_status_code and elb_status_code
# elb=502 + target_status_code=- means the target closed the connection
# elb=502 + target_status_code=500 means target returned 500 which ALB converted to 502
grep " 502 " *.log | head -20
```

**Step 4 — Check security groups.**

The ALB's security group must allow outbound to the target's port. The target's security group must allow inbound from the ALB's security group. If either is missing, ALB can't reach the target → 502.

```bash
# What security group does the ALB use?
aws elbv2 describe-load-balancers --query 'LoadBalancers[*].SecurityGroups'

# Does the target security group allow inbound from the ALB SG?
aws ec2 describe-security-groups --group-ids <target-sg-id> \
  --query 'SecurityGroups[*].IpPermissions'
# Look for inbound rule: port 8080, source = ALB security group ID
```

Missing this rule is the second most common cause of ALB 502s after health check misconfiguration.

**Step 5 — Check for connection timeout vs response timeout.**

ALB has two timeout settings:
- **Idle timeout** (default: 60 seconds) — if no data flows between client and target for 60 seconds, ALB closes the connection
- **Target response timeout** — if the target takes longer than this to respond, ALB returns 504 (not 502)

But if the target resets the TCP connection (crashes, runs out of file descriptors, connection pool exhausted), ALB returns 502.

```bash
# Check if targets are restarting / connection-refusing
# From an EC2 instance in the same VPC, directly hit the target:
curl -v http://<target-private-ip>:8080/health
# "connection refused" or "connection reset" → target is the problem, not ALB
```

**The 30-second troubleshooting sequence:**

```bash
# 1. Target health?
aws elbv2 describe-target-health --target-group-arn <arn>

# 2. Health check config matches app?
aws elbv2 describe-target-groups --target-group-arns <arn> \
  --query 'TargetGroups[*].{Path:HealthCheckPath,Port:HealthCheckPort}'

# 3. Security group allows ALB → target?
aws ec2 describe-security-groups --group-ids <target-sg-id>

# 4. Hit target directly from VPC
curl -v http://<target-ip>:<port>/health
```

**Real scenario:** We had a production 502 that lasted 8 minutes. ALB console showed "Active" and listeners were configured correctly. The target group showed 3 healthy targets. But 502s persisted. The issue: a new deployment had added an HTTP → HTTPS redirect middleware. The ALB health check was on port 80 (HTTP). The middleware returned 301 to every request on port 80, including the health check. The ALB marked the targets as healthy (it accepted the 301 by default — we had `HttpCode: 200-301` in the matcher). But real user requests were also getting 301, and the ALB was configured for HTTP only — it couldn't follow the redirect. Fixed by: changing the ALB to terminate HTTPS and routing to port 80 on targets, removing the middleware redirect. 502s stopped immediately.

---

## 3. Application is deployed on AWS but not accessible from the browser — how do you debug it?

"Not accessible from the browser" is vague — that's intentional. The interviewer wants to see a systematic elimination process, not a guess. The answer moves from the outermost layer (DNS/internet) inward (load balancer → EC2/EKS → application).

**The layered debugging model:**

```
Browser → DNS → Internet → Security Group / NACL → Load Balancer → Target → Application
```

Each layer can be the failure point. You eliminate them in order.

**Layer 1: Is it a DNS problem?**

```bash
# Does the domain resolve?
nslookup api.myapp.com
# Or:
dig api.myapp.com +short

# Expected: returns an IP or ALB CNAME
# If no answer / NXDOMAIN → DNS record missing or propagating
```

If DNS doesn't resolve: check Route53 hosted zone → the record must exist and point at the right ALB/CloudFront/EC2 endpoint.

```bash
# Check Route53 directly (bypassing DNS cache)
aws route53 list-resource-record-sets --hosted-zone-id <zone-id> \
  --query "ResourceRecordSets[?Name=='api.myapp.com.']"
```

**Layer 2: Does the ALB/endpoint respond?**

```bash
# curl with verbose output — shows TLS handshake, redirects, response headers
curl -v https://api.myapp.com/health

# If the domain resolves but no connection:
# → Security Group on ALB not allowing port 443 from 0.0.0.0/0
# → ALB listener not configured for port 443
# → NACL blocking inbound on port 443

# Test HTTP separately (confirm whether it's HTTP or HTTPS config)
curl -v http://api.myapp.com/health
```

**Check the ALB security group:**

```bash
aws ec2 describe-security-groups \
  --group-ids <alb-security-group-id> \
  --query 'SecurityGroups[*].IpPermissions'
# Must see: inbound rule allowing port 443 (and 80) from 0.0.0.0/0
```

**Layer 3: Are the ALB targets healthy?**

The ALB could be reachable but returning 502/503 because all targets are unhealthy.

```bash
# Check target group health
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>

# Look for:
# "State": "unhealthy"
# "Reason": "Target.HealthCheckFailed" or "Target.NotRegistered"
```

Common reasons targets are unhealthy:
- **Health check path is wrong** — app is running but `/health` returns 404
- **Security group on target doesn't allow traffic from ALB** — ALB's security group must be in the inbound rules of the target's security group
- **Port mismatch** — ALB forwards to port 8080 but app listens on 3000
- **App is starting up** — pods/instances in initializing state fail health checks until ready

```bash
# Verify the target security group allows traffic from the ALB SG
aws ec2 describe-security-groups --group-ids <target-sg-id> \
  --query 'SecurityGroups[*].IpPermissions'
# Must see: inbound rule on app port (e.g., 8080) from ALB security group ID
```

**Layer 4: Is the application running on the target?**

If the ALB can't reach the target, test the target directly (from another instance in the same VPC):

```bash
# From a bastion host or another EC2 in the same VPC:
curl -v http://<target-private-ip>:8080/health

# If this fails:
# "Connection refused" → app is not running on that port
# "Connection timed out" → security group blocking or app crashed

# Check what's actually listening on the instance:
ssh ec2-user@<bastion-ip>
ssh ec2-user@<target-private-ip>
ss -tlnp | grep LISTEN   # What ports are open?
```

**Layer 5: Is HTTPS configured correctly?**

Browser shows "not secure" or ERR_SSL_PROTOCOL_ERROR:

```bash
# Check the ALB certificate
aws elbv2 describe-listeners --load-balancer-arn <alb-arn> \
  --query 'Listeners[?Protocol==`HTTPS`].Certificates'

# Verify certificate is valid and not expired
aws acm describe-certificate --certificate-arn <cert-arn> \
  --query 'Certificate.{Status:Status,Expiry:NotAfter,Domain:DomainName}'
# Status must be "ISSUED", Expiry must be in the future
```

If certificate status is `PENDING_VALIDATION`: the DNS validation record was never added to Route53.

**Layer 6: Is there a WAF or CloudFront blocking the request?**

If you're behind CloudFront + WAF:

```bash
# Check CloudFront distribution — is the origin correct?
aws cloudfront get-distribution --id <distribution-id> \
  --query 'Distribution.DistributionConfig.Origins'

# Check WAF — is a rule blocking legitimate traffic?
# WAF → Web ACLs → Sampled Requests → look for blocked requests with rule reason
```

**The 5-minute diagnostic sequence:**

```bash
# 1. DNS resolves?
dig api.myapp.com +short

# 2. Port open?
nc -zv api.myapp.com 443

# 3. HTTP response?
curl -v https://api.myapp.com/health

# 4. ALB targets healthy?
aws elbv2 describe-target-health --target-group-arn <arn>

# 5. ALB SG allows inbound 443?
aws ec2 describe-security-groups --group-ids <alb-sg-id>

# 6. Target SG allows traffic from ALB SG?
aws ec2 describe-security-groups --group-ids <target-sg-id>
```

**Real scenario:** A new service was deployed and the browser showed "This site can't be reached." DNS resolved correctly — dig returned the ALB CNAME. curl to the ALB returned a connection timeout. The ALB listener was configured (port 443). The issue: the engineer had created a new target group and attached it to the ALB, but the target group used port 8080. The pod was listening on port 3000 (the Dockerfile's EXPOSE was 3000, but the Helm chart defaulted to 8080). The ALB was routing to port 8080 → connection refused from pods on port 3000 → ALB returned 502. Fix: updated the Helm chart's `service.targetPort` to 3000. Service was up in 2 minutes.

---

## 4. How would you migrate a running EC2 instance to another region with minimal downtime?

EC2 instances are AZ and region-locked — you can't move a running instance. Migration is a create-in-destination, cutover, retire-source sequence. The goal is to minimise the cutover window to minutes, not hours.

**Step 1 — Create an AMI (golden image) from the running instance.**

```bash
# Create an AMI — this snapshots the root EBS volume while the instance runs
aws ec2 create-image \
  --instance-id i-0abc1234def56789 \
  --name "myapp-migration-$(date +%Y%m%d)" \
  --description "Pre-migration snapshot for region move" \
  --no-reboot    # Take image without stopping the instance (may have slight inconsistency)
  # Omit --no-reboot for a fully consistent image (instance reboots briefly)

# Note the returned AMI ID
# ami-0a1b2c3d4e5f67890
```

`--no-reboot` means the instance keeps serving traffic during the snapshot. For databases or stateful apps, consider stopping writes briefly and using a reboot for full consistency.

**Step 2 — Copy the AMI to the destination region.**

AMIs are region-specific. Use `copy-image` to replicate it:

```bash
# Copy the AMI from ap-south-1 to ap-southeast-1
aws ec2 copy-image \
  --source-region ap-south-1 \
  --source-image-id ami-0a1b2c3d4e5f67890 \
  --region ap-southeast-1 \
  --name "myapp-migration-ap-southeast-1" \
  --encrypted    # Encrypt in the destination region using the default KMS key

# This copies the AMI and all its snapshots — can take 15–60 minutes for large volumes
# Check status:
aws ec2 describe-images --region ap-southeast-1 \
  --image-ids ami-<new-id> \
  --query 'Images[*].State'
# Wait for: "available"
```

**Step 3 — Launch a new instance in the destination region from the copied AMI.**

```bash
# Launch in ap-southeast-1 using the same instance type and config as source
aws ec2 run-instances \
  --region ap-southeast-1 \
  --image-id ami-<new-copied-ami> \
  --instance-type m5.large \
  --subnet-id subnet-<dest-private-subnet> \
  --security-group-ids sg-<dest-sg> \
  --iam-instance-profile Name=myapp-role \
  --key-name my-keypair \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=myapp-prod-new}]'
```

**Step 4 — Validate the new instance before switching traffic.**

```bash
# SSH into the new instance and verify the app is running correctly
ssh -i my-keypair.pem ec2-user@<new-instance-private-ip>
systemctl status myapp
curl http://localhost:8080/health    # Internal health check

# Run smoke tests against the new instance directly (bypassing DNS)
curl -H "Host: api.myapp.com" http://<new-instance-ip>/api/status
```

Do not switch traffic until you're confident the new instance is healthy. This validation can run in parallel while the source instance continues serving.

**Step 5 — Cutover: switch traffic to the new region.**

**If using Route53:**

```bash
# Update the A record to point at the new region's IP or ALB
aws route53 change-resource-record-sets \
  --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "<new-region-alb-ip>"}]
      }
    }]
  }'
```

Pre-lower the TTL before migration day (set to 60 seconds a day in advance). When TTL was 300+ seconds, DNS propagation takes 5 minutes after cutover. With TTL=60, propagation happens in ~1 minute.

**Step 6 — Monitor the new instance post-cutover.**

```bash
# Watch error rate in the new region for 15 minutes
# Watch the old instance — is traffic dropping to zero?
watch -n 10 "aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkIn \
  --dimensions Name=InstanceId,Value=i-0abc1234def56789 \
  --period 60 --statistics Sum \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S)"
```

**Step 7 — Keep the old instance for a rollback window.**

Don't terminate the old instance immediately. Keep it running (stopped to save cost) for 24–48 hours. If the new instance has issues, you can route traffic back instantly.

```bash
# Stop the old instance (saves 95% of cost vs running, keeps EBS)
aws ec2 stop-instances --instance-ids i-0abc1234def56789

# Terminate after validation period
aws ec2 terminate-instances --instance-ids i-0abc1234def56789
```

**For data-bearing instances (RDS is in a different class):**

If the EC2 instance runs a database directly (not managed RDS), you need to sync data during the migration window:

```bash
# Replicate data using rsync between old and new instances (continuous sync during migration)
rsync -avz --delete \
  -e "ssh -i my-keypair.pem" \
  /var/lib/mysql/ \
  ec2-user@<new-instance-ip>:/var/lib/mysql/

# Final rsync at cutover (catches the delta written during the initial sync)
# Then stop writes on old → final rsync → start new → switch DNS
```

**Real scenario:** We migrated a stateless Node.js app from `us-east-1` to `ap-south-1` to reduce latency for Indian users. AMI creation: 4 minutes. AMI copy to `ap-south-1`: 22 minutes. New instance launch and app start: 6 minutes. Validation: 10 minutes. Cutover (DNS change with pre-lowered TTL=60): 90 seconds visible to users. Old instance stopped after 24 hours of monitoring. Total user-visible downtime: under 2 minutes.

---

## 5. Your S3 bucket storing critical backups becomes unavailable — how would you recover?

"Unavailable" in S3 has different meanings — S3 itself is rarely down (99.999999999% durability, 99.99% availability), so "unavailable" usually means: objects were accidentally deleted, the bucket was deleted, access was revoked, or data was corrupted.

**Step 1 — Diagnose the exact failure mode.**

```bash
# Can you list the bucket at all?
aws s3 ls s3://prod-backups/

# Error types and what they mean:
# "NoSuchBucket"       → bucket was deleted (most serious)
# "AccessDenied"       → IAM permissions or bucket policy blocking access
# "NoSuchKey"          → specific object was deleted
# "SlowDown"           → request rate throttled (S3 prefix hot spot)
```

**If "AccessDenied" — check the bucket policy and IAM role:**

```bash
# What does the bucket policy say?
aws s3api get-bucket-policy --bucket prod-backups

# What are my effective permissions?
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789:role/backup-reader \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::prod-backups/*

# Common cause: bucket policy was updated to deny all, or IAM role was rotated
```

**Step 2 — Recover deleted objects using S3 versioning.**

If versioning was enabled and objects were deleted (not the bucket):

```bash
# List all delete markers for a prefix
aws s3api list-object-versions \
  --bucket prod-backups \
  --prefix db-backups/ \
  --query 'DeleteMarkers[*].[Key,VersionId]' \
  --output text

# Remove delete markers to restore the objects
# (automating for many objects)
aws s3api list-object-versions \
  --bucket prod-backups \
  --prefix db-backups/ \
  --query 'DeleteMarkers[*].[Key,VersionId]' \
  --output text | while read key version; do
    aws s3api delete-object \
      --bucket prod-backups \
      --key "$key" \
      --version-id "$version"
  done
# After removing delete markers, objects reappear as current versions
```

**Step 3 — If the bucket itself was deleted.**

S3 bucket deletion is a two-step process: empty the bucket, then delete it. A deleted bucket can be recreated (if the name is available), but objects are gone unless you have cross-region replication or another backup.

```bash
# Recreate the bucket (if name still available)
aws s3 mb s3://prod-backups --region ap-south-1

# Restore from cross-region replication (if configured)
aws s3 sync s3://prod-backups-replica/ s3://prod-backups/
```

**Step 4 — Restore from cross-region replication or S3 Glacier.**

If you had cross-region replication configured before the incident:

```bash
# Replicated bucket in another region — copy back
aws s3 sync s3://prod-backups-dr-us-east-1/ s3://prod-backups/

# If objects are in Glacier (archived tier), initiate a restore first
aws s3api restore-object \
  --bucket prod-backups-dr-us-east-1 \
  --key db-backups/2024-01-14/prod-postgres.dump.gz \
  --restore-request Days=7,GlacierJobParameters=\{Tier=Standard\}
# Standard restore: 3–5 hours. Expedited: 1–5 minutes (higher cost).
```

**Step 5 — Check CloudTrail to understand what happened.**

```bash
# Who deleted the bucket or objects?
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=prod-backups \
  --start-time "2024-01-15T00:00:00Z" \
  --end-time "2024-01-15T23:59:59Z" \
  | jq '.Events[] | {time: .EventTime, name: .EventName, user: .Username}'

# Look for: DeleteBucket, DeleteObject, PutBucketPolicy (access change)
```

This identifies whether it was human error, a script, or a malicious actor.

**Prevention — what to add after this incident:**

```hcl
# 1. S3 versioning — never lose an object to accidental deletion
resource "aws_s3_bucket_versioning" "backups" {
  bucket = aws_s3_bucket.backups.id
  versioning_configuration { status = "Enabled" }
}

# 2. Object Lock COMPLIANCE mode — cannot be deleted for 30 days (regulatory hold)
resource "aws_s3_bucket_object_lock_configuration" "backups" {
  bucket = aws_s3_bucket.backups.id
  rule {
    default_retention {
      mode = "COMPLIANCE"
      days = 30
    }
  }
}

# 3. Cross-region replication — survive a region-level issue
resource "aws_s3_bucket_replication_configuration" "backups" {
  bucket = aws_s3_bucket.backups.id
  role   = aws_iam_role.replication.arn
  rule {
    id     = "replicate-all"
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.backups_dr.arn
      storage_class = "STANDARD_IA"
    }
  }
}

# 4. Block public access — bucket cannot accidentally become public
resource "aws_s3_bucket_public_access_block" "backups" {
  bucket                  = aws_s3_bucket.backups.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

**Real scenario:** A developer ran `aws s3 rm s3://prod-backups/ --recursive` thinking it was a test bucket — the bucket name was similar to a test bucket used earlier that day. 2,400 backup objects deleted. We had versioning enabled with no Object Lock. Recovery: listed all delete markers, removed them in a bulk loop script. All 2,400 objects restored in 8 minutes. Post-incident: added Object Lock COMPLIANCE mode with 30-day retention and cross-region replication. The next deletion attempt failed with: "Object is WORM protected and cannot be deleted."
