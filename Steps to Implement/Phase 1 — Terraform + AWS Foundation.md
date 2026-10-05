# Phase 1 — Terraform + AWS Foundation

We'll keep Phase 1 **safe and cost-free**: no EC2, RDS, NAT Gateway, ALB, or other billable infrastructure yet.

### Phase 1 objectives

By the end, you will have:

```text
aws-community-terraform/
│
├── terraform/
│   ├── main.tf
│   ├── providers.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   └── terraform.tfvars
│
├── modules/
├── app/
├── scripts/
├── docs/
├── .gitignore
└── README.md
```

And you'll successfully run:

```powershell
terraform init
terraform fmt
terraform validate
terraform plan
```

---

## Step 1 — Create the project folder

Since you're using Windows + PowerShell, let's use your existing DevOps workspace.

Run:

```powershell
cd "G:\DevOps-Data\DevOps Project"
```

Create the project:

```powershell
mkdir aws-community-terraform
cd aws-community-terraform
```

Create the directories:

```powershell
mkdir terraform
mkdir modules
mkdir app
mkdir scripts
mkdir docs
```

Check:

```powershell
tree /F
```

You should initially see:

```text
aws-community-terraform
├── app
├── docs
├── modules
├── scripts
└── terraform
```

---

# Step 2 — Check Terraform

Run:

```powershell
terraform version
```

You should get something similar to:

```text
Terraform v1.x.x
on windows_amd64
```

For this lab, Terraform **1.5+** is sufficient.

Also check AWS CLI:

```powershell
aws --version
```

Example:

```text
aws-cli/2.x.x
```

---

# Step 3 — Check AWS credentials

Run:

```powershell
aws sts get-caller-identity
```

Expected:

```json
{
    "UserId": "XXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/..."
}
```

### Important

Do **not** put your AWS access key or secret key inside:

```text
terraform.tfvars
main.tf
providers.tf
GitHub
```

We'll use your existing AWS CLI credentials.

---

# Step 4 — Create `versions.tf`

Go into:

```powershell
cd terraform
```

Create:

```powershell
New-Item versions.tf
```

Open the project in VS Code:

```powershell
code ..
```

Put this in `versions.tf`:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

### What this does

It tells Terraform:

```text
Terraform version >= 1.5
        +
AWS provider 6.x
```

The AWS provider is what allows Terraform to communicate with AWS.

---

# Step 5 — Create `providers.tf`

Create:

```text
terraform/providers.tf
```

Add:

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

The important part is:

```hcl
region = var.aws_region
```

We'll define the region through a variable rather than hard-coding it.

---

# Step 6 — Create `variables.tf`

Create:

```text
terraform/variables.tf
```

Add:

```hcl
variable "aws_region" {
  description = "AWS region where resources will be created"
  type        = string
  default     = "ap-south-1"
}

variable "project_name" {
  description = "Name of the AWS Community project"
  type        = string
  default     = "aws-community"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}
```

We're establishing our first configuration variables:

```text
Region       = ap-south-1
Project      = aws-community
Environment  = dev
EC2          = t3.micro
```

---

# Step 7 — Create `main.tf`

Create:

```text
terraform/main.tf
```

For Phase 1, don't create AWS infrastructure yet.

Use:

```hcl
locals {
  project_name = var.project_name
  environment  = var.environment

  name_prefix = "${local.project_name}-${local.environment}"
}
```

This gives us a reusable naming convention.

For example, later:

```text
aws-community-dev-vpc
aws-community-dev-alb
aws-community-dev-web-1
aws-community-dev-web-2
aws-community-dev-rds
```

---

# Step 8 — Create `outputs.tf`

Create:

```text
terraform/outputs.tf
```

Add:

```hcl
output "project_name" {
  description = "Project name"
  value       = var.project_name
}

output "environment" {
  description = "Environment name"
  value       = var.environment
}

output "aws_region" {
  description = "AWS deployment region"
  value       = var.aws_region
}

output "name_prefix" {
  description = "Resource naming prefix"
  value       = local.name_prefix
}
```

---

# Step 9 — Create `terraform.tfvars`

Create:

```text
terraform/terraform.tfvars
```

Add:

```hcl
aws_region  = "ap-south-1"
project_name = "aws-community"
environment  = "dev"
instance_type = "t3.micro"
```

You can later change:

```hcl
environment = "prod"
```

and Terraform will generate names such as:

```text
aws-community-prod-vpc
aws-community-prod-alb
```

---

# Step 10 — Format Terraform

From:

```text
aws-community-terraform\terraform
```

run:

```powershell
terraform fmt
```

Expected:

```text
providers.tf
variables.tf
versions.tf
main.tf
outputs.tf
```

