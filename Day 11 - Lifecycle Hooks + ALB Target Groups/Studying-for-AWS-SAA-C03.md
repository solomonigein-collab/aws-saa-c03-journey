# Day 11 - Lifecycle Hooks + ALB Target Groups | SAA-C03 Complete Guide

Account: Solomon Igein (573705805662) | us-east-1 | ASG day5-asg (lt-0add66d9a6095a35e v2) | i-0bc45928f3e7b8efc
ALBs: day5-alb (day5-tg) + day11-alb-asg (day11-tg-asg) | DNS: day11-alb-asg-1979065308.us-east-1.elb.amazonaws.com

## PART 1: Lifecycle Hooks (15% exam)

What: Pause ASG in Wait state for custom actions.
- LAUNCHING: Pending:Wait - install software
- TERMINATING: Terminating:Wait - drain connections, S3 logs

Our Hook day11-terminating-hook:
- Transition: autoscaling:EC2_INSTANCE_TERMINATING (Instance terminate)
- Heartbeat: 300 sec (Range 30-7200, Default 3600)
- DefaultResult: CONTINUE = timeout -> continue termination (safe for terminating). ABANDON = timeout -> keep instance (safe for launching)
- Metadata: Optional JSON, NOT target ARN
- Target: SNS day11-lifecycle-topic (optional, hook works without)

CLI:
aws autoscaling put-lifecycle-hook --lifecycle-hook-name day11-terminating-hook --auto-scaling-group-name day5-asg --lifecycle-transition autoscaling:EC2_INSTANCE_TERMINATING --heartbeat-timeout 300 --default-result CONTINUE --notification-target-arn arn:aws:sns:us-east-1:573705805662:day11-lifecycle-topic --role-arn arn:aws:iam::573705805662:role/service-role/AutoScaling-SNS-Role

## PART 2: ALB + TG + ASG (35% exam)

Architecture: Route53 -> ALB (2 AZs) -> TG (/) -> ASG 1-3 t3.micro -> Golden AMI v2 ami-0fffe62b3d58d53c2

TG day11-tg-asg:
- Type Instance (not IP/Lambda), HTTP:80, HTTP1, VPC vpc-0a9725e6138ed7268, ARN .../6d8f7dcfa3694df3
- Health: Path / ONLY, Traffic port, Healthy 2, Unhealthy 2, Timeout 5, Interval 30, Success 200
- 0 at creation -> ASG auto-registers -> 1 Healthy

ALB day11-alb-asg:
- Type Application, Internet-facing, 2 AZs us-east-1a subnet-0c141c3cafed07bd2 + us-east-1b subnet-09d66e432ddac8f0b
- Listener HTTP:80 -> day11-tg-asg
- DNS day11-alb-asg-1979065308.us-east-1.elb.amazonaws.com

GOLDEN RULE: NEVER manual register ASG instances in TG!
Wrong: TG > Register targets > add i-0bc45928f3e7b8efc
Correct: ASG > Integrations > Load balancing > select day11-tg-asg > Update

Health Check Type:
- EC2 only: Running = Healthy even if Apache down -> 502
- EC2, ELB + Grace 300: Recommended! ELB failure -> ASG replaces. Grace 300 ignores checks during boot.

Multiple TGs: day5-asg has day5-tg + day11-tg-asg -> valid, serves both ALBs.

## PART 3: Evidence Checklist
11D2-hook dialog, 11D3-hooks 3 list, 11D4-tg success, 11D5-alb provisioning, 11D6-tg healthy, 11D7-alb apache page, 11D8-asg integrations attached (your latest screenshot!)

## PART 4: 15 Exam Qs
Q1 Hook 300 CONTINUE Lambda crash? A: Terminated after 300s
Q2 Path "/ Healthy threshold..." Unhealthy? A: Path invalid, must be /
Q3 Manual 2 TG, ASG scales to 3? A: 2 only, manual not auto
Q4 EC2 only, Apache crash? A: Keeps unhealthy
Q5 One ASG to 2 TGs? A: Yes
Q6 Hook no SNS? A: Works, pauses, no notification
Q7 ALB single AZ? A: Blocked, need 2
Q8 Heartbeat range? A: 30-7200
Q9 Target type ASG? A: Instance
Q10 Grace 300 fails 2 min? A: Ignores 300s
Q11 Launch fail default? A: ABANDON
Q12 TG Healthy 504? A: SG
Q13 SNS not received? A: Role + topic policy
Q14 Attach new TG need refresh? A: No
Q15 2 ALBs same ASG cost? A: 2x ALB + 1x EC2