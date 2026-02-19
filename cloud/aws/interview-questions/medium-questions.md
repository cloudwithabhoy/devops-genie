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
