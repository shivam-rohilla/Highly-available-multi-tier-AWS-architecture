# Infrastructure Specifications

Complete technical specifications for all AWS resources deployed in this architecture.

## Table of Contents
- [VPC Configuration](#vpc-configuration)
- [Subnet Configuration](#subnet-configuration)
- [Route Tables](#route-tables)
- [Internet Gateway](#internet-gateway)
- [NAT Gateways](#nat-gateways)
- [Security Groups](#security-groups)
- [Load Balancer](#load-balancer)
- [Target Groups](#target-groups)
- [Auto Scaling](#auto-scaling)
- [Launch Template](#launch-template)
- [EC2 Instances](#ec2-instances)
- [RDS Database](#rds-database)
- [IAM Roles & Policies](#iam-roles--policies)

---

## VPC Configuration

### VPC Details
```yaml
Name: ha-vpc-vpc
VPC ID: vpc-03813def90c3df26b
CIDR Block: 10.0.0.0/16
Region: ap-south-1 (Mumbai)
Tenancy: Default
DNS Resolution: Enabled
DNS Hostnames: Enabled
```

### VPC Endpoints
```yaml
# Optional - Can be added for cost optimization
S3 Gateway Endpoint:
  Type: Gateway
  Service: com.amazonaws.ap-south-1.s3
  Route Tables: All private route tables
```

---

## Subnet Configuration

### Public Subnets

**Public Subnet 1**
```yaml
Name: ha-vpc-subnet-public1-ap-south-1a
Subnet ID: subnet-0a87b1bf4d04d0946
CIDR: 10.0.0.0/24
Availability Zone: ap-south-1a
Available IPs: 251
Auto-assign Public IP: Yes
Route Table: ha-vpc-rtb-public
```

**Public Subnet 2**
```yaml
Name: ha-vpc-subnet-public2-ap-south-1b
Subnet ID: subnet-038899f25cfe56d1c
CIDR: 10.0.1.0/24
Availability Zone: ap-south-1b
Available IPs: 251
Auto-assign Public IP: Yes
Route Table: ha-vpc-rtb-public
```

### Private App Tier Subnets

**Private App Subnet 1**
```yaml
Name: ha-vpc-subnet-private1-ap-south-1a
Subnet ID: subnet-0eef64431d9fe3b87
CIDR: 10.0.11.0/24
Availability Zone: ap-south-1a
Available IPs: 251
Auto-assign Public IP: No
Route Table: ha-vpc-rtb-private1-ap-south-1a
```

**Private App Subnet 2**
```yaml
Name: ha-vpc-subnet-private2-ap-south-1b
Subnet ID: subnet-089266d7f32f146e6
CIDR: 10.0.12.0/24
Availability Zone: ap-south-1b
Available IPs: 251
Auto-assign Public IP: No
Route Table: ha-vpc-rtb-private2-ap-south-1b
```

### Private Database Subnets

**DB Subnet 1 (Inferred)**
```yaml
Availability Zone: ap-south-1a
CIDR: 10.0.21.0/24 (example)
Purpose: RDS Primary
Auto-assign Public IP: No
```

**DB Subnet 2 (Inferred)**
```yaml
Availability Zone: ap-south-1b
CIDR: 10.0.22.0/24 (example)
Purpose: RDS Standby
Auto-assign Public IP: No
```

---

## Route Tables

### Public Route Table
```yaml
Name: ha-vpc-rtb-public
Route Table ID: rtb-04de5e7d87fdcaf53
VPC: vpc-03813def90c3df26b

Routes:
  - Destination: 10.0.0.0/16
    Target: local
    Status: Active
  
  - Destination: 0.0.0.0/0
    Target: igw-0f0ebde283b1d165a
    Status: Active

Associated Subnets:
  - subnet-0a87b1bf4d04d0946 (public1-1a)
  - subnet-038899f25cfe56d1c (public2-1b)
```

### Private Route Table 1 (AZ-1a)
```yaml
Name: ha-vpc-rtb-private1-ap-south-1a
Route Table ID: rtb-0a555d2e488ce8c65
VPC: vpc-03813def90c3df26b

Routes:
  - Destination: 10.0.0.0/16
    Target: local
    Status: Active
  
  - Destination: 0.0.0.0/0
    Target: nat-0c85894a73634d5fb
    Status: Active
  
  - Destination: pl-78a54011 (S3 prefix list)
    Target: vpce-0d436b07a6579db2b
    Status: Active

Associated Subnets:
  - subnet-0eef64431d9fe3b87 (private1-1a)
```

### Private Route Table 2 (AZ-1b)
```yaml
Name: ha-vpc-rtb-private2-ap-south-1b
Route Table ID: rtb-0d2b0826dc267592f
VPC: vpc-03813def90c3df26b

Routes:
  - Destination: 10.0.0.0/16
    Target: local
    Status: Active
  
  - Destination: 0.0.0.0/0
    Target: nat-043f071fc43218476
    Status: Active
  
  - Destination: pl-78a54011 (S3 prefix list)
    Target: vpce-0d436b07a6579db2b
    Status: Active

Associated Subnets:
  - subnet-089266d7f32f146e6 (private2-1b)
```

---

## Internet Gateway

```yaml
Name: ha-vpc-igw
Internet Gateway ID: igw-0f0ebde283b1d165a
VPC: vpc-03813def90c3df26b
State: Attached
```

---

## NAT Gateways

### NAT Gateway 1 (AZ-1a)
```yaml
NAT Gateway ID: nat-0c85894a73634d5fb
Subnet: subnet-0a87b1bf4d04d0946 (public1-1a)
Availability Zone: ap-south-1a
State: Available
Elastic IP: (Allocated automatically)
Network Interface: (Created automatically)
Used By: Private route table 1
```

### NAT Gateway 2 (AZ-1b)
```yaml
NAT Gateway ID: nat-043f071fc43218476
Subnet: subnet-038899f25cfe56d1c (public2-1b)
Availability Zone: ap-south-1b
State: Available
Elastic IP: (Allocated automatically)
Network Interface: (Created automatically)
Used By: Private route table 2
```

**Cost Consideration**: Each NAT Gateway costs ~$33/month + data processing charges

---

## Security Groups

### ALB Security Group
```yaml
Name: web-tier-sg (inferred)
Security Group ID: (Not visible in screenshots)
VPC: vpc-03813def90c3df26b
Description: Security group for Application Load Balancer

Inbound Rules:
  - Type: HTTP
    Protocol: TCP
    Port: 80
    Source: 0.0.0.0/0
    Description: Allow HTTP from internet
  
  - Type: HTTPS
    Protocol: TCP
    Port: 443
    Source: 0.0.0.0/0
    Description: Allow HTTPS from internet

Outbound Rules:
  - Type: All Traffic
    Protocol: All
    Port: All
    Destination: 0.0.0.0/0
    Description: Allow all outbound traffic
```

### Application Tier Security Group
```yaml
Name: app-tier-sg (inferred)
Security Group ID: sg-01051589c04c07c48
VPC: vpc-03813def90c3df26b
Description: Security group for application instances

Inbound Rules:
  - Type: HTTP
    Protocol: TCP
    Port: 80
    Source: sg-<alb-security-group>
    Description: Allow HTTP from ALB
  
  - Type: Custom TCP
    Protocol: TCP
    Port: 8080
    Source: sg-<alb-security-group>
    Description: Allow custom app port from ALB

Outbound Rules:
  - Type: All Traffic
    Protocol: All
    Port: All
    Destination: 0.0.0.0/0
    Description: Allow all outbound traffic
```

### Database Security Group
```yaml
Name: rds-mysql-from-app
Security Group ID: sg-0316df5bd8aeb420e
VPC: vpc-03813def90c3df26b
Description: Allow MySQL access from application tier

Inbound Rules:
  - Type: MySQL/Aurora
    Protocol: TCP
    Port: 3306
    Source: sg-01051589c04c07c48 (app-tier-sg)
    Description: Allow MySQL from application tier
  
  - Type: All Traffic
    Protocol: All
    Port: All
    Source: sg-0316df5bd8aeb420e (self-reference)
    Description: Allow traffic within security group

Outbound Rules:
  - Type: All Traffic
    Protocol: All
    Port: All
    Destination: 0.0.0.0/0
    Description: Allow all outbound traffic
```

---

## Load Balancer

### Application Load Balancer
```yaml
Name: ha-alb
ARN: arn:aws:elasticloadbalancing:ap-south-1:951907774892:loadbalancer/app/ha-alb/9cbb3f22aecd4dbc
DNS Name: ha-alb-1207234725.ap-south-1.elb.amazonaws.com
Type: Application
Scheme: Internet-facing
IP Address Type: IPv4
VPC: vpc-0c2b5cc060022dec7
Availability Zones:
  - ap-south-1c (subnet-0895b8e147d4608e9)
  - ap-south-1a (subnet-0a87b1bf4d04d0946)
  - ap-south-1b (subnet-038899f25cfe56d1c)
State: Active
Created: February 9, 2026, 16:12 (UTC+05:30)

Security Settings:
  Security Groups: (ALB security group)
  Deletion Protection: Disabled (enable for production)
  
Access Logs:
  Status: Disabled
  S3 Bucket: (Configure for production)
  
Monitoring:
  CloudWatch Metrics: Enabled
```

### Listeners
```yaml
Listener 1:
  Protocol: HTTP
  Port: 80
  Default Action: Forward to target group
  Target Group: ha-target-group
  Target Group ARN: arn:aws:elasticloadbalancing:ap-south-1:951907774892:targetgroup/ha-target-group/b6a5cd1a5219b993
  Stickiness: Off
  Rules: 1 (Default rule)
```

---

## Target Groups

```yaml
Name: ha-target-group
ARN: arn:aws:elasticloadbalancing:ap-south-1:951907774892:targetgroup/ha-target-group/b6a5cd1a5219b993
Target Type: Instance
Protocol: HTTP
Port: 80
Protocol Version: HTTP1
VPC: vpc-0c2b5cc060022dec7

Health Check Settings:
  Protocol: HTTP
  Path: /
  Port: Traffic port
  Healthy Threshold: 2
  Unhealthy Threshold: 3
  Timeout: 5 seconds
  Interval: 30 seconds
  Success Codes: 200

Attributes:
  Deregistration Delay: 300 seconds
  Stickiness: Disabled
  Load Balancing Algorithm: Round robin

Registered Targets:
  Total: 2
  Healthy: 2
  Unhealthy: 0
  Unused: 0
  Initial: 0
  Draining: 0

Target Details:
  - Instance ID: i-017b4f3a81c8e2ed7
    Zone: ap-south-1c
    Status: Healthy
  
  - Instance ID: i-0b724c1ff23fc3aff
    Zone: ap-south-1a
    Status: Healthy
```

---

## Auto Scaling

### Auto Scaling Group
```yaml
Name: ha-asg
ARN: arn:aws:autoscaling:ap-south-1:951907774892:autoScalingGroup:496da93c-3638-4f4d-a23e-f55d04f580c5:autoScalingGroupName/ha-asg
Launch Template: (Configured)
VPC: (Inherited from subnets)
Subnets:
  - subnet-0eef64431d9fe3b87 (private1-1a)
  - subnet-089266d7f32f146e6 (private2-1b)

Capacity:
  Desired Capacity: 2
  Minimum Capacity: 2
  Maximum Capacity: 4
  Current Instances: 2

Instance Configuration:
  Instance Type: t3.micro
  Instance Distribution: Balanced across AZs
  
Health Check:
  Type: ELB
  Grace Period: 300 seconds
  
Load Balancing:
  Target Groups: ha-target-group
  Health Check Type: ELB
  
Monitoring:
  CloudWatch Metrics: Enabled
  Detailed Monitoring: Disabled (can enable for production)

Tags:
  - Key: Name
    Value: ha-ec2
    Propagate at Launch: Yes

Created: Mon Feb 09 2026 16:00:03 GMT+0530 (India Standard Time)
```

### Scaling Policies
```yaml
Policy Name: Target Tracking Policy
Policy Type: TargetTrackingScaling
Metric: Average CPU Utilization
Target Value: 70%
Warmup Time: 300 seconds

Behavior:
  Scale Out:
    Cooldown: 300 seconds
    
  Scale In:
    Cooldown: 300 seconds
    Protection: Instance scale-in protection disabled
```

---

## Launch Template

```yaml
Name: app-tier-template (inferred)
Version: Latest
AMI: Amazon Linux 2023
Instance Type: t3.micro
Key Pair: (Configured for SSH access)

Network Interfaces:
  Device Index: 0
  Delete on Termination: Yes
  Security Groups: app-tier-sg

IAM Instance Profile:
  Role: EC2-SSM-Core-Role
  
Monitoring:
  Detailed Monitoring: Disabled

Advanced Details:
  User Data: (Application bootstrap script)
  Metadata Version: V2
  Metadata Response Hop Limit: 1
  
Storage:
  Root Volume:
    Volume Type: gp3
    Size: 8 GB
    Delete on Termination: Yes
    Encrypted: Yes (default EBS encryption)
```

### User Data Script (Example)
```bash
#!/bin/bash
# Update system
yum update -y

# Install web server
yum install -y httpd

# Install MySQL client
yum install -y mariadb105

# Start and enable web server
systemctl start httpd
systemctl enable httpd

# Create a simple web page
HOSTNAME=$(hostname -f)
echo "Highly Available App - $HOSTNAME" > /var/www/html/index.html

# Configure application
# (Add your application-specific configuration here)
```

---

## EC2 Instances

### Instance 1
```yaml
Instance ID: i-0b724c1ff23fc3aff
Name: ha-ec2
Instance State: Running
Instance Type: t3.micro
Availability Zone: ap-south-1a
VPC: vpc-0c2b5cc060022dec7
Subnet: subnet-0a87b1bf4d04d0946

Networking:
  Private IPv4 Address: 172.31.34.110
  Public IPv4 Address: 13.235.51.122
  Private DNS: ip-172-31-34-110.ap-south-1.compute.internal
  Public DNS: ec2-13-235-51-122.ap-south-1.compute.amazonaws.com

Security Groups: (App tier security group)
IAM Role: EC2-SSM-Core-Role
Auto Scaling Group: ha-asg

Status Checks:
  System Status: Passed (3/3 checks)
  Instance Status: Passed (3/3 checks)

Monitoring:
  CloudWatch Monitoring: Basic
  Detailed Monitoring: Disabled
```

### Instance 2
```yaml
Instance ID: i-017b4f3a81c8e2ed7
Name: ha-ec2
Instance State: Running
Instance Type: t3.micro
Availability Zone: ap-south-1c
VPC: vpc-0c2b5cc060022dec7
Subnet: (Private subnet in AZ-1c)

Networking:
  Private IPv4 Address: 172.31.20.242
  Public IPv4 Address: (None - in private subnet)
  Private DNS: ip-172-31-20-242.ap-south-1.compute.internal

Security Groups: (App tier security group)
IAM Role: EC2-SSM-Core-Role
Auto Scaling Group: ha-asg

Status Checks:
  System Status: Passed (3/3 checks)
  Instance Status: Passed (3/3 checks)
```

---

## RDS Database

### Database Instance
```yaml
DB Instance Identifier: ha-db-subnet-group
Engine: MySQL
Engine Version: 8.4.7
License Model: general-public-license
DB Instance Class: db.t3.micro
vCPUs: 2
Memory: 1 GiB

Multi-AZ Deployment: Yes
Primary AZ: ap-south-1a
Standby AZ: ap-south-1b

Storage:
  Type: gp3 (General Purpose SSD)
  Allocated: 20 GiB
  IOPS: 3000 (baseline)
  Throughput: 125 MB/s
  Storage Autoscaling: Enabled
  Maximum Storage: 1000 GiB

Connectivity:
  Endpoint: ha-db-subnet-group.cdiq4uegwb4l.ap-south-1.rds.amazonaws.com
  Port: 3306
  VPC: ha-vpc-vpc (vpc-03813def90c3df26b)
  Subnet Group: ha-db-subnet-group
  Publicly Accessible: No
  Availability Zone: ap-south-1a

Subnets:
  - subnet-089266d7f32f146e6 (AZ: ap-south-1b, CIDR: 10.0.12.0/24)
  - subnet-0eef64431d9fe3b87 (AZ: ap-south-1a, CIDR: 10.0.11.0/24)

Security:
  VPC Security Groups: rds-mysql-from-app (sg-0316df5bd8aeb420e) - Active
  Encryption at Rest: Enabled
  Encryption Key: (AWS managed key)
  Certificate Authority: rds-ca-rsa2048-g1
  Certificate Expiry: May 20, 2061, 00:10 (UTC+05:30)
  DB Certificate Expiry: February 10, 2027, 00:00 (UTC+05:30)

Backup:
  Automated Backups: Enabled
  Backup Retention Period: 7 days
  Backup Window: (Automated)
  Latest Restorable Time: (Updated continuously)
  
Maintenance:
  Auto Minor Version Upgrade: Enabled
  Maintenance Window: (Configured)

Monitoring:
  Enhanced Monitoring: Disabled (can enable for production)
  Performance Insights: Disabled (can enable for production)
  CloudWatch Logs Exports: None (can enable error, slow query, general logs)

Database Configuration:
  Master Username: admin
  DB Name: (Default or configured)
  Port: 3306
  Parameter Group: (Default mysql8.4)
  Option Group: (Default mysql-8-4)
```

### DB Subnet Group
```yaml
Name: ha-db-subnet-group
Description: ha-db-subnet-group
VPC: vpc-03813def90c3df26b
Subnets:
  - Subnet: subnet-0eef64431d9fe3b87
    AZ: ap-south-1a
    CIDR: 10.0.11.0/24
    Status: Active
    
  - Subnet: subnet-089266d7f32f146e6
    AZ: ap-south-1b
    CIDR: 10.0.12.0/24
    Status: Active
```

---

## IAM Roles & Policies

### EC2 Instance Role
```yaml
Role Name: EC2-SSM-Core-Role
ARN: arn:aws:iam::951907774892:role/EC2-SSM-Core-Role
Path: /
Max Session Duration: 1 hour
Created: (Date)

Attached Policies:
  - AmazonSSMManagedInstanceCore
    ARN: arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
    Description: AWS managed policy for core Systems Manager functionality

Trust Relationship:
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "ec2.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }

Permissions Summary:
  - SSM Agent communication
  - Systems Manager Session Manager access
  - CloudWatch Logs (for SSM)
  - S3 access for SSM documents
```

### Additional Recommended Policies
```yaml
CloudWatch Logs Access:
  Policy Name: CloudWatchLogsAccess
  Permissions:
    - logs:CreateLogGroup
    - logs:CreateLogStream
    - logs:PutLogEvents
    - logs:DescribeLogStreams

S3 Access (if needed):
  Policy Name: S3ApplicationAccess
  Permissions:
    - s3:GetObject
    - s3:PutObject
    - s3:ListBucket
  Resource: arn:aws:s3:::your-app-bucket/*

Secrets Manager (for DB credentials):
  Policy Name: SecretsManagerAccess
  Permissions:
    - secretsmanager:GetSecretValue
  Resource: arn:aws:secretsmanager:ap-south-1:951907774892:secret:db-credentials-*
```

---

## Resource Tags

### Tagging Strategy
```yaml
Standard Tags (Applied to all resources):
  - Key: Project
    Value: HighlyAvailableWebApp
  
  - Key: Environment
    Value: Production
  
  - Key: ManagedBy
    Value: Manual / Terraform / CloudFormation
  
  - Key: Owner
    Value: Shivam Rohilla
  
  - Key: CostCenter
    Value: Engineering
  
  - Key: Backup
    Value: Daily / Weekly
```

---

## Network Specifications Summary

```yaml
Total Resources:
  VPC: 1
  Subnets: 6 (2 public, 4 private)
  Internet Gateway: 1
  NAT Gateways: 2
  Route Tables: 3
  Security Groups: 3
  Load Balancers: 1
  Target Groups: 1
  Auto Scaling Groups: 1
  EC2 Instances: 2 (can scale to 4)
  RDS Instances: 1 (Multi-AZ = 2 physical instances)

IP Address Allocation:
  Total Available: 65,536 (/16)
  Public Subnets: 512 IPs (2 x /24)
  Private App Subnets: 512 IPs (2 x /24)
  Private DB Subnets: 512 IPs (2 x /24)
  Reserved for future: ~63,000 IPs

Bandwidth:
  NAT Gateway: Up to 45 Gbps
  ALB: Auto-scales based on traffic
  EC2 t3.micro: Up to 5 Gbps network
  RDS db.t3.micro: Moderate network performance
```

---

## Performance Specifications

```yaml
Compute:
  Instance Type: t3.micro
  vCPUs: 2
  Memory: 1 GiB
  Network Performance: Up to 5 Gigabit
  EBS Bandwidth: Up to 2,085 Mbps

Storage:
  EBS Volume Type: gp3
  IOPS: 3,000 (baseline)
  Throughput: 125 MB/s
  RDS Storage: gp3, 20 GB, autoscaling enabled

Load Balancer:
  Connection Timeout: 60 seconds
  Idle Timeout: 60 seconds
  Request Routing: Round robin
  Cross-zone Load Balancing: Enabled

Auto Scaling:
  Scale Out: When CPU > 70% for 5 minutes
  Scale In: When CPU < 70% for 5 minutes
  Cooldown: 300 seconds
```

---

**Last Updated**: February 10, 2026  
**Document Version**: 1.0  
**AWS Account**: 951907774892  
**Region**: ap-south-1
