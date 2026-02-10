# Deployment Checklist

This checklist ensures all components are properly configured for a production-ready deployment.

## Pre-Deployment Checklist

### Planning & Design
- [x] Architecture diagram reviewed
- [x] AWS region selected (ap-south-1)
- [x] Availability Zones identified (ap-south-1a, ap-south-1b)
- [x] IP addressing scheme planned (10.0.0.0/16)
- [ ] Cost estimation completed
- [ ] Security requirements documented
- [ ] Compliance requirements identified
- [ ] Disaster recovery plan created

### AWS Account Setup
- [ ] AWS account created and verified
- [ ] IAM users created with MFA enabled
- [ ] IAM roles and policies defined
- [ ] CloudTrail enabled for audit logging
- [ ] AWS Config enabled (optional but recommended)
- [ ] Billing alerts configured
- [ ] Service limits verified

## Network Infrastructure

### VPC Configuration
- [x] VPC created (10.0.0.0/16)
- [x] Internet Gateway created and attached
- [x] Public subnets created (2 AZs)
- [x] Private app subnets created (2 AZs)
- [x] Private database subnets created (2 AZs)
- [x] NAT Gateways deployed (2 for HA)
- [x] Route tables created and associated
- [x] Public route table: 0.0.0.0/0 → IGW
- [x] Private route tables: 0.0.0.0/0 → NAT Gateway
- [ ] VPC Flow Logs enabled
- [ ] VPC endpoints created (optional)

### DNS (Optional)
- [ ] Route 53 hosted zone created
- [ ] Domain registered or transferred
- [ ] Health checks configured
- [ ] Failover routing policies set

## Security Configuration

### Security Groups
- [x] ALB security group created
  - [x] Inbound: HTTP (80) from 0.0.0.0/0
  - [ ] Inbound: HTTPS (443) from 0.0.0.0/0
  - [x] Outbound: All traffic
- [x] App tier security group created
  - [x] Inbound: HTTP from ALB SG
  - [x] Outbound: All traffic
- [x] Database security group created
  - [x] Inbound: MySQL (3306) from App SG
  - [x] Outbound: All traffic (for updates)

### IAM Configuration
- [x] EC2 instance role created (EC2-SSM-Core-Role)
- [x] SSM permissions attached
- [ ] S3 access policies (if needed)
- [ ] CloudWatch logs permissions
- [ ] Secrets Manager permissions (if using)
- [ ] Parameter Store permissions (if using)

### Additional Security
- [ ] AWS WAF configured on ALB
- [ ] GuardDuty enabled
- [ ] Security Hub enabled
- [ ] Inspector scans scheduled
- [ ] Secrets Manager configured for DB credentials
- [ ] KMS keys created for encryption
- [ ] S3 bucket for logs created (encrypted)

## Database Tier

### RDS Configuration
- [x] DB subnet group created
- [x] RDS instance created
- [x] Engine: MySQL 8.4.7
- [x] Instance class selected (db.t3.micro)
- [x] Multi-AZ deployment enabled
- [x] Storage allocated (20 GB)
- [x] Storage autoscaling enabled
- [x] Backup retention configured (7 days)
- [x] Encryption at rest enabled
- [x] Public accessibility: Disabled
- [x] Security group attached
- [ ] Parameter group customized (if needed)
- [ ] Option group configured (if needed)
- [ ] Enhanced monitoring enabled
- [ ] Performance Insights enabled
- [ ] Automated backups verified
- [ ] Manual snapshot created (baseline)

### Database Setup
- [x] Database connection tested
- [ ] Database schema created
- [ ] Sample data loaded (if applicable)
- [ ] Database users created
- [ ] User permissions configured
- [ ] SSL/TLS connection enforced

## Application Tier

### Launch Template
- [x] AMI selected (Amazon Linux 2023)
- [x] Instance type chosen (t3.micro)
- [x] Key pair created/selected
- [x] Security group attached
- [x] IAM instance profile attached
- [x] User data script created
- [ ] User data script tested
- [ ] Application dependencies installed
- [ ] Application code deployed
- [ ] Environment variables configured

