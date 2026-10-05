
# Phase 2 — AWS VPC & Networking

## What we're building

Our target architecture:

```text
                         INTERNET
                            |
                            |
                    +-------v-------+
                    | Internet      |
                    | Gateway       |
                    +-------+-------+
                            |
              +-------------+-------------+
              |                           |
       PUBLIC SUBNET A              PUBLIC SUBNET B
       ap-south-1a                  ap-south-1b
       10.0.1.0/24                  10.0.2.0/24
              |                           |
              |                     NAT Gateway
              |                           |
              +-------------+-------------+
                            |
                    PRIVATE APP SUBNETS
                            |
              +-------------+-------------+
              |                           |
       APP SUBNET A                 APP SUBNET B
       10.0.11.0/24                 10.0.12.0/24
              |                           |
          Web Server 1                Web Server 2


                    PRIVATE DB SUBNETS
              +-------------+-------------+
              |                           |
       DB SUBNET A                   DB SUBNET B
       10.0.21.0/24                 10.0.22.0/24
              |                           |
              +---------- RDS ------------+
```

---

# Phase 2 objectives

By the end of this phase we'll have:

* VPC
* 6 subnets
* 2 Availability Zones
* Internet Gateway
* Public route table
* Private application route table
* Database route table
* NAT Gateway
* Elastic IP
* Route associations
* Terraform outputs
* Terraform tags

We'll also learn:

```text
VPC
CIDR
Subnet
Availability Zone
Route Table
Internet Gateway
NAT Gateway
Elastic IP
```

---

# Step 2.1 — Check your current region/AZs

Before writing Terraform, let's verify the two Availability Zones available in Mumbai.

Run:

```powershell
aws ec2 describe-availability-zones `
  --region ap-south-1 `
  --query "AvailabilityZones[?State=='available'].ZoneName" `
  --output table
```

You should see something similar to:

```text
--------------------
| Describe...      |
+------------------+
| ap-south-1a      |
| ap-south-1b      |
| ap-south-1c      |
+------------------+
```

We'll use:

```text
ap-south-1a
ap-south-1b
```

---

# Step 2.2 — Create the VPC module

Now we'll start using the `modules` directory we created in Phase 1.

From:

```text
F:\DevOps-Data\Terraform-Project\aws-comunity-terraform\terraform
```

go to the project root:

```powershell
cd ..
```

Then:

```powershell
mkdir modules\vpc
```

Your structure becomes:

```text
aws-comunity-terraform/
│
├── terraform/
│
└── modules/
    └── vpc/
```

---

# Step 2.3 — VPC module variables

Create:

```text
modules\vpc\variables.tf
```

Add:

```hcl
variable "project_name" {
  description = "Project name"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
}

variable "app_subnet_cidrs" {
  description = "CIDR blocks for application private subnets"
  type        = list(string)
}

variable "db_subnet_cidrs" {
  description = "CIDR blocks for database private subnets"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateway"
  type        = bool
  default     = true
}
```

---

# Step 2.4 — Create the VPC

Create:

```text
modules\vpc\main.tf
```

Start with:

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.project_name}-${var.environment}-vpc"
  }
}
```

Our VPC will be:

```text
10.0.0.0/16
```

That gives us:

```text
65,536 IP addresses
```

We won't use all of them immediately.

---

# Step 2.5 — Internet Gateway

Add to the same `main.tf`:

```hcl
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-igw"
  }
}
```

Architecture:

```text
Internet
   |
   v
Internet Gateway
   |
   v
VPC
```

The Internet Gateway allows resources with appropriate public routing/public addressing to communicate with the Internet.

---

# Step 2.6 — Public Subnets

Add:

```hcl
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project_name}-${var.environment}-public-${count.index + 1}"
    Tier = "public"
  }
}
```

Terraform will create:

```text
public-1
10.0.1.0/24
ap-south-1a
```

and:

```text
public-2
10.0.2.0/24
ap-south-1b
```

---

# Step 2.7 — Private Application Subnets

Add:

```hcl
resource "aws_subnet" "app" {
  count = length(var.app_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.app_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.project_name}-${var.environment}-app-${count.index + 1}"
    Tier = "application"
  }
}
```

These become:

```text
app-1
10.0.11.0/24
ap-south-1a
```

and:

```text
app-2
10.0.12.0/24
ap-south-1b
```

Notice:

```hcl
map_public_ip_on_launch
```

isn't enabled.

Our web servers will therefore be private.

---

# Step 2.8 — Private Database Subnets

Add:

```hcl
resource "aws_subnet" "db" {
  count = length(var.db_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.db_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name = "${var.project_name}-${var.environment}-db-${count.index + 1}"
    Tier = "database"
  }
}
```

These become:

```text
db-1
10.0.21.0/24
ap-south-1a
```

and:

```text
db-2
10.0.22.0/24
ap-south-1b
```

---

# Step 2.9 — Public Route Table

Add:

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-public-rt"
  }
}
```

Now add the Internet route:

```hcl
resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.this.id
}
```

This means:

```text
0.0.0.0/0
      |
      v
Internet Gateway
```

---

# Step 2.10 — Associate Public Subnets

Add:

```hcl
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

So:

```text
Public Subnet 1 ──┐
                  ├── Public Route Table
Public Subnet 2 ──┘
```

---

# Step 2.11 — NAT Gateway

Now comes the billable component.

First create an Elastic IP:

