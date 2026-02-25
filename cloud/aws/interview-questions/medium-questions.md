# AWS — Medium Questions

---

## 1. How do AWS IAM policies work, and can you write a sample IAM JSON policy?

IAM is the most tested AWS topic in DevOps interviews because misconfigured IAM is the #1 cause of AWS security incidents. The interviewer is checking whether you understand the model, not just whether you can click in the console.

**The mental model:**

IAM is built on four concepts:
- **Principal** — who is making the request (IAM user, IAM role, AWS service)
- **Action** — what they want to do (`s3:GetObject`, `ec2:DescribeInstances`)
- **Resource** — which specific AWS resource (a specific S3 bucket, all EC2 instances, etc.)
- **Condition** — optional constraints (only from a specific IP, only with MFA, only in a specific region)

**Evaluation logic:** By default, everything is **denied**. A request is allowed only if there's an explicit `Allow` that covers it AND no explicit `Deny` overrides it. Explicit Deny always wins.

---

**Sample 1: S3 read-only for a specific bucket (least privilege)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowListBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::my-prod-data-bucket"
    },
    {
      "Sid": "AllowReadObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::my-prod-data-bucket/*"
    }
  ]
}
```

Notice the two separate ARNs: `s3:::bucket-name` for bucket-level actions (ListBucket), and `s3:::bucket-name/*` for object-level actions (GetObject). This is the most common IAM mistake — people put `s3:ListBucket` on `/*` and it silently fails because ListBucket applies to the bucket, not the objects.

---

**Sample 2: EC2 developer policy with MFA condition**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2ReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:List*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowEC2StartStopWithMFA",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "arn:aws:ec2:ap-south-1:123456789012:instance/*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    },
    {
      "Sid": "DenyTerminate",
      "Effect": "Deny",
      "Action": "ec2:TerminateInstances",
      "Resource": "*"
    }
  ]
}
```

The `DenyTerminate` statement uses an explicit Deny — developers can start/stop but can never terminate instances, regardless of any other policy that might grant it. Explicit Deny overrides everything.

---

**Sample 3: IRSA policy for a pod to read from Secrets Manager**

This is what we actually write in production for Kubernetes pods using IAM Roles for Service Accounts (IRSA):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowGetSecret",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:prod/my-app/*"
    }
  ]
}
```

The trust policy for this role (allows the K8s service account to assume it):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/ABCDEF123456"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/ABCDEF123456:sub": "system:serviceaccount:prod:my-app-sa"
        }
      }
    }
  ]
}
```

The `StringEquals` condition on the service account name is critical. Without it, any service account in the cluster can assume this role. We had a security finding where a role's trust policy used `StringLike` with a wildcard — any pod in any namespace could assume it.

---

**Real production mistake we made:**

We gave a CI/CD service account `iam:*` permissions so it could create roles for new services. An auditor flagged it immediately — `iam:*` includes `iam:CreateUser`, `iam:AttachUserPolicy`, `iam:CreateAccessKey`. With those permissions, a compromised CI system could create an admin user and exfiltrate credentials. We scoped it down to exactly the IAM actions the pipeline needed: `iam:CreateRole`, `iam:AttachRolePolicy`, `iam:PassRole` — and only on roles with a specific naming pattern using a condition:

```json
{
  "Condition": {
    "StringLike": {
      "iam:RoleArn": "arn:aws:iam::123456789012:role/app-*"
    }
  }
}
```

**Key insight:** IAM policy writing is a skill. The principle of least privilege isn't just a concept — it's something you actively enforce by asking "what's the minimum set of actions on the minimum set of resources this principal actually needs?" Start with nothing and add only what's required.

---

## 2. Explain VPC concepts in AWS.

> **Also asked as:** "Tell me about the VPC structure setup in your project."

VPC (Virtual Private Cloud) is your isolated network inside AWS. Every EC2 instance, RDS database, Lambda in a VPC, and EKS node lives inside one. Understanding VPC is non-negotiable for any DevOps role.

**The core building blocks:**

**VPC** — A logically isolated network. You define the CIDR block (e.g., `10.0.0.0/16` = 65,536 IPs). Think of it as your private data center inside AWS.

**Subnets** — Subdivisions of the VPC CIDR, tied to a specific Availability Zone. You split your VPC into public and private subnets:

```
VPC: 10.0.0.0/16

Public subnets (internet-facing):
  10.0.1.0/24  → ap-south-1a
  10.0.2.0/24  → ap-south-1b
  10.0.3.0/24  → ap-south-1c

Private subnets (no direct internet):
  10.0.11.0/24 → ap-south-1a
  10.0.12.0/24 → ap-south-1b
  10.0.13.0/24 → ap-south-1c
```

**Internet Gateway (IGW)** — Attaches to the VPC. Allows resources in public subnets to reach the internet and be reached from the internet. Without an IGW, nothing in the VPC can talk to the internet.

**NAT Gateway** — Lives in a public subnet. Allows resources in private subnets to make outbound internet requests (pull Docker images, call external APIs) without being directly reachable from the internet. It's like a one-way door for private resources.

```
Private subnet resource → NAT Gateway (in public subnet) → Internet Gateway → Internet
Internet                → Cannot initiate connection to private subnet resource
```

**Route Tables** — Control where traffic goes. Public subnets have a route: `0.0.0.0/0 → igw-xxx`. Private subnets have: `0.0.0.0/0 → nat-gateway-xxx`.

**Security Groups** — Stateful, instance-level firewall. You define which ports and IP ranges can reach an instance. "Stateful" means if you allow inbound on port 443, the response traffic is automatically allowed outbound — you don't need an explicit outbound rule.

**NACLs (Network ACLs)** — Stateless, subnet-level firewall. You explicitly allow both inbound AND outbound. Applied before security groups. Most teams use Security Groups for most things and only touch NACLs for subnet-level deny rules.

**VPC Peering** — Private network connection between two VPCs. Traffic stays on AWS backbone, doesn't go over the internet. We use this to connect our app VPC to our shared-services VPC (where central logging and monitoring live).

**VPC Endpoints** — Allow private resources to access AWS services (S3, DynamoDB, ECR, Secrets Manager) without going through the internet or a NAT Gateway. Two types:
- Gateway endpoint (free) — for S3 and DynamoDB
- Interface endpoint (costs money) — for everything else

We added a VPC endpoint for ECR and Secrets Manager because our EKS nodes in private subnets were pulling Docker images through NAT Gateway. Each NAT Gateway charges per GB of data processed. ECR endpoints eliminated that cost — image pulls stay on the private network.

**What our production VPC looks like:**

```
VPC: 10.0.0.0/16
├── Public subnets (ALB, NAT Gateway, Bastion)
│   ├── 10.0.1.0/24 (ap-south-1a)
│   ├── 10.0.2.0/24 (ap-south-1b)
│   └── 10.0.3.0/24 (ap-south-1c)
└── Private subnets (EKS nodes, RDS, ElastiCache)
    ├── 10.0.11.0/24 (ap-south-1a)
    ├── 10.0.12.0/24 (ap-south-1b)
    └── 10.0.13.0/24 (ap-south-1c)
```

EKS nodes live in private subnets. Users reach services via ALB in the public subnet. Nodes reach the internet for image pulls via NAT Gateway. RDS is in private subnets with no route to the internet at all — accessible only from within the VPC.

**Real incident caused by VPC misconfiguration:**

We launched a new EKS node group in private subnets, but the subnets didn't have the tag `kubernetes.io/role/internal-elb: 1`. EKS uses this tag to know which subnets to use for internal load balancers. When we created an internal ALB, it had no subnets to deploy into and silently failed. New services couldn't get endpoints. 45 minutes of debugging before we found the missing tag in AWS docs.

---

## 3. What load balancer have you used in AWS, and why?

AWS has three types — **ALB, NLB, CLB** — and the choice matters. The interviewer wants to know if you understand the trade-offs, not just that you've clicked "Create Load Balancer."

**ALB (Application Load Balancer) — what we use for 90% of services.**

Operates at Layer 7 (HTTP/HTTPS). Can route based on:
- Path: `/api/*` → service-a, `/admin/*` → service-b
- Hostname: `api.myapp.com` → service-a, `admin.myapp.com` → service-b
- HTTP headers, query parameters, request methods

We use ALB for all our microservices exposed via EKS Ingress. One ALB serves all services — traffic is routed by path and hostname rules. This is cost-effective (one ALB instead of one per service) and manageable via the AWS Load Balancer Controller (Kubernetes ingress annotations control ALB rules).

```yaml
# Kubernetes Ingress that creates ALB rules automatically
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - host: api.myapp.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 80
```

**NLB (Network Load Balancer) — used for specific cases.**

Operates at Layer 4 (TCP/UDP). No HTTP awareness, no path routing. Routes based on IP and port only.

**When we use NLB:**
1. **Non-HTTP traffic** — we have a service that accepts raw TCP connections for a legacy protocol. ALB only understands HTTP/HTTPS.
2. **Ultra-low latency** — NLB adds ~microseconds of latency vs ALB's ~milliseconds. For high-frequency trading or real-time gaming, that matters.
3. **Static IP requirement** — NLB gives you a fixed Elastic IP per AZ. ALB IPs change. Some customers' firewalls whitelist by IP, so we use NLB in front of those services.
4. **Preserve source IP** — NLB passes the original client IP to the backend. ALB can do this via `X-Forwarded-For` header, but some apps need the actual IP in the TCP connection.

**CLB (Classic Load Balancer) — we don't use it.**

The original AWS LB, now legacy. No path-based routing, limited features. Any CLB we find in our account is from before my time — we migrate them to ALB/NLB as we encounter them.

**Real decision we made:** Our payment service initially used NLB because the previous team wanted "maximum performance." We measured latency and found the difference between ALB and NLB was 0.3ms — meaningless for our use case. We migrated to ALB because we needed path-based routing and WAF integration (WAF only works with ALB). The migration cut our load balancer count from 4 to 1 (shared ALB with path routing), saving $180/month.

**Key insight:** Don't choose NLB for "better performance" by default. If you need Layer 7 features (SSL termination, path routing, WAF, authentication), you need ALB. NLB is for specific use cases where Layer 4 is genuinely required.

> **Also asked as:** "What is the difference between an L4 and L7 load balancer?" — L4 (NLB) routes by IP/TCP port only; L7 (ALB) reads HTTP headers, URLs, and cookies to make routing decisions. See `networking/basic-questions.md` for the pure networking concept.

---

## 4. How do you connect and manage services such as databases, EC2, EKS, and ECS?

"Connect and manage" means: authentication, networking, and operational access — not just "I use the AWS console."

**Databases (RDS):**

RDS instances live in private subnets — no public endpoint. Connection from application pods:

- **Networking:** Security group on RDS allows inbound port 5432 (Postgres) or 3306 (MySQL) only from the EKS node security group or pod CIDR. Not from 0.0.0.0/0.
- **Credentials:** We use AWS Secrets Manager. The app reads the DB password at startup via the AWS SDK — never from environment variables baked into the image.
- **From a developer's machine:** Through a bastion host or AWS SSM Session Manager port forwarding — no RDS public endpoint, no SSH keys needed:

```bash
# SSM port forward to RDS — no bastion, no open SSH port
aws ssm start-session \
  --target i-0abc123def456 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["prod-db.abc123.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["5432"]}'

# Now connect as if RDS is local
psql -h localhost -p 5432 -U app_user -d mydb
```

**EC2:**

We don't SSH to EC2 instances. All access is via AWS SSM Session Manager — no bastion, no open port 22, no SSH keys to manage or rotate:

```bash
aws ssm start-session --target i-0abc123def456
```

For bulk operations (run a command on 20 instances): SSM Run Command or SSM State Manager. For patching: AWS Systems Manager Patch Manager.

**EKS:**

```bash
# Authenticate kubectl via IAM role — no static kubeconfig stored on disk
aws eks update-kubeconfig --region ap-south-1 --name prod-cluster --role-arn arn:aws:iam::123456789:role/eks-admin-role
```

Engineers authenticate via AWS SSO. Their SSO role maps to an EKS RBAC role via the `aws-auth` ConfigMap. Developers get read-only access to prod. CI/CD gets scoped write access via a service account IAM role (IRSA).

**ECS:**

ECS tasks connect to databases the same way — Secrets Manager for credentials, security groups for network access. For managing tasks operationally: ECS Exec.

---

## 5. What is the command to connect to a running ECS container?

The answer most people give is "ECS Exec." The full answer explains how it works, how to enable it, and the real command.

**ECS Exec** — lets you run commands inside a running ECS task the same way `kubectl exec` does for Kubernetes pods. It uses SSM Session Manager under the hood, so no SSH, no open ports, no bastion.

**Enable ECS Exec on the task definition:**

```json
{
  "containerDefinitions": [{
    "name": "my-app",
    "image": "...",
    "linuxParameters": {
      "initProcessEnabled": true    // Required for ECS Exec — init process handles signal forwarding
    }
  }],
  "taskRoleArn": "arn:aws:iam::123456789:role/ecs-task-role"
}
```

The task IAM role needs SSM permissions:
```json
{
  "Effect": "Allow",
  "Action": [
    "ssmmessages:CreateControlChannel",
    "ssmmessages:CreateDataChannel",
    "ssmmessages:OpenControlChannel",
    "ssmmessages:OpenDataChannel"
  ],
  "Resource": "*"
}
```

**Enable exec on the ECS service:**

```bash
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --enable-execute-command
```

**The actual command to connect:**

```bash
# Get the task ID first
aws ecs list-tasks --cluster my-cluster --service-name my-service

# Connect to the running container
aws ecs execute-command \
  --cluster my-cluster \
  --task arn:aws:ecs:ap-south-1:123456789:task/my-cluster/abc123def456 \
  --container my-app \
  --interactive \
  --command "/bin/bash"
```

This drops you into a shell inside the running container — exactly like `kubectl exec -it <pod> -- /bin/bash`.

**Real scenario:** We had an ECS task that was failing health checks intermittently. Logs showed the app starting correctly. We used ECS Exec to get inside the container and ran `curl localhost:8080/health` manually — it returned a redirect (301) instead of 200. The health check was following redirects, but the ALB health check wasn't. Fix: changed the health check path to `/health/` (with trailing slash) which the app handled without redirect. We never would have found this from logs alone — ECS Exec was essential.

---

## 6. How do you create and deploy AWS Lambda functions?

Lambda is different from containers — the deployment artifact is a ZIP (or container image), and "deployment" means updating the function code, not managing instances.

**Creating a Lambda with Terraform (what we use for all new functions):**

```hcl
# The function code packaged as a ZIP
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/src"
  output_path = "${path.module}/dist/function.zip"
}

resource "aws_lambda_function" "process_events" {
  function_name    = "process-events-${var.environment}"
  role             = aws_iam_role.lambda_exec.arn
  handler          = "handler.main"
  runtime          = "python3.12"
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  environment {
    variables = {
      DB_SECRET_ARN = aws_secretsmanager_secret.db.arn
      ENVIRONMENT   = var.environment
    }
  }

  timeout     = 30
  memory_size = 256

  vpc_config {
    subnet_ids         = var.private_subnet_ids
    security_group_ids = [aws_security_group.lambda.id]
  }
}
```

`source_code_hash` is critical — Terraform uses it to detect when the ZIP has changed and trigger a redeployment. Without it, Terraform won't redeploy even if your code changed.

**What we actually deploy (CI/CD flow for Lambda):**

```groovy
// Jenkins pipeline for Lambda
stage('Test') {
  sh 'pytest tests/'
}
stage('Package') {
  sh '''
    pip install -r requirements.txt -t dist/
    cp -r src/ dist/
    cd dist && zip -r ../function.zip .
  '''
}
stage('Deploy') {
  sh '''
    aws s3 cp function.zip s3://my-artifacts/lambda/process-events/$GIT_COMMIT.zip
    aws lambda update-function-code \
      --function-name process-events-prod \
      --s3-bucket my-artifacts \
      --s3-key lambda/process-events/$GIT_COMMIT.zip
  '''
}
stage('Publish version') {
  sh '''
    aws lambda publish-version \
      --function-name process-events-prod \
      --description "Deploy $GIT_COMMIT"
  '''
}
```

**Lambda container images (for larger functions):**

For functions with large dependencies (ML models, heavy libraries), we use container images instead of ZIP — same Docker workflow as our EKS services, up to 10GB image size.

```bash
docker build -t my-lambda .
docker tag my-lambda $ECR_REPO:$GIT_COMMIT
docker push $ECR_REPO:$GIT_COMMIT

aws lambda update-function-code \
  --function-name process-events-prod \
  --image-uri $ECR_REPO:$GIT_COMMIT
```

---

## 7. What are the different ways to push artifacts to AWS Lambda?

This question is checking whether you know the trade-offs, not just that Lambda can be updated.

**Method 1: Direct upload via AWS CLI (small functions, < 50MB)**

```bash
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip
```

Limit: 50MB compressed. For anything larger, this fails with a size error.

**Method 2: S3 artifact (most common in CI/CD)**

```bash
aws s3 cp function.zip s3://my-artifacts/lambda/my-function/$GIT_COMMIT.zip

aws lambda update-function-code \
  --function-name my-function \
  --s3-bucket my-artifacts \
  --s3-key lambda/my-function/$GIT_COMMIT.zip
```

Limit: 250MB uncompressed. Artifacts are versioned in S3 by commit SHA — rollback is `update-function-code` with the previous S3 key.

**Method 3: Container image from ECR (largest functions, ML workloads)**

```bash
aws lambda update-function-code \
  --function-name my-function \
  --image-uri 123456789.dkr.ecr.ap-south-1.amazonaws.com/my-lambda:$GIT_COMMIT
```

Limit: 10GB. Same ECR workflow as regular containers — build, scan with Trivy, push, deploy.

**Method 4: Terraform (infrastructure as code — what we prefer)**

Terraform manages the function resource including the code. `source_code_hash` detects changes.

```hcl
resource "aws_lambda_function" "my_function" {
  filename         = "function.zip"
  source_code_hash = filebase64sha256("function.zip")
  ...
}
```

`terraform apply` updates the function if the ZIP changed. All infra changes (memory, timeout, env vars) and code changes go through the same `terraform apply` — one source of truth.

**Method 5: AWS SAM / Serverless Framework**

Higher-level tools that abstract over CloudFormation or Terraform. SAM is AWS-native and integrates with CodePipeline. We don't use it — we already have Jenkins + Terraform and don't want another deployment abstraction layer.

**What we use in practice:**

New Lambda functions: Terraform manages the resource, CI pipeline updates the code via S3 + `update-function-code`. For large ML inference functions: container image via ECR. All functions use Lambda versioning + aliases — `prod` alias always points to the latest stable version, and we can point it to a previous version for instant rollback without redeployment.

---

## 9. How do you reduce or manage cloud costs effectively?

Cloud costs grow silently. Services get provisioned, traffic patterns change, and the bill keeps climbing unless you actively manage it. Here's how I approach cost reduction systematically.

**Start with visibility — you can't cut what you can't see.**

```bash
# AWS Cost Explorer — spot which services and accounts are driving spend
# Enable cost allocation tags: every resource gets tagged with Team, Service, Environment
# Tag policy example (Terraform):
resource "aws_resourcegroups_group" "cost_center" { ... }
```

We enforce tagging via AWS Config rules — any resource without mandatory tags (`Team`, `Service`, `Environment`) generates a compliance finding. Untagged spend is the enemy of cost control.

**Right-size compute first — the biggest lever.**

```bash
# AWS Compute Optimizer recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --account-ids 123456789012 \
  --output json | jq '.instanceRecommendations[] | {instance: .instanceName, current: .currentInstanceType, recommended: .recommendationOptions[0].instanceType}'
```

We found instances consistently running at 12% CPU with 4GB RAM allocated and 400MB used. Compute Optimizer recommended moving from `m5.large` to `t3.small` — 70% cost reduction for those instances.

In Kubernetes — VPA (Vertical Pod Autoscaler) in recommendation mode:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
spec:
  updatePolicy:
    updateMode: "Off"    # Recommendation only — don't auto-apply
```

`kubectl describe vpa <name>` shows actual p95 CPU/memory vs what's requested. We had pods requesting 1 CPU and using 80m. Tuning requests based on VPA recommendations reduced our EKS node count by 30%.

**Use spot/preemptible instances for non-critical workloads.**

Spot instances are 60–90% cheaper than on-demand. Risk: they can be interrupted with 2-minute notice.

```hcl
# EKS node group with mixed on-demand + spot
resource "aws_eks_node_group" "workers" {
  capacity_type = "SPOT"
  instance_types = ["m5.large", "m5a.large", "m4.large"]  # Multiple types = fewer interruptions
}
```

What runs on spot: CI/CD build agents, dev/staging environments, batch processing jobs, non-critical background workers. What runs on on-demand: production web services, databases (RDS is already managed — use Reserved Instances instead).

**Reserved Instances and Savings Plans for predictable workloads.**

```
On-demand: $0.096/hr for m5.large
1-year No Upfront RI: $0.057/hr  → 40% savings
3-year No Upfront RI: $0.037/hr  → 61% savings
Compute Savings Plan: ~40% savings, flexible across instance families
```

We buy Compute Savings Plans (not instance-specific RIs) because they're flexible — if we move from `m5.large` to `m6i.large`, the Savings Plan still applies. RIs lock you to a specific instance type.

**Eliminate waste — orphaned resources are free money wasted.**

```bash
# Unattached EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,CreatedOn:CreateTime}'

# Unused Elastic IPs (charged when not attached)
aws ec2 describe-addresses \
  --query 'Addresses[?InstanceId==null].PublicIp'

# Old snapshots (automated but manual ones accumulate)
aws ec2 describe-snapshots --owner-ids self \
  --query 'Snapshots[?StartTime<`2024-01-01`].SnapshotId'

# Idle load balancers (no healthy targets for 30+ days)
# Check via CloudWatch: RequestCount = 0 for 30 days
```

We run a weekly Lambda that reports orphaned resources to Slack. In the first month, we found 47 unattached EBS volumes (some from instances terminated 8 months ago), 12 unused Elastic IPs, and 3 load balancers with no targets. Total cleanup: $340/month.

**NAT Gateway — often the hidden cost driver.**

NAT Gateway charges $0.045/GB of processed data. If your EKS nodes in private subnets pull Docker images through NAT Gateway every build, costs add up fast.

```
Fix: VPC Endpoints for ECR and S3
→ Image pulls go through private network, not NAT Gateway
→ Cost: $7.30/month per endpoint
→ Savings: $800/month in NAT data processing fees (at our image pull volume)
```

**Real outcome:** We ran a 3-week cost optimization sprint:
- Right-sized EC2/pods using Compute Optimizer + VPA: -28%
- Moved CI build agents to spot: -$400/month
- Purchased Compute Savings Plans for baseline load: -35%
- VPC endpoints for ECR/S3/Secrets Manager: -$800/month
- Cleaned up orphaned resources: -$340/month
- Total reduction: 42% lower AWS bill, from $18,000/month to $10,400/month

**Key principle:** Cost optimization is a continuous process, not a one-time event. We review Cost Explorer weekly, have Compute Optimizer recommendations as a standing agenda item in sprint planning, and budget alerts trigger when spend exceeds forecast by more than 10%.

---

## 8. NLB vs ALB in a microservices architecture — which do you choose and when?

In a microservices architecture, this is not a binary choice. We use both, for different services, for specific reasons.

**The fundamental difference:**

- **ALB (Layer 7)** — understands HTTP/HTTPS. Can route based on URL path, hostname, headers, query params, and HTTP method. Supports WebSocket, HTTP/2, WAF, and authentication (Cognito/OIDC)
- **NLB (Layer 4)** — understands TCP/UDP/TLS. Routes based on IP and port only. No HTTP awareness. Extremely low latency, preserves source IP, supports static Elastic IPs

**In microservices, ALB covers ~90% of use cases:**

Most microservices communicate over HTTP/gRPC. ALB's path-based and hostname-based routing lets you serve all your services from a single load balancer:

```yaml
# Single ALB, multiple services via path routing
Rules:
  /api/orders/*    → order-service target group
  /api/payments/*  → payment-service target group
  /api/users/*     → user-service target group
  /                → frontend target group
```

One ALB instead of one NLB per service cuts load balancer costs significantly. ALB integrates with WAF — one WAF ACL protects all services. ALB terminates TLS and handles certificates via ACM.

**When NLB is the right choice in microservices:**

**1. gRPC / HTTP/2 with long-lived streams.**
ALB supports gRPC, but NLB handles long-lived TCP streams more efficiently. If your microservices communicate via gRPC streaming (bi-directional streams, server-push), NLB passes TCP through without interference. ALB inspects and potentially resets long-lived HTTP connections.

**2. Non-HTTP protocols.**
A microservice exposing a raw TCP protocol (message broker, custom binary protocol, database proxy). ALB doesn't understand non-HTTP protocols. NLB just forwards TCP packets.

**3. Static IP requirement for firewall whitelisting.**
NLB gives you one static Elastic IP per AZ. ALB IPs are dynamic and change. If a customer's firewall must whitelist your IP, you need NLB. We had a financial services customer who required IP whitelisting for compliance — we put NLB in front of that service specifically.

**4. Extreme low latency (sub-millisecond matters).**
NLB operates at layer 4 with minimal processing overhead — microseconds vs ALB's milliseconds. For high-frequency trading, real-time gaming, or services where request latency is measured in microseconds, NLB wins. For typical API services where requests take 20–500ms, the latency difference is irrelevant.

**5. Preserving client source IP in the TCP connection itself.**
ALB adds `X-Forwarded-For` headers (HTTP layer). If your service needs the real client IP in the TCP socket (not in a header) — for IP-based rate limiting in legacy apps that don't read headers — NLB passes it through via Proxy Protocol.

**Our microservices architecture decision:**

```
Internet
    ↓
CloudFront (CDN + DDoS protection)
    ↓
WAF (SQL injection, rate limiting)
    ↓
ALB (HTTP/HTTPS, path routing, TLS termination)
    ↓
EKS pods (order, payment, user, notification services)

Exceptions:
- Internal gRPC service mesh → NLB (long-lived streams)
- Legacy TCP-based reporting tool → NLB (non-HTTP)
- Customer-facing API for IP-whitelisting customer → NLB (static IP)
```

**Real cost comparison:** Before consolidating, we had 6 ALBs (one per service) and 1 NLB. After: 1 ALB for HTTP services (path routing) + 1 NLB for TCP services. Saved $180/month in load balancer hourly charges. More importantly, WAF was only on 1 ALB instead of 6 — one rule update protects all services.

> **Also asked as:** "When do you use ALB vs NLB?" — covered above (ALB for HTTP/HTTPS with L7 routing and WAF; NLB for TCP/UDP, static IP, ultra-low latency, or non-HTTP protocols).

---

## 10. What is Autoscaling ? How you used in your previous projects without UI based ?

> **Also asked as:** "What is AWS Auto Scaling Groups and what are its use cases?"
> **Also asked as:** "What is Autoscaling ? How you used in your previous projects without UI based ?"

An Auto Scaling Group (ASG) is AWS's mechanism for automatically adjusting the number of EC2 instances in a group based on demand, schedules, or metrics. The ASG ensures you always have the right number of instances — not too few (degraded availability) and not too many (wasted cost).

**How to use it "Without UI based" (Infrastructure as Code)**
In modern DevOps, we never configure Autoscaling in the AWS Management Console (the UI). Clicking through the console is error-prone, untrackable, and non-repeatable.

Instead, we define the entire Autoscaling configuration using Terraform. 

**The Terraform Implementation:**
To create an ASG without the UI, you need two main components: a **Launch Template** (what to launch) and the **Auto Scaling Group** itself (how/when to launch).

```hcl
# 1. First, define WHAT to launch (The Launch Template)
resource "aws_launch_template" "app_server" {
  name_prefix   = "app-server-"
  image_id      = "ami-0abcdef1234567890" # Amazon Linux 2023
  instance_type = "t3.medium"
  
  # Connect the EC2 instances to the correct Security Group
  vpc_security_group_ids = ["sg-0123456789abcdef0"]

  # Give the EC2 instances an IAM role (e.g., to read from S3)
  iam_instance_profile {
    name = "app-server-role"
  }

  # User Data script that runs on startup (installs Docker and pulls the app)
  user_data = base64encode(<<-EOF
              #!/bin/bash
              yum update -y
              amazon-linux-extras install docker -y
              service docker start
              docker run -d -p 80:80 my-registry/my-app:latest
              EOF
  )
}

# 2. Second, define the Auto Scaling Group behavior
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  vpc_zone_identifier = ["subnet-abc", "subnet-def"] # Private subnets
  
  # The scaling bounds
  min_size         = 2  # Never drop below 2 instances (High Availability)
  desired_capacity = 2  # Start with 2
  max_size         = 10 # Never exceed 10 instances (Cost Control)

  # Attach it to an Application Load Balancer
  target_group_arns = ["arn:aws:elasticloadbalancing:region:account-id:targetgroup/my-tg/xyz"]

  launch_template {
    id      = aws_launch_template.app_server.id
    version = "$Latest"
  }
}

# 3. Third, define WHEN to scale (Target Tracking Policy)
resource "aws_autoscaling_policy" "cpu_tracking" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    # Tell AWS: "Maintain the average CPU across all instances at exactly 70%"
    target_value = 70.0
    # If average CPU hits 75%, AWS provisions a new instance
    # If average CPU drops to 30%, AWS terminates an instance
  }
}
```

By deploying this Terraform code from a CI/CD pipeline, the Auto Scaling Group is provisioned entirely programmatically ("without UI based"). If we need to change the instance type from `t3.medium` to `m5.large`, we simply update the Terraform code, raise a PR, and the pipeline rolls out the change automatically.

**The core model:**

```
ASG Configuration
├── Launch Template  — which AMI, instance type, IAM role, user-data, security group
├── VPC + Subnets    — which AZs to spread instances across
├── Min / Desired / Max size  — floor, current target, ceiling
└── Scaling Policies — what triggers adding or removing instances
```

The ASG continuously checks: are the running instances equal to the desired count? If an instance fails a health check, the ASG terminates it and launches a replacement automatically — this is self-healing.

**Scaling policies:**

**1. Target Tracking (simplest — 80% of use cases).**

Tell the ASG to maintain a target metric value. It figures out when to add/remove.

```hcl
resource "aws_autoscaling_policy" "cpu_tracking" {
  name                   = "cpu-target-tracking"
  autoscaling_group_name = aws_autoscaling_group.app.name
  policy_type            = "TargetTrackingScaling"

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }
    target_value = 70.0   # Keep average CPU at 70%
    # ASG adds instances when CPU > 70%, removes when CPU < 70%
  }
}
```

Other common targets: `ALBRequestCountPerTarget` (requests per instance), `ASGAverageNetworkIn`.

**2. Step Scaling (when you need tiered responses).**

Different scale amounts at different thresholds — add 2 instances at 70% CPU, add 5 at 90% CPU.

**3. Scheduled Scaling (predictable traffic patterns).**

```hcl
resource "aws_autoscaling_schedule" "morning_scale_up" {
  scheduled_action_name  = "morning-scale-up"
  autoscaling_group_name = aws_autoscaling_group.app.name
  min_size               = 4
  max_size               = 20
  desired_capacity       = 8
  recurrence             = "0 8 * * 1-5"   # 8 AM weekdays (IST)
}

resource "aws_autoscaling_schedule" "night_scale_down" {
  scheduled_action_name  = "night-scale-down"
  autoscaling_group_name = aws_autoscaling_group.app.name
  min_size               = 1
  max_size               = 20
  desired_capacity       = 2
  recurrence             = "0 20 * * 1-5"  # 8 PM weekdays — save cost overnight
}
```

**Use cases:**

**1. Stateless application tiers — the primary use case.**

Web servers, API servers, worker processes — anything that doesn't store state locally. Put them in an ASG behind an ALB. Traffic spikes: ASG scales out. Traffic drops: scales in. Instance fails: replaced automatically. This is the foundation of every resilient EC2 architecture.

**2. Cost optimization with mixed instances + spot.**

```hcl
resource "aws_autoscaling_group" "app" {
  mixed_instances_policy {
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.app.id
      }
      override {
        instance_type     = "m5.large"
        weighted_capacity = "1"
      }
      override {
        instance_type     = "m5a.large"    # AMD variant — often 10% cheaper
        weighted_capacity = "1"
      }
      override {
        instance_type     = "m6i.large"
        weighted_capacity = "1"
      }
    }
    instances_distribution {
      on_demand_base_capacity                  = 2    # Always keep 2 on-demand (baseline)
      on_demand_percentage_above_base_capacity = 20   # 20% on-demand, 80% spot above base
      spot_allocation_strategy                 = "price-capacity-optimized"
    }
  }
}
```

This runs your baseline on reliable on-demand instances and scales with cheap spot instances — typically 60–70% cheaper than on-demand for the spot portion.

**3. Self-healing infrastructure.**

ASG integrates with EC2 health checks and ALB health checks. If an instance starts failing health checks:
1. ASG marks it unhealthy
2. Terminates the failing instance
3. Launches a replacement in the same AZ
4. ALB drains connections before termination (connection draining / deregistration delay)

No manual intervention. Our on-call engineers stopped getting paged for "dead instance — please replace" alerts after we moved to ASGs.

**4. Multi-AZ high availability.**

```hcl
resource "aws_autoscaling_group" "app" {
  vpc_zone_identifier = [
    aws_subnet.private_a.id,  # ap-south-1a
    aws_subnet.private_b.id,  # ap-south-1b
    aws_subnet.private_c.id,  # ap-south-1c
  ]
  # ASG spreads instances across AZs
  # If one AZ has an outage, ASG launches replacements in other AZs
}
```

**ASG lifecycle hooks — for graceful operations:**

```hcl
resource "aws_autoscaling_lifecycle_hook" "terminating" {
  name                   = "drain-before-termination"
  autoscaling_group_name = aws_autoscaling_group.app.name
  lifecycle_transition   = "autoscaling:EC2_INSTANCE_TERMINATING"
  heartbeat_timeout      = 300   # 5 minutes to complete in-flight requests
  default_result         = "CONTINUE"
}
```

Before terminating an instance during scale-in, the lifecycle hook pauses and waits — giving the instance 5 minutes to finish processing any in-flight requests. Critical for worker processes that shouldn't be killed mid-job.

**Real scenario:** Our batch processing workers ran on a fixed fleet of 10 EC2 instances. During month-end reporting, the queue had 50,000 jobs and 10 workers took 8 hours. We moved to an ASG scaling on SQS queue depth: 0 messages → 1 instance (cost: ~$0.10/hour). Queue depth > 1,000 → scale to 20 instances. Month-end processing finished in 40 minutes instead of 8 hours. Cost for that burst: $6 instead of running 10 instances 24/7 at $48/day.

> **Also asked as:** "Explain a scenario where scheduled scaling is more appropriate than dynamic scaling." — covered above. Scheduled scaling fits predictable patterns: scale up at 08:00 before business hours, scale down at 20:00. Dynamic (target tracking) fits unpredictable load. Use scheduled when you know *when* load hits; use dynamic when you only know *if* it hits.

---

## 11. Why do we use RDS and how do you scale it?

RDS (Relational Database Service) is AWS's managed database offering. The question has two parts — "why" (when to choose RDS over alternatives) and "how to scale" (the actual operational work).

**Why RDS over self-managed databases on EC2:**

Running PostgreSQL on an EC2 instance means you own: installation, patching, backups, replication setup, failover handling, monitoring, storage management, SSL certificate rotation. That's a full-time job per database cluster.

RDS manages all of that:

| Concern | EC2 self-managed | RDS managed |
|---|---|---|
| Patches | You apply them | AWS applies them (with maintenance windows) |
| Backups | You set up cron + S3 | Automated daily backups, point-in-time recovery |
| Multi-AZ failover | You configure replication + Route53 | One checkbox — automatic failover < 60 seconds |
| Monitoring | You set up Prometheus exporters | CloudWatch metrics out of the box |
| Storage | You provision and resize volumes | Autoscaling storage (no manual resize) |
| Read replicas | You configure streaming replication | One API call, AWS manages lag monitoring |

The tradeoff: RDS costs ~20–40% more than equivalent EC2 compute. That premium buys engineering time. For a 3-person team running 5 databases, the math is obvious — the alternative is one engineer spending 30% of their time on database operations.

**When NOT to use RDS:**

- **Need unsupported extensions or custom compiled PostgreSQL** — RDS restricts superuser and some extensions
- **Extreme performance requirements** — self-managed on `i3en.metal` with NVMe may outperform RDS for specific workloads
- **Cost at massive scale** — very large RDS instances are expensive; at scale, Aurora Serverless or self-managed may win
- **Non-relational workloads** — use DynamoDB (key-value), ElastiCache (caching), OpenSearch (search) instead

**How to scale RDS:**

**Vertical scaling (scale up — bigger instance).**

```bash
# Modify the instance class — takes 5–10 minutes with a reboot
aws rds modify-db-instance \
  --db-instance-identifier prod-postgres \
  --db-instance-class db.r6g.2xlarge \   # Was db.r6g.xlarge
  --apply-immediately
```

In Terraform:
```hcl
resource "aws_db_instance" "prod" {
  instance_class = "db.r6g.2xlarge"   # Change here, terraform apply triggers modification
}
```

Vertical scaling handles CPU and memory constraints but has limits — the largest RDS instance is `db.r6g.16xlarge` (64 vCPU, 512 GB RAM). Beyond that: Aurora.

**Horizontal scaling for reads — read replicas.**

```hcl
resource "aws_db_instance" "prod_replica_1" {
  replicate_source_db = aws_db_instance.prod.identifier
  instance_class      = "db.r6g.xlarge"
  publicly_accessible = false

  tags = { Role = "read-replica" }
}
```

Application code routes read queries to replica endpoints, writes to primary:

```python
# Application reads from replica — reduces load on primary
db_read  = connect(host=os.getenv("DB_READ_ENDPOINT"))   # replica
db_write = connect(host=os.getenv("DB_WRITE_ENDPOINT"))  # primary

def get_orders(user_id):
    return db_read.execute("SELECT * FROM orders WHERE user_id = %s", user_id)

def create_order(data):
    return db_write.execute("INSERT INTO orders ...", data)
```

Read replicas have **replication lag** — reads might be slightly behind writes. Acceptable for report queries, not acceptable for "immediately after write" reads (use primary for those).

**Aurora for higher scale:**

Aurora is AWS's cloud-native MySQL/PostgreSQL-compatible database. For large-scale workloads, it replaces standard RDS:

- **Storage scales automatically** up to 128 TB (no manual intervention)
- **Up to 15 read replicas** vs RDS's 5, with sub-10ms replication lag
- **Aurora Serverless v2** — capacity auto-scales from 0.5 ACUs to 128 ACUs, billed per second. Zero-scale when idle (dev/test environments: near-zero cost when unused)

```hcl
resource "aws_rds_cluster" "prod" {
  cluster_identifier = "prod-aurora-postgres"
  engine             = "aurora-postgresql"
  engine_version     = "15.4"

  serverlessv2_scaling_configuration {
    min_capacity = 0.5
    max_capacity = 64
  }
}
```

**Storage scaling:**

```hcl
resource "aws_db_instance" "prod" {
  allocated_storage     = 100    # GB initial
  max_allocated_storage = 1000   # GB — autoscaling ceiling
  # RDS automatically expands when free storage < 10%
  # No downtime, no manual action required
}
```

**Real scenario:** Our prod PostgreSQL (db.r6g.xlarge) was hitting 90% CPU during peak hours — report queries were competing with application writes. Solution: promoted one read replica for reporting and pointed our BI tool's connection string at the replica endpoint. Primary CPU dropped to 45%. Cost: +$0.20/hour for the replica instance. Alternative was a vertical scale to `db.r6g.2xlarge` (+$0.40/hour) which wouldn't have helped because the CPU was split between reads and writes — only separation fixed it.

---

## 12. How does Lambda establish communication with S3, and what are the common patterns?

Lambda and S3 integrate in two directions: Lambda **triggered by** S3 events, and Lambda **reading from / writing to** S3 directly. Both patterns are common, and the interviewer often wants to see that you understand the IAM and network dimensions, not just the happy path.

**Direction 1: S3 triggers Lambda (event-driven).**

An S3 event notification calls Lambda automatically when objects are created, deleted, or modified.

```hcl
# 1. Lambda function
resource "aws_lambda_function" "image_processor" {
  function_name = "image-processor"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "index.handler"
  runtime       = "python3.12"
  filename      = "function.zip"

  environment {
    variables = {
      OUTPUT_BUCKET = aws_s3_bucket.processed.id
    }
  }
}

# 2. Allow S3 to invoke this Lambda
resource "aws_lambda_permission" "s3_invoke" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.image_processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.uploads.arn
}

# 3. S3 notification that triggers the Lambda
resource "aws_s3_bucket_notification" "upload_trigger" {
  bucket = aws_s3_bucket.uploads.id

  lambda_function {
    lambda_function_arn = aws_lambda_function.image_processor.arn
    events              = ["s3:ObjectCreated:*"]
    filter_prefix       = "uploads/"     # Only trigger for objects in uploads/ prefix
    filter_suffix       = ".jpg"         # Only for JPEG files
  }

  depends_on = [aws_lambda_permission.s3_invoke]
}
```

Lambda receives the S3 event and processes it:

```python
import boto3
import json

s3 = boto3.client('s3')

def handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key    = record['s3']['object']['key']

        # Download the uploaded file
        response = s3.get_object(Bucket=bucket, Key=key)
        image_data = response['Body'].read()

        # Process (resize, generate thumbnail, etc.)
        thumbnail = generate_thumbnail(image_data)

        # Upload result to output bucket
        s3.put_object(
            Bucket=os.environ['OUTPUT_BUCKET'],
            Key=f"thumbnails/{key}",
            Body=thumbnail,
            ContentType='image/jpeg'
        )
```

**Direction 2: Lambda calls S3 directly (IAM-controlled).**

Lambda uses the AWS SDK to read/write S3. The only requirement: the Lambda execution role must have the necessary S3 permissions.

```hcl
# Lambda execution role with S3 access
resource "aws_iam_role_policy" "lambda_s3_access" {
  name = "lambda-s3-access"
  role = aws_iam_role.lambda_exec.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:PutObject"]
        Resource = "${aws_s3_bucket.data.arn}/*"
      },
      {
        Effect   = "Allow"
        Action   = ["s3:ListBucket"]
        Resource = aws_s3_bucket.data.arn
      }
    ]
  })
}
```

No access keys needed — Lambda automatically uses the execution role's credentials via the instance metadata service.

**Network consideration: VPC Lambdas need a VPC endpoint or NAT Gateway to reach S3.**

Lambda functions NOT in a VPC reach S3 directly over the internet (via AWS's internal network).

Lambda functions IN a VPC (needed to access RDS, ElastiCache) cannot reach S3 by default — the VPC's private subnets have no internet route. Two options:

```hcl
# Option 1: VPC Gateway Endpoint for S3 (free, recommended)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.ap-south-1.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [aws_route_table.private.id]
  # S3 traffic from the VPC now routes via this endpoint, not NAT Gateway
}

# Option 2: NAT Gateway (costs money per GB processed)
# Everything in private subnets routes internet-bound traffic through NAT
```

Gateway endpoints are free and add a route to the VPC's route table. Every S3 request from a VPC Lambda goes through the endpoint instead of NAT — eliminating NAT Gateway data processing charges for S3 traffic.

**Common patterns:**

| Pattern | Trigger direction | Use case |
|---|---|---|
| S3 → Lambda | S3 event → Lambda | File processing on upload (images, CSVs, videos) |
| Lambda → S3 (write) | Lambda invoked by other event | Store results, generate reports, write logs |
| Lambda → S3 (read) | Lambda invoked by other event | Load config files, process reference data |
| S3 → SQS → Lambda | S3 event → SQS → Lambda | High-volume file processing with backpressure |

The S3 → SQS → Lambda pattern is preferred over direct S3 → Lambda for high-volume scenarios because:
- SQS provides a buffer — Lambda can process at its own pace
- SQS provides retry logic — failed Lambda invocations go back to the queue
- S3 can deliver to SQS with guaranteed delivery; direct S3 → Lambda has a small chance of missed events at high volume

**Real scenario:** We had a data pipeline where clients upload CSV files to S3. Direct S3 → Lambda worked fine at low volume. When a client uploaded 10,000 files in a batch, Lambda hit the concurrency limit (1,000 concurrent executions by default) and S3 events were throttled — some files never got processed. We moved to S3 → SQS → Lambda with a concurrency limit on the Lambda of 100. The SQS queue absorbed all 10,000 file events. Lambda processed them at 100 concurrent executions. No files were missed. Processing time: ~10 minutes instead of near-instant, which was acceptable for this batch pipeline.

---

## 12. What are the use cases of CodeBuild, CodePipeline, and CodeDeploy?

These are AWS's native CI/CD services. Each handles a different stage of the pipeline. Understanding when to use them (vs Jenkins, GitHub Actions, ArgoCD) is what the interviewer wants.

**CodeBuild — managed build service (CI).**

CodeBuild runs your build commands (compile, test, scan, build Docker image) in a managed container. You don't manage build servers.

```yaml
# buildspec.yml — goes in your app repo
version: 0.2
phases:
  pre_build:
    commands:
      - echo Logging in to ECR
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REPO
  build:
    commands:
      - docker build -t $ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker push $ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION
  post_build:
    commands:
      - echo Build completed
artifacts:
  files:
    - appspec.yml
    - taskdef.json
```

**Use cases:** Building Docker images, running unit tests, compiling code, running security scans (Trivy, SonarQube). Replaces Jenkins build agents — no EC2 instances to manage, pay per build minute.

**CodePipeline — orchestration (glue between stages).**

CodePipeline connects Source → Build → Deploy stages. It doesn't build or deploy itself — it triggers CodeBuild and CodeDeploy at the right time.

```
CodePipeline:
  Source Stage:  GitHub repo (webhook trigger on push)
       ↓
  Build Stage:   CodeBuild (build image, run tests)
       ↓
  Approval:      Manual approval gate
       ↓
  Deploy Stage:  CodeDeploy (to EC2/ECS/Lambda)
```

**Use cases:** Orchestrating multi-stage deployments, adding manual approval gates, connecting AWS-native services together. Replaces Jenkins pipeline orchestration if you're fully in the AWS ecosystem.

**CodeDeploy — deployment automation (CD).**

CodeDeploy handles rolling out new code to EC2 instances, ECS services, or Lambda functions. It manages health checks, rollback, and traffic shifting.

```yaml
# appspec.yml for EC2 deployment
version: 0.0
os: linux
files:
  - source: /
    destination: /opt/myapp
hooks:
  BeforeInstall:
    - location: scripts/stop_server.sh
  AfterInstall:
    - location: scripts/start_server.sh
  ValidateService:
    - location: scripts/health_check.sh
      timeout: 60
```

**Deployment strategies in CodeDeploy:**

| Strategy | How it works | Use case |
|---|---|---|
| **AllAtOnce** | Deploy to all instances simultaneously | Dev/test environments |
| **OneAtATime** | Deploy to one instance at a time | Small fleet, zero-risk |
| **HalfAtATime** | Deploy to 50% of instances, then the other 50% | Balanced speed/safety |
| **Blue/Green** | Spin up new instances, shift traffic, terminate old | Production (EC2 or ECS) |
| **Canary (Lambda)** | 10% traffic to new version, then 100% | Lambda functions |

**When to use AWS-native vs third-party:**

| Scenario | Choose |
|---|---|
| AWS-only, ECS/Lambda, team prefers managed services | CodeBuild + CodePipeline + CodeDeploy |
| Multi-cloud, complex pipelines, existing Jenkins | Jenkins + ArgoCD |
| GitHub-centric, open-source, simple pipelines | GitHub Actions |
| Kubernetes-first, GitOps | Jenkins/GitHub Actions (CI) + ArgoCD (CD) |

**Real scenario:** We used CodePipeline + CodeDeploy for an EC2-based legacy app. The pipeline was: GitHub → CodeBuild (build JAR) → Manual Approval → CodeDeploy (rolling deploy to EC2 fleet). CodeDeploy's Blue/Green strategy spun up new instances with the new version, ran health checks, then shifted the ALB target group. Rollback was automatic if health checks failed — CodeDeploy kept the old instances alive for 1 hour.

> **Also asked as:** "How to autoscale in EC2?" — covered in Q10 (Auto Scaling Groups with target tracking, step scaling, and scheduled scaling).

