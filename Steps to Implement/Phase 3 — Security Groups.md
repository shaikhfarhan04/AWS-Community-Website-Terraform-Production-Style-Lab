# Phase 3 — Security Groups 🔐

We’ll keep the design production-style but simple enough for the lab.

### Target traffic flow

```text
                    INTERNET
                       │
                    TCP 80
                       ▼
              ┌─────────────────┐
              │     ALB SG      │
              │  Inbound: 80    │
              └────────┬────────┘
                       │
                    TCP 80
                       ▼
              ┌─────────────────┐
              │   WEB/APP SG    │
              │ Inbound: 80     │
              │ From ALB SG     │
              └────────┬────────┘
                       │
                    TCP 3306
                       ▼
              ┌─────────────────┐
              │     RDS SG      │
              │ Inbound: 3306   │
              │ From WEB SG     │
              └─────────────────┘

        EC2 administration
               │
               ▼
        AWS SSM Session Manager
        No public SSH :22
```

The important security principle is:

> **Each layer only accepts traffic from the layer immediately in front of it.**

So we will **not** use `0.0.0.0/0` for the application or database security groups.

---

# Step 3.1 — Create the Security Group module

From PowerShell:

```powershell
cd "F:\DevOps-Data\Terraform-Project\aws-comunity-terraform"
```

Create the module:

```powershell
New-Item -ItemType Directory -Force -Path ".\modules\security"
```

Our structure will become:

```text
aws-comunity-terraform
│
├── modules
│   ├── vpc
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   └── security
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
└── terraform
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── providers.tf
    ├── versions.tf
    └── terraform.tfvars
```

---

# Step 3.2 — `modules/security/variables.tf`

Create:

```powershell
code ".\modules\security\variables.tf"
```

Add:

```hcl
variable "name_prefix" {
  description = "Name prefix for security group resources"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID where security groups will be created"
  type        = string
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}
```

### Why these variables?

`vpc_id` tells AWS:

> Create these security groups inside our Phase 2 VPC.

Your VPC is:

```text
10.0.0.0/16
```

and currently has:

```text
vpc-07df451f36c44d612
```

We don't hard-code that ID. Terraform will obtain it from the VPC module.

---

# Step 3.3 — `modules/security/main.tf`

Create:

```powershell
code ".\modules\security\main.tf"
```

Add:

```hcl
# ---------------------------------------------------------
# ALB Security Group
# ---------------------------------------------------------

resource "aws_security_group" "alb" {
  name        = "${var.name_prefix}-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = var.vpc_id

  tags = merge(
    var.tags,
    {
      Name = "${var.name_prefix}-alb-sg"
    }
  )
}

resource "aws_vpc_security_group_ingress_rule" "alb_http" {
  security_group_id = aws_security_group.alb.id

  description = "Allow HTTP from the internet"

  cidr_ipv4   = "0.0.0.0/0"
  from_port   = 80
  to_port     = 80
  ip_protocol = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "alb_all" {
  security_group_id = aws_security_group.alb.id

  description = "Allow outbound traffic"

  cidr_ipv4   = "0.0.0.0/0"
  ip_protocol = "-1"
}


# ---------------------------------------------------------
# Web / Application Security Group
# ---------------------------------------------------------

resource "aws_security_group" "web" {
  name        = "${var.name_prefix}-web-sg"
  description = "Security group for web/application EC2 instances"
  vpc_id      = var.vpc_id

  tags = merge(
    var.tags,
    {
      Name = "${var.name_prefix}-web-sg"
    }
  )
}

resource "aws_vpc_security_group_ingress_rule" "web_http_from_alb" {
  security_group_id = aws_security_group.web.id

  description                  = "Allow HTTP from ALB"
  referenced_security_group_id = aws_security_group.alb.id

  from_port   = 80
  to_port     = 80
  ip_protocol = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "web_all" {
  security_group_id = aws_security_group.web.id

  description = "Allow outbound traffic"

  cidr_ipv4   = "0.0.0.0/0"
  ip_protocol = "-1"
}


# ---------------------------------------------------------
# RDS Security Group
# ---------------------------------------------------------

resource "aws_security_group" "rds" {
  name        = "${var.name_prefix}-rds-sg"
  description = "Security group for RDS MySQL"
  vpc_id      = var.vpc_id

  tags = merge(
    var.tags,
    {
      Name = "${var.name_prefix}-rds-sg"
    }
  )
}

resource "aws_vpc_security_group_ingress_rule" "rds_mysql_from_web" {
  security_group_id = aws_security_group.rds.id

  description                  = "Allow MySQL from web/application servers"
  referenced_security_group_id = aws_security_group.web.id

  from_port   = 3306
  to_port     = 3306
  ip_protocol = "tcp"
}

resource "aws_vpc_security_group_egress_rule" "rds_all" {
  security_group_id = aws_security_group.rds.id

  description = "Allow outbound traffic"

  cidr_ipv4   = "0.0.0.0/0"
  ip_protocol = "-1"
}
```

---

## Understand the important part

### ALB

