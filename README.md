# Multi-Region AWS Disaster Recovery System with Amazon CloudFront

## 📌 Project Overview

This project implements a multi-region disaster recovery architecture on Amazon Web Services (AWS) to improve application availability, support business continuity, and reduce downtime during application or regional failures.

The primary environment is deployed in **Mumbai (`ap-south-1`)**, while the disaster recovery environment is deployed in **US East (N. Virginia, `us-east-1`)**.

Amazon Route 53 provides DNS-based failover between the primary and disaster recovery Application Load Balancers. Amazon Aurora cross-region replication and Amazon S3 Cross-Region Replication support data protection across regions.

**Amazon CloudFront is planned as an additional global content delivery layer** to improve content delivery performance and provide HTTPS access for users around the world.

The infrastructure was built primarily through the AWS Management Console.

## 🎯 Project Objectives

- Build a multi-region disaster recovery architecture using AWS.
- Deploy applications across Mumbai and N. Virginia.
- Improve availability with Application Load Balancers and Auto Scaling Groups.
- Configure Aurora cross-region database replication.
- Configure S3 Cross-Region Replication.
- Implement Route 53 primary and secondary failover routing.
- Monitor infrastructure using Amazon CloudWatch.
- Explore Amazon CloudFront for global content delivery.
- Validate application failover through a simulated primary application failure.

## 🏗️ Architecture Overview

**Primary Region:** Mumbai — `ap-south-1`
**DR Region:** N. Virginia — `us-east-1`

```text
                         Global Users
                              |
                              v
                      Amazon CloudFront
                    (Planned CDN Integration)
                              |
                              v
                         Amazon Route 53
                       DNS / Failover
                         /         \
                        v           v
                 Mumbai Region    N. Virginia
                  ap-south-1       us-east-1
                      |                |
                     ALB              ALB
                      |                |
                     ASG              ASG
                      |                |
                  EC2 Instances   EC2 Instances
                      |                |
                      v                v
               Aurora Primary ---> Aurora DR
                 Replication       Secondary

                 S3 Primary -----> S3 DR
                       Cross-Region Replication

                     Amazon CloudWatch
                        Monitoring
```

**Architecture status:** CloudFront is a planned enhancement. The existing Route 53 failover, ALBs, application infrastructure, Aurora replication, and S3 replication were configured and tested separately.

The final production traffic flow will depend on the CloudFront origin and DNS configuration selected during implementation.

## 🛠️ AWS Services Used

| AWS Service                     | Purpose                                     |
| ------------------------------- | ------------------------------------------- |
| Amazon VPC                      | Isolated networking in both regions         |
| Public and Private Subnets      | Network segmentation                        |
| Internet Gateway                | Internet connectivity for public resources  |
| NAT Gateway                     | Outbound access for private resources       |
| Security Groups                 | Network traffic control                     |
| Amazon EC2                      | Application hosting                         |
| EC2 Auto Scaling                | Maintains desired application capacity      |
| Application Load Balancer       | Distributes HTTP requests                   |
| Amazon Aurora MySQL             | Relational database                         |
| Aurora Cross-Region Replication | Replicates database data to the DR region   |
| Amazon S3                       | Object storage                              |
| S3 Cross-Region Replication     | Replicates new objects to the DR bucket     |
| Amazon Route 53                 | DNS and failover routing                    |
| Amazon CloudWatch               | Infrastructure monitoring and alarms        |
| AWS IAM                         | Identity and access management              |
| AWS Systems Manager             | Secure access to private EC2 instances      |
| AWS Backup                      | Optional backup vault                       |
| Amazon CloudFront               | Planned global content delivery and caching |

## 🌐 Network Architecture

### Mumbai Primary Region

- VPC: `DR-Primary-VPC`
- CIDR: `10.0.0.0/16`
- Public subnets for the internet-facing ALB.
- Private application subnets across two Availability Zones.
- Private database subnets across two Availability Zones.
- NAT Gateways and an S3 Gateway Endpoint.
- Separate security groups for ALB, application, and database tiers.

### N. Virginia DR Region

- VPC: `DR-Secondary-VPC`
- CIDR: `10.1.0.0/16`
- Public subnets for the DR ALB.
- Private application subnets across two Availability Zones.
- Private database subnets across two Availability Zones.
- NAT Gateways and an S3 Gateway Endpoint.
- Separate security groups for application and database tiers.

Different VPC CIDR ranges prevent overlapping address space between the regions.

## 🖥️ Application Deployment

The application is hosted on Amazon Linux EC2 instances using Nginx.

Each region includes:

- An internet-facing Application Load Balancer.
- A target group with HTTP health checks.
- An Auto Scaling Group across two Availability Zones.
- Private EC2 application instances.
- AWS Systems Manager access without requiring public SSH.

The application pages identify the serving region, making it possible to distinguish the Mumbai primary application from the N. Virginia DR application.

## 🗄️ Database Disaster Recovery

Amazon Aurora MySQL was configured in Mumbai as the primary database, with a secondary cluster in N. Virginia.

The `drapp` database and its `users` table were used to validate replication.

A test record was created in the primary database:

| ID | Name  | Email                                          |
| -- | ----- | ---------------------------------------------- |
| 1  | Vijay | [vijay@example.com](mailto\:vijay@example.com) |

The DR database was accessed to verify that the database and test record had replicated successfully.

**Important:** Database replication is separate from application failover. A regional recovery procedure must promote the secondary database and update application database connectivity when write access is required in the DR region.

