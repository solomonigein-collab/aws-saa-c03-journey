# AWS SAA-C03 - 60 Day Challenge Journey
### By Solomon Igein - solomonigein-collab

> Building real AWS infrastructure daily — from EC2 basics to Event-Driven Auto Remediation

---

## 🚀 Progress Overview

- ✅ **Day 1:** EC2 Setup - Launch, SSH, Security Groups
- ✅ **Day 2:** Security Hardening - IAM Users, MFA, Security Groups Lockdown
- ✅ **Day 3:** CloudWatch Basics - Metrics, Dashboards, CPU Monitoring
- ✅ **Day 4:** CloudWatch Agent Deep Dive - Logs, Custom Metrics
- ✅ **Day 5:** Full Tutorial - AMI + Launch Template + ALB + Auto Scaling Group
- ✅ **Day 6:** ALB Health Checks and Target Tracking Scaling Policies
- ✅ **Day 7:** SNS + CloudWatch Notifications - Alarm `day7-high-cpu-70` → SNS Topic `day7-alarms` → Email Subscription
- ✅ **Day 8: Lambda + EventBridge Auto Remediation - SNS Fanout + Event-Driven Architecture** **[NEW - Today]**

---

### Day 8: Lambda + EventBridge Auto Remediation ✅
**Goal:** Build automated remediation with SNS Fanout + EventBridge Events

**What I Built:**
1. **IAM Role:** `day8-lambda-role` - AWSLambdaBasicExecutionRole + CloudWatch Logs permissions
   - Proof: `8D1-lambda-role-created.png` - Role created with green banner

2. **Lambda Function:** `day8-alarm-handler` - Python 3.12 runtime
   - Handler parses SNS message, logs 🚨 ALERT, checks High CPU logic
   - Proof: `8D2-lambda-created.png` - Function ARN + Python 3.12 visible
   - Proof: `8D3-lambda-code-deployed.png` - Code deployed successfully

3. **SNS Fanout (SAA-C03 Key Concept):**
   - Topic: `day7-alarms` (Standard)
   - Subscriptions: 3 Total → 2 Email + 1 Lambda ALL Confirmed
     - `arn:aws:lambda:us-east-1:573705805662:function:day8-alarm-handler` - Confirmed LAMBDA
     - `solomonigein@gmail.com` - Confirmed EMAIL
   - Publish: 1 message `Day8 Lambda fanout test - Solomon Igein`
   - Result: Delivered to BOTH Email (Gmail) + Lambda (CloudWatch Logs) simultaneously
   - Proof: `8D4-sns-lambda-subscription.png` - Subscriptions (3) Fanout
   - Proof: `8D1-publish-success.png` - Message published to topic day7-alarms successfully
   - Proof: Gmail inbox AWS Notifications Day 8 + CloudWatch Logs `=== DAY 8 ALARM HANDLER TRIGGERED ===`

4. **Lambda Execution Proof:**
   - CloudWatch Logs: `/aws/lambda/day8-alarm-handler`
   - Log shows: `🚨 ALERT: Day8 Lambda fanout test...`, `High CPU detected! Would trigger scale-up`, `Duration: 2.62ms`
   - Proof: `8D5-lambda-logs-success.png`

5. **EventBridge Rule:** `day8-asg-events`
   - Event Bus: `default`, Status: Enabled, Type: Standard
   - Event Pattern:
     ```json
     {
       "source": ["aws.autoscaling"],
       "detail-type": ["EC2 Instance Launch Successful", "EC2 Instance Terminate Successful", "EC2 Instance Launch Unsuccessful"]
     }
     ```
   - Target: Lambda `day8-alarm-handler` with role `Amazon_EventBridge_Invoke_Lambda_292310891`
   - Proof: `8D6-eventbridge-rule-created.png` - Rule created successfully + Target Lambda

**Architecture:**
```
CloudWatch Alarm (day7-high-cpu-70) 
   → SNS Topic (day7-alarms) 
      → Fanout → Email + Lambda (day8-alarm-handler) → CloudWatch Logs 🚨

ASG Launch/Terminate Events 
   → EventBridge Rule (day8-asg-events) 
      → Lambda (day8-alarm-handler) → Auto Remediation
```

**SAA-C03 Concepts Covered:**
- SNS Fanout Pattern (1 Publisher → Many Subscribers)
- Lambda Event Source (SNS + EventBridge)
- EventBridge vs CloudWatch Events (Modern event bus)
- IAM Execution Roles for Lambda
- CloudWatch Logs for Lambda monitoring

**Folder:** [Day 8 Lambda EventBridge Auto Remediation Full Tutorial](./Day%208%20Lambda%20EventBridge%20Auto%20Remediation%20Full%20Tutorial/)

---

### Day 7: SNS + CloudWatch Notifications ✅
- CloudWatch Alarm: `day7-high-cpu-70`
- SNS Topic: `day7-alarms`
- Email subscription confirmed
- Folder: [day7-sns-cloudwatch-notifications](./day7-sns-cloudwatch-notifications/)

### Day 6: ALB Health Checks and Target Tracking
- ALB + Health Checks + Auto Scaling Group
- Folder: [day6-alb-health-checks-and-target-tracking](./day6-alb-health-checks-and-target-tracking/)

### Day 5: Full Tutorial - AMI + Launch Template + ALB + ASG
- File: `Day-5-Full-Tutorial-Download-(MARKDOWN-...`

---

## 🛠 Tech Stack
- EC2, AMI, Launch Templates
- ALB, Target Groups, Health Checks
- Auto Scaling Groups, Scaling Policies
- CloudWatch Metrics, Alarms, Logs, Agent, Dashboards
- SNS Topics, Subscriptions, Fanout
- Lambda (Python 3.12), IAM Roles, EventBridge Rules
- Git, GitHub, GitHub Desktop

## 📚 Daily Proofs
Every day includes screenshots with green success banners, ARNs, and real execution logs — no mocks.

## Next Up
- **Day 9:** Auto Scaling Policies - Target Tracking + Scheduled Scaling + Predictive + Lifecycle Hooks
- **Day 10:** Cost Optimization + Cleanup

---

**Connect:** GitHub - solomonigein-collab | Journey: aws-saa-c03-journey
