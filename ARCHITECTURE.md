# Architecture Diagram

## High-Level Architecture

```
                                    Internet
                                       |
                                       |
                        +--------------+---------------+
                        |                              |
                   Internet Gateway              Route 53 (Optional)
                        |                              |
                        +------------------------------+
                                       |
                        +--------------v--------------+
                        |   Application Load Balancer |
                        |         (ha-alb)            |
                        |   Internet-facing           |
                        +--------------+--------------+
                                       |
                +----------------------+----------------------+
                |                                             |
    +-----------v-----------+                    +-----------v-----------+
    | Availability Zone 1a  |                    | Availability Zone 1b  |
    |                       |                    |                       |
    |  +----------------+   |                    |   +----------------+  |
    |  | Public Subnet  |   |                    |   | Public Subnet  |  |
    |  | 10.0.0.0/24    |   |                    |   | 10.0.1.0/24    |  |
    |  +-------+--------+   |                    |   +--------+-------+  |
    |          |            |                    |            |          |
    |  +-------v--------+   |                    |   +--------v-------+  |
    |  | NAT Gateway    |   |                    |   | NAT Gateway    |  |
    |  +-------+--------+   |                    |   +--------+-------+  |
    |          |            |                    |            |          |
    |  +-------v--------+   |                    |   +--------v-------+  |
    |  | Private Subnet |   |                    |   | Private Subnet |  |
    |  | App Tier       |   |                    |   | App Tier       |  |
    |  | 10.0.11.0/24   |   |                    |   | 10.0.12.0/24   |  |
    |  +-------+--------+   |                    |   +--------+-------+  |
    |          |            |                    |            |          |
    |  +-------v--------+   |                    |   +--------v-------+  |
    |  | EC2 Instance   |   |                    |   | EC2 Instance   |  |
    |  | (Auto Scaling) |   |                    |   | (Auto Scaling) |  |
    |  +-------+--------+   |                    |   +--------+-------+  |
    |          |            |                    |            |          |
    |  +-------v--------+   |                    |   +--------v-------+  |
    |  | Private Subnet |   |                    |   | Private Subnet |  |
    |  | DB Tier        |   |                    |   | DB Tier        |  |
    |  +-------+--------+   |                    |   +--------+-------+  |
    |          |            |                    |            |          |
    +-----------+-----------+                    +-----------+-----------+
                |                                             |
                +--------------------+------------------------+
                                     |
                            +--------v--------+
                            |   RDS MySQL     |
                            |   Multi-AZ      |
                            |   Primary: AZ1a |
                            |   Standby: AZ1b |
                            +-----------------+
```

## Network Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                           Internet                              │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Internet Gateway│
                    └────────┬────────┘
                             │
                             ▼
            ┌────────────────────────────────┐
            │  Application Load Balancer     │
            │  Security Group: Allow 80/443  │
            └────────────┬───────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         ▼                               ▼
┌──────────────────┐          ┌──────────────────┐
│   EC2 Instance   │          │   EC2 Instance   │
│   AZ: 1a         │          │   AZ: 1b         │
│   Private Subnet │          │   Private Subnet │
│   SG: From ALB   │          │   SG: From ALB   │
└────────┬─────────┘          └─────────┬────────┘
         │                               │
         └───────────────┬───────────────┘
                         │
                         ▼
              ┌────────────────────┐
              │    RDS MySQL       │
              │    Multi-AZ        │
              │    Private Subnets │
              │    SG: From App    │
              └────────────────────┘
```

## Security Groups Flow

```
                    Internet (0.0.0.0/0)
                            │
                            │ HTTP:80, HTTPS:443
                            ▼
                    ┌───────────────┐
                    │   ALB-SG      │
                    └───────┬───────┘
                            │
                            │ HTTP:80, Custom TCP:8080
                            ▼
                    ┌───────────────┐
                    │   App-SG      │
                    │ (sg-01051...) │
                    └───────┬───────┘
                            │
                            │ MySQL:3306
                            ▼
                    ┌───────────────┐
                    │   DB-SG       │
                    │(rds-mysql-... )│
                    └───────────────┘
```

## Auto Scaling Architecture

```
                    ┌─────────────────────┐
                    │  CloudWatch Alarms  │
                    │  CPU > 70%          │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │  Auto Scaling Group │
                    │  Min: 2             │
                    │  Desired: 2         │
                    │  Max: 4             │
                    └──────────┬──────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
                ▼              ▼              ▼
         ┌──────────┐   ┌──────────┐   ┌──────────┐
         │Instance 1│   │Instance 2│   │Instance N│
         │   AZ-1a  │   │   AZ-1b  │   │ (Scaling)│
         └──────────┘   └──────────┘   └──────────┘
                │              │              │
                └──────────────┼──────────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │   Target Group      │
                    │   Health Checks     │
                    └─────────────────────┘
