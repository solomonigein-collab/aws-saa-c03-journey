# Day 10 - Advanced Auto Scaling: Warm Pools & Instance Refresh
### AWS Certified Solutions Architect Associate SAA-C03
**Student: Solomon Igein (573705805662) | Region: us-east-1 | Date: 2026-08-03**
**ASG: day5-asg | Launch Template: day5-lt-golden (lt-0add66d9a6095a35e) | Golden AMI: ami-0fffe62b3d58d53c2**

---

## 1. Objectives of Day 10
- Understand why standard ASG scale-out is slow (2-3 min UserData)
- Implement Warm Pools to achieve 10x faster scale-out (10 seconds)
- Master Launch Template versioning (v1 → v2)
- Execute Instance Refresh (Rolling, Min 90%, Warmup 60s)
- Portfolio: Prove Zero-downtime deployment

## 2. Part 1: Warm Pool - Theory (SAA-C03 High-Yield)

### Problem without Warm Pool:
Scale-out event → ASG launches new EC2 from LT → Install Apache + CW Agent + UserData → ELB Health Checks → InService = **120-180 seconds**. User waits.

### Solution - Warm Pool:
Pool of **pre-initialized, Stopped instances** sitting ready.
Scale-out → ASG just **Starts** a Stopped instance → Already has Apache + CW Agent installed → **10-15 seconds InService**.

**Cost Benefit:** Stopped instances = you pay ONLY EBS volume (8GB gp3 ~ $0.80/mo), NOT CPU. Running = free tier t3.micro.

### SAA-C03 Exam Questions:
- Q: Application needs fastest scale-out, pre-warmed with app installed? **Answer: Warm Pool**
- Q: How to reduce scale-out time without keeping extra Running instances? **Warm Pool Stopped**
- Q: Difference between Warmed:Running vs Warmed:Stopped? Running = fastest but pay CPU, Stopped = slightly slower (10s start) but cheapest.

