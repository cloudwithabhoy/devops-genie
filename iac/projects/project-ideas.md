# IaC Project Ideas

> Hands-on projects to build real Terraform and Ansible skills.

## Beginner

- **Terraform EC2 Instance** — Write a Terraform config to provision an EC2 instance with a security group and output its public IP.
- **Ansible Web Server Setup** — Write an Ansible playbook to install and configure Nginx on a remote VM.

## Intermediate

<!-- Add intermediate project ideas here -->

- **Terraform VPC Module** — Build a reusable Terraform module that creates a VPC with public and private subnets.
- **Remote State with S3** — Configure Terraform to use an S3 bucket and DynamoDB table for remote state and locking.
- **Ansible Role for App Deployment** — Write an Ansible role that deploys a Node.js or Python app, manages config, and ensures the service is running.

## Advanced

<!-- Add advanced project ideas here -->

- **Full AWS Environment** — Use Terraform to provision a complete environment: VPC, EC2 ASG, ALB, RDS, and S3 — all in modules.
- **Terraform CI/CD Pipeline** — Run `terraform plan` on PRs and `terraform apply` on merge via GitHub Actions with Atlantis or direct pipeline.
- **Policy as Code** — Integrate Checkov or OPA into a CI pipeline to enforce Terraform security policies.

## Project Template

For each project, document:
1. **Goal** — What infrastructure you're provisioning
2. **Tools** — Terraform version, Ansible version, cloud provider
3. **Files** — Directory structure and key file contents
4. **Steps** — How to initialise and apply
5. **Cleanup** — `terraform destroy` or playbook teardown steps
