# Day 6 - ALB Deep Dive + Health Checks + Target Groups + Scaling Policies
## AWS SAA-C03 60-Day Challenge

**Goal:** Make ALB highly available with proper health checks and enable intelligent auto scaling with Target Tracking.

---

### Architecture Recap
```
Internet -> ALB (day5-alb) -> Listener HTTP:80 -> Rule /* -> Target Group day5-tg (Port 80) -> EC2 instances in ASG
                                      |
                                      -> Health Checks every 30s to / on each instance
ASG -> Scaling Policy cpu-50-target-tracking -> CloudWatch CPU metric -> Add/remove instances (min 1, max 3)
```

---

### Part 1: Target Group Health Checks (The Heartbeat)

**Location:** EC2 -> Target Groups -> day5-tg -> Health checks tab

| Setting | Value | Exam Meaning |
|---------|-------|--------------|
| Protocol | HTTP | ALB talks HTTP to instance |
| Path | `/` | Calls http://instance:80/ — must return 200 |
| Port | Traffic port | Uses same port as target (80) |
| Healthy threshold | 5 | Needs 5 consecutive 200 OK to become Healthy |
| Unhealthy threshold | 2 | 2 failures = Unhealthy quickly |
| Timeout | 5 sec | Wait 5s for response |
| Interval | 30 sec | Check every 30s |
| Success codes | 200 | Only 200 is healthy |

**SAA-C03 Trap:** If your app health is at `/health` but TG checks `/`, instance will be Unhealthy forever even though app works! Change Path to match app.

**Verified:** 2 instances Healthy ✅
- i-074ede40c333e9079 - us-east-1a - Healthy
- i-0df0ee2f1c844e10a - us-east-1b - Healthy

Screenshots:
- `6D1.png` - Health check settings
- `6D2.png` - Targets 2/2 Healthy

---

### Part 2: ALB Deep Dive

**Location:** EC2 -> Load Balancers -> day5-alb

**Listeners:** Front door of ALB
- HTTP:80 -> Forward to day5-tg
- Rule: IF Path is /* THEN Forward to day5-tg

**Exam Concepts:**
- ALB is Layer 7 (HTTP) - can route by Path (/images/*, /api/*) and Host (app.example.com)
- NLB is Layer 4 (TCP) - ultra low latency, no path routing
- ALB terminates SSL - Add HTTPS:443 listener with ACM cert -> Forward to TG (instances stay HTTP)
- Cross-Zone Load Balancing: Enabled by default on ALB - distributes evenly across AZs

---

### Part 3: Auto Scaling Policies - The Brain

**Location:** EC2 -> Auto Scaling Groups -> day5-asg -> Automatic scaling

#### Created: Target Tracking Policy
- **Name:** cpu-50-target-tracking
- **Type:** Target Tracking Scaling (BEST for exam - 80% of questions)
- **Metric:** Average CPU utilization
- **Target:** 50%
- **Warmup:** 60 seconds - instance needs 60s to boot before counting CPU
- **Scale-in:** Enabled - can both add AND remove

**How it works:**
1. CloudWatch collects avg CPU from all instances every 1 min
2. If avg > 50% for 3 consecutive mins -> ASG launches new instance (up to max 3)
3. If avg < 50% for 15 mins -> ASG terminates one instance (down to min 1)

Screenshots:
- `6D3.png` - Create dynamic scaling policy form
- `6D4.png` - Policy created success
- `6D5.png` - Policy details - Enabled, Maintain 50%, Warmup 60s, Scale-in Enabled

#### Tested:
Stress test to trigger scale-out:
```bash
sudo yum install stress -y
stress --cpu 2 --timeout 400 &
# Watch: EC2 -> ASG -> Activity -> Launching new instance
pkill stress # After test, ASG will scale-in after 15 min
```

Result: ASG scaled from 2 -> 1 after low CPU - proves policy works!

---

### Part 4: Health Check Type - EC2 and ELB (CRITICAL)

**Location:** EC2 -> ASG -> day5-asg -> Details -> Health checks -> Edit

**Before:** Health check type = EC2 only
- Only checks if hypervisor says instance is running
- App could be crashed returning 500, but EC2 says Running -> ASG does nothing

**After (Day 6 Update):**
- **Turn on Elastic Load Balancing health checks = CHECKED ✅**
- **Type now = EC2, ELB**
- **Grace period = 300 sec** - Give new instance 5 min to boot before ELB checks kill it

**Why this is exam GOLD:**
If TG marks instance Unhealthy (2 failed checks), ASG with ELB health checks will TERMINATE and REPLACE it automatically! Self-healing architecture.

Screenshots:
- `6D5(1).png` - Edit page with ELB checkbox checked
- `6D6.png` - Updated successfully - EC2, ELB

---

### Day 6 Trophy Checklist

- [x] Target Group day5-tg - Path / - Thresholds 5/2 - 2/2 Healthy
- [x] ALB day5-alb - Listener 80 -> Forward to TG
- [x] Scaling Policy cpu-50-target-tracking - Target Tracking - CPU 50% - Enabled
- [x] ASG Health Check Type = EC2, ELB - Grace 300s
- [x] Verified scaling activity - 2 -> 1 scale-in
- [x] Pushed to GitHub - 8 Commits

---

### SAA-C03 Exam Traps to Memorize

1. **Cooldown:** Default 300s. After scaling action, ASG ignores metrics for cooldown period. Prevents flapping.
2. **Warmup vs Cooldown:** Warmup = new instance time to become useful (60s). Cooldown = time after scaling before next scaling (300s).
3. **Termination Policies:** OldestLaunchConfig, NewestInstance, ClosestToNextHour - which instance ASG kills on scale-in.
4. **Instance Protection:** Enable scale-in protection for DB migration instance so ASG doesn't kill it mid-migration.
5. **Health Check Grace:** If 300s too low, new instance still booting might be marked Unhealthy and terminated in loop.
6. **Target Tracking vs Scheduled vs Predictive:** Target Tracking = keep metric at value (CPU 50%). Scheduled = cron (scale to 5 on Monday 9am). Predictive = AI forecast.

---

### Next: Day 7 - CloudWatch Alarms + SNS Notifications + ASG Lifecycle Hooks
