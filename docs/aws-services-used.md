# AWS Services Used

## 1. Project Overview

This project implements a multi-region disaster recovery architecture using Amazon Web Services (AWS). The Mumbai region (`ap-south-1`) is the primary environment, and US East (N. Virginia, `us-east-1`) is the disaster recovery environment.

The project focuses on application availability, cross-region data replication, DNS-based failover, monitoring, and recovery testing.

## 2. AWS Services and Their Roles

| AWS Service | Purpose | Project Usage |
|---|---|---|
| Amazon VPC | Provides isolated networking | Separate VPCs in Mumbai and N. Virginia |
| Public Subnets | Host internet-facing resources | Application Load Balancers |
| Private Subnets | Isolate application and database resources | EC2 application instances and database subnets |
| Internet Gateway | Enables internet connectivity for public resources | Attached to both VPCs |
| NAT Gateway | Enables outbound internet access from private subnets | Configured in both regions |
| Route Tables | Control subnet traffic routing | Public and private subnet routing |
| Security Groups | Control network traffic | Separate ALB, application, and database security groups |
| Amazon EC2 | Runs the web application | Amazon Linux instances with Nginx |
| EC2 Auto Scaling | Maintains desired application capacity | Two application instances per region |
| Application Load Balancer | Distributes HTTP traffic | Separate ALBs for Mumbai and N. Virginia |
| Amazon Aurora MySQL | Relational database service | Primary database in Mumbai and secondary cluster in N. Virginia |
| Aurora Cross-Region Replication | Replicates database data | Verified replication to the DR cluster |
| Amazon S3 | Stores objects | Source and destination buckets |
| S3 Cross-Region Replication | Replicates new objects across regions | Mumbai source bucket to N. Virginia destination bucket |
| Amazon Route 53 | DNS resolution and failover routing | Primary and secondary alias records |
| Amazon CloudWatch | Monitors metrics and alarms | Primary Auto Scaling Group CPU alarm |
| AWS IAM | Controls permissions | EC2 role for Systems Manager access |
| AWS Systems Manager | Secure instance management | Access to private EC2 instances without public SSH |
| AWS Backup | Provides backup management capabilities | A backup vault was created as an optional component |

## 3. Regional Configuration

### Mumbai — Primary Region

- Region: `ap-south-1`
- VPC: `DR-Primary-VPC`
- VPC CIDR: `10.0.0.0/16`
- Application Load Balancer: `DR-Primary-ALB`
- Target Group: `DR-Primary-TG`
- Auto Scaling Group: `DR-Primary-ASG`
- Launch Template: `DR-Primary-App-LT`
- Aurora Cluster: `dr-primary-aurora`
- Backup Vault: `DR-Primary-Backup-Vault`

### N. Virginia — Disaster Recovery Region

- Region: `us-east-1`
- VPC: `DR-Secondary-VPC`
- VPC CIDR: `10.1.0.0/16`
- Application Load Balancer: `DR-Secondary-ALB`
- Target Group: `DR-Secondary-TG`
- Auto Scaling Group: `DR-Secondary-ASG`
- Launch Template: `DR-Secondary-App-LT`
- Aurora Cluster: `dr-secondary-aurora`

## 4. High Availability and Disaster Recovery

The application layer uses Application Load Balancers and Auto Scaling Groups across two Availability Zones in each region.

Aurora cross-region replication provides a secondary copy of database data. S3 Cross-Region Replication provides a separate copy of replicated objects.

Route 53 failover records route application traffic to the primary ALB while it is healthy and can route traffic to the secondary ALB when the primary is considered unhealthy.

## 5. Security Configuration

- EC2 application instances are placed in private subnets.
- Database clusters are configured without public access.
- Security groups restrict communication between application and database tiers.
- S3 Block Public Access is enabled.
- S3 bucket versioning is enabled for replication.
- IAM roles and Systems Manager provide controlled instance access.
- Credentials and private keys are not stored in the repository.

## 6. Project Scope Notes

- CloudFront was not configured as part of the tested implementation.
- AWS Backup was optional; creating a vault alone does not mean scheduled backups were configured.
- Database promotion and automatic write recovery were not validated.
- No measured RTO or RPO values are claimed.