Terraform will format the files.

---

# Step 11 — Initialize Terraform

Run:

```powershell
terraform init
```

You should see something similar to:

```text
Initializing the backend...

Initializing provider plugins...

- Finding hashicorp/aws versions matching "~> 6.0"...
- Installing hashicorp/aws...

Terraform has been successfully initialized!
```

Terraform will create:

```text
.terraform/
.terraform.lock.hcl
```

### Don't delete `.terraform.lock.hcl`

We'll commit:

```text
.terraform.lock.hcl
```

to Git.

We will **not** commit:

```text
.terraform/
```

---

# Step 12 — Validate

Run:

```powershell
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

If you get this, your Terraform configuration is syntactically correct.

---

# Step 13 — Run Plan

Now:

```powershell
terraform plan
```

This is important.

Because we haven't created any AWS resources yet, you should see something similar to:

```text
No changes.
Your infrastructure matches the configuration.
```

This is **good**.

We haven't created:

```text
❌ EC2
❌ RDS
❌ ALB
❌ NAT Gateway
❌ VPC
```

So there should be nothing for Terraform to create.

---

# Step 14 — Test Terraform Outputs

Run:

```powershell
terraform output
```

You should get something similar to:

```text
aws_region = "ap-south-1"
environment = "dev"
name_prefix = "aws-community-dev"
project_name = "aws-community"
```

This confirms our variables and locals are working.

---

# Step 15 — Create `.gitignore`

Go back to the project root:

```powershell
cd ..
```

Create:

```powershell
New-Item .gitignore
```

Add:

```gitignore
# Terraform
.terraform/
*.tfstate
*.tfstate.*
crash.log
crash.*.log
*.tfplan

# Terraform variable files containing secrets
*.tfvars
!terraform/terraform.tfvars.example

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# AWS
.aws/

# Secrets
*.pem
*.key
*.crt
.env

# Python
__pycache__/
*.py[cod]
.venv/
venv/

# VS Code
.vscode/

# OS
Thumbs.db
.DS_Store
```

One important point: because our current `terraform.tfvars` contains only non-secret configuration, you **could** commit it, but I recommend keeping it ignored and creating an example file.

---

# Step 16 — Create `terraform.tfvars.example`

Create:

```text
terraform/terraform.tfvars.example
```

Use:

```hcl
aws_region   = "ap-south-1"
project_name = "aws-community"
environment  = "dev"
instance_type = "t3.micro"
```

This file **can be committed to GitHub**.

---

# Step 17 — Create README

At project root:

```text
README.md
```

Use:

````markdown
# AWS Community Terraform Project

Production-style AWS Community website infrastructure built using Terraform.

## Architecture

The project will eventually contain:

- Amazon VPC
- Public and private subnets
- Internet Gateway
- NAT Gateway
- Application Load Balancer
- Two EC2 web servers
- Amazon RDS MySQL
- IAM
- AWS Systems Manager
- AWS Secrets Manager
- Amazon S3
- Amazon CloudWatch

## Region

ap-south-1

## Environment

dev

## Terraform

Terraform >= 1.5

## AWS Provider

AWS Provider 6.x

## Project Structure

```text
aws-community-terraform/
├── terraform/
├── modules/
├── app/
├── scripts/
└── docs/
````

## Current Phase

Phase 1 - Terraform and AWS Foundation

````

---

# Phase 1 Expected Structure

At the end, your project should look like:

```text
aws-community-terraform/
│
├── terraform/
│   │
│   ├── .terraform/
│   │
│   ├── .terraform.lock.hcl
│   ├── main.tf
│   ├── providers.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   ├── terraform.tfvars
│   └── terraform.tfvars.example
│
├── modules/
│
├── app/
│
├── scripts/
│
├── docs/
│
├── .gitignore
└── README.md
````

---

# Phase 1 Verification Checklist

Run these **in this order**:

```powershell
terraform version
```

```powershell
aws sts get-caller-identity
```

```powershell
terraform fmt
```

```powershell
terraform init
```

```powershell
terraform validate
```

```powershell
terraform plan
```

Then:

```powershell
terraform output
```

### Expected result

```text
Terraform version     ✅
AWS authentication   ✅
Terraform formatting  ✅
Provider initialized  ✅
Configuration valid   ✅
Plan successful       ✅
Outputs working       ✅
AWS resources created  0
```

**Do not run `terraform apply` in Phase 1.** There is nothing to apply yet, and keeping this phase resource-free lets us establish the Terraform foundation without AWS charges.

Once these checks pass, **Phase 2 will be the interesting part: we'll build the complete VPC with 6 subnets, route tables, Internet Gateway, and NAT Gateway**, and I'll explain every resource before we create it.