### Configuration Used:
- ASG: day5-asg, Min 1, Max 3, Desired 1
- Warm Pool: Min 1, Max 3 (Equal to group's max), State: Stopped, Reuse: Reuse on scale-in
- Result: 1 Running (InService) + 2 Warm Pool (Warmed:Stopped) = 3 total instances managed.

## 3. Part 1: Warm Pool - Hands-On Implementation (What you did)

**Initial State (Before):**
- EC2 Instances (4): 1 Running day5-asg (i-0df0ee2f1c844e10a), 1 Stopped warm pool candidate (i-03df17e337e929255), day2-test-server, other.

**Steps:**
1. EC2 → Auto Scaling Groups → day5-asg → Instance management tab
2. Warm pool section → Edit → Create warm pool: Min 1, Warm pool size 3, State Stopped, Reuse on scale-in
3. Save → ASG launches warm pool instances in background
4. Proof: Warm pool (2) instances appear, Current warm pool size: 2, State Warmed:Stopped, Healthy

**Proofs to keep:**
- EC2 Instances showing Stopped instance with tag aws:autoscaling:groupName=day5-asg (your 10D4)
- ASG console showing Warm pool instances (2) Healthy, Launch template Version 1 (your 10D5)

## 4. Part 2: Launch Template Versioning

**Why versioning?**
Golden AMI is immutable. To deploy new app version, you create NEW LT version, not edit running instances. ASG Instance Refresh uses new version.

**Steps you did:**
1. EC2 → Launch Templates → day5-lt-golden → Versions tab → Versions (1)
2. Actions → Create new version → Source Version 1
3. Version description: day10-v2-instance-refresh
4. (Optional) Tags: Version=v2-Day10
5. Create template version → Success banner
6. Result: Versions (2) - Version 1 Default Yes, Version 2 Description day10-v2-instance-refresh, AMI ami-0fffe62b3d58d53c2 same, Created 2026-08-03

**Exam tip:** Default version controls what NEW launches use. ASG can be pinned to specific version. Instance Refresh can override to newer version without changing default.

## 5. Part 3: Instance Refresh - Theory (Most Tested)

**What is Instance Refresh?**
Rolling update to replace all instances in ASG with new Desired Configuration (new LT version / new AMI) with ZERO downtime.

**Strategies:**
- Rolling: Replace in batches, keep Min Healthy % alive. **Cheapest, exam default.** Uses Terminate and launch.
- Launch before terminating: Launch new first, then terminate old. Faster but costs double capacity temporarily.

**Key Parameters (SAA-C03):**
- Minimum healthy percentage: 90% → During refresh, keep 90% healthy. With Desired 1, means 0 can be unhealthy, so replaces 1 by 1.
- Instance warmup: 60s → Time to wait after instance is InService before considering next batch. Must match target tracking warmup. Prevents flapping.
- Skip matching: DISABLED → Must be disabled to force replacement even if instance matches. If Enabled, instances already matching desired config are skipped → Refresh does 0 work (common trap!).
- Desired configuration - Update launch template: Check + Select Version 2 → Tells ASG to roll to v2.

**Lifecycle during your refresh:**
Total 3 instances to update (1 Running + 2 Warm Pool)
- 0% → 33% → 50% → 100% → Successful
- Each instance: Terminating → Launching → Warmed:Pending → Warmed:Stopped (or Running) → Warmup 60s → Next

## 6. Part 3: Instance Refresh - Hands-On Implementation (What you did)

1. ASG → day5-asg → Instance refresh tab → Start instance refresh
2. Replacement type: Replace instances, Instance replacement method: Terminate and launch (Control costs)
3. Set healthy percentage: Min 90% Max 100%
4. Fallback: Violate min healthy percentage
5. Instance warmup: 60 seconds
6. Skip matching: Disabled (UNCHECKED) - CRITICAL
7. Standby: Ignore
8. Desired configuration: Update launch template → day5-lt-golden | Version 2 (day10-v2-instance-refresh)
9. Start → Active instance refresh: ID 21293f83-22bb-46bf-b4dc-f67574e22d25, In progress 0% → 50% → 100%
10. Status reason: Waiting for instances to warm up before continuing. For example: i-0e3585030bfb8da1f is warming up
11. Final: Terminated old Running i-0df0ee2f1c844e10a and old warm i-03df17e337e929255, Created new Running i-0bc45928f3e7b8efc 3/3 checks + new Warm Pool 2 instances (i-04732751369b0e38d, i-0e3585030bfb8da1f) both Version 2 Healthy

**Final State (After):**
- ASG: day5-asg At desired capacity, Launch template day5-lt-golden | Version 2, 1/1 Healthy Desired 1
- Warm pool instances (2): Both Warmed:Stopped, t3.micro, Version 2, Healthy

## 7. Portfolio Screenshots - Which to Include

### Folder: /Day-10-Warm-Pool-Instance-Refresh/

**MANDATORY - Keep these 8 (in order):**

1. **10D1-ec2-instances-warm-pool-candidate.png** = Your first screenshot - 4 instances, 1 Running + 1 Stopped same AZ us-east-1b (proves warm pool candidate exists)
2. **10D2-ec2-tags-asg-ownership.png** = Your 10D4.png - Instance Details Tags aws:autoscaling:groupName=day5-asg, IAM EC2-SSM-Role, launch template id lt-0add66d9a6095a35e - proves ASG owns Stopped instance
3. **10D3-asg-warm-pool-2-instances.png** = Your 10D5.png - ASG console At desired capacity, Current warm pool size 2 instances, Min 1, Warm pool size 3, State Stopped, Reuse on scale in - MAIN DAY 10 PART 1 PROOF
4. **10D4-launch-template-versions-1.png** = Your 10D7.png - Before versioning, Versions (1) Default Version 1
5. **10D5-launch-template-v2-success.png** = Your 10D12.png - Success banner Successfully created day5-lt-golden(lt-0add66d9a6095a35e) green
6. **10D6-launch-template-versions-2-confirmed.png** = Your 10D13.png - Versions (2) Latest version 2, Description day10-v2-instance-refresh, Created 2026-08-03T13:41:42Z - MAIN PART 2 PROOF
7. **10D7-instance-refresh-config-v2.png** = Your 10D20.png - Desired configuration Update launch template checked, Launch template day5-lt-golden Version 2 day10-v2-instance-refresh selected
8. **10D8-instance-refresh-50-100-percent.png** = Combine 10D22 (50% Waiting for instances to warm up) + 10D23 (100% 1 instance left) - Shows rolling progress
9. **10D9-final-asg-v2-warm-pool-healthy.png** = Your 10D25.png - FINAL BOSS PROOF: ASG At desired capacity day5-lt-golden | Version 2 1/1 Healthy, Warm pool instances (2) both Warmed:Stopped Version 2 Healthy
10. **10D10-ec2-final-running-v2.png** = Your 1024.png - EC2 Instances (7) showing old Terminated + new Running i-0bc45928f3e7b8efc 3/3 checks passed 44.202.239.2 + new Stopped warm pool i-04732751369b0e38d - proves replacement worked

**Optional but nice:**
- Instance refresh config page showing Min 90% Max 100% Warmup 60s Skip matching Disabled
- Activity history tab showing Terminate/Launch events

**DELETE / Don't include:**
- Duplicates of same page
- Day2-test-server alone (not relevant)
- Any screenshot with sensitive IP fully visible (blur Public IPv4 if portfolio public)

## 8. Architecture Diagram

```
                    [Internet]
                        |
                [day5-asg - Min1 Max3 Desired1]
                        |
        +---------------+---------------+
        |                               |
  Running Fleet                 Warm Pool (Stopped)
  i-0bc45928f3e7b8efc          i-04732751... Stopped Healthy v2
  InService 3/3 checks        i-0e358503... Stopped Healthy v2
  Version 2                    Version 2
  44.202.239.221               Pre-warmed, pay EBS only

Scale-out event:
Warm Pool Stopped → Start → Running InService in 10s (vs 120s from scratch)

Instance Refresh:
ASG Rolling: Terminate v1 → Launch v2 → 60s warmup → Next
Result: Zero downtime deployment
```

## 9. Interview / Exam Q&A

**Q1: Why Warm Pool Stopped not Running?**
A: Stopped = cheapest, 10s start penalty acceptable. Running = instant but pay CPU always.

**Q2: What happens if you enable Skip matching during Instance Refresh?**
A: If instance already matches desired config, it's skipped. If you forgot to set desired config to v2, refresh does nothing. Must Disable skip matching to force replacement.

**Q3: Difference between Instance Refresh and Auto Scaling Rolling Update via CloudFormation?**
A: Instance Refresh is native ASG feature for LT/AMI updates, preserves AZ balance. CFN rolling update is via CFN stack.

**Q4: How does Warm Pool interact with Instance Refresh?**
A: Refresh updates BOTH running and warm pool instances to new version. Ensures future scale-outs use new version.

**Q5: Cost optimization?**
A: Warm Pool Stopped + t3.micro free tier + 8GB gp3 = <$1/mo for 2 warm instances, but gives 90% faster scale-out. Perfect for production spiky workloads.

## 10. Next Steps - Day 11 Preview

- Lifecycle Hooks: Custom actions before terminate (drain connections, upload logs)
- Mixed Instances Policy: Spot + On-Demand
- Predictive Scaling
- Attach ALB Target Group + Health Checks

---

**Instructor Notes:** Solomon executed Day 10 perfectly: Warm Pool 2 instances, LT v2 created 2026-08-03, Instance Refresh ID 21293f83-22bb-46bf-b4dc-f67574e22d25 Rolling 0%→100% Successful, Final ASG Version 2 1/1 Healthy. Ready for SAA-C03 advanced ASG questions.
