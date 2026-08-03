# Day 9: Auto Scaling Policies + Lifecycle Hooks Full Tutorial
AWS SAA-C03 60-Day Challenge | Solomon Igein | 03/08/2026

Portfolio Proofs in this folder:
- 9D1-target-tracking-created.png - Target Tracking cpu-50-target-tracking Enabled 50% CPU
- 9D2-scheduled-action-created.png - Scheduled Action day9-morning-scale-out Desired 3
- 9D3-lifecycle-launch-hook-created.png - Launch Hook day9-launch-hook
- 9D4-lifecycle-hooks-both-created.png - Both Hooks (2) Launch + Terminate 300s CONTINUE

Part 1: Target Tracking Scaling Policy
What: ASG maintains Average CPU at 50%
Steps: EC2 > ASG > day5-asg > Automatic scaling > Create dynamic scaling policy > Target tracking > Avg CPU 50 > Warmup 60 > Scale-in Enabled
Result: Dynamic scaling policies (1) cpu-50-target-tracking Enabled As required to maintain Average CPU utilization at 50

Part 2: Scheduled Scaling
What: Scale at known time 9AM to 3 instances
Steps: Scheduled actions (0) > Create scheduled action > Name day9-morning-scale-out > Desired 3 > Once > 2026/08/03 09:00 UTC > Create > Green banner success
Result: Scheduled actions (1) Desired 3

Part 3: Lifecycle Hooks
What: Pause launch/terminate to run custom actions
Hook 1 Launch: Instance management > Lifecycle hooks (0) > Create > Name day9-launch-hook > Transition Instance launch (autoscaling:EC2_INSTANCE_LAUNCHING) > Heartbeat 300 > Default CONTINUE > Create
Hook 2 Terminate: Create > Name day9-terminate-hook > Transition Instance terminate (autoscaling:EC2_INSTANCE_TERMINATING) > 300 > CONTINUE > Create
Result: Lifecycle hooks (2) day9-launch-hook LAUNCHING 300 CONTINUE + day9-terminate-hook TERMINATING 300 CONTINUE

SAA-C03 Cheat Sheet:
Target Tracking = Keep metric at target, AWS manages alarms, Preferred
Scheduled = Time-based predictable traffic
Predictive = AI forecast based on history
Lifecycle Hook = Pause for custom actions, Heartbeat timeout 30-7200, Default CONTINUE/ABANDON, Can notify SNS/SQS/EventBridge (Day 8 integration)

Architecture Day 9:
ALB > ASG day5-asg (1-3) > Policies: Target Tracking 50% + Scheduled 09:00 Desired 3 + Hooks Launch 300s + Terminate 300s + Day 8 SNS Fanout + EventBridge

Account: Solomon Igein 573705805662 us-east-1
