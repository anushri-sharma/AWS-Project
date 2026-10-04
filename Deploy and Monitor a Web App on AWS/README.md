# AWS Web App Deployment & Monitoring Project

![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)
![Nginx](https://img.shields.io/badge/nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![Status](https://img.shields.io/badge/status-complete-brightgreen?style=for-the-badge)

A hands-on cloud infrastructure project demonstrating core AWS skills: networking (VPC), compute (EC2), storage (S3), databases (RDS), monitoring (CloudWatch/SNS), and IAM security — plus real troubleshooting scenarios modeled on cloud support tickets.

---

## 📐 Architecture

![Architecture Diagram](./architecture-diagram.png)

**Components:**
- **VPC** with public and private subnets across an Availability Zone
- **EC2** instance (public subnet) running Nginx, reachable via HTTP/HTTPS
- **RDS** MySQL instance (private subnet), only reachable from EC2's security group
- **S3** bucket for static assets with mixed public/private access policies
- **CloudWatch + SNS** for CPU monitoring and email alerting
- **IAM** users/roles scoped with least-privilege policies

```
                          ┌─────────────────────┐
                          │      Internet        │
                          └──────────┬───────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   Internet Gateway    │
                          └──────────┬───────────┘
                                     │
                    ┌────────────────▼────────────────┐
                    │              VPC                 │
                    │  ┌────────────────────────────┐  │
                    │  │  Public Subnet (10.0.1.0/24)│  │
                    │  │   ┌────────────────────┐   │  │
                    │  │   │  EC2 (Nginx/Flask) │   │  │
                    │  │   └──────────┬─────────┘   │  │
                    │  └──────────────┼─────────────┘  │
                    │                 │                 │
                    │  ┌──────────────▼─────────────┐  │
                    │  │ Private Subnet (10.0.2.0/24)│  │
                    │  │   ┌────────────────────┐   │  │
                    │  │   │   RDS (MySQL)      │   │  │
                    │  │   └────────────────────┘   │  │
                    │  └─────────────────────────────┘  │
                    └────────────────────────────────────┘
                                     │
                          ┌──────────▼───────────┐
                          │   S3 Bucket (assets)  │
                          └───────────────────────┘
                                     │
                          ┌──────────▼───────────┐
                          │ CloudWatch → SNS      │
                          │ (CPU alarm → email)   │
                          └───────────────────────┘
```

---

## 🛠️ Tech Stack

| Category | Service/Tool |
|---|---|
| Compute | Amazon EC2 (t2.micro) |
| Web Server | Nginx |
| Networking | Amazon VPC, Subnets, Route Tables, IGW |
| Storage | Amazon S3 |
| Database | Amazon RDS (MySQL) |
| Monitoring | CloudWatch, SNS |
| Security | IAM, Security Groups, NACLs |

---

## 🚀 What This Project Covers

- [x] Custom VPC with public/private subnet segmentation
- [x] IAM users and policies following least-privilege principles
- [x] EC2 instance deployment with a live web server
- [x] S3 bucket with mixed public/private access configuration
- [x] RDS database isolated in a private subnet
- [x] CloudWatch alarm + SNS email notification on high CPU
- [x] Deliberate misconfiguration + troubleshooting exercises (see below)

---

## 🔧 Setup Steps

### 1. IAM & Account Setup
- Enabled MFA on root account
- Created an admin IAM user for daily operations (no root usage)
- Created a scoped `app-deploy` IAM user limited to EC2, S3, and RDS actions

### 2. VPC & Networking
- Created VPC (`10.0.0.0/16`) with one public subnet (`10.0.1.0/24`) and one private subnet (`10.0.2.0/24`)
- Attached an Internet Gateway and configured route tables accordingly
- Public subnet routes `0.0.0.0/0` → IGW; private subnet has no internet route

### 3. EC2 & Web Server
- Launched a `t2.micro` EC2 instance in the public subnet
- Installed and configured Nginx
- Security group restricts SSH to my IP only; allows HTTP/HTTPS from anywhere

```bash
sudo yum update -y
sudo yum install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

### 4. S3 Storage
- Created a bucket for static assets
- Configured a bucket policy allowing public read only on a specific prefix, while keeping the rest of the bucket private

### 5. RDS Database
- Deployed a MySQL instance in the private subnet
- Security group only allows inbound `3306` from the EC2 instance's security group (not from any IP)

### 6. Monitoring & Alerting
- Created a CloudWatch alarm on `CPUUtilization > 80%` for 5 minutes
- Linked the alarm to an SNS topic subscribed to my email
- Verified the alert by stress-testing the instance:

```bash
sudo yum install -y stress
stress --cpu 2 --timeout 300
```

---

## 🐞 Troubleshooting Log

Simulated real support-ticket scenarios by deliberately breaking the environment, then diagnosing and fixing each issue. Full write-ups in [`troubleshooting-log.md`](./troubleshooting-log.md).

| # | Symptom | Root Cause | Resolution |
|---|---|---|---|
| 1 | Site unreachable over HTTP | Port 80 removed from security group | Re-added inbound rule for port 80 |
| 2 | S3 "Access Denied" on public assets | Overly restrictive bucket policy | Corrected policy to scope public read to the right prefix |
| 3 | App couldn't connect to RDS | RDS security group referenced wrong source SG | Updated inbound rule to reference EC2's security group |
| 4 | SSH connection timing out | Network ACL blocking inbound traffic | Identified NACL vs. security group behavior (stateless vs. stateful) and corrected NACL rule |
| 5 | CloudWatch alarm fired but no email received | SNS subscription never confirmed | Resent confirmation, confirmed subscription, verified alert delivery |

---

## 📁 Repository Structure

```
aws-web-app-project/
├── README.md
├── architecture-diagram.png
├── troubleshooting-log.md
├── scripts/
│   ├── ec2-setup.sh
│   └── app.py
└── terraform/          # stretch goal: infrastructure as code
```

---

## 📚 Key Takeaways

- Practiced designing network segmentation with public/private subnets
- Learned to scope IAM and security group permissions tightly rather than defaulting to broad access
- Built muscle memory for diagnosing the most common categories of cloud support tickets: connectivity, access, and alerting failures
- Understood the practical difference between stateful (security groups) and stateless (NACLs) network controls

---

## 🔜 Next Steps

- [ ] Add an Application Load Balancer + Auto Scaling Group
- [ ] Automate infrastructure with Terraform
- [ ] Add a CI/CD pipeline (GitHub Actions) for automated deployment

---

## 📬 Contact

**[Your Name]**
[LinkedIn](#) · [GitHub](#) · [Email](#)
