# Testing and Validation

## 1. Overview

This document records the tests performed on the Multi-Region AWS Disaster Recovery System.

The project uses Mumbai (`ap-south-1`) as the primary region and N. Virginia (`us-east-1`) as the disaster recovery region.

The tests focus on application availability, database replication, object replication, Auto Scaling, and DNS-based failover.

## 2. Test Summary

| Test Case | Expected Result | Observed Result | Status |
|---|---|---|---|
| Mumbai ALB access | Primary application loads | Mumbai application was accessible | Passed |
| N. Virginia ALB access | DR application loads | N. Virginia application was accessible | Passed |
| EC2-to-Aurora connectivity | Application instance can connect to Aurora | Connection succeeded | Passed |
| Aurora replication | DR database contains replicated data | `drapp` and the test user record were verified | Passed |
| S3 Cross-Region Replication | New source object appears in the destination bucket | Replicated object was verified | Passed |
| Auto Scaling self-healing | Replacement instance launches after termination | Replacement instance was created | Passed |
| Route 53 primary routing | Domain resolves to the primary application | Mumbai application loaded | Passed |
| Route 53 failover | Domain serves the DR application when the primary is unhealthy | N. Virginia application loaded after failover convergence | Passed |
| CloudWatch alarm creation | CPU alarm is configured | Alarm created; triggering not tested | Created |
| CloudFront distribution | CDN serves the application | Not configured as part of the tested implementation | Not tested |

## 3. Application Availability Tests

### Test 1: Mumbai Application

**Objective:** Verify that the primary application is reachable through the Mumbai Application Load Balancer.

**Procedure:**
1. Open the Mumbai ALB DNS endpoint.
2. Verify that the web page loads.
3. Confirm that the page identifies the Mumbai region.
4. Verify the target group health.

**Result:** The Mumbai application was accessible through its ALB.

**Status:** Passed.

### Test 2: N. Virginia Application

**Objective:** Verify that the disaster recovery application can serve requests independently.

**Procedure:**
1. Open the N. Virginia ALB DNS endpoint.
2. Verify that the web page loads.
3. Confirm that the page identifies `us-east-1`.
4. Verify the DR target group health.

**Result:** The N. Virginia application was accessible through its ALB.

**Status:** Passed.

## 4. Auto Scaling Test

**Objective:** Verify that the primary Auto Scaling Group replaces a terminated application instance.

**Procedure:**
1. Open EC2 Auto Scaling Groups.
2. Select `DR-Primary-ASG`.
3. Terminate an instance managed by the group.
4. Wait for Auto Scaling to launch a replacement.
5. Verify that the replacement registers with the target group and becomes healthy.

**Result:** A replacement instance was created successfully.

**Status:** Passed.

## 5. Aurora Database Replication Test

**Objective:** Verify that database data from Mumbai is available in the N. Virginia secondary cluster.

**Test database:** `drapp`  
**Test table:** `users`

Example verification query:

```sql
USE drapp;
SELECT * FROM users;
```

Expected test record:

| ID | Name | Email |
|---|---|---|
| 1 | Vijay | vijay@example.com |

**Result:** The database and test record were visible in the N. Virginia secondary cluster.

**Status:** Passed.

**Limitation:** This test verified replication. It did not verify automatic promotion of the secondary database or application write recovery during a regional database failure.

## 6. S3 Cross-Region Replication Test

**Objective:** Verify that newly uploaded objects are replicated from Mumbai to N. Virginia.

**Procedure:**
1. Upload a new test object to the primary S3 bucket.
2. Open the destination bucket in N. Virginia.
3. Locate the replicated object.
4. Confirm that the replication succeeded.

**Result:** A newly uploaded test object appeared in the destination bucket.

**Status:** Passed.

**Limitation:** Objects created before the replication rule was enabled were not automatically replicated by default.

## 7. Route 53 Failover Test

**Objective:** Verify that the custom domain can route users to the DR application when the primary application becomes unhealthy.

**Domain:** `app.vijaysammeta.online`

**Primary endpoint:** Mumbai ALB  
**Secondary endpoint:** N. Virginia ALB

### Procedure

1. Confirm that the custom domain serves the Mumbai application under normal conditions.
2. Make the Mumbai application instances unavailable to simulate an application failure.
3. Observe that the Mumbai ALB temporarily returns `502 Bad Gateway` when its targets are unavailable.
4. Wait for Route 53 health evaluation and DNS/failover convergence.
5. Open the custom domain in a browser.
6. Verify that the N. Virginia application page appears.

### Observed Result

The domain initially displayed a `502 Bad Gateway` response during the failure simulation. After waiting and refreshing, the domain served the N. Virginia DR application.

**Status: Passed.**

This demonstrated application traffic failover from Mumbai to N. Virginia.

### Recovery Note

After the test, restore `DR-Primary-ASG` to its intended capacity:

- Minimum: 2
- Desired: 2
- Maximum: 4

Verify that the primary target group is healthy again. DNS caching and health evaluation can cause a delay before normal routing is observed.

## 8. CloudWatch Monitoring

**Alarm name:** `DR-Primary-ASG-High-CPU`

Configuration:

- Metric: `CPUUtilization`
- Statistic: Average
- Evaluation period: 5 minutes
- Threshold: Greater than 70%

**Result:** The CloudWatch alarm was created.

**Status:** Created; alarm triggering was not validated.

## 9. Overall Results

The following core disaster recovery capabilities were validated:

- Primary application availability.
- DR application availability.
- Auto Scaling instance replacement.
- Aurora cross-region data replication.
- S3 Cross-Region Replication.
- Route 53 application failover.

The failover test demonstrated that the custom domain served the N. Virginia application after the Mumbai application targets became unavailable.

## 10. Known Limitations

- Database promotion and automatic write recovery were not tested.
- Complete regional recovery was not tested.
- RTO and RPO were not measured.
- CloudFront was not deployed or tested.
- CloudWatch alarm triggering was not tested.
- Restoration of the primary Auto Scaling Group should be confirmed after the failure simulation.

## 11. Conclusion

The project successfully demonstrated a multi-region AWS application architecture with replicated data services and DNS-based application failover.

Further work could include automated database promotion, end-to-end recovery testing, measured RTO/RPO, alert notifications, and infrastructure provisioning through Terraform.
