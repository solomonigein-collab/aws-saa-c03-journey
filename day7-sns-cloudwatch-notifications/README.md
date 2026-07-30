# Day 7 - SNS + CloudWatch Notifications - Full Tutorial

## Objective
Build an end-to-end alerting system: Auto Scaling Group -> CloudWatch Alarm -> SNS Topic -> Email notification. Get notified when CPU > 70% and when instances launch/terminate.

## Architecture
```
CloudWatch Alarm (day7-high-cpu-70) 
   monitors -> ASG: day5-asg (CPUUtilization)
   on breach -> Action -> SNS Topic: day7-alarms
   SNS -> Subscriptions -> Email: solomonigein@gmail.com

ASG day5-asg -> Activity Notifications -> SNS Topic: day7-alarms -> Email
```

## Prerequisites
- Day 5 ASG `day5-asg` running (1/1 Healthy)
- SNS Topic from previous steps deleted or ready to create new

## Part 1: Create SNS Topic (7D1)

1. AWS Console -> Search **Simple Notification Service (SNS)**
2. Left menu -> **Topics** -> **Create topic**
3. Type: **Standard**
4. Name: `day7-alarms`
5. Display name: `day7-alarms`
6. Click **Create topic**
7. Green banner: "Successfully created topic..."

**Screenshot:** `7D1-sns-topic-created.png.png`
- Shows topic created successfully with ARN `arn:aws:sns:us-east-1:573705805662:day7-alarms`

## Part 2: Confirm Email Subscription (7D2 + 7D3)

1. Inside topic `day7-alarms` -> Tab **Subscriptions** -> **Create subscription**
2. Protocol: **Email**
3. Endpoint: `your-email@gmail.com`
4. Click **Create subscription** -> Status: Pending confirmation
5. Go to Gmail -> Open email from AWS Notifications -> Click **Confirm subscription**
6. Browser shows: `Subscription confirmed! 573705805662:day7-alarms:xxxx...`

7. Back to SNS -> Subscriptions -> Refresh -> Status: **Confirmed**
   - You should see `Subscriptions (2)` if you have BillingAlarmTopic too

**Screenshots:**
- `7D2-sns-subscription-confirmed.png.png` -> Browser "Subscription confirmed!"
- `7D3-sns-subscriptions-list.png.png` -> List showing 2 subscriptions with Status Confirmed

## Part 3: Create CloudWatch Alarm (7D4)

This alarm triggers the SNS email when ASG CPU is high.

1. Search **CloudWatch** -> Left -> **Alarms** -> **All alarms** -> **Create alarm**
2. **Select metric** -> Browse -> **EC2** -> **By Auto Scaling Group** (or Per-Instance Metrics)
3. Search `CPUUtilization` -> Check box for `AutoScalingGroupName = day5-asg` -> **Select metric**
4. **Specify metric and conditions:**
   - Metric: `day5-asg CPUUtilization`
   - Statistic: Average
   - Period: **1 minute**
   - Threshold type: **Static**
   - Whenever: **Greater** than **70**
   - Additional configuration -> Datapoints to alarm: **2 out of 2**
   - Missing data: Ignore
5. **Next** -> Configure actions:
   - When In alarm -> Notification -> Select an existing SNS topic -> Choose **`day7-alarms`**
   - Email will show below
6. **Next** -> Add name: `day7-high-cpu-70` -> Description: `Email when CPU > 70% for 2 mins`
7. **Next** -> Preview -> **Create alarm**
8. Green banner: `Successfully created alarm day7-high-cpu-70.`

**Initial state:** `Insufficient data` (for 1-2 mins) -> Then becomes `OK` (green)

**Screenshot:** `7D4-cloudwatch-alarm-created.png.png`
- Shows green banner + alarm details: Period 1 min, Threshold 70, Datapoints 2/2, Notification to day7-alarms

## Part 4: ASG Activity Notifications (7D5)

Get email when ASG launches or terminates an instance.

1. Search **EC2** -> Left -> **Auto Scaling Groups** -> Select `day5-asg`
2. Bottom tabs -> **Activity** (new console) -> **Activity notifications (0)** section
3. Click **Create notification**
4. Popup **Create notification**:
   - SNS Topic dropdown: **Select SNS Topic** -> Choose `day7-alarms`
   - Event types -> Check ALL 6:
     - Launch
     - Terminate
     - Replace root volume
     - Fail to launch
     - Fail to terminate
     - Fail to replace root volume
5. Click **Create** (orange)
6. Green banner: `Notifications created or edited successfully`
7. Table now shows: `Activity notifications (1)` -> `Send to: day7-alarms (your-email)` -> `Launch, Terminate, Fail to launch...`

**Screenshot:** `7D5-asg-notification-created.png.png`
- Shows green success banner + 1 notification linked to day7-alarms with all 6 events

## Verification & Testing (Optional but Gold)

**Test SNS:**
SNS -> Topics -> day7-alarms -> **Publish message** -> Subject: Test Day 7 -> Message: Hello from Day 7! -> Publish -> Check Gmail inbox for email from AWS

**Test CloudWatch (Stress):**
SSH into EC2 instance from day5-asg -> Run `sudo stress --cpu 2 --timeout 300` or `sudo yum install stress && stress --cpu 2 --timeout 180`
Watch CloudWatch alarm go `In alarm` (red) and receive email.

## Why This Matters for SAA-C03
- SNS is push-based, fully managed pub/sub
- CloudWatch Alarms can trigger: SNS, Auto Scaling actions, EC2 actions
- ASG Notifications use SNS to inform about lifecycle events
- Decoupling: CloudWatch doesn't send email directly, it triggers SNS
- Exam loves: What happens when CPU > threshold? -> Alarm -> SNS -> Notify

## Cleanup (Keep for next days)
DO NOT DELETE day7-alarms topic or day7-high-cpu-70 alarm yet. Needed for Day 8 integration.

## GitHub Structure
```
day7-sns-cloudwatch-notifications/
├── 7D1-sns-topic-created.png.png
├── 7D2-sns-subscription-confirmed.png.png
├── 7D3-sns-subscriptions-list.png.png
├── 7D4-cloudwatch-alarm-created.png.png
└── 7D5-asg-notification-created.png.png
```

## Commands Used
No CLI needed - all console. Optional CLI:
```bash
aws sns create-topic --name day7-alarms
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:573705805662:day7-alarms --protocol email --notification-endpoint your-email@gmail.com
aws cloudwatch put-metric-alarm --alarm-name day7-high-cpu-70 --metric-name CPUUtilization --namespace AWS/EC2 --statistic Average --period 60 --threshold 70 --comparison-operator GreaterThanThreshold --dimensions Name=AutoScalingGroupName,Value=day5-asg --evaluation-periods 2 --alarm-actions arn:aws:sns:us-east-1:573705805662:day7-alarms
```
