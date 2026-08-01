# Day 8 - Lambda + EventBridge Auto-Remediation - Full Tutorial

## Objective
Make your Day 7 alarm ACTIONABLE. When CPU alarm fires or instance launches/terminates, trigger a Lambda function automatically to log, tag, or notify. No human needed.

## Architecture Today
```
Before Day 7: CloudWatch Alarm -> SNS -> Email (you manually check)

After Day 8:
CloudWatch Alarm day7-high-cpu-70 -> SNS day7-alarms -> TWO subscribers:
   1) Email (solomonigein@gmail.com) -> You
   2) Lambda day8-alarm-handler -> Logs to CloudWatch Logs + auto-action

ASG day5-asg Launch/Terminate -> EventBridge Rule day8-asg-events -> Lambda day8-alarm-handler -> Log + Tag

EventBridge = Event Bus that listens to ALL AWS events
Lambda = Serverless code that runs when triggered
```

## Why This Matters for SAA-C03
- Exam favorite: "How to run code when alarm triggers without servers?" -> Lambda + SNS + CloudWatch
- EventBridge vs CloudWatch Events: EventBridge is new (exam loves it)
- Decoupling: SNS fan-out pattern (1 topic -> Email + Lambda + SQS + etc)
- Serverless remediation: Alarm -> Lambda -> Stop/Restart/Tag instance

## Part 1: Create IAM Role for Lambda (2 mins)

1. IAM -> Roles -> Create role
2. Trusted entity: **AWS service** -> Use case: **Lambda**
3. Permissions: Add:
   - `AWSLambdaBasicExecutionRole` (for CloudWatch Logs)
   - `AmazonSNSReadOnlyAccess` (optional)
4. Role name: `day8-lambda-role`
5. Create role
6. Click role -> Trust relationships -> Should show lambda.amazonaws.com

**Screenshot:** `8D1-lambda-role-created.png`

## Part 2: Create Lambda Function (5 mins)

1. Search **Lambda** -> Functions -> **Create function**
2. Author from scratch
   - Function name: `day8-alarm-handler`
   - Runtime: **Python 3.12** (latest)
   - Architecture: x86_64
   - Execution role: **Use an existing role** -> `day8-lambda-role`
3. Click **Create function** -> Green banner "Successfully created..."

4. Inside function -> **Code** tab -> Replace code with this Python:

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    logger.info("=== DAY 8 ALARM HANDLER TRIGGERED ===")
    logger.info(f"Full Event: {json.dumps(event, indent=2)}")
    
    # Check if from SNS
    if 'Records' in event:
        for record in event['Records']:
            sns_message = record['Sns']['Message']
            logger.info(f"SNS Message: {sns_message}")
            print(f"🚨 ALERT: {sns_message}")
            
            # Auto-remediation logic (Day 8 - just log, Day 9 will add action)
            if 'ALARM' in sns_message or 'CPU' in sns_message:
                logger.info("High CPU detected! Would trigger scale-up or notification")
    
    # Check if from EventBridge
    if 'source' in event and event['source'] == 'aws.autoscaling':
        detail = event.get('detail', {})
        logger.info(f"ASG Event: {detail.get('Description', 'No description')}")
        print(f"🔄 ASG Event: {event['detail-type']}")
    
    return {
        'statusCode': 200,
        'body': json.dumps('Day 8 Handler executed successfully!')
    }
