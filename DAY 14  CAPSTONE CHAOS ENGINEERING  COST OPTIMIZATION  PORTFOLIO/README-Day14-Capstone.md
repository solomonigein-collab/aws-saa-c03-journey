# Day 14 Capstone - Self-Healing ASG Stack - Solomon
## Chaos Engineering + Cost Optimization + Portfolio

![Architecture](day14_self_healing_asg_stack_diagram.png)

### 🎯 What I Built (Days 5-14)

**Production-Ready Auto Scaling Stack:**
- **ALB:** `day11-alb-asg-1979065308` (Application Load Balancer) - Internet facing, health checks 3/3 Healthy
- **Target Group:** `day11-tg-asg` Port 80 HTTP
- **ASG:** `day5-asg` Min 1 Max 5 Desired 1 - Self-healing across AZs
- **Launch Template:** `day5-lt-golden` Version 3 `v3-Day13-Apache-v2` - Golden AMI pattern with Apache
- **Instance Proof:** `ip-172-31-12-57` `t3.micro` Apache serving `Hello from Day13 - v3`

### 🔥 Chaos Engineering - Self-Healing Proof

**Test:** Manually terminated instance `i-0b11b05020b8d2046`
- **ASG Activity:** Detected Unhealthy → Terminating → Launching new from Warm Pool
- **Recovery Time:** <60 seconds
- **Result:** New instance `i-0259bac23aa433340` `3/3 Checks Passed` - Zero Downtime, ALB continued serving traffic
- **Proof:** ALB DNS still returned `ip-172-31-12-57` equivalent page during failover

### ⚡ Advanced Scaling - All 3 Types

1. **Target Tracking:** CPU 50% - Auto scale on load
2. **Predictive Scaling:** `cpu-predictive-50` - ML forecast based on history (Status: Need more data initially, normal)
3. **Scheduled Scaling:** 
   - `scale-out-8am` Cron `0 8 * * *` Desired 2 (Morning peak)
   - `scale-in-8pm` Cron `0 20 * * *` Desired 1 (Night save)

### ❄️ Warm Pool - Fast Scale

- **Config:** 4 instances `Stopped` `Reuse on scale in` `Max Group Prepared Capacity 4`
- **Benefit:** Saves 70% launch time vs cold launch - Enterprise pattern

### 💰 Cost Optimization - $32.80 → $0.00

**Before Cleanup:**
- 1 Running t3.micro $7.60 + 4 Warm Pool Stopped $2.56 + day2-test-server $0.64 + ALB $22 = $32.80/mo

**Cleanup Order (Critical):**
1. Deleted Scheduled Actions (2) → `0 scheduled actions`
2. Deleted Warm Pool → `Warm pool deleted successfully` `Status Deleting`
3. Deleted ASG `day5-asg` → `Deleting`
4. Deleted ALB `day11-alb-asg` → `Successfully deleted load balancer`
5. Deleted TG `day11-tg-asg` → `Successfully deleted target group`
6. Deleted LT Version 2 → `Launch template version deleted`
7. Terminated `day2-test-server i-0cae82dc78d9694d` → `Terminated`

**After:** Bills `USD 0.00` - All services $0.00 - Free Tier credits covered learning

### 🛠 Tech Stack

- EC2 t3.micro, Launch Templates Versioning, Auto Scaling Groups, ALB, Target Groups, Warm Pools, Predictive + Scheduled + Target Tracking, CloudWatch, Cost Explorer

### 📚 Lessons

- Always delete Scheduled Actions BEFORE ASG or they recreate ASG
- Warm Pool must be deleted before ASG
- Stopped instances still cost EBS, only Terminated = $0
- ALB is biggest cost $22/mo - delete when not demoing

#AWS #DevOps #ChaosEngineering #FinOps #AutoScaling

---
Ready for Day 15: RDS + ASG + CloudFormation/Terraform $0 templates in repo.
