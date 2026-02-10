# Troubleshooting Guide

Common issues and solutions for the Highly Available Multi-Tier Architecture.

## Table of Contents
- [Quick Diagnostics](#quick-diagnostics)
- [Load Balancer Issues](#load-balancer-issues)
- [Auto Scaling Issues](#auto-scaling-issues)
- [Database Connection Issues](#database-connection-issues)
- [Network Connectivity Issues](#network-connectivity-issues)
- [Performance Issues](#performance-issues)
- [Security Group Issues](#security-group-issues)
- [Monitoring & Logging](#monitoring--logging)

---

## Quick Diagnostics

### Health Check Script
```bash
#!/bin/bash
# Quick health check for the architecture

echo "=== Architecture Health Check ==="
echo ""

# Check ALB
echo "1. Load Balancer Status:"
aws elbv2 describe-load-balancers \
  --names ha-alb \
  --query 'LoadBalancers[0].State.Code' \
  --output text

# Check Target Health
echo ""
echo "2. Target Health:"
aws elbv2 describe-target-health \
  --target-group-arn $(aws elbv2 describe-target-groups \
    --names ha-target-group \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text) \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]' \
  --output table

# Check ASG
echo ""
echo "3. Auto Scaling Group:"
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names ha-asg \
  --query 'AutoScalingGroups[0].[DesiredCapacity,MinSize,MaxSize]' \
  --output table

# Check RDS
echo ""
echo "4. Database Status:"
aws rds describe-db-instances \
  --db-instance-identifier ha-db-subnet-group \
  --query 'DBInstances[0].[DBInstanceStatus,MultiAZ,PubliclyAccessible]' \
  --output table

echo ""
echo "=== Health Check Complete ==="
```

---

## Load Balancer Issues

### Issue 1: Unhealthy Targets

**Symptoms:**
- Targets showing as "Unhealthy" in target group
- 502/503 errors when accessing the ALB
- Intermittent application availability

**Diagnosis:**
```bash
# Check target health
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>

# Check target health details
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn> \
  --targets Id=<instance-id> \
  --query 'TargetHealthDescriptions[0].TargetHealth'
```

**Common Causes & Solutions:**

1. **Application not running on instance**
```bash
# SSH to instance via Session Manager
aws ssm start-session --target <instance-id>

# Check if web server is running
sudo systemctl status httpd
# or
sudo systemctl status nginx

# If not running, start it
sudo systemctl start httpd
sudo systemctl enable httpd
```

2. **Security group blocking ALB traffic**
```bash
# Verify security group rules
aws ec2 describe-security-groups \
  --group-ids <app-tier-sg-id> \
  --query 'SecurityGroups[0].IpPermissions'

# Required: Inbound rule allowing port 80/8080 from ALB security group
```

**Fix:**
- Edit app tier security group
- Add inbound rule: Type: HTTP, Source: ALB security group

3. **Health check path returns error**
```bash
# Test health check path from instance
curl localhost/
curl -I localhost/

# Should return HTTP 200
```

**Fix:**
- Ensure health check path exists and returns 200
- Update application to respond to health checks
- Or change health check path in target group

4. **Health check settings too aggressive**
```bash
# View current health check settings
aws elbv2 describe-target-groups \
  --target-group-arns <target-group-arn> \
  --query 'TargetGroups[0].HealthCheckIntervalSeconds,HealthCheckTimeoutSeconds,HealthyThresholdCount,UnhealthyThresholdCount'
```

**Fix:**
- Increase timeout from 5 to 10 seconds
- Increase interval from 30 to 60 seconds
- Increase unhealthy threshold from 3 to 5

### Issue 2: 502 Bad Gateway

**Symptoms:**
- ALB returns 502 error
- Targets are healthy but requests fail

**Diagnosis:**
```bash
# Check ALB logs (if enabled)
aws s3 ls s3://<log-bucket>/AWSLogs/<account-id>/elasticloadbalancing/ap-south-1/

# Check CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/ha-alb/<id> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum
```

**Common Causes & Solutions:**

1. **Application crashing or timing out**
```bash
# Check application logs
sudo journalctl -u httpd -n 100

# Check for out of memory
free -h
top

# Check disk space
df -h
```

2. **Backend timeout**
- Increase idle timeout on target group (default 60s)
- Optimize application response time

---

## Auto Scaling Issues

### Issue 1: Instances Not Scaling Out

**Symptoms:**
- CPU is high but no new instances launched
- Desired capacity not increasing

**Diagnosis:**
```bash
# Check scaling activities
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name ha-asg \
  --max-records 20

# Check scaling policies
aws autoscaling describe-policies \
  --auto-scaling-group-name ha-asg

# Check CloudWatch alarms
aws cloudwatch describe-alarms \
  --alarm-names <scaling-alarm-name>
```

**Common Causes & Solutions:**

1. **Reached maximum capacity**
```bash
# Check current capacity
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names ha-asg \
  --query 'AutoScalingGroups[0].[DesiredCapacity,MaxSize]'
```

**Fix:** Increase MaxSize if appropriate

2. **Cooldown period active**
```bash
# Check recent scaling activities
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name ha-asg \
  --max-records 5
```

**Fix:** Wait for cooldown to expire (default 300s) or reduce cooldown

3. **Insufficient capacity in AZ**
- AWS may not have available instances
- Try different instance type or AZ

4. **IAM permissions issue**
- Ensure Auto Scaling has permission to launch instances

### Issue 2: Instances Launching But Immediately Terminating

**Symptoms:**
- New instances appear but terminate quickly
- Scaling activities show "Launching" then "Terminating"

**Diagnosis:**
```bash
# Check scaling activities for error messages
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name ha-asg \
  --max-records 10 \
  --query 'Activities[*].[StatusCode,StatusMessage,Cause]'

# Check instance system logs
aws ec2 get-console-output --instance-id <instance-id>
```

**Common Causes:**
1. User data script failing
2. Health check failing immediately
3. AMI issue
4. EBS volume mounting failure

**Fix:**
- Review and fix user data script
- Increase health check grace period
- Test AMI separately
- Check EBS snapshot availability

---

## Database Connection Issues

### Issue 1: Cannot Connect to RDS from EC2

**Symptoms:**
- Application cannot connect to database
- Connection timeout errors
- Access denied errors

**Diagnosis:**
```bash
# From EC2 instance, test connection
mysql -h ha-db-subnet-group.cdiq4uegwb4l.ap-south-1.rds.amazonaws.com -u admin -p

# Test network connectivity
telnet ha-db-subnet-group.cdiq4uegwb4l.ap-south-1.rds.amazonaws.com 3306

# Check if port is open
nc -zv ha-db-subnet-group.cdiq4uegwb4l.ap-south-1.rds.amazonaws.com 3306
```

**Common Causes & Solutions:**

1. **Security group not allowing traffic**
```bash
# Check DB security group
aws ec2 describe-security-groups \
  --group-ids sg-0316df5bd8aeb420e \
  --query 'SecurityGroups[0].IpPermissions'
```

**Fix:**
- Ensure inbound rule allows MySQL (3306) from app tier security group
- Verify source is sg-01051589c04c07c48 (app-tier-sg)

2. **Wrong endpoint**
```bash
# Get correct endpoint
aws rds describe-db-instances \
  --db-instance-identifier ha-db-subnet-group \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text
```

3. **Database credentials incorrect**
- Verify username and password
- Check if user has appropriate permissions

4. **Database not in same VPC**
```bash
# Verify VPC
aws rds describe-db-instances \
  --db-instance-identifier ha-db-subnet-group \
  --query 'DBInstances[0].DBSubnetGroup.VpcId'
```

### Issue 2: Connection Pool Exhausted

**Symptoms:**
- "Too many connections" error
- Application slow or hanging

**Diagnosis:**
```bash
# Check current connections
mysql -h <endpoint> -u admin -p -e "SHOW PROCESSLIST;"
mysql -h <endpoint> -u admin -p -e "SHOW STATUS LIKE 'Threads_connected';"
mysql -h <endpoint> -u admin -p -e "SHOW STATUS LIKE 'Max_used_connections';"
```

**Solutions:**
1. Increase max_connections parameter
2. Implement connection pooling in application
3. Close idle connections
4. Scale up RDS instance class

---

## Network Connectivity Issues

### Issue 1: Instances Cannot Reach Internet

**Symptoms:**
- Cannot update packages (yum/apt)
- Cannot download from internet
- Outbound connections fail

**Diagnosis:**
```bash
# Test internet connectivity
ping -c 4 8.8.8.8
curl -I https://www.google.com

# Check route table
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=<subnet-id>" \
  --query 'RouteTables[0].Routes'
```

**Common Causes & Solutions:**

1. **NAT Gateway not in route table**
```bash
# Verify NAT Gateway route exists
# Should see: 0.0.0.0/0 → nat-<id>
```

**Fix:**
- Add route: Destination: 0.0.0.0/0, Target: NAT Gateway

2. **NAT Gateway in wrong state**
```bash
# Check NAT Gateway status
aws ec2 describe-nat-gateways \
  --nat-gateway-ids <nat-id> \
  --query 'NatGateways[0].State'
```

**Fix:**
- If "failed" or "deleted", create new NAT Gateway
- Associate with public subnet
- Update route table

3. **Security group blocking outbound traffic**
- Ensure outbound rule allows all traffic (0.0.0.0/0)

### Issue 2: Cannot SSH to Instance

**Symptoms:**
- Connection timeout when trying to SSH
- Connection refused

**Diagnosis:**
```bash
# Check if instance has public IP
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --query 'Reservations[0].Instances[0].PublicIpAddress'

# Check security group
aws ec2 describe-instances \
  --instance-ids <instance-id> \
  --query 'Reservations[0].Instances[0].SecurityGroups'
```

**Solutions:**

**Best Practice:** Use Systems Manager Session Manager instead of SSH
```bash
# Connect via SSM
aws ssm start-session --target <instance-id>
```

If SSH is required:
1. Instance must be in public subnet or have bastion host
2. Security group must allow SSH (22) from your IP
3. Key pair must be correct

---

## Performance Issues

### Issue 1: Slow Application Response

**Symptoms:**
- High latency
- Slow page loads
- Target response time high

**Diagnosis:**
```bash
# Check ALB target response time
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/ha-alb/<id> \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum

# Check CPU utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=AutoScalingGroupName,Value=ha-asg \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

**Common Causes:**
1. Insufficient compute capacity → Scale up/out
2. Database slow queries → Add indexes, optimize queries
3. No caching → Implement Redis/Memcached
4. Network latency → Check AZ placement

### Issue 2: High Database Latency

**Diagnosis:**
```bash
# Check RDS metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReadLatency \
  --dimensions Name=DBInstanceIdentifier,Value=ha-db-subnet-group \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

**Solutions:**
1. Enable read replicas for read-heavy workloads
2. Add database indexes
3. Implement query caching
4. Scale up database instance class
5. Use connection pooling

---

## Security Group Issues

### Common Security Group Problems

**Issue:** Connectivity works initially, then breaks

**Cause:** Security group rules modified

**Diagnosis:**
```bash
# Review security group rules
aws ec2 describe-security-groups --group-ids <sg-id>

# Check security group changes in CloudTrail
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=<sg-id> \
  --max-results 10
```

**Issue:** Can't determine which security group is blocking traffic

**Diagnostic Approach:**
1. Start with most restrictive (database)
2. Temporarily allow all traffic to test
3. Narrow down to specific port/protocol
4. Restore proper security group rules

**Example Test:**
```bash
# Temporarily allow all inbound (for testing only!)
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol all \
  --source-group <source-sg-id>

# Test connectivity
# Then remove the rule and add specific port
```

---

## Monitoring & Logging

### Enable Enhanced Monitoring

**ALB Access Logs:**
```bash
# Create S3 bucket
aws s3 mb s3://my-alb-logs-<account-id>

# Enable ALB access logs
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn <alb-arn> \
  --attributes Key=access_logs.s3.enabled,Value=true \
              Key=access_logs.s3.bucket,Value=my-alb-logs-<account-id>
```

**VPC Flow Logs:**
```bash
# Enable VPC flow logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids <vpc-id> \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn <role-arn>
```

**CloudWatch Agent on EC2:**
```bash
# Install CloudWatch agent
sudo yum install amazon-cloudwatch-agent -y

# Configure agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Start agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

### Useful CloudWatch Insights Queries

**Find 5xx errors in ALB logs:**
```sql
fields @timestamp, @message
| filter @message like /5\d{2}/
| stats count() by bin(5m)
```

**Application errors:**
```sql
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

---

## Emergency Procedures

### Rollback Procedure

If deployment causes issues:

1. **Identify last known good configuration**
```bash
# Check ASG launch template versions
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names ha-asg \
  --query 'AutoScalingGroups[0].LaunchTemplate'
```

2. **Revert to previous launch template**
```bash
# Update ASG with previous template version
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name ha-asg \
  --launch-template LaunchTemplateId=<id>,Version=<previous-version>
```

3. **Force instance refresh**
```bash
# Terminate instances to force new launches
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name ha-asg
```

### Scale Down for Maintenance

```bash
# Set desired capacity to 0
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name ha-asg \
  --desired-capacity 0

# Suspend Auto Scaling processes
aws autoscaling suspend-processes \
  --auto-scaling-group-name ha-asg \
  --scaling-processes Launch Terminate
```

### Resume After Maintenance

```bash
# Resume Auto Scaling
aws autoscaling resume-processes \
  --auto-scaling-group-name ha-asg \
  --scaling-processes Launch Terminate

# Restore capacity
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name ha-asg \
  --desired-capacity 2
```

---

## Getting Help

### AWS Support Resources

1. **AWS Support**: Create a support case in AWS Console
2. **AWS Forums**: https://forums.aws.amazon.com/
3. **AWS re:Post**: https://repost.aws/
4. **AWS Documentation**: https://docs.aws.amazon.com/

### Useful Commands for Support

**Collect diagnostics:**
```bash
#!/bin/bash
# Save to diagnostics.txt

echo "=== System Information ===" > diagnostics.txt
uname -a >> diagnostics.txt
cat /etc/os-release >> diagnostics.txt

echo -e "\n=== Network Configuration ===" >> diagnostics.txt
ip addr >> diagnostics.txt
ip route >> diagnostics.txt

echo -e "\n=== DNS Configuration ===" >> diagnostics.txt
cat /etc/resolv.conf >> diagnostics.txt

echo -e "\n=== Active Connections ===" >> diagnostics.txt
netstat -tulpn >> diagnostics.txt

echo -e "\n=== Disk Usage ===" >> diagnostics.txt
df -h >> diagnostics.txt

echo -e "\n=== Memory Usage ===" >> diagnostics.txt
free -h >> diagnostics.txt

echo -e "\n=== Running Processes ===" >> diagnostics.txt
ps aux >> diagnostics.txt

echo -e "\n=== Recent Logs ===" >> diagnostics.txt
tail -n 100 /var/log/messages >> diagnostics.txt
```

---

**Document Version:** 1.0  
**Last Updated:** February 10, 2026  
**Maintained By:** DevOps Team