```hcl
resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? 1 : 0

  domain = "vpc"

  tags = {
    Name = "${var.project_name}-${var.environment}-nat-eip"
  }
}
```

Then:

```hcl
resource "aws_nat_gateway" "this" {
  count = var.enable_nat_gateway ? 1 : 0

  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id

  tags = {
    Name = "${var.project_name}-${var.environment}-nat"
  }

  depends_on = [
    aws_internet_gateway.this
  ]
}
```

We'll use **one NAT Gateway** for this practice lab.

A production architecture might use one NAT Gateway per AZ for better availability, but that would increase cost.

---

# Step 2.12 — Private Application Route Table

Add:

```hcl
resource "aws_route_table" "app" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-app-rt"
  }
}
```

Now:

```hcl
resource "aws_route" "app_nat" {
  count = var.enable_nat_gateway ? 1 : 0

  route_table_id         = aws_route_table.app.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.this[0].id
}
```

This creates:

```text
Private EC2
    |
    v
Private Route Table
    |
    v
NAT Gateway
    |
    v
Internet Gateway
    |
    v
Internet
```

Notice the difference:

### Public

```text
EC2 → IGW → Internet
```

### Private

```text
EC2 → NAT → IGW → Internet
```

The private EC2 doesn't receive a public IP.

---

# Step 2.13 — Associate Application Subnets

```hcl
resource "aws_route_table_association" "app" {
  count = length(aws_subnet.app)

  subnet_id      = aws_subnet.app[count.index].id
  route_table_id = aws_route_table.app.id
}
```

---

# Step 2.14 — Database Route Table

Create:

```hcl
resource "aws_route_table" "db" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.project_name}-${var.environment}-db-rt"
  }
}
```

Associate the DB subnets:

```hcl
resource "aws_route_table_association" "db" {
  count = length(aws_subnet.db)

  subnet_id      = aws_subnet.db[count.index].id
  route_table_id = aws_route_table.db.id
}
```

### Important

We intentionally don't add:

```text
0.0.0.0/0 → NAT
```

to the database route table.

So the database subnet has no direct Internet route.

That's what we want.

---

# Step 2.15 — VPC Outputs

Create:

```text
modules\vpc\outputs.tf
```

Add:

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "VPC CIDR"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "app_subnet_ids" {
  description = "Application subnet IDs"
  value       = aws_subnet.app[*].id
}

output "db_subnet_ids" {
  description = "Database subnet IDs"
  value       = aws_subnet.db[*].id
}

output "nat_gateway_id" {
  description = "NAT Gateway ID"
  value       = var.enable_nat_gateway ? aws_nat_gateway.this[0].id : null
}
```

---

# Step 2.16 — Connect the VPC Module

Now go to:

```text
terraform/main.tf
```

Replace the current `locals` block with:

```hcl
locals {
  project_name = var.project_name
  environment  = var.environment

  name_prefix = "${local.project_name}-${local.environment}"
}

module "vpc" {
  source = "../modules/vpc"

  project_name = var.project_name
  environment  = var.environment

  vpc_cidr = "10.0.0.0/16"

  availability_zones = [
    "ap-south-1a",
    "ap-south-1b"
  ]

  public_subnet_cidrs = [
    "10.0.1.0/24",
    "10.0.2.0/24"
  ]

  app_subnet_cidrs = [
    "10.0.11.0/24",
    "10.0.12.0/24"
  ]

  db_subnet_cidrs = [
    "10.0.21.0/24",
    "10.0.22.0/24"
  ]

  enable_nat_gateway = true
}
```

---

# Step 2.17 — Add VPC Outputs

Modify:

```text
terraform/outputs.tf
```

to:

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

output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "vpc_cidr" {
  description = "VPC CIDR"
  value       = module.vpc.vpc_cidr
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = module.vpc.public_subnet_ids
}

output "app_subnet_ids" {
  description = "Application subnet IDs"
  value       = module.vpc.app_subnet_ids
}

output "db_subnet_ids" {
  description = "Database subnet IDs"
  value       = module.vpc.db_subnet_ids
}

output "nat_gateway_id" {
  description = "NAT Gateway ID"
  value       = module.vpc.nat_gateway_id
}
```

---

# Step 2.18 — Format everything

From the `terraform` directory:

```powershell
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

---

# Step 2.19 — Initialize again

Because we've introduced a module:

```powershell
terraform init
```

Terraform should detect:

```text
- ../modules/vpc
```

---

# Step 2.20 — VERY IMPORTANT: Plan before Apply

Run:

```powershell
terraform plan
```

You should see resources such as:

```text
Plan: 15 to add, 0 to change, 0 to destroy.
```

The exact number may differ slightly depending on provider behavior.

Look carefully for:

```text
aws_vpc
aws_internet_gateway
aws_subnet
aws_route_table
aws_route
aws_route_table_association
aws_eip
aws_nat_gateway
```

---

# ⚠️ Before `terraform apply`

**Stop at this point and send me your `terraform plan` output.**

I want us to inspect the plan before creating the VPC and especially before creating the **NAT Gateway**, because unlike the Terraform configuration itself, the NAT Gateway can incur AWS charges.

Once the plan is correct, we'll do the apply together and then verify the actual AWS networking using:

```powershell
aws ec2 describe-vpcs
aws ec2 describe-subnets
aws ec2 describe-route-tables
aws ec2 describe-nat-gateways
```

That verification step is important: **we're learning AWS networking, not just learning to make Terraform say "Apply complete."**
