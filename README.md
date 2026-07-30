# AWS SAA-C03 60-Day Journey - Day 7 Complete

### By Solomon Igein - Zero to CloudWatch + Snapshot + ALB + ASG + SNS

**Region:** us-east-1 (N. Virginia) | **Instance:** day2-test-server i-0cae82d2c78d9694d | **ASG:** day5-asg (1/1 Healthy) | **ALB:** day5-alb | **SNS:** day7-alarms | **Alarm:** day7-high-cpu-70

**Progress:** Day 1 ✅ Day 2 ✅ Day 3 ✅ Day 4 ✅ Day 5 ✅ Day 6 ✅ Day 7 ✅ SNS + CloudWatch Notifications TODAY!

---
---

## Day 1: Launch Your First EC2 Server

You should see `active (running)` in green after this.

1. Go to **AWS Console -> EC2 -> Launch instance**
2. Name: `day2-test-server`, AMI: **Amazon Linux 2023**, Type: `t3.micro` (Free tier)
3. Key pair: **Proceed without a key pair** - we will use Session Manager (more secure for SAA-C03)
4. Network: **Create security group** -> Allow SSH from My IP only for now
5. Storage: `8 GiB gp3` (default)
6. User data - paste this to install Apache:
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "Hello from SAA-C03 Day 1 - $(hostname -f)" > /var/www/html/index.html
```
7. Click **Launch instance**
8. Wait 30 sec -> Go to EC2 -> Instances -> you should see `day2-test-server` -> State: `running` in green.

**SAA-C03 concept:** EC2 is a virtual server in AWS. t3.micro is burstable, free tier eligible.

---

## Day 2: Security Hardening with IAM Roles + SSM

You should see `Connected` in Session Manager, no SSH key needed.

1. Go to **IAM -> Roles -> Create role** -> Trusted entity: **AWS Service** -> Use case: **EC2** -> Next
2. Add policies: Search `AmazonSSMManagedInstanceCore` -> Check it -> Search `CloudWatchAgentServerPolicy` -> Check it -> Next -> Role name: `EC2-SSM-CW-Role` -> Create role
3. Go to **EC2 -> Instances -> i-0cae82d2c78d9694d -> Actions -> Security -> Modify IAM role** -> Select `EC2-SSM-CW-Role` -> Update IAM role
4. Go to **EC2 -> Instances -> Select instance -> Connect -> Session Manager tab -> Connect**
5. You should see a black terminal with `sh-5.2$` prompt. If you see `Connection failed`, wait 2 min for role to attach.

**SAA-C03 concept:** Never use SSH keys if you can use SSM. IAM Roles give EC2 permissions to talk to AWS services securely.

---

## Day 3: CloudWatch Basics

You should see `CPUUtilization` graph only, no RAM or Disk.

1. Go to **CloudWatch -> Metrics -> All metrics -> EC2 -> Per-Instance Metrics**
2. Search `i-0cae82d2c78d9694d` -> You will see only `CPUUtilization`, `StatusCheckFailed`, `NetworkIn` etc.
3. Question: Where is RAM? Where is Disk? Answer: EC2 hypervisor cannot see inside OS. You need CloudWatch Agent.

---

## Day 4: CloudWatch Agent - RAM 28.8% + Disk + Snapshot (The Hard Day)

### Step A: Install Agent via SSM
In Session Manager terminal:
```
sudo yum install -y amazon-cloudwatch-agent
```
If you see `(END)` in green at bottom - press `q`.

### Step B: Fix IAM if no data
Go to **IAM -> Roles -> EC2-SSM-CW-Role -> Add permissions -> Attach policies** -> `CloudWatchAgentServerPolicy` -> Attach. Wait 60 sec.

### Step C: Create Config File (fixes fail to fetch json config error)
Run in ONE paste:
```
sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/etc && sudo bash -c 'cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json <<EOF
{
  "metrics": {
    "metrics_collected": {
      "mem": { "measurement": ["mem_used_percent"], "metrics_collection_interval": 60 },
      "disk": { "measurement": ["used_percent"], "metrics_collection_interval": 60, "resources": ["/"] }
    }
  }
}
EOF
'
```

### Step D: Start Agent - You Should See Validation Succeeded (H6.png)
```
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```
You should see `Configuration validation succeeded` - Screenshot it.

### Step E: Verify Live Metrics - Trophy Shot (H10.png)
Wait 2 minutes, then:
1. Go to **CloudWatch -> Metrics -> All metrics**
2. You should now see `CWAgent 2` -> Click it.
3. Check `mem_used_percent ~28.8%` and `disk_used_percent ~27%` - THIS IS YOUR H10.png trophy.

### Step F: Create EBS Snapshot (H13.png)
1. Go to **EC2 -> Volumes -> Select vol-0f347c9b043a489a9 8 GiB**
2. **Actions -> Create snapshot** -> Description: `day4-before-autoscaling` -> **Create snapshot**
3. You will see `snap-0289891c0e472868e` - Go to **Snapshots** -> Status `Completed`, Size `2.26 GiB` (H13.png)

### Step G: Cost Optimization - Stop Instance (H15.png)
Go to **EC2 -> Instances -> Select day2-test-server -> Instance state -> Stop instance** -> Stop. State becomes `Stopped`.

You are done with Day 4!