### Auto Scaling Group
- [x] Launch template selected
- [x] VPC and subnets configured
- [x] Desired capacity: 2
- [x] Minimum capacity: 2
- [x] Maximum capacity: 4
- [x] Health check type: ELB
- [x] Health check grace period: 300s
- [x] Target tracking policy configured (70% CPU)
- [ ] Scaling notifications configured
- [ ] Instance refresh configured
- [ ] Lifecycle hooks (if needed)

### Application Configuration
- [ ] Application logging configured
- [ ] Log rotation set up
- [ ] CloudWatch agent installed
- [ ] Application monitoring configured
- [ ] Environment-specific configs deployed
- [ ] Database connection pooling configured
- [ ] Caching configured (if applicable)

## Load Balancer Tier

### Application Load Balancer
- [x] ALB created (internet-facing)
- [x] Subnets: Public subnets in both AZs
- [x] Security group attached
- [x] HTTP listener configured (Port 80)
- [ ] HTTPS listener configured (Port 443)
- [ ] SSL certificate attached (from ACM)
- [ ] HTTP to HTTPS redirect configured
- [x] Target group created
- [x] Health check path configured (/)
- [x] Health check interval: 30s
- [x] Healthy threshold: 2
- [x] Unhealthy threshold: 3
- [ ] Stickiness configured (if needed)
- [ ] Access logs enabled
- [ ] Connection draining configured
- [ ] Deletion protection enabled

### SSL/TLS Configuration
- [ ] Certificate requested from ACM
- [ ] Certificate validated
- [ ] Certificate attached to HTTPS listener
- [ ] Strong cipher policy selected
- [ ] SSL/TLS version enforced (TLS 1.2+)

## Monitoring & Alerting

### CloudWatch Setup
- [ ] Dashboard created
- [ ] ALB metrics added to dashboard
- [ ] ASG metrics added to dashboard
- [ ] RDS metrics added to dashboard
- [ ] EC2 metrics added to dashboard
- [ ] Custom application metrics configured

### CloudWatch Alarms
- [ ] Unhealthy target count alarm
- [ ] High CPU utilization alarm (ASG)
- [ ] High memory utilization alarm
- [ ] RDS CPU alarm
- [ ] RDS storage alarm
- [ ] RDS connection count alarm
- [ ] ALB 5xx error rate alarm
- [ ] ALB target response time alarm

### Logging
- [ ] CloudWatch log groups created
- [ ] Application logs forwarded to CloudWatch
- [ ] RDS logs enabled (error, slow query, general)
- [ ] VPC Flow Logs configured
- [ ] ALB access logs configured
- [ ] Log retention policies set
- [ ] Log insights queries created

### SNS Configuration
- [ ] SNS topics created for alarms
- [ ] Email subscriptions confirmed
- [ ] SMS subscriptions configured (if needed)
- [ ] Integration with ticketing system (if applicable)

## Backup & Disaster Recovery

### Backup Configuration
- [x] RDS automated backups enabled (7 days)
- [ ] AWS Backup plan created
- [ ] Backup vault created
- [ ] AMI creation scheduled
- [ ] Snapshot lifecycle policies defined
- [ ] Cross-region backup configured (if needed)

### Disaster Recovery Testing
- [ ] RDS restore tested
- [ ] Point-in-time recovery tested
- [ ] AMI restoration tested
- [ ] Failover to secondary AZ tested
- [ ] Recovery time objective (RTO) measured
- [ ] Recovery point objective (RPO) measured
- [ ] DR runbook created

## Testing & Validation

### Functional Testing
- [x] Application accessible via ALB DNS
- [x] Load balancer distributing traffic
- [x] Database connectivity verified
- [x] Health checks passing
- [ ] All application features working
- [ ] SSL/TLS connection working
- [ ] Session persistence working (if enabled)

### Performance Testing
- [ ] Load testing performed
- [ ] Stress testing performed
- [ ] Response time measured
- [ ] Throughput measured
- [ ] Scalability verified
- [ ] Database performance tested

### Reliability Testing
- [x] Instance termination tested (Auto Scaling recovery)
- [ ] AZ failure simulated
- [ ] Database failover tested
- [ ] Network partition tested
- [ ] Security group changes tested
- [ ] Rolling deployment tested