```

## Data Flow for a Request

```
1. User Request
   │
   ├─> DNS Resolution (Optional Route 53)
   │
   ├─> ALB receives request on Port 80
   │   └─> Security Group Check: Allow from 0.0.0.0/0
   │
   ├─> ALB forwards to Target Group
   │   └─> Health Check: Only healthy targets
   │
   ├─> Request routed to EC2 instance
   │   └─> Security Group Check: Allow from ALB SG
   │   └─> Instance processes request
   │
   ├─> Application queries database
   │   └─> Security Group Check: Allow MySQL from App SG
   │   └─> RDS returns data
   │
   ├─> Response sent back through ALB
   │
   └─> User receives response
```

## High Availability Flow

```
Normal Operation:
    ALB
     │
     ├──> Instance 1 (AZ-1a) ✓ Healthy
     │
     └──> Instance 2 (AZ-1b) ✓ Healthy


Failure Scenario (Instance 1 Fails):
    ALB
     │
     ├──> Instance 1 (AZ-1a) ✗ Unhealthy
     │    └─> Auto Scaling detects failure
     │        └─> Launches new instance
     │
     └──> Instance 2 (AZ-1b) ✓ Healthy (Handling all traffic)


AZ Failure Scenario:
    ALB
     │
     ├──> AZ-1a ✗ Entire AZ Down
     │    └─> All instances unavailable
     │
     └──> AZ-1b ✓ Operating
          └─> Instances handle all traffic
          └─> Auto Scaling launches additional instances in AZ-1b

Database Multi-AZ:
    Primary (AZ-1a) ✗ Failed
         │
         ├─> Automatic Failover (1-2 minutes)
         │
         ▼
    Standby (AZ-1b) ✓ Promoted to Primary
         │
         └─> Applications reconnect automatically
```

## Component Relationships

```
┌────────────────────────────────────────────────────┐
│                      VPC                           │
│                  (10.0.0.0/16)                     │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │           Availability Zone 1a               │ │
│  │                                              │ │
│  │  ┌────────────────┐  ┌───────────────────┐  │ │
│  │  │ Public Subnet  │  │ Private App Sub   │  │ │
│  │  │   (10.0.0/24)  │  │  (10.0.11/24)     │  │ │
│  │  └────────────────┘  └───────────────────┘  │ │
│  │           │                    │             │ │
│  │           └────────────────────┘             │ │
│  │                                              │ │
│  │  ┌────────────────────────────────────────┐ │ │
│  │  │      Private DB Subnet                 │ │ │
│  │  └────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │           Availability Zone 1b               │ │
│  │                                              │ │
│  │  ┌────────────────┐  ┌───────────────────┐  │ │
│  │  │ Public Subnet  │  │ Private App Sub   │  │ │
│  │  │   (10.0.1/24)  │  │  (10.0.12/24)     │  │ │
│  │  └────────────────┘  └───────────────────┘  │ │
│  │           │                    │             │ │
│  │           └────────────────────┘             │ │
│  │                                              │ │
│  │  ┌────────────────────────────────────────┐ │ │
│  │  │      Private DB Subnet                 │ │ │
│  │  └────────────────────────────────────────┘ │ │
│  └──────────────────────────────────────────────┘ │
│                                                    │
└────────────────────────────────────────────────────┘

External Resources:
┌──────────────────┐  ┌──────────────────┐
│  Internet Gateway│  │  NAT Gateways    │
└──────────────────┘  └──────────────────┘

┌──────────────────┐  ┌──────────────────┐
│  Route Tables    │  │  Security Groups │
└──────────────────┘  └──────────────────┘
```

## Monitoring & Management

```
┌─────────────────────────────────────────────────────┐
│              CloudWatch Monitoring                  │
│                                                     │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │ ALB Metrics│  │ ASG Metrics│  │ RDS Metrics │  │
│  └────────────┘  └────────────┘  └─────────────┘  │
│                                                     │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐  │
│  │   Alarms   │  │ Dashboards │  │   Logs      │  │
│  └────────────┘  └────────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │   SNS Topics     │
              │ (Notifications)  │
              └──────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │ Email / SMS      │
              │ Administrators   │
              └──────────────────┘
```

## Deployment Pipeline (Future Enhancement)

```
┌──────────┐     ┌───────────┐     ┌──────────┐     ┌──────────┐
│          │     │           │     │          │     │          │
│  GitHub  │────▶│CodePipeline│────▶│CodeBuild │────▶│  Deploy  │
│          │     │           │     │          │     │          │
└──────────┘     └───────────┘     └──────────┘     └──────────┘
                                                            │
                                                            ▼
                                                   ┌─────────────────┐
                                                   │ Auto Scaling    │
                                                   │ Rolling Update  │
                                                   └─────────────────┘
```

## Legend

```
┌─────────┐
│ Symbols │
└─────────┘

│  = Connection
▼  = Direction of flow
✓  = Healthy/Active
✗  = Unhealthy/Failed
──▶ = Data flow
┌──┐ = Component/Service
```
