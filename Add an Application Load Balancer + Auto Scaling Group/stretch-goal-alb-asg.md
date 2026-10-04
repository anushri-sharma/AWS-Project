# Stretch Goal: Application Load Balancer + Auto Scaling Group

This extends the base [AWS Web App Deployment & Monitoring Project](./README.md) by replacing the single EC2 instance with a resilient, self-healing, auto-scaling fleet behind a load balancer.

**Why this matters:** a single EC2 instance is a single point of failure. This stretch goal demonstrates production-grade thinking — the infrastructure can survive an instance failure and scale automatically under load, without manual intervention.

---

## 📐 Updated Architecture

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
                    │  ┌─────────────────────────────┐ │
                    │  │   Application Load Balancer  │ │
                    │  │     (spans 2 public subnets) │ │
                    │  └───────────────┬─────────────┘ │
                    │                  │                 │
                    │   ┌──────────────┼──────────────┐  │
                    │   │              │              │  │
                    │  ┌▼─────────┐  ┌─▼────────┐     │  │
                    │  │  Public   │  │  Public   │    │  │
                    │  │ Subnet AZ1│  │ Subnet AZ2│    │  │
                    │  │  ┌─────┐  │  │  ┌─────┐  │    │  │
                    │  │  │ EC2 │  │  │  │ EC2 │  │    │  │
                    │  │  └─────┘  │  │  └─────┘  │    │  │
                    │  └───────────┘  └───────────┘    │  │
                    │     (Auto Scaling Group: 2–4)      │
                    │                                     │
                    │  ┌─────────────────────────────┐  │
                    │  │ Private Subnet (RDS, MySQL)  │  │
                    │  └─────────────────────────────┘  │
                    └────────────────────────────────────┘
```

---

## 🛠️ Additional Tech Stack

| Category | Service/Tool |
|---|---|
| Load Balancing | Application Load Balancer (ALB) |
| Scaling | Auto Scaling Group (ASG) |
| Instance Config | Launch Template + user-data bootstrap script |
| Health Checks | ALB Target Group health checks |

---

## 🚀 What This Adds

- [x] Second public subnet in a different Availability Zone (ALB requires 2+ AZs)
- [x] Launch Template that bootstraps Nginx automatically via user-data
- [x] Target Group with health checks
- [x] Application Load Balancer distributing traffic across AZs
- [x] Auto Scaling Group (min 2, max 4) with CPU-based target tracking
- [x] Revised security group chain: internet → ALB → EC2 (no more direct public access to instances)

---

## 🔧 Setup Steps

### 1. Bootstrap script (Launch Template user-data)

```bash
#!/bin/bash
yum update -y
yum install -y nginx
systemctl start nginx
systemctl enable nginx
echo "<h1>Served by $(hostname -f)</h1>" > /usr/share/nginx/html/index.html
```

The hostname output makes it easy to visually confirm requests are being distributed across instances.

### 2. Second public subnet

```
- Add public subnet #2: 10.0.3.0/24 in a different AZ than subnet #1
- Associate it with the same route table (0.0.0.0/0 → IGW)
```

### 3. Launch Template

```
EC2 → Launch Templates → Create
- AMI: Amazon Linux 2023
- Instance type: t2.micro
- Security group: new "web-sg" — allow port 80 from alb-sg ONLY
- User data: bootstrap script above
```

### 4. Target Group

```
EC2 → Target Groups → Create
- Target type: Instances
- Protocol: HTTP, port 80
- Health check path: /
- VPC: existing project VPC
```

### 5. Application Load Balancer

```
EC2 → Load Balancers → Create → Application Load Balancer
- Scheme: internet-facing
- Subnets: both public subnets (both AZs)
- Security group: new "alb-sg" — allow 80/443 from 0.0.0.0/0
- Listener: HTTP:80 → forward to target group
```

### 6. Auto Scaling Group

```
EC2 → Auto Scaling Groups → Create
- Launch template: from step 3
- Subnets: both public subnets
- Attach to existing load balancer → select target group
- Desired capacity: 2 | Min: 2 | Max: 4
- Scaling policy: target tracking, ~50% average CPU
```

### 7. Security group chain (critical detail)

```
alb-sg    : inbound 80/443 from 0.0.0.0/0
web-sg    : inbound 80 from alb-sg only (not from the internet directly)
```

This is a meaningful architectural shift worth calling out in interviews: the EC2 instances are no longer directly internet-facing — the ALB is the only entry point.

### 8. Decommission the original standalone instance

The original single EC2 instance from the base project is now redundant since the ASG manages instance lifecycle. Terminate it once the ASG is healthy.

### 9. Verify traffic distribution

```bash
for i in {1..10}; do curl -s http://<alb-dns-name> | grep hostname; done
```

You should see requests landing on different instance hostnames across runs.

---

## 🐞 Additional Troubleshooting Scenarios

| # | Symptom | Root Cause | Resolution |
|---|---|---|---|
| 1 | ALB shows targets as "unhealthy" | Health check path returns non-200, or web-sg doesn't allow traffic from alb-sg | Confirm health check path is valid; fix security group reference |
| 2 | All instances land in one AZ | ASG subnet config missing the second AZ | Add both public subnets to ASG settings |
| 3 | Scaling never triggers under load | Target tracking policy misconfigured, or metric/cooldown delay | Review scaling policy metric, threshold, and cooldown period |
| 4 | ALB DNS resolves but times out | alb-sg missing inbound rule, or listener not forwarding to target group | Verify alb-sg inbound rules and listener → target group association |

---

## 📚 Key Takeaways

- Learned the difference between a single point of failure and a self-healing, multi-AZ architecture
- Practiced the "narrow the blast radius" security pattern: instances only accept traffic from the ALB, not the open internet
- Understood how Launch Templates + user-data enable repeatable, automated instance provisioning
- Built comfort with the full request path: Internet → IGW → ALB → Target Group → EC2 instance

---

## 🔜 Next Steps

- [ ] Automate this entire stack with Terraform
- [ ] Add HTTPS via ACM certificate on the ALB listener
- [ ] Add a CI/CD pipeline that updates the Launch Template and triggers an instance refresh