```hcl
cidr_ipv4 = "0.0.0.0/0"
from_port = 80
```

This means:

```text
Internet
   │
   │ HTTP :80
   ▼
ALB
```

That's appropriate because the ALB is the public entry point.

---

### Web/App EC2

Notice that we **do not** use:

```hcl
cidr_ipv4 = "0.0.0.0/0"
```

Instead:

```hcl
referenced_security_group_id = aws_security_group.alb.id
```

Meaning:

```text
ALB SG
  │
  │ TCP 80
  ▼
Web SG
```

Only resources associated with the ALB security group can reach port 80 on the application servers.

---

### RDS

Again, we don't expose MySQL publicly.

```hcl
referenced_security_group_id = aws_security_group.web.id
```

Therefore:

```text
Web/App SG
     │
     │ TCP 3306
     ▼
 RDS SG
```

Not:

```text
Internet ──X──> RDS :3306
```

This is one of the most important security concepts in AWS.

---

# Step 3.4 — Security module outputs

Create:

```powershell
code ".\modules\security\outputs.tf"
```

Add:

```hcl
output "alb_security_group_id" {
  description = "Security group ID for the Application Load Balancer"
  value       = aws_security_group.alb.id
}

output "web_security_group_id" {
  description = "Security group ID for web/application EC2 instances"
  value       = aws_security_group.web.id
}

output "rds_security_group_id" {
  description = "Security group ID for RDS"
  value       = aws_security_group.rds.id
}
```

---

# Step 3.5 — Call the module from root Terraform

Open:

```powershell
code ".\terraform\main.tf"
```

You already have the VPC module there.

Keep the existing VPC module and add the security module **after it**:

```hcl
module "vpc" {
  source = "../modules/vpc"

  name_prefix = local.name_prefix
  vpc_cidr    = "10.0.0.0/16"

  public_subnets = {
    "ap-south-1a" = "10.0.1.0/24"
    "ap-south-1b" = "10.0.2.0/24"
  }

  app_subnets = {
    "ap-south-1a" = "10.0.11.0/24"
    "ap-south-1b" = "10.0.12.0/24"
  }

  db_subnets = {
    "ap-south-1a" = "10.0.21.0/24"
    "ap-south-1b" = "10.0.22.0/24"
  }

  enable_nat_gateway = true

  tags = local.common_tags
}


# ---------------------------------------------------------
# Security Groups
# ---------------------------------------------------------

module "security" {
  source = "../modules/security"

  name_prefix = local.name_prefix
  vpc_id      = module.vpc.vpc_id

  tags = local.common_tags
}
```

### Important

The critical connection is:

```hcl
vpc_id = module.vpc.vpc_id
```

Terraform understands:

```text
VPC Module
     │
     │ vpc_id
     ▼
Security Module
```

So we don't manually copy:

```text
vpc-07df451f36c44d612
```

into our Terraform code.

---

# Step 3.6 — Root outputs

Open:

```powershell
code ".\terraform\outputs.tf"
```

Keep your existing outputs and add:

```hcl
output "alb_security_group_id" {
  description = "ALB security group ID"
  value       = module.security.alb_security_group_id
}

output "web_security_group_id" {
  description = "Web/application security group ID"
  value       = module.security.web_security_group_id
}

output "rds_security_group_id" {
  description = "RDS security group ID"
  value       = module.security.rds_security_group_id
}
```

---

# Step 3.7 — Format and validate

Now run:

```powershell
cd ".\terraform"

terraform fmt -recursive
```

Then:

```powershell
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

Now generate the plan:

```powershell
terraform plan
```

### Expected change

Because Phase 2 is already applied, Terraform should now show approximately:

```text
Plan: 9 to add, 0 to change, 0 to destroy.
```

That's because we have:

```text
3 Security Groups
3 Ingress rules
3 Egress rules
----------------
9 resources
```

Do **not** apply yet if the plan shows unexpected changes to your VPC/NAT/subnets.

---

## Phase 3 security model

After this step, the AWS architecture will look like:

```text
                         INTERNET
                            │
                            │ :80
                            ▼
                  ┌──────────────────┐
                  │       ALB        │
                  │     ALB-SG       │
                  └────────┬─────────┘
                           │
                           │ :80
                           │ ALB-SG only
                           ▼
              ┌─────────────────────────┐
              │      EC2 APP #1         │
              │      EC2 APP #2         │
              │        WEB-SG            │
              └────────────┬────────────┘
                           │
                           │ :3306
                           │ WEB-SG only
                           ▼
                  ┌──────────────────┐
                  │       RDS        │
                  │      RDS-SG      │
                  └──────────────────┘
```

And administration will later be:

```text
Your PC
   │
   │ AWS SSM
   ▼
Private EC2
```

**No `0.0.0.0/0 → TCP 22`.** 🔒

### Your immediate task

Run these three commands from `terraform`:

```powershell
terraform fmt -recursive
terraform validate
terraform plan
```

**Send me the complete `terraform plan` result.** We'll review it together before you run `terraform apply`.
