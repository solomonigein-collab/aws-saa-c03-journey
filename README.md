# AWS SAA-C03 - 60 Day Challenge - Solomon Igein
**Region:** us-east-1 (N. Virginia)
**Instance:** day2-test-server i-0cae82d2c78d9694d t3.micro us-east-1d
**Public IP (when running):** 18.205.112.162

This repo documents everything we troubleshot together - from IAM to CloudWatch Agent to EBS Snapshot.

## Day 4 Complete Proof
- ✅ IAM Fix: Added CloudWatchAgentServerPolicy to role
- ✅ Agent Validation: Configuration validation succeeded (H6.png)
- ✅ Live Metrics: CWAgent mem_used_percent 28.8% + disk_used_percent (H10.png)
- ✅ Snapshot: snap-0289891c0e472868e 2.26 GiB Completed - day4-before-autoscaling (H13.png)
- ✅ Cost Saving: Instance Stopped (H15.png)

## Structure
- /day1-EC2-setup.md
- /day2-security-hardening.md
- /day3-cloudwatch-basics.md
- /day4-cloudwatch-agent-deep-dive.md
- /scripts/amazon-cloudwatch-agent.json
- /scripts/user-data.sh

## How to push to GitHub (2 min)
1. Create new repo on github.com named aws-saa-c03-journey (public)
2. Upload these files OR run:
```
git init
git add .
git commit -m "Day 4 Complete - CloudWatch Agent + Snapshot"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/aws-saa-c03-journey.git
git push -u origin main
```
