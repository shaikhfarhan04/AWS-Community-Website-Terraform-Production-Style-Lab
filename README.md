Absolutely. We can turn this into a **complete Terraform + AWS hands-on project** based on your earlier AWS Community Registration application, but make the infrastructure more realistic and suitable for DevOps interview practice.

![Image](https://images.openai.com/static-rsc-4/KyP_KphsMNT2cCsPYygjZfYkBoj2kMv68FUP3Di0X1U7JMVVTVStiw55zg2z8MbXgUt1qpQIN9gi9IiXr-AgcwLTwmokM__887YS6dgHFFiSxMlW2wmlRTkonbTj331XLxStF6SdbngUgz4t5QzQdy8F8qqcYK7GbKOu59W6s6KIC4hG6SSrPurVVPbxzTOm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/vXHEpsN1vL1kmrh3Q3vgWL31WLAh8-8spiEO8pZwiGo7uEOOajiRRNeLyyY33EhSXc6fdkpHBmZQ-fPnkAyi4698zSdH_X7R1-YlZT_RceZhH59v8miL3d3zpHXFJP1pGe1_K1E7Ry5oUlwPybYouI2d07mdqt79LSiQpoQSuH8-TsTGrafykQ0LD-PWA3o-?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/V1aQibsRVzYhrBbDBancuK-PGlInPlbyoI_OTQRNQobhY92jRk3YZ93z1a6i5yZUJ4ZFdebo5FoIhAqX6zDFgY1kx2qyPDUf0E5R41J1yv61jZ36oQ-NqKSULtkpnPiFasbSyRs49b-73KsmzmEXaU_v_ZPljbks6LNv7jzRkpmqOf-119ZZIKPh6Vpytimq?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/-oWOdt3ULjlr-C7t9adyGk3YDRdGIS5s3VHtvvfCoIOCCWKp7l1A9IOD393X3eqbowO51yrYZXFUsat35EJUXquOQpy23wdoGPrP1nFVfgs6eszccd_Kziuelb92Z6zppUNYUrED9Egk7nPtGHsjpztGtRTOYe-P_7LU9RWBRs68QmpJvZnqbYXhhnscq0-6?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/mp9yauME6kaSzKLiQLiPd4ERfOBCB4adKwMbqYjseVB5YZ4ELzETrJ4HIyQ6gVeOVfBgJnmxPGoA3uu04M5SQMjuUVz9We2jAzGlypfGHhvAmBpLIWBT-oyybBVRcfv-GCWRtFeH_v9-3vYDLwgW1w4-tRxbgPFjQZDbDyhLpsmNeyOHjqgY8apMjcSodf8X?purpose=fullsize)

# AWS Community Website — Terraform Production-Style Lab

### Project goal

Build an **AWS Community website** with:

* 🌐 Public Application Load Balancer
* 🖥️ **2 Web/Application Servers**
* 🗄️ Private **RDS MySQL Database**
* 🔐 VPC with public/private subnets
* 🌍 Internet Gateway
* 🚪 NAT Gateway
* ⚖️ Application Load Balancer
* 👤 IAM roles
* 📦 S3 bucket
* 📊 CloudWatch monitoring
* 🔥 Security Groups
* 🔑 AWS Secrets Manager
* 🏗️ Terraform
* 🐧 Amazon Linux 2023
* 🐳 Docker — later phase
* ☸️ Kubernetes/EKS — optional advanced phase

We'll initially keep it **cost-conscious**, using `t3.micro` where practical and a single NAT Gateway.

---

# 1. Final Architecture

```text
                         INTERNET
                            |
                            |
                     +---------------+
                     | Internet      |
                     | Gateway       |
                     +-------+-------+
                             |
                             |
                    PUBLIC SUBNETS
                 +-----------+-----------+
                 |                       |
        +--------v--------+     +--------v--------+
        | Public Subnet 1 |     | Public Subnet 2 |
        | AZ-1            |     | AZ-2            |
        |                 |     |                 |
        |      ALB        |     |      ALB        |
        +--------+--------+     +--------+--------+
                 \                 /
                  \               /
                   \             /
                    +-----+-----+
                          |
                          |
                 PRIVATE APP SUBNETS
              +-----------+-----------+
              |                       |
       +------v------+         +------v------+
       | Web Server 1|         | Web Server 2|
       | EC2         |         | EC2         |
       | t3.micro    |         | t3.micro    |
       +------+------+\         /+------+------+
              |       \       /       |
              |        \     /        |
              |         \   /         |
              |          \ /          |
              |       Application     |
              |         Traffic       |
              |                       |
              +-----------+-----------+
                          |
                          |
                  PRIVATE DB SUBNETS
                    +-----+-----+
                          |
                    +-----v-----+
                    | RDS MySQL |
                    |           |
                    | Community |
                    | Database  |
                    +-----------+

             PRIVATE SUBNETS
                    |
              +-----v------+
              | NAT Gateway|
              +------------+
                    |
             Internet Gateway
```

---

# 2. What We Will Build

We'll divide the project into **10 phases** so you actually learn Terraform rather than simply copying a huge Terraform configuration.

| Phase | Topic                          | Main Learning                  |
| ----- | ------------------------------ | ------------------------------ |
| 1     | Project + Terraform setup      | Terraform fundamentals         |
| 2     | VPC                            | Networking                     |
| 3     | Security Groups                | AWS security                   |
| 4     | EC2 Web Servers                | Compute                        |
| 5     | ALB                            | Load balancing                 |
| 6     | RDS MySQL                      | Database                       |
| 7     | IAM + Secrets Manager          | Security                       |
| 8     | S3 + CloudWatch                | Storage/Monitoring             |
| 9     | Website deployment             | Application deployment         |
| 10    | Terraform production practices | Modules, remote state, outputs |

---

# 3. Target AWS Architecture

We'll use:

### Region

For this lab, use:

```text
ap-south-1
```

Mumbai is convenient for your existing AWS practice.

---

## Availability Zones

We'll use two AZs:

```text
ap-south-1a
ap-south-1b
```

---

# 4. VPC Design

We'll create:

```text
VPC
10.0.0.0/16
```

### Public subnets

```text
10.0.1.0/24    → AZ-a
10.0.2.0/24    → AZ-b
```

Used for:

```text
ALB
NAT Gateway
```

### Private application subnets

```text
10.0.11.0/24   → AZ-a
10.0.12.0/24   → AZ-b
```

Used for:

```text
Web Server 1
Web Server 2
```

### Private database subnets

```text
10.0.21.0/24   → AZ-a
10.0.22.0/24   → AZ-b
```

Used for:

```text
RDS MySQL
```

So we have:

```text
                    VPC
                 10.0.0.0/16
                       |
       +---------------+---------------+
       |               |               |
       v               v               v

     PUBLIC          APP             DATABASE
     SUBNETS         SUBNETS         SUBNETS

  10.0.1.0/24    10.0.11.0/24    10.0.21.0/24
  10.0.2.0/24    10.0.12.0/24    10.0.22.0/24
```

This is much better practice than putting everything into one subnet.

---

# 5. Security Architecture

We'll deliberately avoid opening everything to the Internet.

### Internet → ALB

```text
Internet
   |
   | TCP 80
   v
ALB
```

ALB Security Group:

```text
Inbound:
80  ← 0.0.0.0/0
443 ← 0.0.0.0/0   [later]
```

---

### ALB → Web Servers

Web server Security Group:

```text
Inbound:
80 ← ALB Security Group
```

Not:

```text
80 ← 0.0.0.0/0
```

This is important.

---

### Web Servers → RDS

RDS Security Group:

```text
Inbound:
3306 ← Web Server Security Group
```

Therefore:

```text
Internet
    |
    v
   ALB
    |
    v
Web Server 1
Web Server 2
    |
    v
RDS MySQL
```

The database is **never publicly accessible**.

---

# 6. AWS Services

The lab will eventually use:

### Core

```text
Amazon VPC
EC2
ALB
RDS MySQL
IAM
S3
CloudWatch
Secrets Manager
```

### Terraform

```text
Terraform
AWS Provider
Terraform Modules
Remote State
S3 Backend
```

### Application

```text
Python
Flask
MySQL
Nginx
Gunicorn
```

### Later DevOps

```text
Git
GitHub
Docker
ECR
Jenkins/GitHub Actions
```

---

# 7. AWS Community Website

We'll make the application realistic.

## Website

```text
AWS Community
```

Pages:

```text
/
├── Home
├── About
├── Events
├── Members
└── Register
```

Registration form:

```text
Name
Email
Phone
City
AWS Experience
Interested Services
```

Example:

```text
Farhan
farhan@example.com
Pune
3 Years
EC2, VPC, Terraform, Kubernetes
```

Data will be stored in:

```text
RDS MySQL
```

---

# 8. Application Flow

```text
User
 |
 | HTTP
 v
ALB
 |
 +-------------------+
 |                   |
 v                   v
Web Server 1      Web Server 2
 |
 +---------+---------+
           |
           v
        RDS MySQL
```

The ALB automatically distributes traffic:

```text
Request 1 → Web Server 1
Request 2 → Web Server 2
Request 3 → Web Server 1
Request 4 → Web Server 2
```

---

# 9. Terraform Project Structure

I recommend this structure rather than putting everything into one `main.tf`.

```text
aws-community-terraform/
│
├── terraform/
│   │
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── providers.tf
│   ├── versions.tf
│   │
│   ├── terraform.tfvars
│   ├── terraform.tfvars.example
│   │
│   └── backend.tf
│
├── modules/
│   │
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── security/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── ec2/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── alb/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── rds/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   ├── s3/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   │
│   └── monitoring/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── app/
│   │
│   ├── app.py
│   ├── requirements.txt
│   ├── templates/
│   │   ├── index.html
│   │   ├── register.html
│   │   └── success.html
│   │
│   └── static/
│
├── scripts/
│   ├── web-server.sh
│   └── deploy.sh
│
├── docs/
│   ├── architecture.md
│   ├── deployment.md
│   └── troubleshooting.md
│
├── .gitignore
└── README.md
```

This gives you excellent Terraform practice.

---

# 10. Phase 1 — Terraform Foundation

We'll start with:

```text
AWS Provider
Region
Terraform version
Variables
Outputs
```

### `versions.tf`

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

---

### `providers.tf`

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "AWS-Community"
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  }
}
```

---

### `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "aws-community"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}
```

---

# 11. Phase 2 — VPC

Terraform will create:

```text
VPC
 |
 +-- Internet Gateway
 |
 +-- Public Subnet A
 |
 +-- Public Subnet B
 |
 +-- Private App Subnet A
 |
 +-- Private App Subnet B
 |
 +-- Private DB Subnet A
 |
 +-- Private DB Subnet B
 |
 +-- Public Route Table
 |
 +-- Private Route Table
 |
 +-- DB Route Table
 |
 +-- NAT Gateway
