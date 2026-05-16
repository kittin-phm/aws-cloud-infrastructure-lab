# ⚡ AWS Cloud Infrastructure Lab

> A production-grade AWS cloud infrastructure featuring VPC networking, auto-scaling compute, managed database, S3 storage, and a cost-saving Lambda scheduler — all built and debugged hands-on.

---

## 🗺️ Architecture Overview

```
Internet
    │
    ▼
[Internet Gateway: cloud-lab-igw]
    │
    ▼
[ALB: cloud-lab-alb]  ← HTTP :80  (cloud-lab-alb-sg)
    │
    ▼  (private subnets)
[Auto Scaling Group: cloud-lab-asg]
  EC2 t2.micro · Amazon Linux · Apache httpd
  cloud-lab-ec2-sg
    │
    ▼
[RDS MySQL: cloud-lab-db]
  db.t3.micro · no public access
  cloud-lab-rds-sg
```

```
S3: cloud-lab-bucket-kittin          Lambda Scheduler (Terraform)
  Versioning ON                        🟢 9 AM  → ASG/RDS ON
  Lifecycle → Glacier after 90 days    🔴 6 PM  → ASG/RDS OFF
```

**Live URL:** [http://cloud-lab-alb-521128514.ap-southeast-1.elb.amazonaws.com](http://cloud-lab-alb-521128514.ap-southeast-1.elb.amazonaws.com)

---

## 📦 What Was Built

### Phase 1 — VPC & Networking
> Build the private network that isolates all resources. Define IP ranges, split into public (internet-facing) and private (internal-only) subnets across two availability zones, then connect public subnets to the internet via an Internet Gateway and Route Table.

| Resource | Value |
|---|---|
| VPC | `cloud-lab-vpc` · CIDR `10.0.0.0/16` |
| Public Subnet 1 | `cloud-lab-public-1` · `10.0.1.0/24` · ap-southeast-1a |
| Public Subnet 2 | `cloud-lab-public-2` · `10.0.2.0/24` · ap-southeast-1b |
| Private Subnet 1 | `cloud-lab-private-1` · `10.0.3.0/24` · ap-southeast-1a |
| Private Subnet 2 | `cloud-lab-private-2` · `10.0.4.0/24` · ap-southeast-1b |
| Internet Gateway | `cloud-lab-igw` → attached to VPC |
| Route Table | `cloud-lab-public-rt` → `0.0.0.0/0` → IGW → public subnets |

### Phase 2 — Security Groups
> Act as virtual firewalls that control exactly what traffic is allowed in and out of each resource. Traffic flows in one direction only — internet → ALB → EC2 → RDS — so nothing is exposed more than it needs to be.

| Security Group | Rules |
|---|---|
| `cloud-lab-alb-sg` | HTTP 80 + HTTPS 443 from internet |
| `cloud-lab-ec2-sg` | HTTP 80 from `alb-sg` · SSH 22 |
| `cloud-lab-rds-sg` | MySQL 3306 from `ec2-sg` only |

### Phase 3 — Database
> Provision a managed MySQL database inside the private subnets so it is never directly reachable from the internet. Only EC2 instances with the correct security group can connect to it on port 3306.

- **RDS Subnet Group:** `cloud-lab-db-subnet-group` (both private subnets)
- **RDS Instance:** `cloud-lab-db`
  - Engine: MySQL · Class: `db.t3.micro` · Free tier
  - Public access: **disabled**
  - Security group: `cloud-lab-rds-sg`

### Phase 4 — Compute
> Launch EC2 web servers through an Auto Scaling Group so the number of instances automatically grows or shrinks based on CPU load. The Application Load Balancer sits in front and distributes incoming HTTP traffic evenly across healthy instances.

- **Launch Template:** `cloud-lab-lt`
  - AMI: Amazon Linux · Instance type: `t2.micro`
  - Key pair: `cloud-lab-key`
  - Security group: `cloud-lab-ec2-sg`
  - Auto-assign public IP: enabled *(fixed after initial bug)*

- **Auto Scaling Group:** `cloud-lab-asg`
  - Min: 1 · Max: 3 · Desired: 1
  - CPU scaling policy at 50%
  - ALB: `cloud-lab-alb` + target group created during setup

### Phase 5 — Storage
> Store files and static assets in S3 with versioning enabled so every change is recoverable. A lifecycle rule automatically moves older objects to Glacier (cold storage) after 90 days to cut storage costs. CloudFront (CDN) was planned but skipped pending account verification.

- **S3 Bucket:** `cloud-lab-bucket-kittin`
  - Versioning: **enabled**
  - Lifecycle rule: move to **Glacier** after 90 days
- **CloudFront:** skipped (account not yet verified at time of lab)

### Phase 6 — Cost Scheduler (Terraform)
> Automatically shut down EC2 and RDS every evening and restart them each morning using Lambda functions triggered by EventBridge cron rules. The entire scheduler stack is written as Terraform infrastructure-as-code so it can be deployed or destroyed in one command. This saves cost by ensuring resources only run during working hours.

Folder: `cloud-lab-scheduler/`

```
cloud-lab-scheduler/
├── main.tf              # All AWS resources (Lambda, EventBridge, IAM)
├── variables.tf         # Variable declarations
├── terraform.tfvars     # Real ASG + RDS resource names
└── lambdas/
    ├── asg_start.py     # Start EC2 instances via ASG
    ├── asg_stop.py      # Stop EC2 instances via ASG
    ├── rds_start.py     # Start RDS instance
    └── rds_stop.py      # Stop RDS instance
```

**Schedule (Bangkok time, weekdays):**

| Time | Action |
|---|---|
| 🟢 09:00 | ASG desired capacity → 1 · RDS starts |
| 🔴 18:00 | ASG desired capacity → 0 · RDS stops |

**Terraform commands run:**

```bash
terraform init    # ✅ Initialized
terraform plan    # ✅ 16 resources planned
terraform apply   # ✅ 16 resources created
```

---

## Phase 7 — Monitoring 

Set up a CloudWatch dashboard to visualise EC2 and RDS CPU metrics in real time. An alarm watches EC2 CPU utilisation and fires an SNS notification email whenever it exceeds 70% — so any unexpected load spike is caught immediately.

| Resource | Value |
|---|---|
| Dashboard | `cloud-lab-dashboard` |
| EC2 Widget | CPUUtilization · Auto Scaling Group `cloud-lab-asg` |
| RDS Widget | CPUUtilization · DB instance `cloud-lab-db` |
| Alarm | `cloud-lab-cpu-alarm` · triggers when CPU > 70% for 5 minutes |
| Notification | SNS topic `cloud-lab-alerts` · email alert on breach |

## 🛠️ AWS Services Used

| Service | Purpose |
|---|---|
| **VPC** | Isolated network with public/private subnets |
| **EC2** | Web server (Amazon Linux + Apache httpd) |
| **ALB** | Load balancer distributing HTTP traffic |
| **Auto Scaling** | Scale EC2 based on CPU (min 1, max 3) |
| **RDS MySQL** | Managed relational database (private subnet) |
| **S3** | Object storage with versioning + Glacier lifecycle |
| **Lambda** | Serverless functions to start/stop ASG and RDS |
| **EventBridge** | Cron triggers for the Lambda scheduler |
| **IAM** | Roles and policies for Lambda permissions |
| **Terraform** | Infrastructure-as-code for the scheduler stack |
| CloudWatch | CPU dashboards for EC2 and RDS + SNS email alarm at 70% threshold |
| SNS | Email notification topic triggered by CloudWatch alarm |
---

## ✅ Final Result

- 🌐 **Website live** at [http://cloud-lab-alb-521128514.ap-southeast-1.elb.amazonaws.com](http://cloud-lab-alb-521128514.ap-southeast-1.elb.amazonaws.com)
- ⏰ **Auto ON/OFF** — 9 AM to 6 PM weekdays (Bangkok time) to save cost
- 🏗️ **Full production-pattern AWS architecture** running on a free-tier budget