### Security Testing
- [ ] Security group rules validated
- [ ] IAM permissions tested (least privilege)
- [ ] Encryption verified (at rest and in transit)
- [ ] Vulnerability scanning performed
- [ ] Penetration testing completed (if required)
- [ ] SSL/TLS configuration tested
- [ ] Public accessibility verified (database should not be public)

## Cost Optimization

### Cost Controls
- [ ] Resource tagging strategy implemented
- [ ] Cost allocation tags applied
- [ ] AWS Budgets configured
- [ ] Cost anomaly detection enabled
- [ ] Reserved Instance analysis performed
- [ ] Savings Plan analysis performed
- [ ] Right-sizing recommendations reviewed

### Optimization Opportunities
- [ ] Instance types optimized
- [ ] Storage types optimized
- [ ] NAT Gateway usage reviewed
- [ ] Data transfer costs analyzed
- [ ] Auto Scaling policies optimized
- [ ] Idle resources identified and removed

## Documentation

### Technical Documentation
- [x] Architecture diagram created
- [x] Network diagram created
- [x] Security architecture documented
- [ ] Deployment guide created
- [ ] Operations runbook created
- [ ] Troubleshooting guide created
- [ ] API documentation (if applicable)

### Operational Documentation
- [ ] Incident response procedures
- [ ] Escalation matrix
- [ ] On-call rotation schedule
- [ ] Change management process
- [ ] Maintenance windows defined
- [ ] SLA/SLO definitions

## Compliance & Governance

### Compliance Checks
- [ ] Data residency requirements met
- [ ] Encryption requirements met
- [ ] Audit logging requirements met
- [ ] Access control requirements met
- [ ] Data retention policies implemented
- [ ] Privacy requirements (GDPR, etc.) met

### Governance
- [ ] Tagging policies enforced
- [ ] Resource naming conventions applied
- [ ] Service Control Policies reviewed
- [ ] AWS Organizations structure documented
- [ ] Billing account structure documented

## Pre-Production Checklist

- [ ] All infrastructure code in version control
- [ ] Infrastructure as Code validated (Terraform/CloudFormation)
- [ ] CI/CD pipeline tested
- [ ] Rollback procedures documented
- [ ] Database migration scripts tested
- [ ] Data seeding scripts ready
- [ ] Configuration management tested
- [ ] Secrets rotation tested

## Go-Live Checklist

### Final Validations
- [ ] All checklist items completed
- [ ] Stakeholder sign-off received
- [ ] Change request approved
- [ ] Maintenance window scheduled
- [ ] Rollback plan prepared
- [ ] Communication plan ready
- [ ] Monitoring dashboards ready
- [ ] On-call team briefed

### Deployment Steps
- [ ] DNS cutover plan ready
- [ ] Load testing in production environment
- [ ] Smoke tests passed
- [ ] User acceptance testing completed
- [ ] Performance baselines established
- [ ] Alerts tested and working

### Post-Deployment
- [ ] Application monitoring verified
- [ ] Performance metrics within SLAs
- [ ] No critical errors in logs
- [ ] Backup verification completed
- [ ] Security scanning completed
- [ ] Post-deployment review scheduled

## Post-Production Monitoring (First 7 Days)

- [ ] Day 1: Hourly monitoring and validation
- [ ] Day 2-3: Monitor for anomalies
- [ ] Day 4-7: Review metrics and optimize
- [ ] Week 2: Post-implementation review
- [ ] Week 4: Cost analysis and optimization

## Continuous Improvement

- [ ] Metrics reviewed weekly
- [ ] Performance optimization ongoing
- [ ] Security patches applied regularly
- [ ] Cost optimization reviews monthly
- [ ] Architecture reviews quarterly
- [ ] Disaster recovery drills quarterly
- [ ] Documentation kept up-to-date

---

## Status Legend
- [x] = Completed
- [ ] = Not Started / Pending
- [~] = In Progress (use this to track current work)

## Notes

Add any specific notes, exceptions, or customizations for your deployment:

```
Date: _____________
Deployed By: _____________
Environment: Production
AWS Account: 951907774892
Region: ap-south-1

Notes:
- 
- 
- 
```