```

The important learning objective is understanding:

```text
Route Table
    ↓
Subnet
    ↓
Internet Gateway / NAT Gateway
```

---

# 12. Phase 3 — EC2 Web Servers

We'll create exactly:

```text
web-server-1
web-server-2
```

Initially:

```text
Amazon Linux 2023
t3.micro
```

They will live in:

```text
Private App Subnet A
Private App Subnet B
```

No public IP.

---

# 13. How will we manage the private servers?

Instead of exposing SSH publicly, we'll use **AWS Systems Manager Session Manager**.

Architecture:

```text
Your PC
   |
   v
AWS Systems Manager
   |
   +----------------+
   |                |
   v                v
EC2-1            EC2-2
```

This is a very useful DevOps skill.

We'll give the EC2 instances:

```text
AmazonSSMManagedInstanceCore
```

IAM role.

---

# 14. Phase 4 — Application Load Balancer

We'll create:

```text
aws_lb
```

Then:

```text
Target Group
       |
       +---- EC2-1
       |
       +---- EC2-2
```

ALB listener:

```text
HTTP : 80
```

Traffic:

```text
Client
  |
  v
ALB :80
  |
  +--------+
  |        |
  v        v
EC2-1    EC2-2
```

---

# 15. Health Checks

This is important for your interview.

We'll configure:

```text
Health check path:

