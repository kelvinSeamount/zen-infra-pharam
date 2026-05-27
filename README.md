# zen-infra — Implementation Guide

![Infra Setup](docs/infra.png)
# Test


This guide walks you through setting up the zen-pharma infrastructure on your own AWS account from scratch using this repository. Follow each section in order.


----

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Repository Structure](#3-repository-structure)
4. [Step 1 — AWS Account Setup](#4-step-1--aws-account-setup)
5. [Step 2 — S3 State Backend Setup](#5-step-2--s3-state-backend-setup)


---

## 1. Architecture Overview

This repository provisions a complete Kubernetes based platform on AWS for the zen-pharma
application. All infrastructure is defined as code in Terraform and deployed automatically
via GitHub Actions no manual AWS console clicks required after initial setup.


---

### AWS Resources Created by Terraform

```
AWS Account (eu-central-1)
│
├── S3 Bucket  (created manually — state backend for Terraform)
│   └── zen-pharma-terraform-state-<your-username>
│       ├── envs/dev/terraform.tfstate
│       ├── envs/qa/terraform.tfstate
│       └── envs/prod/terraform.tfstate
│
├── VPC  (10.0.0.0/16)
│   ├── Public Subnets        10.0.1.0/24  (us-east-1a)  ]  NAT Gateway,
│   │                         10.0.2.0/24  (us-east-1b)  ]  NLB, Ingress
│   ├── Private EKS Subnets   10.0.3.0/24  (us-east-1a)  ]  EKS worker
│   │                         10.0.4.0/24  (us-east-1b)  ]  nodes (private)
│   └── Private RDS Subnets   10.0.5.0/24  (us-east-1a)  ]  RDS PostgreSQL
│                             10.0.6.0/24  (us-east-1b)  ]  (private)
│
├── EKS Cluster  (pharma-dev-cluster, Kubernetes 1.33)
│   ├── Managed Node Group
│   │   ├── Instance type : t3.small
│   │   ├── Desired       : 3 nodes
│   │   ├── Min / Max     : 1 / 4
│   │   └── Subnets       : private EKS subnets (no public IP)
│   └── OIDC Provider
│       └── Enables IRSA — pods assume IAM roles without static credentials
│
├── RDS PostgreSQL  (pharma-dev-postgres)
│   ├── Engine        : PostgreSQL 15.7
│   ├── Instance      : db.t3.micro
│   ├── Storage       : 20 GB gp2, encrypted
│   ├── Access        : private subnet only, port 5432 from EKS SG only
│   └── DB name       : pharmadb  /  Master user: pharmaadmin
│
├── ECR Repositories  (8 repos, one per service)
│   ├── api-gateway               (Spring Cloud Gateway, port 8080)
│   ├── auth-service              (JWT auth, port 8081)
│   ├── drug-catalog-service      (drug catalogue, port 8082)
│   ├── inventory-service         (stock management, port 8083)
│   ├── supplier-service          (vendor management, port 8084)
│   ├── manufacturing-service     (batch tracking, port 8085)
│   ├── notification-service      (Node.js, port 3000)
│   └── pharma-ui                 (React frontend, port 80)
│   │
│   └── Each repo has:
│       ├── scan_on_push = true   (automatic CVE scan on every push)
│       └── Lifecycle policy      (keep last 10 images, expire older ones)
│
├── IAM
│   ├── EKS Cluster Role          (allows EKS control plane to manage AWS resources)
│   ├── EKS Node Group Role       (allows worker nodes to pull from ECR, join cluster)
│   │
│   ├── GitHub Actions OIDC Role  (pharma-dev-gitlab-runner-role)
│   │   ├── Trust policy : repo zen-pharma-frontend and zen-pharma-backend only
│   │   └── Permissions  : ECR push/pull, EKS describe
│   │   └── How it works : GitHub OIDC token -> AWS STS -> short-lived credentials
│   │                      No AWS_ACCESS_KEY_ID stored in GitHub
│   │
│   ├── ESO IRSA Role             (pharma-dev-eso-role)
│   │   ├── Trust policy : EKS service account external-secrets/external-secrets
│   │   └── Permissions  : secretsmanager:GetSecretValue on /pharma/* paths only
│   │
│   └── ArgoCD IRSA Role          (pharma-dev-argocd-role)
│       └── Trust policy : EKS service account argocd/argocd-application-controller
│
└── AWS Secrets Manager
    ├── /pharma/dev/db-credentials   {"username": "pharmaadmin", "password": "..."}
    └── /pharma/dev/jwt-secret       {"secret": "..."}
```

---

### Terraform Module Structure

```
zen-infra/
├── envs/
│   ├── dev/    <-- calls all modules with dev-specific values
│   ├── qa/     <-- same modules, different sizing
│   └── prod/   <-- same modules, production sizing + HA settings
│
└── modules/
    ├── vpc/            creates VPC, subnets, IGW, NAT GW, route tables
    ├── eks/            creates EKS cluster, node group, OIDC provider
    ├── rds/            creates RDS instance, subnet group, security group
    ├── ecr/            creates ECR repos with lifecycle policies
    ├── iam/            creates OIDC roles for GitHub Actions, ESO, ArgoCD
    └── secrets-manager/ stores DB password and JWT secret in Secrets Manager
```


Each environment directory (`envs/dev`) calls the modules like functions:

```
envs/dev/main.tf
    |
    |-- module "vpc"              --> modules/vpc/
    |-- module "eks"              --> modules/eks/   (depends on vpc outputs)
    |-- module "rds"              --> modules/rds/   (depends on vpc + eks outputs)
    |-- module "ecr"              --> modules/ecr/
    |-- module "iam"              --> modules/iam/   (depends on eks OIDC outputs)
    └-- module "secrets_manager"  --> modules/secrets-manager/
```

Modules share data via outputs — for example, `module.eks.oidc_provider_arn` is passed
into `module.iam` so the IAM trust policy references the exact OIDC provider created
for this cluster, not a hardcoded ARN.

---

### Network Traffic Flow

```
Internet
    |
    v
AWS Network Load Balancer  (created by NGINX Ingress Controller Helm chart)
    |  routes by URL path
    |-- /          -->  pharma-ui       (React, port 80)
    |-- /api/*     -->  api-gateway     (port 8080)
                           |
                           |-- /api/auth/*          --> auth-service        (8081)
                           |-- /api/catalog/*       --> drug-catalog-svc    (8082)
                           |-- /api/inventory/*     --> inventory-service   (8083)
                           |-- /api/suppliers/*     --> supplier-service    (8084)
                           |-- /api/manufacturing/* --> manufacturing-svc   (8085)
                           └-- /api/notifications/* --> notification-svc    (3000)
                                                            |
                                                    All backend services
                                                    pull secrets from
                                                    AWS Secrets Manager
                                                    via ESO (no passwords
                                                    in pod spec or config)
                                                            |
                                                            v
                                               RDS PostgreSQL (private subnet)
```

---
### GitHub Actions CI/CD Flow for Infrastructure

```
Developer creates feature branch in zen-infra
    |
    v
git push origin feature/my-change
    |
    v
Open Pull Request  -->  zen-infra GitHub Actions runs automatically:
    |
    |   [Terraform Plan job]
    |   1. Checkout code
    |   2. Setup Terraform 1.10.0
    |   3. Configure AWS credentials  (static IAM keys from GitHub Secrets)
    |   4. terraform fmt -check       --> fails if code is not formatted
    |   5. terraform init -backend-config=backend.tfvars  --> connects to S3 backend
    |   6. terraform validate         --> syntax and logic check
    |   7. terraform plan             --> shows what will change (saved as artifact)
    |
    |   Plan output is visible in the Actions tab. PR is blocked if plan fails.
    |
    v
PR reviewed and merged to main
    |
    v
[Terraform Plan job runs again on main - fresh plan]
    |
    v
[Terraform Apply job - PAUSES for manual approval]
    |
    |   Go to: Actions --> running workflow --> "Review deployments" --> Approve
    |
    v
[Terraform Apply runs]
    |   8. terraform apply tfplan     --> provisions/updates AWS resources
    |      (takes 15-25 min for EKS + RDS)
    |
    v
Infrastructure updated in AWS
    |
    v
EKS cluster is ready for Stage 2 (install NGINX Ingress, ArgoCD, ESO)
```

---

### Security Design Decisions

| Decision | Why |
|---|---|
| Worker nodes in private subnets | Nodes not reachable from internet; only NLB is public |
| RDS in private subnets | Database never exposed to internet; only EKS nodes can connect (port 5432 via SG rule) |
| No static AWS keys in GitHub CI | GitHub Actions uses OIDC; credentials are short-lived (1 hour) and scoped to specific repos |
| IRSA for pods | Pods never hold AWS credentials; they exchange a projected K8s token for short-lived STS credentials |
| Secrets Manager (not ConfigMap) | DB passwords and JWT secret never live in Git or Kubernetes config; ESO syncs them at runtime |
| ECR scan on push | Every image is automatically scanned for CVEs when pushed; results visible in ECR console |
| S3 state with versioning | Terraform state is versioned — accidental corruption can be rolled back |

---

## 2. Prerequisites

Ensure the following tools are installed on your local machine before starting.

### Required Tools

| Tool | Minimum Version | Install |
|---|---|---|
| Terraform | 1.10.0+ | https://developer.hashicorp.com/terraform/install |
| AWS CLI | 2.x | https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html |
| Git | 2.x | https://git-scm.com/downloads |

### Verify Installations

```bash
terraform version
# Terraform v1.10.x

aws --version
# aws-cli/2.x.x

git --version
# git version 2.x.x
```

### Required Access

- An AWS account with administrator access (or sufficient permissions — see Step 1)
- A GitHub account
- The zen-infra repository forked to your GitHub account

---

## 3. Repository Structure

```
zen-infra/
├── .github/
│   ├── dependabot.yml                    # Automated dependency update config
│   └── workflows/
│       └── terraform.yml                 # CI/CD pipeline — plan + apply + destroy
│
├── envs/
│   ├── dev/
│   │   ├── backend.tf                    # S3 remote state config for dev (bucket + key only)
│   │   ├── backend.tfvars                # Backend region — passed to terraform init
│   │   ├── providers.tf                  # AWS, Kubernetes, TLS provider config
│   │   ├── main.tf                       # Module calls with dev-specific values
│   │   ├── variables.tf                  # Input variable declarations (includes aws_region)
│   │   └── outputs.tf                    # Output values (cluster name, RDS endpoint)
│   ├── qa/                               # QA environment (structure mirrors dev)
│   └── prod/                             # Prod environment (structure mirrors dev)
│
└── modules/
    ├── vpc/                              # VPC, subnets, IGW, NAT Gateway, route tables
    ├── eks/                              # EKS cluster, node group, OIDC provider
    ├── rds/                              # RDS PostgreSQL, subnet group, security group
    ├── ecr/                              # ECR repositories and lifecycle policies
    ├── iam/                              # GitHub Actions OIDC role and policy
    └── secrets-manager/                  # Secrets Manager secrets for app credentials
```

**Key design decisions:**
- **Directory-per-environment** (`envs/dev`, `envs/qa`, `envs/prod`) — complete isolation, separate state files, different resource sizing per environment
- **Shared modules** — all environments call the same modules with different input values
- **No `terraform.tfvars`** — secrets are never stored on disk, passed at runtime from GitHub Secrets

---

## 4. Step 1 — AWS Account Setup

### 4.1 Create an IAM User for Terraform (if not using OIDC)

For the initial bootstrap (before OIDC is set up via Terraform), you need an IAM user with programmatic access.

Go to **AWS Console → IAM → Users → Create user**:
- Username: `terraform-ci`
- Access type: Programmatic access
- Permissions: Attach the following managed policies:
  - `AdministratorAccess` (simplest for learning — scope down in production)

Save the **Access Key ID** and **Secret Access Key** — you will need these in Step 5.

> **Note for production**: Scope IAM permissions to only what Terraform needs — EC2, EKS, RDS, ECR, IAM, Secrets Manager, S3, VPC.

### 4.2 Configure AWS CLI Locally

```bash
aws configure
# AWS Access Key ID: <your-access-key-id>
# AWS Secret Access Key: <your-secret-access-key>
# Default region name: us-east-1
# Default output format: json
```

Verify it works:

```bash
aws sts get-caller-identity
# Should return your account ID, user ARN, and user ID
```

---

## 5. Step 2 — S3 State Backend Setup

Terraform requires an S3 bucket to store its state file. This bucket must exist **before** running Terraform. Create it manually — you only do this once.

### 5.1 Create the S3 Bucket

Replace `YOUR-GITHUB-USERNAME` with your actual GitHub username to make the bucket name unique.

```bash
# Create the bucket
aws s3api create-bucket \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --region us-east-1

# Enable versioning (allows state rollback)
aws s3api put-bucket-versioning \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# Block all public access
aws s3api put-public-access-block \
  --bucket zen-pharma-terraform-state-YOUR-GITHUB-USERNAME \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

### 5.2 Verify the Bucket

```bash
aws s3 ls s3://zen-pharma-terraform-state-YOUR-GITHUB-USERNAME
# Should return empty (no error)
```