```

5. Click **Deploy** (Ctrl+S) -> Wait for green "Changes deployed"

**Screenshots:**
- `8D2-lambda-created.png` -> Shows function created with Python 3.12 + role
- `8D3-lambda-code-deployed.png` -> Code editor with our code + Deploy button success

## Part 3: Subscribe Lambda to SNS Topic day7-alarms (3 mins) - FANOUT PATTERN

This is key SAA pattern: 1 SNS -> 2 subscribers (Email + Lambda)

1. Go to **SNS** -> Topics -> `day7-alarms` -> Tab **Subscriptions** -> **Create subscription**
2. Protocol: **AWS Lambda**
3. Endpoint: Select `day8-alarm-handler` (arn:aws:lambda:us-east-1:...)
4. Enable raw message delivery: **Leave unchecked** (we want JSON wrapper)
5. Click **Create subscription** -> Status **Confirmed** instantly (Lambda auto-confirms, no email needed!)

6. Verify: Subscriptions (3) now:
   - Email: Confirmed
   - Lambda: Confirmed

**Screenshot:** `8D4-sns-lambda-subscription.png` -> Shows 3 subscriptions: 1 Email billing, 1 Email yours, 1 Lambda day8-alarm-handler all Confirmed

## Part 4: Test SNS -> Lambda -> CloudWatch Logs (3 mins)

1. SNS -> Topics -> day7-alarms -> **Publish message**
   - Subject: `Test Day 8 Lambda Trigger`
   - Message body: `{"AlarmName":"day7-high-cpu-70","NewStateValue":"ALARM","Reason":"Testing Day 8 Lambda integration"}`
2. Click **Publish message** -> Green banner "Message published"

3. Wait 10 seconds -> Go to Lambda -> day8-alarm-handler -> Tab **Monitor** -> **View CloudWatch logs** -> Click latest Log Stream

4. You should see:
```
=== DAY 8 ALARM HANDLER TRIGGERED ===
SNS Message: {"AlarmName":"day7-high-cpu-70"...}
🚨 ALERT: ...
High CPU detected! Would trigger scale-up
```

**Screenshot:** `8D5-lambda-logs-success.png` -> CloudWatch Logs showing our log messages + 🚨 ALERT

**Also check email:** You should still get email (fan-out working!)

## Part 5: Create EventBridge Rule for ASG Events (5 mins) - Advanced

Listen to ASG events directly without SNS

1. Search **EventBridge** -> Left -> **Rules** -> **Create rule**
2. Name: `day8-asg-events`
3. Description: `Capture ASG Launch/Terminate for Day 8`
4. Event bus: default
5. Rule type: **Rule with an event pattern**
6. Next -> Event source: **AWS events**
7. Creation method: **Custom pattern (JSON editor)** -> Paste:

```json
{
  "source": ["aws.autoscaling"],
  "detail-type": ["EC2 Instance Launch Successful", "EC2 Instance Terminate Successful", "EC2 Instance Launch Unsuccessful"],
  "detail": {
    "AutoScalingGroupName": ["day5-asg"]
  }
}
```

OR use guided:
- Event pattern builder -> AWS services -> Auto Scaling
- Event Type -> EC2 Instance State-change Notification

8. Next -> Target types -> **AWS service** -> Select **Lambda function** -> Choose `day8-alarm-handler`
9. Next -> Tags -> Skip
10. Create rule -> Green banner

**Screenshot:** `8D6-eventbridge-rule-created.png` -> Shows rule day8-asg-events with pattern + target Lambda

## Part 6: Test EventBridge (Optional - Terminate one instance)

1. EC2 -> Auto Scaling Groups -> day5-asg -> Instance management -> Select 1 instance -> **Detach** OR set Desired to 2 then back to 1 to trigger Launch
2. Wait 1-2 mins -> ASG launches new instance
3. Check Lambda Logs again -> Should see new log with `source: aws.autoscaling`

**Screenshot (optional):** `8D7-eventbridge-triggered-log.png`

## GitHub Structure for Day 8
```
day8-lambda-eventbridge-remediation/
├── 8D1-lambda-role-created.png
├── 8D2-lambda-created.png
├── 8D3-lambda-code-deployed.png
├── 8D4-sns-lambda-subscription.png
├── 8D5-lambda-logs-success.png
├── 8D6-eventbridge-rule-created.png
├── lambda-code.py
└── README.md (this file)
```

## Cleanup (Keep)
Keep Lambda, Role, EventBridge Rule, SNS subscriptions - needed for Day 9 (SQS + DLQ)

## CLI Commands (Optional)
```bash
aws iam create-role --role-name day8-lambda-role --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name day8-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws lambda create-function --function-name day8-alarm-handler --runtime python3.12 --role arn:aws:iam::573705805662:role/day8-lambda-role --handler lambda_function.lambda_handler --zip-file fileb://function.zip
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:573705805662:day7-alarms --protocol lambda --notification-endpoint arn:aws:lambda:us-east-1:573705805662:function:day8-alarm-handler
aws lambda add-permission --function-name day8-alarm-handler --statement-id sns-invoke --action lambda:InvokeFunction --principal sns.amazonaws.com --source-arn arn:aws:sns:us-east-1:573705805662:day7-alarms
```