/health
```

Application returns:

```json
{
  "status": "healthy"
}
```

If:

```text
EC2-1 → unhealthy
```

ALB stops sending traffic to it.

```text
                ALB
                 |
          +------+------+
          |             |
       HEALTHY       UNHEALTHY
          |             X
          v
        EC2-1         EC2-2
```

---

# 16. Phase 5 — RDS MySQL

We'll create:

```text
Amazon RDS MySQL
```

Inside:

```text
Private DB Subnet A
Private DB Subnet B
```

Example:

```text
DB Name:

awscommunity
```

Tables:

```text
members
events
users
```

Example:

```sql
CREATE TABLE members (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(150),
    city VARCHAR(100),
    aws_experience VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

# 17. Database Security

The RDS instance will have:

```text
Publicly accessible = false
```

And:

```text
Port 3306
```

will only accept connections from:

```text
Web Server Security Group
```

Not:

```text
0.0.0.0/0
```

This is an important AWS security principle.

---

# 18. Phase 6 — Secrets Manager

Instead of putting:

```hcl
db_password = "MyPassword123"
```

inside Terraform files, we'll eventually use:

```text
AWS Secrets Manager
```

Example secret:

```text
aws-community/dev/database
```

Containing:

```json
{
  "username": "community_admin",
  "password": "********",
  "host": "aws-community.xxxxxx.ap-south-1.rds.amazonaws.com",
  "database": "awscommunity"
}
```

The application retrieves the secret through IAM.

---

# 19. Phase 7 — S3

We'll create an S3 bucket for:

```text
AWS Community
Application Assets
```

Potential contents:

```text
uploads/
event-images/
member-documents/
```

We'll enable:

```text
Versioning
Encryption
Block Public Access
```

This also gives you S3 Terraform practice.

---

# 20. Phase 8 — CloudWatch

We'll monitor:

```text
EC2 CPU
EC2 status
ALB requests
ALB 5xx
Target health
RDS CPU
RDS connections
RDS storage
```

Eventually:

```text
CloudWatch Alarm
       |
       v
SNS
       |
       v
Email
```

---

# 21. Phase 9 — Application Deployment

Initially we'll deploy:

```text
Flask
+
Gunicorn
+
Nginx
```

on both EC2 instances.

Architecture:

```text
ALB
 |
 v
Nginx
 |
 v
Gunicorn
 |
 v
Flask
 |
 v
RDS
```

---

# 22. Website

The home page could look like:

```text
+------------------------------------------------+
|              AWS COMMUNITY                     |
+------------------------------------------------+
| Home | About | Events | Members | Register     |
+------------------------------------------------+

        Welcome to AWS Community

Learn AWS
Practice Cloud
Build DevOps Projects
Share Knowledge

       [ Join Community ]

-------------------------------------------------

Latest Events

AWS Beginners Workshop
Terraform Hands-on Lab
Kubernetes Community Meetup

-------------------------------------------------

           AWS Community India
```

Registration:

```text
+--------------------------------+
|       Join AWS Community       |
+--------------------------------+

Name:              [____________]

Email:             [____________]

City:              [____________]

AWS Experience:    [____________]

Interested In:

[ ] EC2
[ ] VPC
[ ] Terraform
[ ] Docker
[ ] Kubernetes
[ ] DevOps

             [ REGISTER ]
```

---

# 23. Phase 10 — Advanced Terraform

After the infrastructure works, we'll refactor it into:

```text
modules/
```

and learn:

### Terraform modules

```text
module "vpc" {
   source = "../modules/vpc"
}
```

### Variables

```text
variables.tf
```

### Outputs

```text
outputs.tf
```

### Locals

```text
locals.tf
```

### Data sources

```text
data.aws_ami
```

### Dependencies

```text
depends_on
```

### Terraform lifecycle

```text
create_before_destroy
prevent_destroy
ignore_changes
```

---

# 24. Remote Terraform State

At the end we'll move from:

```text
terraform.tfstate
```

on your laptop to:

```text
S3
 |
 +-- Terraform State
```

and use:

```text
S3 Backend
```

with state locking supported by the current Terraform/AWS approach we choose during the lab.

This gives you a very realistic DevOps workflow.

---

# 25. Git Workflow

We'll also practice:

```text
GitHub
   |
   v
feature branch
   |
   v
Pull Request
   |
   v
main
```

Repository:

```text
aws-community-terraform
```

Example branches:

```text
main
develop
feature/vpc
feature/ec2
feature/alb
feature/rds
feature/monitoring
```

---

# 26. Terraform Commands We'll Practice

You will repeatedly use:

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

```powershell
terraform apply
```

```powershell
terraform output
```

```powershell
terraform show
```

```powershell
terraform state list
```

```powershell
terraform destroy
```

And later:

```powershell
terraform import
```

```powershell
terraform taint
```

```powershell
terraform graph
```

---

# 27. Important Troubleshooting Practice

I'll deliberately make this a **troubleshooting lab**, not just a deployment exercise.

We'll practice problems such as:

### Problem 1

```text
ALB → 503
```

Find:

```text
Target health
Security Group
Application
Port
```

### Problem 2

```text
EC2 cannot reach Internet
```

Investigate:

```text
Route Table
NAT Gateway
Subnet
Internet Gateway
```

### Problem 3

```text
Application cannot connect to RDS
```

Check:

```text
RDS SG
EC2 SG
Port 3306
DNS
Credentials
```

### Problem 4

```text
ALB only sends traffic to one server
```

Investigate:

```text
Target Group
Health Checks
Availability
```

### Problem 5

```text
Terraform wants to recreate RDS
```

Investigate:

```text
terraform plan
lifecycle
state
configuration
```

These are excellent interview scenarios.

---

# 28. Final Project Architecture

At completion:

```text
                         USERS
                           |
                           v
                    +-------------+
                    |     ALB     |
                    |   HTTP/HTTPS|
                    +------+------+
                           |
                 +---------+---------+
                 |                   |
                 v                   v
          +-------------+     +-------------+
          | Web Server 1|     | Web Server 2|
          | EC2         |     | EC2         |
          | Private     |     | Private     |
          +------+------+     +------+------+
                 |                   |
                 +---------+---------+
                           |
                           v
                    +-------------+
                    | RDS MySQL   |
                    | Private     |
                    +-------------+

Additional services:

        EC2
         |
         +---- IAM Role
         |
         +---- SSM
         |
         +---- CloudWatch
         |
         +---- Secrets Manager
         |
         +---- S3

Networking:

        VPC
         |
         +---- Public Subnets
         |
         +---- Private App Subnets
         |
         +---- Private DB Subnets
         |
         +---- IGW
         |
         +---- NAT Gateway
```

---

# 29. What You Will Be Able to Say in an Interview

After completing this project, you can explain:

> "I provisioned a highly available AWS three-tier application architecture using Terraform. The application uses an internet-facing Application Load Balancer across two Availability Zones, two private EC2 application servers, and an Amazon RDS MySQL database deployed in private database subnets. I implemented security groups using least-privilege access, IAM roles for EC2, Systems Manager for administration without public SSH, Secrets Manager for database credentials, S3 for application assets, and CloudWatch for monitoring."

That's a **strong DevOps project explanation**.

---

# 30. Our Learning Approach

I recommend we **do not create the entire infrastructure in one shot**.

We'll build it like this:

```text
PHASE 1
Terraform + AWS setup
        ↓
PHASE 2
VPC + 6 subnets
        ↓
PHASE 3
Security Groups + IAM
        ↓
PHASE 4
2 EC2 Web Servers
        ↓
PHASE 5
ALB + Target Group
        ↓
PHASE 6
RDS MySQL
        ↓
PHASE 7
Flask AWS Community Website
        ↓
PHASE 8
Secrets Manager + S3
        ↓
PHASE 9
CloudWatch + Monitoring
        ↓
PHASE 10
Terraform Modules
        ↓
PHASE 11
Remote State
        ↓
PHASE 12
Docker + ECR
        ↓
PHASE 13
CI/CD
```

**Phase 1 should be our starting point:** create the Windows/PowerShell project structure, configure the AWS provider, create the Terraform variables, verify AWS credentials, and run `terraform init`, `validate`, and `plan` **without creating expensive AWS resources yet**.