## 📦 S3 Cross-Region Replication

Two S3 buckets were configured:

- Source bucket in Mumbai.
- Destination bucket in N. Virginia.

Configuration includes:

- Versioning enabled on both buckets.
- Block Public Access enabled.
- SSE-S3 encryption.
- An enabled S3 Cross-Region Replication rule.

Replication was verified by uploading a new object to the source bucket and checking that it appeared in the destination bucket.

Objects created before the replication rule was enabled are not automatically replicated by default.

## 🔀 Route 53 DNS Failover

A public hosted zone was configured for:

`app.vijaysammeta.online`

Two failover alias records were created:

| Record    | Role        | Target             |
| --------- | ----------- | ------------------ |
| Primary   | Mumbai      | `DR-Primary-ALB`   |
| Secondary | N. Virginia | `DR-Secondary-ALB` |

Evaluate Target Health was enabled for both records.

During the failover test, the primary application targets became unavailable. After health evaluation and DNS convergence, the custom domain served the N. Virginia DR application.

## 🌍 Amazon CloudFront — Planned Enhancement

Amazon CloudFront is planned as a global content delivery network in front of the application.

### Expected Benefits

- Deliver cacheable content from edge locations closer to users.
- Improve performance for eligible static and cacheable content.
- Support HTTPS for viewers using an appropriately configured certificate.
- Provide an opportunity to learn CloudFront origins, cache behaviors, and origin failover.

### Planned Implementation

1. Create a CloudFront distribution using the Mumbai ALB as the initial origin.
2. Test the distribution using its generated CloudFront domain.
3. Configure cache behaviors appropriate for the application.
4. Configure HTTPS and a custom CloudFront hostname if required.
5. Evaluate a safe design for the N. Virginia DR origin.
6. Validate origin failure behavior before changing the existing Route 53 records.

**Status:** Not yet deployed. CloudFront must be configured and tested before it is described as an implemented component.

CloudFront origin failover and Route 53 DNS failover are different mechanisms. Adding CloudFront does not automatically guarantee that all application traffic or database writes will fail over correctly.

## 📊 Monitoring

An Amazon CloudWatch alarm named `DR-Primary-ASG-High-CPU` was created to monitor primary EC2 CPU utilization.

- Metric: `CPUUtilization`
- Statistic: Average
- Evaluation period: 5 minutes
- Threshold: Greater than 70%

The alarm was created, but an alarm-triggering test was not confirmed.

## 🧪 Testing and Validation

| Test                                       | Result |
| ------------------------------------------ | ------ |
| Mumbai application access through ALB      | Passed |
| N. Virginia application access through ALB | Passed |
| EC2-to-Aurora connectivity                 | Passed |
| Aurora cross-region data replication       | Passed |
| S3 Cross-Region Replication                | Passed |
| Auto Scaling replacement test              | Passed |
| Route 53 application failover              | Passed |
| CloudFront distribution                    | passed |

### Disaster Recovery Test

1. The Mumbai application instances were stopped to simulate primary application failure.
2. The Mumbai ALB temporarily returned `502 Bad Gateway` while its targets were unavailable.
3. After health evaluation and DNS convergence, the custom domain served the N. Virginia application.
4. The N. Virginia application was confirmed accessible through the domain.

This validated application traffic failover. It did not establish automatic database promotion, full write recovery, or measured RTO/RPO values.

## 🔐 Security Considerations

- Deploy application instances in private subnets.
- Restrict traffic using security groups.
- Keep database instances private.
- Enable S3 Block Public Access.
- Use IAM roles and Systems Manager for administrative access.
- Never commit AWS credentials, database passwords, private keys, or access tokens.
- Use appropriate HTTPS and caching policies when deploying CloudFront.

## 💡 Challenges and Solutions

### Subnet CIDR Overlap

Resolved subnet CIDR overlap by selecting non-overlapping ranges within the VPC CIDR.

### Aurora Replication Configuration

Configured the required Aurora MySQL binary logging settings, applied the parameter changes, rebooted the writer instance, and retried cross-region replication.

### DNS Resolution

Corrected the hosted-zone and domain nameserver delegation configuration to resolve the custom domain.

### Application Failover

Verified that the custom domain served the N. Virginia application after the Mumbai application targets became unavailable and DNS failover converged.

## 🚀 Future Enhancements

- Deploy and validate Amazon CloudFront.
- Automate infrastructure provisioning using Terraform.
- Add CI/CD using Jenkins or AWS developer tools.
- Configure automated database promotion and endpoint switching.
- Add SNS notifications for monitoring alarms.
- Document a disaster recovery runbook.
- Measure recovery time objective (RTO) and recovery point objective (RPO).

## 💰 Cost Management

This project uses billable resources such as EC2 instances, NAT Gateways, ALBs, Aurora, S3 storage, and potentially CloudFront.

After testing:

- Remove resources that are no longer required.
- Review snapshots, backups, and S3 object versions before deletion.
- Delete dependent resources in the correct order.
- Review AWS Billing and Cost Management for ongoing charges.

## 👨‍💻 Author

**Vijay Kumar Sammeta**
B.Tech — Computer Science and Engineering

**Project:** Multi-Region AWS Disaster Recovery System with Amazon CloudFront

## 📄 License

Add an appropriate license file if you intend to publish this project for reuse. If you choose the MIT License, include the actual MIT `LICENSE` file in the repository.
