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

**Live URL:** `http://cloud-lab-alb-521128514.ap-southeast-1.elb.amazonaws.com`

---

## 📦 What Was Built

### Phase 1 — VPC & Networking

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

| Security Group | Rules |
|---|---|
| `cloud-lab-alb-sg` | HTTP 80 + HTTPS 443 from internet |
| `cloud-lab-ec2-sg` | HTTP 80 from `alb-sg` · SSH 22 |
| `cloud-lab-rds-sg` | MySQL 3306 from `ec2-sg` only |

### Phase 3 — Database

- **RDS Subnet Group:** `cloud-lab-db-subnet-group` (both private subnets)
- **RDS Instance:** `cloud-lab-db`
  - Engine: MySQL · Class: `db.t3.micro` · Free tier
  - Public access: **disabled**
  - Security group: `cloud-lab-rds-sg`

### Phase 4 — Compute

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

- **S3 Bucket:** `cloud-lab-bucket-kittin`
  - Versioning: **enabled**
  - Lifecycle rule: move to **Glacier** after 90 days
- **CloudFront:** skipped (account not yet verified at time of lab)

### Phase 6 — Cost Scheduler (Terraform)

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

## 🐛 Bugs Encountered & Fixed

| # | Bug | Fix |
|---|---|---|
| 1 | VPC created with `/24` CIDR (too small) | Deleted and recreated with `/16` |
| 2 | EC2 had no public IP | Enabled auto-assign public IP in launch template → instance refresh |
| 3 | Browser showed `ERR_CONNECTION_REFUSED` | Installed Apache (`httpd`) on EC2 |
| 4 | Target group showed **Unhealthy** | Fixed after web server was installed |
| 5 | Browser still not loading | Used `http://` instead of `https://` |
| 6 | `echo` failed due to `!` character | Removed `!` from HTML string (bash history expansion) |
| 7 | Semicolons `;` in `main.tf` invalid | Rewrote entire Terraform file correctly |
| 8 | CloudFront distribution failed | Account not yet verified — skipped for now |

---

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

---

## ✅ Final Result

- 🌐 **Website live** at `http://cloud-lab-alb-521128514.ap-southeast-1.elb.amazonaws.com`
- ⏰ **Auto ON/OFF** — 9 AM to 6 PM weekdays (Bangkok time) to save cost
- 🏗️ **Full production-pattern AWS architecture** running on a free-tier budget

---

## 👤 Author

**Kittin Phummarawong** · [@kittin-phm](https://github.com/kittin-phm)

*Built as a hands-on AWS cloud infrastructure lab — from VPC to auto-scaling to cost automation.*
