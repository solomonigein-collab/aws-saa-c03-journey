# Day 1: EC2 Launch
- Launched t3.micro day2-test-server in us-east-1d
- AMI: Amazon Linux 2023
- Key: No SSH key - using SSM Session Manager for SAA-C03 best practice
- Security Group: Allow 80, 443 from 0.0.0.0/0
- User Data: httpd install
