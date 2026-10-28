# Scalable Web Application on AWS — ALB, Auto Scaling, CloudFront, and Multi-AZ RDS

A production-oriented reference architecture for hosting an EC2-based web application across two Availability Zones (AZs), with private application instances, a public Application Load Balancer (ALB), automatic scaling, a Multi-AZ relational database, edge caching, and centralized operations.

## Table of Contents

- [1. Project Overview](#1-project-overview)
- [2. Request and Traffic Flow](#2-request-and-traffic-flow)
- [3. Network Design](#3-network-design)
- [4. AWS Services and Responsibilities](#4-aws-services-and-responsibilities)
- [5. Security Design](#5-security-design)
- [6. High Availability and Scaling](#6-high-availability-and-scaling)
- [7. Database Design and Failover](#7-database-design-and-failover)
- [8. Monitoring, Logging, and Notifications](#8-monitoring-logging-and-notifications)
- [9. Implementation Plan](#9-implementation-plan)
- [10. Validation and Test Plan](#10-validation-and-test-plan)
- [11. Cost Considerations](#11-cost-considerations)

---

## 1. Project Overview

This project deploys a scalable web application on Amazon Web Services (AWS). The application runs on Amazon EC2 instances distributed across two Availability Zones.

An internet-facing Application Load Balancer distributes incoming application traffic across healthy instances, while an EC2 Auto Scaling Group maintains the desired capacity and adjusts the number of instances based on demand.

Amazon CloudFront provides edge delivery and caching for static content. AWS WAF protects the web entry point using managed web application firewall rules.

Amazon RDS runs in a Multi-AZ deployment to provide managed database high availability and automatic failover.

The application and database tiers are deployed in private subnets. AWS Systems Manager Session Manager provides administrative access without requiring a bastion host or inbound SSH access from the internet.

Amazon CloudWatch collects metrics and logs, evaluates alarms, and sends notifications through Amazon SNS.

### 1.1 Architecture Diagram

The following diagram illustrates the overall architecture, including the network layout, application tier, database tier, and AWS services involved in request handling and operations.

**Architecture diagram:**

<!-- Replace the placeholder below with your architecture image. -->

![AWS Scalable Web Application Architecture](./AWS%20VPC%20Multi-AZ%20Architecture%20Diagram.png)

_Figure 1: Scalable Web Application Architecture on AWS._

### 1.2 Architecture Principles

- **High availability:** Distribute the load balancer and application capacity across two AZs.
- **Private compute:** Application EC2 instances have no public IPv4 addresses.
- **Elastic capacity:** Use an Auto Scaling Group and health checks to replace unhealthy instances and adjust capacity.
- **Layered security:** Use CloudFront, WAF, security groups, network ACLs, IAM, and private subnets.
- **Managed database availability:** Use RDS Multi-AZ rather than implementing database replication manually.
- **Secure administrative access:** Use Session Manager with least-privilege IAM permissions.
- **Observability:** Collect metrics, logs, alarms, and notifications.
- **Recoverability:** Enable database backups, define retention, and test restoration procedures.

### 1.3 Regional and Availability Zone Boundaries

- The VPC and its subnets exist in one AWS Region.
- Each AZ is an independent failure domain within that Region.
- Public and private subnets are created in both AZs.
- The ALB uses public subnets in both AZs.
- The Auto Scaling Group uses private application subnets in both AZs.
- The RDS DB subnet group includes private database subnets in at least two AZs.

In a standard Multi-AZ DB instance deployment, AWS manages the primary and standby placement and synchronous replication.

### 1.4 CIDR Plan

| Network              | Example CIDR   | Purpose                           |
| -------------------- | -------------- | --------------------------------- |
| VPC                  | `10.0.0.0/16`  | VPC address space                 |
| Public Subnet A      | `10.0.1.0/24`  | ALB nodes and NAT Gateway A       |
| Public Subnet B      | `10.0.2.0/24`  | ALB nodes and NAT Gateway B       |
| Private App Subnet A | `10.0.11.0/24` | Application EC2 instances in AZ A |
| Private App Subnet B | `10.0.12.0/24` | Application EC2 instances in AZ B |
| Private DB Subnet A  | `10.0.21.0/24` | RDS subnet group                  |
| Private DB Subnet B  | `10.0.22.0/24` | RDS subnet group                  |

---

## 2. Request and Traffic Flow

### 2.1 User Request

1. A user requests the application domain.
2. Route 53 resolves the domain to the CloudFront distribution using an Alias record.
3. CloudFront serves a cached object when the request matches a valid cache behavior and cache entry.
4. If the request is not served from cache, CloudFront forwards it to the configured origin.
5. For dynamic application requests, the origin is the internet-facing ALB. AWS WAF evaluates requests at the CloudFront distribution.
6. The ALB listener evaluates its rules and forwards the request to the target group.
7. The target group forwards the request to a healthy EC2 instance in a private application subnet.
8. The application accesses RDS through the database endpoint when database access is required.
9. The response returns through the application, ALB, and CloudFront to the user. The cache policy determines whether CloudFront stores the response.

### 2.2 Origin Protection

If CloudFront is the intended public entry point, protect the ALB from direct unwanted access.

Possible approaches include:

- Allowing the CloudFront origin-facing managed prefix list in the ALB security group, where supported and appropriate.
- Configuring a custom origin header in CloudFront and validating it at the origin or application layer.
- Combining network restrictions with application-level validation and carefully configured WAF rules.

---

## 3. Network Design

### 3.1 VPC

The VPC provides an isolated network boundary for the application.

Create the VPC with a non-overlapping CIDR block and enable DNS support and DNS hostnames as required by the design.

### 3.2 Public Subnets

Public subnets host the ALB nodes and NAT Gateways.

Their route tables include:

- The local VPC route.
- `0.0.0.0/0` routed to the Internet Gateway.

A route to the Internet Gateway makes internet routing possible. Security groups, network ACLs, public addressing, and resource configuration still control whether traffic can flow.

### 3.3 Private Application Subnets

EC2 instances run in private application subnets without public IP addresses.

Each subnet's route table includes:

- The local VPC route.
- `0.0.0.0/0` routed to the NAT Gateway in the same AZ, when internet egress is required.

The NAT Gateway enables outbound connections initiated from private instances. It does not allow arbitrary unsolicited inbound internet connections to EC2 instances.

For improved resilience and reduced cross-AZ data transfer, deploy a NAT Gateway in each AZ and route each private application subnet to its same-AZ NAT Gateway.

NAT Gateways incur hourly and data-processing charges.

### 3.4 Private Database Subnets

Database subnets should not have a direct route to an Internet Gateway or a default route through a NAT Gateway unless a documented requirement justifies an exceptional design.

The database should not be publicly accessible.

Create an RDS DB subnet group that includes private subnets in at least two AZs. RDS uses the subnet group for placement and failover within the supported deployment model.

### 3.5 Internet Gateway vs. NAT Gateway

| Component              | Primary Purpose                                            | Direction / Usage                                                                       |
| ---------------------- | ---------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Internet Gateway (IGW) | Connects a VPC to the internet                             | Used by public resources with suitable routes, addresses, and security rules            |
| NAT Gateway            | Enables outbound IPv4 internet access from private subnets | Private instances initiate connections; unsolicited inbound connections are not enabled |

### 3.6 Route Table Summary

| Route Table       | Routes                                            |
| ----------------- | ------------------------------------------------- |
| Public RT A/B     | VPC local route; default route to IGW             |
| Private App RT A  | VPC local route; default route to NAT Gateway A   |
| Private App RT B  | VPC local route; default route to NAT Gateway B   |
| Private DB RT A/B | VPC local route; no direct internet default route |

---

## 4. AWS Services and Responsibilities

| Service                   | Role in This Project                    | Important Configuration                                                         |
| ------------------------- | --------------------------------------- | ------------------------------------------------------------------------------- |
| Amazon VPC                | Network isolation                       | CIDR, DNS settings, subnets, route tables                                       |
| Availability Zones        | Fault isolation                         | Distribute application and database networking across two AZs                   |
| Internet Gateway          | Internet routing for public subnets     | Attach to VPC and add public default routes                                     |
| NAT Gateway               | Private subnet outbound IPv4 access     | One per AZ for resilient same-AZ routing                                        |
| Application Load Balancer | Layer 7 HTTP/HTTPS routing              | Public subnets, HTTPS listener, target group, health checks                     |
| Amazon EC2                | Application compute                     | Private instances, supported AMI, IAM instance profile                          |
| Launch Template           | Defines instance configuration          | AMI, instance type, security group, IAM profile, user data                      |
| Auto Scaling Group        | Maintains and adjusts EC2 capacity      | Private subnets in both AZs, minimum/desired/maximum capacity, health checks    |
| Target Tracking           | Scales capacity against a target metric | Average CPU or ALB requests per target                                          |
| Amazon CloudFront         | Edge delivery and caching               | Origins, cache behaviors, TLS certificate, cache policies                       |
| AWS WAF                   | Web request filtering                   | CloudFront association, managed rule groups, metrics, logging                   |
| Amazon RDS                | Managed relational database             | Private DB subnet group, encryption, backups, Multi-AZ                          |
| Amazon Route 53           | DNS                                     | Alias record to CloudFront                                                      |
| AWS Systems Manager       | Secure instance administration          | SSM Agent, IAM role, network connectivity, Session Manager                      |
| Amazon CloudWatch         | Observability                           | Metrics, logs, dashboards, alarms                                               |
| Amazon SNS                | Notifications                           | Topic, subscriptions, permissions, delivery testing                             |
| AWS Certificate Manager   | TLS certificates                        | CloudFront viewer certificate in `us-east-1`; ALB certificate in the ALB Region |
| AWS IAM                   | Access control                          | Least-privilege roles and policies                                              |

---

## 5. Security Design

### 5.1 Security Groups

Security groups are stateful virtual firewalls associated with network interfaces and resources.

Prefer references to other security groups rather than broad CIDR ranges for internal traffic.

| Security Group             | Inbound                               | Source                                                                     |
| -------------------------- | ------------------------------------- | -------------------------------------------------------------------------- |
| ALB Security Group         | TCP 443                               | Appropriate CloudFront origin-facing source restriction, where implemented |
| Application Security Group | Application port, e.g., TCP 8080      | ALB security group                                                         |
| Database Security Group    | PostgreSQL TCP 5432 or MySQL TCP 3306 | Application security group                                                 |

**Notes:**

- Add TCP 80 to the ALB only if required to redirect HTTP to HTTPS.
- Do not expose application ports directly to the internet.
- Do not allow database traffic from `0.0.0.0/0`.
- Configure egress rules according to the application's actual dependencies.
- The application must listen on the port configured in the target group.

### 5.2 Network ACLs

Network ACLs apply at the subnet level and are stateless. They provide an additional layer of filtering but are not a substitute for security groups.

If custom restrictive NACLs are used, ensure both request and return traffic are allowed, including the necessary ephemeral ports.

Incorrect NACL rules can break otherwise valid connections and health checks.

Begin with the default NACL for initial connectivity testing, then introduce least-privilege rules with a documented test plan.

### 5.3 AWS WAF

Associate a web ACL with CloudFront for the intended edge protection.

A reasonable starting point includes:

- AWS Managed Rules Common Rule Set.
- Known Bad Inputs Rule Set.
- SQL injection protection, where relevant.
- IP reputation or rate-based rules, where justified.
- Logging and sampled request inspection.

### 5.4 IAM

The EC2 instance profile should grant only the required permissions, such as Systems Manager access and narrowly scoped access to required AWS APIs.

If the application reads secrets from Secrets Manager or Parameter Store, grant access only to the necessary secret ARNs and KMS keys.

### 5.5 Encryption and Secrets

- Redirect HTTP to HTTPS and use valid TLS certificates.
- Enable RDS storage encryption and use encrypted snapshots.
- Use encrypted EBS volumes.
- Store database credentials in AWS Secrets Manager or Systems Manager Parameter Store with appropriate encryption and access policies.
- Never commit credentials, private keys, `.env` files, or unredacted connection strings to Git.
- Rotate credentials according to the application's supported rotation process.

### 5.6 Systems Manager Session Manager

Session Manager enables interactive access without opening inbound SSH port 22 or operating a bastion host.

To configure it:

1. Use a supported AMI with SSM Agent.
2. Attach an IAM instance profile with the required Systems Manager permissions.
3. Provide network access to the required Systems Manager endpoints through NAT or VPC interface endpoints.
4. Ensure endpoint security groups, DNS, and policies permit the connection.
5. Use IAM policies to control which users can start sessions.
6. Configure session logging to CloudWatch Logs or S3 if required by the security policy.

Session Manager removes the need for an inbound SSH path. It does not remove the need for IAM governance and audit controls.

---

## 6. High Availability and Scaling

### 6.1 Application Load Balancer

Configure the ALB in public subnets in both AZs.

Create an HTTPS listener on port 443 and forward requests to the application target group. Optionally, configure an HTTP listener on port 80 to redirect to HTTPS.

The target group should use the instance target type when registering EC2 instances.

Configure:

- Protocol and application port.
- Health-check path, such as `/health`.
- Expected success codes, such as `200`.
- Healthy and unhealthy thresholds and intervals appropriate for the application.
- A health endpoint that reports whether the application can serve requests.

### 6.2 Launch Template

A Launch Template defines how Auto Scaling launches instances.

Include:

- A maintained AMI, such as a supported Amazon Linux image.
- An appropriate instance type.
- Application security group.
- IAM instance profile.
- Encrypted EBS volume settings.
- User data or a controlled bootstrap mechanism.
- Metadata service settings, preferably requiring IMDSv2.
- Tags for ownership, environment, and cost allocation.

### 6.3 Auto Scaling Group

Create an ASG spanning the two private application subnets.

Configure minimum, desired, and maximum capacity based on availability, expected load, and budget.

For example, a lab might start with:

- Minimum capacity: 2 instances.
- Desired capacity: 2 instances.
- Maximum capacity: 4 instances.

These are illustrative values, not universal production recommendations.

A production baseline should consider whether the application can withstand the loss of an AZ and still meet capacity and availability objectives.

Enable ELB health checks so that the ASG can replace instances reported unhealthy by the load balancer.

Configure instance warm-up and health-check grace periods to reflect actual startup time.

### 6.4 Scaling Policies

**Target tracking** adjusts capacity to keep a selected metric near a target.

Possible metrics include:

- Average CPU utilization.
- ALB requests per target, when appropriate for the workload.

Select a metric that correlates with application saturation. CPU is not always a good proxy for queue depth, latency, memory pressure, or downstream bottlenecks.

**Step scaling** can respond to CloudWatch alarms with different capacity adjustments for different alarm severities.

It is useful where thresholds and step adjustments have been designed and tested.

---

## 7. Database Design and Failover

### 7.1 RDS Multi-AZ

Use Amazon RDS for PostgreSQL or MySQL, according to the application.

Place it in a DB subnet group spanning at least two private subnets in different AZs.

Configure:

- Public access: **No**.
- Storage encryption: Enabled.
- Automated backups: Enabled.
- Multi-AZ: Enabled.
- Database security group: Allow traffic only from the application tier.

In a standard RDS Multi-AZ DB instance deployment:

- The primary handles database reads and writes.
- AWS maintains a standby in another AZ.
- RDS replicates changes synchronously to the standby.
- RDS can fail over to the standby during supported failure scenarios.
- The database endpoint is used by the application; the endpoint resolves to the current primary after failover.

The standby in a standard Multi-AZ DB instance deployment is for high availability and is not normally used to serve read traffic.

If read scaling is required, evaluate read replicas or a Multi-AZ DB cluster configuration based on the database engine and requirements.

### 7.2 Application Connectivity

Use the RDS endpoint DNS name rather than hardcoding an IP address.

Configure the application with a secret or secure parameter containing the database connection information.

Ensure the DB security group allows only the database engine port from the application security group.

Applications should implement connection retries and reconnect behavior because database failover can interrupt existing connections.

---

## 8. Monitoring, Logging, and Notifications

### 8.1 CloudWatch Metrics

Monitor at least the following metrics.

**Application Load Balancer**

- Request count.
- Target response time.
- HTTP 4xx and 5xx errors.
- Healthy and unhealthy host counts.
- Rejected connections, where relevant.

**EC2 and Auto Scaling**

- CPU utilization.
- Instance health and ASG desired/in-service capacity.
- Scaling activities and launch failures.
- Memory and disk metrics through the CloudWatch Agent, if required. These are not all available as default EC2 metrics.

**RDS**

- CPU utilization.
- Database connections.
- Free storage space.
- Read/write latency and IOPS, where relevant.
- Failover and event notifications.

**CloudFront and WAF**

- Requests, cache hit ratio, origin errors, and latency-related metrics.
- WAF allowed/blocked requests and rule metrics.

### 8.2 Logs

Centralize relevant logs, such as application logs, web server logs, and operating-system logs, using the CloudWatch Agent or another supported logging path.

Enable ALB access logs and CloudFront logs if the project requires request-level analysis.

Configure retention periods and protect logs from unauthorized access.

### 8.3 Alarms and SNS

Create alarms with meaningful thresholds and evaluation periods.

Examples include:

- No healthy ALB targets.
- Sustained high target response time.
- Elevated 5xx response rate.
- Sustained high CPU.
- ASG unable to reach desired capacity.
- Low RDS free storage or unusually high connection counts.
- Database failover events, where available through the selected monitoring mechanism.

Configure CloudWatch alarms to publish to an SNS topic and subscribe an email address or other approved endpoint.

Confirm the subscription and test a notification.

An alarm is useful only if someone owns the response and knows what action to take.

---

## 9. Implementation Plan

The following order reduces troubleshooting complexity. Exact console labels may change.

### Phase 1 — Plan the Environment

1. Choose an AWS Region and two available AZs.
2. Select a non-overlapping VPC CIDR.
3. Choose the database engine and application port.
4. Define environment tags and naming conventions.
5. Estimate costs before creating NAT Gateways, ALB, RDS, and CloudFront resources.
6. Decide whether to provision manually, with Terraform, or with another infrastructure-as-code tool.

### Phase 2 — Build the VPC

1. Create the VPC and enable DNS support and hostnames.
2. Create two public, two private application, and two private database subnets across the selected AZs.
3. Attach an Internet Gateway.
4. Create public route tables with a default route to the IGW and associate them with public subnets.
5. Create one NAT Gateway in each public subnet, each with an Elastic IP.
6. Create private application route tables and route each to its same-AZ NAT Gateway if internet egress is required.
7. Keep database subnets without a direct internet route.
8. Verify every subnet association and route.

### Phase 3 — Configure Security

1. Create separate ALB, application, and database security groups.
2. Permit HTTPS to the ALB from the intended source.
3. Permit the application port to EC2 only from the ALB security group.
4. Permit the DB port only from the application security group.
5. Create an EC2 IAM instance profile with required Systems Manager permissions.
6. Decide whether to use NAT or Systems Manager VPC endpoints.
7. Add custom NACL rules only if required and test ephemeral return traffic.

### Phase 4 — Prepare the Application

1. Ensure the application can start reliably from a clean instance.
2. Provide a health endpoint, such as `/health`.
3. Externalize configuration and retrieve secrets securely.
4. Confirm that the app listens on the selected port and binds to an address reachable through the instance network interface.
5. Define bootstrap steps using user data, a baked AMI, or a deployment agent.
6. Make startup idempotent and log bootstrap failures.

### Phase 5 — Deploy the ALB

1. Create an internet-facing ALB in both public subnets.
2. Create a target group using the instance target type and the correct application port.
3. Configure the health-check path and success codes.
4. Create an HTTPS listener with a valid ACM certificate.
5. Optionally create an HTTP listener that redirects to HTTPS.
6. Confirm that the ALB security group allows the intended source and its egress can reach the application security group.

### Phase 6 — Deploy EC2 and Auto Scaling

1. Create a Launch Template with the AMI, instance type, security groups, IAM profile, EBS settings, and bootstrap process.
2. Create an ASG spanning both private application subnets.
3. Attach the target group.
4. Configure minimum, desired, and maximum capacity and health-check settings.
5. Configure target-tracking scaling based on an appropriate metric.
6. Verify that instances launch without public IP addresses and register as healthy targets.
7. Confirm that Session Manager can connect to each instance.

### Phase 7 — Deploy RDS

1. Create a DB subnet group containing the private database subnets in both AZs.
2. Create the RDS database with Multi-AZ enabled for the selected deployment type.
3. Disable public accessibility.
4. Enable encryption and automated backups.
5. Apply the DB security group.
6. Store credentials in Secrets Manager or Parameter Store.
7. Configure the application's database connection and test queries from the private application tier.

### Phase 8 — Configure CloudFront, WAF, and DNS

1. Create a CloudFront distribution with the ALB as the dynamic origin.
2. Optionally add a private S3 origin for static assets using Origin Access Control (OAC).
3. Define cache behaviors for static and dynamic paths.
4. Configure the viewer certificate and alternate domain name, if using a custom domain.
5. Associate a WAF web ACL with CloudFront and configure managed rules.
6. Test WAF rules in count mode before switching appropriate rules to block mode.
7. Create a Route 53 Alias record pointing the application domain to CloudFront.
8. Protect the ALB origin from unintended direct access.
9. Test HTTPS, redirects, cache behavior, and application/API behavior.

**Certificate note:** CloudFront viewer certificates using ACM must be in `us-east-1`. An ACM certificate used by an ALB must be in the same Region as that ALB.

### Phase 9 — Configure Observability

1. Configure application and system logs.
2. Enable ALB access logs and CloudFront/WAF logging as required.
3. Create a CloudWatch dashboard.
4. Create alarms for unhealthy targets, elevated errors/latency, capacity issues, and database pressure.
5. Create an SNS topic and confirm subscriptions.
6. Trigger a test alarm and verify notification delivery.

---

## 10. Validation and Test Plan

| Test                 | Procedure                                                             | Expected Result                                                                          |
| -------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Public Entry Point   | Open the application domain over HTTPS                                | Application responds with a valid certificate                                            |
| HTTP Redirect        | Request HTTP if port 80 redirect is configured                        | Redirect to HTTPS                                                                        |
| Target Health        | Inspect target group health                                           | Expected instances are healthy                                                           |
| Instance Isolation   | Inspect application instance public IP and inbound rules              | No public IP; no direct internet ingress to the application port                         |
| Database Isolation   | Inspect RDS public access and DB security group                       | Not publicly accessible; DB port restricted to the application security group            |
| Session Manager      | Start a session to a private EC2 instance                             | Access works without public SSH                                                          |
| Auto Scaling         | Generate controlled load or change desired capacity in a test window  | ASG launches or terminates instances as configured                                       |
| Instance Replacement | Stop or terminate a test instance under controlled conditions         | ASG restores capacity and the target group returns to a healthy state                    |
| AZ Resilience        | Use a planned, safe test or controlled simulation                     | Remaining AZ has sufficient capacity; no unsupported assumptions about forced AZ outages |
| Database Failover    | Perform an approved RDS failover test in a non-production environment | Application reconnects using the RDS endpoint after recovery                             |
| WAF                  | Send safe test requests that match configured rules                   | Expected requests are counted or blocked and logged                                      |
| Static Caching       | Request the same static asset repeatedly                              | Cache behavior and hit ratio match the policy                                            |
| Dynamic Response     | Request personalized or API endpoints                                 | Responses are not cached incorrectly                                                     |
| Alarm Notification   | Trigger a test CloudWatch alarm                                       | SNS subscriber receives notification                                                     |
| Recovery             | Restore a snapshot or point-in-time recovery in a test environment    | Recovery procedure works and timing is recorded                                          |

---

## 11. Cost Considerations

Common cost drivers include:

- NAT Gateway hourly charges and data processing.
- Application Load Balancer hours and capacity units.
- EC2 instance hours and EBS volumes.
- RDS instance hours, storage, backups, and I/O, depending on configuration.
- CloudFront data transfer and request charges.
- AWS WAF web ACL, rule, and request charges.
- Public IPv4 address charges, where applicable.
- CloudWatch metrics, log ingestion, storage, and alarms.
- Data transfer between AZs and out of AWS.
