# Implementation Steps

## 1. Project Overview

The Multi-Region AWS Disaster Recovery System was implemented using the AWS Management Console.

The architecture contains two AWS regions:

- **Primary:** Mumbai (`ap-south-1`)
- **Disaster Recovery:** US East (N. Virginia, `us-east-1`)

The goal is to maintain an application environment in a separate region and validate DNS-based application failover.

## 2. Create the Primary VPC

1. Select the Mumbai region.
2. Create the VPC `DR-Primary-VPC`.
3. Assign CIDR block `10.0.0.0/16`.
4. Create public subnets across two Availability Zones.
5. Create private application subnets across two Availability Zones.
6. Create private database subnets across two Availability Zones.
7. Configure an Internet Gateway and NAT Gateways.
8. Configure route tables and an S3 Gateway Endpoint.
9. Enable DNS resolution and DNS hostnames.

## 3. Configure Primary Security Groups

Create three security groups:

- `DR-Primary-ALB-SG`: Allows inbound HTTP and HTTPS from the internet.
- `DR-Primary-App-SG`: Allows inbound HTTP from the ALB security group.
- `DR-Primary-DB-SG`: Allows MySQL/Aurora traffic on port 3306 from the application security group.

Avoid exposing application instances and the database directly to the public internet.

## 4. Configure IAM and Application Instances

1. Create the EC2 IAM role `DR-Primary-EC2-SSM-Role`.
2. Attach `AmazonSSMManagedInstanceCore`.
3. Launch Amazon Linux 2023 EC2 instances in private application subnets.
4. Associate the application security group and IAM role.
5. Install Nginx using instance user data.
6. Configure the application page to identify the Mumbai region.
7. Verify access using AWS Systems Manager.

## 5. Configure the Primary Load Balancer

1. Create target group `DR-Primary-TG`.
2. Configure HTTP port 80 and health check path `/`.
3. Register the application instances.
4. Create internet-facing ALB `DR-Primary-ALB`.
5. Select the two public subnets.
6. Associate `DR-Primary-ALB-SG`.
7. Configure an HTTP listener on port 80 to forward traffic to `DR-Primary-TG`.
8. Confirm that both targets become healthy.
9. Test the ALB DNS endpoint.

## 6. Configure the Primary Auto Scaling Group

1. Create launch template `DR-Primary-App-LT`.
2. Configure Amazon Linux 2023, the application security group, the EC2 IAM role, and Nginx user data.
3. Create `DR-Primary-ASG` in the private application subnets.
4. Set minimum capacity to 2, desired capacity to 2, and maximum capacity to 4.
5. Attach `DR-Primary-TG`.
6. Enable load balancer health checks.
7. Configure CPU-based target tracking.
8. Verify that the application targets become healthy.
9. Test instance replacement by terminating an ASG-managed instance.

## 7. Configure Aurora MySQL in Mumbai

1. Create database subnets across two Availability Zones.
2. Create DB subnet group `dr-primary-db-subnet-group`.
3. Create Aurora MySQL cluster `dr-primary-aurora`.
4. Select the primary VPC and database subnet group.
5. Disable public access.
6. Associate `DR-Primary-DB-SG`.
7. Enable encryption and configure automated backup retention.
8. Wait until the cluster and instances are available.

## 8. Create and Verify the Database

Connect to the primary database from a permitted application instance and create a test database and table.

```sql
CREATE DATABASE drapp;
USE drapp;

CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(150)
);

INSERT INTO users (name, email)
VALUES ('Vijay', 'vijay@example.com');

SELECT * FROM users;
```

Verify that the record is present in the primary database.

## 9. Configure S3 Replication

1. Create the primary S3 bucket in Mumbai.
2. Create the destination S3 bucket in N. Virginia.
3. Enable versioning on both buckets.
4. Keep Block Public Access enabled.
5. Configure SSE-S3 encryption.
6. Create an enabled S3 Cross-Region Replication rule from the source bucket to the destination bucket.
7. Allow the replication role to access the required objects.
8. Upload a new test object to the source bucket.
9. Confirm that the object appears in the destination bucket.

## 10. Configure Monitoring

Create the CloudWatch alarm `DR-Primary-ASG-High-CPU`.

Configuration:

- Metric: `CPUUtilization`
- Statistic: Average
- Evaluation period: 5 minutes
- Threshold: Greater than 70%

The alarm was created to monitor the primary application's CPU utilization.

## 11. Build the N. Virginia DR Environment

1. Switch to `us-east-1`.
2. Create `DR-Secondary-VPC` with CIDR `10.1.0.0/16`.
3. Create public, private application, and private database subnets.
4. Configure the Internet Gateway, NAT Gateways, route tables, and S3 Gateway Endpoint.
5. Create `DR-Secondary-ALB-SG`, `DR-Secondary-App-SG`, and `DR-Secondary-DB-SG`.
6. Create launch template `DR-Secondary-App-LT`.
7. Configure Nginx to identify the N. Virginia region.
8. Create `DR-Secondary-ASG` with two desired application instances.
9. Create target group `DR-Secondary-TG`.
10. Create internet-facing ALB `DR-Secondary-ALB`.
11. Attach the target group to the Auto Scaling Group.
12. Verify that the targets are healthy and the DR ALB endpoint works.

## 12. Configure Aurora Cross-Region Replication

1. Create the DR DB subnet group `dr-secondary-db-subnet-group`.
2. Configure the required Aurora MySQL binary logging settings on the primary cluster.
3. Apply the appropriate parameter group changes and reboot the writer instance when required.
4. Create the secondary cluster `dr-secondary-aurora` in N. Virginia using the supported cross-region replication procedure.
5. Associate the DR VPC, DB subnet group, and database security group.
6. Wait for the secondary cluster to become available.
7. Connect to the secondary database and verify the replicated `drapp` database and `users` record.

The DR cluster is a secondary database. This project verified replication but did not validate automatic database promotion or application write recovery.

## 13. Configure Route 53 Failover

1. Create or select the public hosted zone for `vijaysammeta.online`.
2. Ensure the domain is delegated to the hosted zone's Route 53 nameservers.
3. Create an alias A record for `app.vijaysammeta.online` with failover routing set to Primary and target the Mumbai ALB.
4. Create a matching Secondary failover alias record targeting the N. Virginia ALB.
5. Enable Evaluate Target Health on both alias records.
6. Confirm that the custom domain opens the Mumbai application while the primary is healthy.

## 14. Validate Disaster Recovery

1. Confirm that the primary and DR ALBs work independently.
2. Confirm that Aurora replication and S3 replication are working.
3. Simulate primary application failure by making the Mumbai application targets unavailable.
4. Wait for health evaluation and DNS convergence.
5. Open `app.vijaysammeta.online` and confirm that the N. Virginia application is served.
6. Restore the Mumbai Auto Scaling Group to its normal desired capacity.
7. Confirm that the Mumbai target group becomes healthy again.

## 15. Optional Components

An AWS Backup vault was created as an optional component. A vault alone does not configure scheduled backups.

CloudFront, Terraform, and a CI/CD pipeline were not part of the validated implementation.

## 16. Important Notes

- AWS services can incur ongoing charges.
- Avoid committing credentials, access keys, private keys, or database passwords.
- Remove temporary test changes after failover validation.
- Follow an appropriate cleanup order before deleting project resources.
