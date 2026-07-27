# Day 4: CloudWatch Agent - RAM & Disk Monitoring (The Hard Day)

## Problem
Default EC2 only sends CPU metrics. SAA-C03 requires custom metrics: mem_used_percent and disk_used_percent.

## Issue 1: Terminal stuck at (END)
**Symptom:** Black page shows `amazon-cloudwatch-agent` green text + `(END)` bottom, pressing q does nothing.
**Root Cause:** Terminal in pager mode (less), lost focus.
**Fix:**
1. Click inside black terminal to focus
2. Press `q` (lowercase)
3. If stuck, Ctrl+C then q
4. Fresh session: EC2 -> Instances -> day2-test-server -> Connect -> Session Manager -> Connect

## Issue 2: fail to fetch config - no such file or directory (H5.png)
**Symptom:** `fail to fetch json config: open /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json: no such file or directory`
**Root Cause:** Config file not created / deleted after reboot.
**Fix:** Recreate config in ONE paste:
```bash
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

## Issue 3: IAM Permission Missing (H4.png root cause)
**Symptom:** Agent installed but no CWAgent namespace appears.
**Fix:** IAM -> Roles -> Role attached to day2-test-server -> Add permission -> `CloudWatchAgentServerPolicy` -> Attach.
Attach also `AmazonSSMManagedInstanceCore` for Session Manager.

## Start Agent (H6.png success)
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```
Expected: `Configuration validation second phase succeeded` + `Configuration validation succeeded` (H6.png)

## Verify (H7.png -> H10.png)
1. CloudWatch -> Metrics -> Custom namespaces -> You will see `CWAgent 2` (H7.png) - this is VICTORY
2. Click CWAgent tile -> you see `device, fstype, host, path 1` and `host 1` (H8.png)
3. Click `host` -> check `mem_used_percent` -> Graph shows ~28.8% (H9.png)
4. Breadcrumb: Click `All > CWAgent` to go back -> Click `device, fstype, host, path` -> check `disk_used_percent`
5. Graphed metrics (2) shows both lines (H10.png) - Day 4 trophy screenshot!

## Snapshot (H11.png -> H13.png)
EC2 -> Volumes -> vol-0f347c9b043a489a9 8 GiB gp3 -> Actions -> Create snapshot -> Description: day4-before-autoscaling
Result: snap-0289891c0e472868e 2.26 GiB Standard Completed (H13.png)
Banner: Successfully created snapshot (H12.png)

## Cost Optimization (H14.png -> H15.png)
EC2 -> Instances -> day2-test-server -> Instance state -> Stop -> Stop (H14.png)
Result: Successfully initiated stopping (H15.png) -> Stopped. No CPU charge, only EBS.
