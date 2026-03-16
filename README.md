# Multi-Tier Web Application Architecture on AWS

A production-ready, highly available multi-tier architecture on AWS, provisioned with Terraform and deployed via GitHub Actions CI/CD.

## Architecture Overview

```
Internet
   │
   ▼
[ALB] ── public subnets (2 AZs)
   │
   ▼
[EC2 Auto Scaling Group] ── private app subnets (2 AZs)
   │
   ▼
[RDS PostgreSQL Multi-AZ] ── private DB subnets (2 AZs)
```

**Network layout per AZ:**
- Public subnet → ALB, NAT Gateway
- Private App subnet → EC2 instances (no direct internet exposure)
- Private DB subnet → RDS (no internet access at all)

## Repository Structure

```
cloud-architecture/
├── terraform/
│   ├── main.tf                  # VPC, subnets, IGW, NAT, ALB, ASG, RDS, IAM
│   ├── variables.tf             # All input variables
│   ├── outputs.tf               # Key output values (ALB DNS, RDS endpoint, etc.)
│   ├── terraform.tfvars.example # Example variable values — copy and fill in
│   └── userdata.sh              # EC2 bootstrap script (CloudWatch Agent + app service)
├── github-actions/
│   └── deploy.yml               # Full CI/CD pipeline (test → scan → plan → build → deploy)
└── cloudwatch/
    ├── cloudwatch.tf            # Dashboards, alarms, log groups, SNS
    └── variables.tf             # alert_email variable
```

## Prerequisites

- AWS CLI configured with appropriate credentials
- Terraform >= 1.5.0
- An S3 bucket + DynamoDB table for Terraform remote state
- An ACM certificate in `us-east-1` for your domain
- GitHub repository secrets configured (see CI/CD section)

## Quick Start

### 1. Bootstrap Terraform Remote State

```bash
# Create S3 bucket for state
aws s3api create-bucket \
  --bucket your-terraform-state-bucket \
  --region us-east-1

aws s3api put-bucket-versioning \
  --bucket your-terraform-state-bucket \
  --versioning-configuration Status=Enabled

# Create DynamoDB table for state locking
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

### 2. Configure Variables

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values
```

### 3. Deploy Infrastructure

```bash
terraform init
terraform plan
terraform apply
```

## CI/CD Pipeline (GitHub Actions)

The pipeline in `github-actions/deploy.yml` runs these stages in order:

| Stage | Trigger | Description |
|---|---|---|
| **Test & Lint** | All pushes/PRs | Unit tests + coverage |
| **Security Scan** | After tests | Trivy vulnerability scan (CRITICAL/HIGH) |
| **Terraform Plan** | PRs only | Posts plan diff as PR comment |
| **Build & Push** | `main` / `develop` | Docker image → ECR |
| **Deploy Staging** | `develop` branch | `terraform apply` + ASG instance refresh |
| **Deploy Prod** | `main` branch | Manual approval gate → `terraform apply` + rolling refresh |

### Required GitHub Secrets

```
AWS_ROLE_ARN              # IAM role ARN (OIDC) for staging
AWS_ROLE_ARN_STAGING      # IAM role ARN for staging deploys
AWS_ROLE_ARN_PROD         # IAM role ARN for prod deploys
ECR_REPOSITORY            # ECR repository name
ACM_CERT_ARN              # ACM certificate ARN (staging)
ACM_CERT_ARN_PROD         # ACM certificate ARN (prod)
DB_USERNAME               # RDS username
DB_PASSWORD               # RDS password
DB_USERNAME_PROD          # RDS username (prod)
DB_PASSWORD_PROD          # RDS password (prod)
SLACK_WEBHOOK_URL         # Slack notifications
```

## CloudWatch Monitoring

The `cloudwatch/cloudwatch.tf` file creates:

**Dashboards:**
- `{project}-overview` — single-pane view of ALB, EC2, and RDS metrics

**Alarms (→ SNS → Email):**
| Alarm | Threshold |
|---|---|
| ALB 5XX errors | > 10 per minute |
| Target 5XX errors | > 10 per minute |
| ALB response time (p99) | > 2 seconds |
| Unhealthy hosts | > 0 |
| EC2 CPU | > 80% |
| EC2 memory | > 85% |
| RDS CPU | > 75% |
| RDS free storage | < 5 GB |
| RDS connections | > 80 |

## Security Highlights

- EC2 instances have **no public IPs** — access via SSM Session Manager only
- RDS is in a **private subnet with no internet route**
- ALB enforces **TLS 1.3** and redirects all HTTP → HTTPS
- Terraform state is **encrypted at rest** in S3
- GitHub Actions uses **OIDC** (no long-lived AWS keys)
- EBS volumes are **encrypted**
