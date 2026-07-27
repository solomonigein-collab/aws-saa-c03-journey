# Day 5: AMI + Launch Template + ALB + Auto Scaling Group - Full Tutorial

### By Solomon Igein
**Prereq:** Day 4 snapshot snap-0289891c0e472868e and instance i-0cae82d2c78d9694d in us-east-1

### SAA-C03 Goal: Learn Golden AMI pattern, Launch Templates, and Auto Scaling for high availability

You should see `Available` in green for AMI, and later `Healthy` instances in ASG.

---

## Part 1: Create Golden AMI from Day 4 Instance

1. Go to **AWS Console -> EC2 -> Instances**
2. You should see `day2-test-server i-0cae82d2c78d9694d` in `Stopped` state (we stopped it on Day 4 H15.png). If running, Stop it.
3. Select it -> **Actions -> Image and templates -> Create image**
4. Image name: `day4-golden-ami-v1`
5. Description: `Golden AMI with Apache + CloudWatch Agent mem_used_percent 28.8% - SAA-C03 Day 5 - snapshot snap-0289891c0e472868e`
6. **UNCHECK** `No reboot`? Actually since instance is Stopped, you don't need No reboot - leave unchecked.
7. Instance volumes: 8 GiB gp3 - keep as is
8. Click **Create image**
9. Go to **EC2 -> AMIs** (left menu under Images) -> You should see `day4-golden-ami-v1` -> State: `Pending` -> Wait 3-4 min until `Available` (green). This is your trophy 1 - screenshot it.

**SAA-C03 Concept:** AMI = Amazon Machine Image - template with OS + software + config. Golden AMI pattern = you bake all software once, then launch 100s of identical servers in seconds.

---

## Part 2: Create Launch Template (LT)

1. Go to **EC2 -> Launch templates -> Create launch template**
2. Launch template name: `day5-lt-golden`
3. Template version description: `v1 with t3.micro + SSM role + CW Agent`
4. **Application and OS Images:** Click `My AMIs` -> `Owned by me` -> Select `day4-golden-ami-v1` (ami-xxxxxxxx). You should see your AMI ID.
5. Instance type: `t3.micro` - Select from list
6. Key pair: **Don't include in launch template** (we use Session Manager - more secure)
7. **Network settings:**
   - Security groups: Select existing SG `day2-test-server SG` - or create new allowing HTTP 80 from 0.0.0.0/0 and HTTPS 443.
   - For ALB later we need HTTP open to world.
8. **Advanced details -> IAM instance profile:** Select `EC2-SSM-CW-Role` (the role from Day 2)
9. **Advanced details -> User data - Optional - paste this (ensures Apache starts even if AMI reboot):**
```bash
#!/bin/bash
systemctl start httpd
systemctl enable httpd
echo "Hello from Day5 ASG - $(hostname -f) - AZ $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)" > /var/www/html/index.html
```
10. Click **Create launch template** -> You should see `Successfully created...`

---

## Part 3: Create Application Load Balancer (ALB) + Target Group FIRST (Best practice)

### 3A: Create Target Group
1. Go to **EC2 -> Target Groups -> Create target group**
2. Target type: **Instances**
3. Target group name: `day5-tg`
4. Protocol: HTTP Port 80, VPC: select your default VPC (same as your instance)
5. Health checks: Path `/` , Healthy threshold 2, Interval 30 sec
6. Click Next -> No targets to register now -> **Create target group**

### 3B: Create ALB
1. Go to **EC2 -> Load Balancers -> Create Load Balancer -> Application Load Balancer**
2. Name: `day5-alb`
3. Scheme: Internet-facing, IP address type IPv4
4. Network mapping: Select at least 2 AZs - e.g., us-east-1a and us-east-1b (must select subnets in 2 AZs)
5. Security groups: Select same SG that allows HTTP 80 from 0.0.0.0/0 - or create `day5-alb-sg` allowing HTTP 80 inbound from anywhere.
6. Listeners: HTTP:80 -> Forward to `day5-tg`
7. Click **Create load balancer** -> Wait until State: `Active` (not Provisioning). Copy DNS name: `day5-alb-xxxxxxxx.us-east-1.elb.amazonaws.com` - trophy 2 screenshot.

---

## Part 4: Create Auto Scaling Group (ASG) - The Magic

1. Go to **EC2 -> Auto Scaling groups -> Create Auto Scaling group**
2. Name: `day5-asg`
3. Launch template: Select `day5-lt-golden` -> Version `$Latest` -> Next
4. VPC: Default VPC
5. Availability Zones and subnets: Select at least 2 subnets (same AZs you used for ALB) - e.g., us-east-1a and us-east-1b
6. Next -> Load balancing: **Attach to an existing load balancer** -> Choose existing target groups -> Select `day5-tg`
7. Health checks: Turn on **ELB health checks** + EC2 (default)
8. Next -> Group size:
   - Desired capacity: `2`
   - Min: `1`
   - Max: `3`
9. Scaling policies: **Target tracking** -> Name `cpu50` -> Metric type `Average CPU utilization` -> Target value `50` -> Next
10. Add notifications? Skip -> Next -> Add tags: Key `Name` Value `day5-asg-instance` -> Next
11. **Create Auto Scaling group**

**What to expect:** Go to EC2 -> Instances -> You should see 2 NEW instances launching: `day5-asg-instance` with state `Pending` -> `Running`. Wait 3-4 min.

Go to **Target Groups -> day5-tg -> Targets** -> You should see 2 instances with `Initial` -> `Healthy` (green). Screenshot this - trophy 3!

Go to **Load Balancer -> day5-alb -> Copy DNS** -> Paste in browser -> You should see `Hello from Day5 ASG - ip-10-0-x-x - AZ us-east-1a` - Refresh and AZ changes between 1a and 1b because ALB load balances. Screenshot - trophy 4!

---

## Part 5: Test Self-Healing (SAA-C03 Favorite Interview Question)

1. Go to **EC2 -> Instances** -> Select ONE of the 2 ASG instances -> **Instance state -> Terminate instance**
2. Watch **ASG -> Activity** tab -> You should see `Terminating...` -> `Launching new instance` - ASG automatically replaces it to maintain Desired=2. This is self-healing!
3. Wait 3 min -> You should again have 2 Running instances.

**SAA-C03 Concept:** ASG ensures desired count, ALB distributes traffic, ELB health checks remove unhealthy instances. This is High Availability + Fault Tolerance.

---

## Part 6: Cost Optimization - Save Money

1. Go to **EC2 -> Auto Scaling groups -> day5-asg -> Edit** -> Set Desired, Min, Max all to `0` -> Update. This terminates all ASG instances but keeps ASG config for tomorrow.
2. Go to **EC2 -> Load Balancers** -> Keep ALB? For SAA cost, you can delete ALB (ALB costs ~$0.02/hr). Or keep but know cost.
3. Keep AMI `day4-golden-ami-v1` and snapshot `snap-0289891c0e472868e` - they cost pennies.

**Screenshots for Day 5 portfolio (5 trophies):**
- H1: AMI `day4-golden-ami-v1` state `Available`
- H2: Launch Template `day5-lt-golden` with your AMI
- H3: ALB `day5-alb` state `Active` + DNS
- H4: Target Group `day5-tg` with 2 Healthy instances
- H5: Browser showing ALB DNS page with AZ name, and ASG Activity showing self-healing launch

You are now building production-like infra!
