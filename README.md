# Highly Available Multi-Tier Web Application on AWS

A production-ready, highly available three-tier web application architecture deployed on AWS with automated scaling, load balancing, and secure database implementation.

## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Features](#features)
- [Architecture Components](#architecture-components)
- [Prerequisites](#prerequisites)
- [Deployment Guide](#deployment-guide)
- [Network Architecture](#network-architecture)
- [Security Configuration](#security-configuration)
- [Testing & Validation](#testing--validation)
- [Monitoring](#monitoring)
- [Cost Considerations](#cost-considerations)
- [Troubleshooting](#troubleshooting)
- [Future Enhancements](#future-enhancements)
- [Screenshots](#screenshots)
- [License](#license)

## 🏗️ Architecture Overview

This project implements a secure, highly available three-tier architecture on AWS with the following characteristics:

- **Multi-AZ Deployment**: Resources distributed across 2 Availability Zones (ap-south-1a, ap-south-1b)
- **Auto Scaling**: Automatic scaling between 2-4 instances based on CPU utilization
- **Load Balancing**: Application Load Balancer distributing traffic across healthy instances
- **Database High Availability**: RDS MySQL with Multi-AZ deployment
- **Security**: Defense-in-depth with security groups, private subnets, and IAM roles

## ✨ Features

- ✅ **High Availability**: Multi-AZ deployment ensures 99.99% uptime
- ✅ **Auto Scaling**: Automatically adjusts capacity based on demand
- ✅ **Secure Database**: RDS MySQL isolated in private subnets with no public access
- ✅ **Load Balancing**: Traffic distributed evenly across healthy instances
- ✅ **Network Isolation**: Three-tier security with public, private app, and private database subnets
- ✅ **Fault Tolerance**: System continues operating even if one AZ fails
- ✅ **Managed Services**: Leverages AWS managed services for reduced operational overhead

## 🔧 Architecture Components

### 1. Network Layer (VPC)
- **VPC CIDR**: 10.0.0.0/16
- **Availability Zones**: 2 (ap-south-1a, ap-south-1b)
- **Subnets**: 4 total
  - 2 Public subnets (for load balancer)
  - 2 Private subnets (for application tier)
  - 2 Private subnets (for database tier)

### 2. Web Tier
- **Application Load Balancer (ALB)**: `ha-alb`
  - Scheme: Internet-facing
  - Listener: HTTP:80
  - Health checks: Enabled
  - Cross-zone load balancing: Enabled

### 3. Application Tier
- **Auto Scaling Group**: `ha-asg`
  - Desired Capacity: 2
  - Minimum: 2
  - Maximum: 4
  - Instance Type: t3.micro
  - Scaling Policy: Target Tracking (70% CPU)
- **Launch Template**: `app-tier-template`
  - AMI: Amazon Linux 2023
  - Security Group: Allows traffic from ALB only
  - IAM Role: EC2-SSM-Core-Role

### 4. Database Tier
- **RDS Instance**: MySQL 8.4.7
  - Instance Class: db.t3.micro
  - Multi-AZ: Enabled
  - Storage: 20 GB SSD with autoscaling
  - Backup Retention: 7 days
  - Public Access: Disabled
  - Encryption: Enabled

### 5. Security Components
- **Security Groups**:
  - ALB Security Group: Allows HTTP/HTTPS from internet
  - App Tier Security Group: Allows traffic from ALB only
  - Database Security Group: Allows MySQL from App tier only
- **NAT Gateways**: 2 (one per AZ for high availability)
- **Internet Gateway**: 1 for public subnet internet access

## 📦 Prerequisites

- AWS Account with appropriate permissions
- AWS CLI configured
- Basic understanding of:
  - VPC and networking concepts
  - EC2 and Auto Scaling
  - RDS and database management
  - Load balancing concepts

## 🚀 Deployment Guide

### Step 1: VPC Setup

```bash
# VPC Configuration
VPC Name: ha-vpc-vpc
CIDR Block: 10.0.0.0/16
Region: ap-south-1
Availability Zones: ap-south-1a, ap-south-1b

# Subnet Configuration
Public Subnets:
  - ha-vpc-subnet-public1-ap-south-1a (10.0.0.0/24)
  - ha-vpc-subnet-public2-ap-south-1b (10.0.1.0/24)

Private App Subnets:
  - ha-vpc-subnet-private1-ap-south-1a (10.0.11.0/24)
  - ha-vpc-subnet-private2-ap-south-1b (10.0.12.0/24)

Private DB Subnets:
  - Custom naming based on your setup
```

### Step 2: Security Groups

**ALB Security Group** (`web-tier-sg`):
```
Inbound:
  - HTTP (80) from 0.0.0.0/0
  - HTTPS (443) from 0.0.0.0/0

Outbound:
  - All traffic
```

**App Tier Security Group** (`app-tier-sg`):
```
Inbound:
  - HTTP (80) from ALB Security Group
  - Custom TCP (8080) from ALB Security Group

Outbound:
  - All traffic
```

**Database Security Group** (`rds-mysql-from-app`):
```
Inbound:
  - MySQL (3306) from App Tier Security Group (sg-01051589c04c07c48)

Outbound:
  - All traffic
```

### Step 3: RDS Database

```bash
# Database Configuration
Engine: MySQL 8.4.7
Instance Identifier: ha-db-subnet-group
Master Username: admin
Instance Class: db.t3.micro
Storage: 20 GB (Autoscaling enabled)
Multi-AZ: Yes
Subnet Group: ha-db-subnet-group
Security Group: rds-mysql-from-app
Public Access: No
Backup Retention: 7 days
Encryption: Enabled
```

### Step 4: Launch Template

```bash
# Launch Template Configuration
Name: app-tier-template
AMI: Amazon Linux 2023
Instance Type: t3.micro
Security Group: app-tier-sg
IAM Role: EC2-SSM-Core-Role

# User Data (example)
#!/bin/bash
yum update -y
yum install -y httpd mariadb105
systemctl start httpd
systemctl enable httpd
echo "Highly Available App - $(hostname -f)" > /var/www/html/index.html
```

### Step 5: Auto Scaling Group

```bash
# ASG Configuration
Name: ha-asg
Launch Template: app-tier-template
VPC: ha-vpc-vpc
Subnets: Private app subnets in both AZs
Desired Capacity: 2
Minimum: 2
Maximum: 4
Health Check Type: ELB
Health Check Grace Period: 300 seconds
Scaling Policy: Target Tracking - 70% CPU
```

### Step 6: Application Load Balancer

```bash
# ALB Configuration
Name: ha-alb
Scheme: Internet-facing
IP Address Type: IPv4
VPC: ha-vpc-vpc
Subnets: Public subnets in both AZs
Security Group: web-tier-sg

# Target Group
Name: ha-target-group
Protocol: HTTP
Port: 80
VPC: ha-vpc-vpc
Health Check Path: /
Health Check Interval: 30 seconds
Healthy Threshold: 2
Unhealthy Threshold: 3

# Listener
Protocol: HTTP
Port: 80
Default Action: Forward to ha-target-group
```

## 🌐 Network Architecture

### Route Tables

**Public Route Table** (`ha-vpc-rtb-public`):
```
Destination: 0.0.0.0/0 → Internet Gateway
Destination: 10.0.0.0/16 → Local
```

**Private Route Table 1** (`ha-vpc-rtb-private1-ap-south-1a`):
```
Destination: 0.0.0.0/0 → NAT Gateway (ap-south-1a)
Destination: 10.0.0.0/16 → Local
```

**Private Route Table 2** (`ha-vpc-rtb-private2-ap-south-1b`):
```
Destination: 0.0.0.0/0 → NAT Gateway (ap-south-1b)
Destination: 10.0.0.0/16 → Local
```

### Network Flow

```
Internet → ALB (Public Subnets) → EC2 Instances (Private App Subnets) → RDS (Private DB Subnets)
                                                    ↓
                                            NAT Gateway (Outbound Internet)
```

## 🔒 Security Configuration

### Defense in Depth Layers

1. **Network Layer**:
   - VPC isolation
   - Public/Private subnet segmentation
   - Network ACLs (default)

2. **Instance Layer**:
   - Security Groups with least privilege
   - No direct internet access to app/database tiers
   - IAM roles for AWS service access

3. **Database Layer**:
   - No public accessibility
   - Encryption at rest enabled
   - Encryption in transit (SSL/TLS)
   - Automated backups

4. **Application Layer**:
   - IAM role for EC2 (EC2-SSM-Core-Role)
   - Systems Manager for secure access (no SSH keys needed)

### IAM Roles

**EC2-SSM-Core-Role**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ssm:UpdateInstanceInformation",
        "ssmmessages:CreateControlChannel",
        "ssmmessages:CreateDataChannel",
        "ssmmessages:OpenControlChannel",
        "ssmmessages:OpenDataChannel"
      ],
      "Resource": "*"
    }
  ]
}
```

## 🧪 Testing & Validation

### Health Check Verification

1. **Load Balancer Health**:
```bash
# Check target health
aws elbv2 describe-target-health \
  --target-group-arn <target-group-arn>
```

2. **Application Accessibility**:
```bash
# Test ALB endpoint
curl http://ha-alb-1207234725.ap-south-1.elb.amazonaws.com

# Expected Output:
# Highly Available App - ip-172-31-20-242.ap-south-1.compute.internal
# or
# Highly Available App - ip-172-31-34-110.ap-south-1.compute.internal
```

3. **Database Connectivity**:
```bash
# From EC2 instance
mysql -h ha-db-subnet-group.cdiq4uegwb4l.ap-south-1.rds.amazonaws.com -u admin -p

# Verify connection
mysql> SELECT VERSION();
# Output: 8.4.7
```

### Failover Testing

1. **Instance Failure Test**:
```bash
# Stop one instance
aws ec2 stop-instances --instance-ids <instance-id>

# Verify:
# - ALB marks instance as unhealthy
# - Traffic routes only to healthy instance
# - ASG launches replacement instance
```

2. **AZ Failure Simulation**:
   - Manually terminate all instances in one AZ
   - Verify application remains accessible via ALB
   - Confirm ASG launches new instances

## 📊 Monitoring

### CloudWatch Metrics to Monitor

**ALB Metrics**:
- `TargetResponseTime`
- `HealthyHostCount`
- `UnHealthyHostCount`
- `RequestCount`

**Auto Scaling Metrics**:
- `GroupDesiredCapacity`
- `GroupInServiceInstances`
- `GroupMinSize`, `GroupMaxSize`

**RDS Metrics**:
- `CPUUtilization`
- `DatabaseConnections`
- `FreeableMemory`
- `ReadLatency`, `WriteLatency`

### Recommended CloudWatch Alarms

```bash
# Unhealthy Targets Alarm
Metric: UnHealthyHostCount
Threshold: >= 1
Period: 60 seconds
Evaluation Periods: 2

# High CPU Alarm
Metric: CPUUtilization (EC2)
Threshold: >= 80%
Period: 300 seconds
Evaluation Periods: 2

# Database Connection Alarm
Metric: DatabaseConnections (RDS)
Threshold: >= 80% of max connections
Period: 300 seconds
Evaluation Periods: 2
```

## 💰 Cost Considerations

### Estimated Monthly Costs (ap-south-1 region)

| Service | Configuration | Estimated Cost |
|---------|--------------|----------------|
| EC2 Instances (t3.micro) | 2 instances × 730 hours | ~$15 |
| Application Load Balancer | 1 ALB + LCU charges | ~$20 |
| NAT Gateway | 2 NAT Gateways | ~$66 |
| RDS MySQL (db.t3.micro) | Multi-AZ | ~$30 |
| Data Transfer | Moderate usage | ~$10 |
| **Total** | | **~$141/month** |

### Cost Optimization Tips

1. **Development/Testing**:
   - Use single NAT Gateway instead of 2
   - Use single-AZ RDS instead of Multi-AZ
   - Reduce to t3.micro or t3.nano instances

2. **Production**:
   - Consider Reserved Instances for predictable workloads (up to 72% savings)
   - Use Savings Plans
   - Implement Auto Scaling to scale down during off-peak hours

3. **Alternative Approaches**:
   - Use NAT Instances instead of NAT Gateway (cheaper but less reliable)
   - Consider AWS App Runner or Elastic Beanstalk for simpler deployments

## 🔧 Troubleshooting

### Common Issues

**1. Unhealthy Targets in ALB**

```bash
# Check security group rules
# Ensure App SG allows traffic from ALB SG on port 80/8080

# Check application is running
ssh to instance (via Systems Manager)
sudo systemctl status httpd

# Check health check path
curl localhost/
```

**2. Database Connection Issues**

```bash
# Verify security group
# Ensure DB SG allows traffic from App SG on port 3306

# Verify endpoint
aws rds describe-db-instances \
  --db-instance-identifier ha-db-subnet-group \
  --query 'DBInstances[0].Endpoint.Address'

# Test from EC2
telnet <rds-endpoint> 3306
```

**3. Auto Scaling Not Working**

```bash
# Check scaling policies
aws autoscaling describe-policies \
  --auto-scaling-group-name ha-asg

# Check CloudWatch alarms
aws cloudwatch describe-alarms

# Review scaling activities
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name ha-asg
```

**4. Instances Can't Reach Internet**

```bash
# Verify NAT Gateway in route table
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=<private-subnet-id>"

# Check NAT Gateway status
aws ec2 describe-nat-gateways
```

## 🚀 Future Enhancements

### Short-term Improvements

1. **SSL/TLS Implementation**:
   - Request SSL certificate from AWS Certificate Manager
   - Add HTTPS:443 listener to ALB
   - Redirect HTTP to HTTPS

2. **Enhanced Monitoring**:
   - Set up CloudWatch Dashboard
   - Configure SNS notifications for alarms
   - Enable VPC Flow Logs

3. **Backup & Recovery**:
   - Configure AWS Backup
   - Test RDS restore procedures
   - Document recovery time objectives (RTO)

### Long-term Enhancements

1. **Security**:
   - Implement AWS WAF on ALB
   - Enable GuardDuty for threat detection
   - Use AWS Secrets Manager for database credentials
   - Enable AWS Config for compliance

2. **CI/CD Pipeline**:
   - Implement CodePipeline for automated deployments
   - Use CodeBuild for application builds
   - Implement blue-green deployments

3. **Caching Layer**:
   - Add ElastiCache (Redis/Memcached)
   - Implement CloudFront CDN

4. **Containerization**:
   - Migrate to ECS/Fargate
   - Implement service mesh (AWS App Mesh)

5. **Observability**:
   - Implement distributed tracing (AWS X-Ray)
   - Centralized logging (CloudWatch Logs Insights)
   - Application Performance Monitoring (APM)

## 📸 Screenshots

Project screenshots are available in the `/screenshots` directory:

- `alb-details-page.png` - Application Load Balancer configuration
- `alb-listeners.png` - ALB listener rules
- `asg-desired-capacity.jpeg` - Auto Scaling Group capacity settings
- `database-security-group-rule.jpeg` - RDS security group configuration
- `database-subnet-group.png` - RDS subnet group spanning multiple AZs
- `ec2-instance-iam-role-attached.png` - IAM role attached to EC2
- `ec2-instance-page.png` - EC2 instances running in ASG
- `ec2-ssh.png` - SSH access to EC2 instance
- `healthy.png` - Target group health status
- `RDS_instance_page__public_access___NO_.png` - RDS configuration with public access disabled
- `route-table-private-1a.png` - Private route table for AZ 1a
- `route-table-private-1b.png` - Private route table for AZ 1b
- `route-table-public.png` - Public route table with IGW
- `vpc-resource-map.jpeg` - VPC resource visualization

## 📝 Project Information

- **Project Name**: Highly Available Multi-Tier Web Application
- **AWS Region**: ap-south-1 (Mumbai)
- **Deployment Date**: February 9, 2026
- **Architecture Pattern**: Three-tier (Web, Application, Database)
- **High Availability**: Multi-AZ deployment across 2 AZs

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## 📄 License

This project is licensed under the MIT License - see the LICENSE file for details.

## 👤 Author

**Shivam Rohilla**
- AWS Account: xxxxxxxxxxxx
- Region: Asia Pacific (Mumbai)

## 📝 Project Status

**Note**: This architecture was deployed and tested in February 2026. All AWS resources have been terminated to avoid costs while maintaining free tier eligibility. The documentation and screenshots serve as proof of concept and implementation.

For reproduction, follow the deployment guide in this README.

## 🙏 Acknowledgments

- AWS Well-Architected Framework
- AWS Solution Architects
- AWS Documentation

---

**Note**: This architecture is designed for learning and demonstration purposes. For production deployments, additional security hardening, compliance requirements, and organizational policies should be considered.
