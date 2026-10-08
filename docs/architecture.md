# IFIS POC – AWS Architecture

## 1. Document Purpose

This document describes the AWS architecture implemented and validated for the IFIS project Proof of Concept (POC).

The purpose of this POC is to:

- Understand the existing IFIS AWS architecture.
- Recreate the architecture in a separate AWS account/environment.
- Validate the architecture using actual AWS resources.
- Use reusable AWS CloudFormation templates.
- Provide a repeatable deployment approach for future environments.
- Document networking, security, compute, CloudFront, WAF, monitoring, backup, and operational considerations.

This document describes the POC architecture and should not be treated as a production deployment specification without reviewing environment-specific requirements.

---

## 2. Project Information

| Item | Value |
|---|---|
| Project | W2P-Japan / IFIS |
| Environment | POC |
| AWS Account | 963910217596 |
| Primary POC Region | ap-south-1 |
| CloudFront WAF Region | us-east-1 |
| VPC | vpc-06900f62513eff63 |
| VPC CIDR | 172.31.0.0/16 |
| Repository | sohan_IFIS_project |
| Infrastructure as Code | AWS CloudFormation |

---

## 3. Architecture Overview

The target architecture provides a public entry point through Amazon CloudFront while keeping the application EC2 instance private inside the VPC.

The main traffic flow is:

```text
Internet User
     |
     v
Route 53
     |
     v
CloudFront
     |
     v
AWS WAF
     |
     v
CloudFront VPC Origin
     |
     v
Private EC2
     |
     v
NAT Gateway
     |
     v
Internet / AWS Services
```

AWS Backup and CloudWatch operate as supporting services.

```text
                         Internet
                            |
                            v
                       Route 53
                            |
                            v
                      CloudFront
                            |
                            v
                     AWS WAF
                            |
                            v
                    VPC Origin
                            |
                            v
                +---------------------+
                |       VPC           |
                |                     |
                |  Private Subnet     |
                |                     |
                |   +-------------+   |
                |   | Private EC2 |   |
                |   +-------------+   |
                |          |          |
                |          v          |
                |   NAT Gateway       |
                |          |          |
                +----------|----------+
                           |
                           v
                       Internet
```

---

## 4. Architecture Components

The POC contains the following major AWS components:

1. Amazon VPC
2. Public subnet
3. Private subnet
4. Private route table
5. NAT Gateway
6. Elastic IP
7. Security Group
8. Amazon EC2
9. IAM Role
10. IAM Instance Profile
11. AWS Systems Manager
12. Amazon CloudWatch
13. Amazon CloudFront
14. CloudFront VPC Origin
15. AWS WAF
16. AWS Backup
17. Amazon Route 53

Not every component is deployed in every stage of the POC.

CloudFront resource creation is currently dependent on AWS account verification.

---

# 5. Network Architecture

## 5.1 VPC

The POC uses an existing VPC.

| Property | Value |
|---|---|
| VPC ID | vpc-06900f62513eff63 |
| CIDR | 172.31.0.0/16 |
| Region | ap-south-1 |

The VPC is not created by the POC CloudFormation network stack.

Instead, the existing VPC ID is supplied as a parameter.

This makes the CloudFormation template reusable in another AWS account or environment.

---

## 5.2 Existing Public Subnet

The NAT Gateway is deployed into an existing public subnet.

| Property | Value |
|---|---|
| Subnet ID | subnet-0d6445bd9f644383b |
| CIDR | 172.31.0.0/20 |
| Availability Zone | ap-south-1b |
| Purpose | NAT Gateway |

The subnet already has public routing through an Internet Gateway.

---

## 5.3 Private Subnet

A dedicated private subnet is created for the application EC2 instance.

| Property | Value |
|---|---|
| CIDR | 172.31.48.0/20 |
| Availability Zone | ap-south-1b |
| Public IP | Disabled |
| Route Table | Dedicated private route table |

The EC2 instance is launched into this subnet.

---

## 5.4 Private Route Table

The private subnet uses a dedicated route table.

The important routes are:

```text
Destination        Target
--------------------------------
172.31.0.0/16      local
0.0.0.0/0          NAT Gateway
```

The local route allows communication within the VPC.

The default route sends outbound Internet traffic through the NAT Gateway.

---

## 5.5 NAT Gateway

The POC uses a NAT Gateway for outbound connectivity from the private subnet.

| Property | Value |
|---|---|
| NAT Gateway | nat-0702c9152a11907d3 |
| Location | Public subnet |
| Purpose | Outbound connectivity |

The NAT Gateway is an outbound-only path from the private subnet.

It does not provide inbound Internet access to the EC2 instance.

---

## 5.6 NAT Traffic Flow

The traffic flow is:

```text
Private EC2
    |
    v
Private Route Table
    |
    v
NAT Gateway
    |
    v
Internet Gateway
    |
    v
Internet
```

The Internet cannot directly initiate a connection to the private EC2 instance through the NAT Gateway.

---

# 6. Private EC2 Architecture

## 6.1 EC2 Purpose

The EC2 instance represents the private application/server workload.

The instance does not have a public IP address.

CloudFront is intended to provide the public entry point.

---

## 6.2 Current POC EC2

| Property | Value |
|---|---|
| Instance ID | i-043ba12e8364ece96 |
| Region | ap-south-1 |
| Network | Private subnet |
| Public IP | None |
| Operating System | Amazon Linux 2023 |
| Root Volume | Encrypted gp3 |
| Management | AWS Systems Manager |

The instance ID is environment-specific and must not be hardcoded in reusable templates.

---

## 6.3 Private IP Address

The EC2 private IP address is allocated by AWS.

The architecture does not depend on a fixed EC2 private IP.

CloudFormation obtains the instance information dynamically.

This is important for reusability because a future deployment can create a different private IP.

---

## 6.4 EC2 Internet Access

The EC2 instance does not have a public IP.

When the instance needs outbound connectivity, traffic follows:

```text
EC2
 |
 v
Private Route Table
 |
 v
NAT Gateway
 |
 v
Internet Gateway
 |
 v
Internet
```

This outbound connectivity is also important for initial configuration and AWS service communication when VPC endpoints are not being used.

---

# 7. EC2 Security Group

The EC2 security group controls inbound and outbound traffic.

## 7.1 Inbound HTTP

HTTP TCP port 80 is allowed from the VPC CIDR.

```text
Protocol: TCP
Port: 80
Source: 172.31.0.0/16
```

This supports internal VPC connectivity and POC testing.

---

## 7.2 CloudFront Origin Access

HTTP TCP port 80 is also allowed from the CloudFront managed prefix list.

```text
Protocol: TCP
Port: 80
Source:
com.amazonaws.global.cloudfront.origin-facing
```

Current managed prefix list ID:

```text
pl-9aa247f3
```

This allows CloudFront origin-facing traffic to reach the private EC2 origin.

---

## 7.3 Outbound Traffic

Outbound traffic is allowed.

The POC security group uses:

```text
Outbound:
All traffic
Destination:
0.0.0.0/0
```

This supports outbound connectivity through the NAT Gateway.

---

# 8. CloudFront Architecture

## 8.1 Purpose

Amazon CloudFront provides the public application entry point.

The intended traffic flow is:

```text
User
 |
 v
CloudFront
 |
 v
VPC Origin
 |
 v
Private EC2
```

The EC2 instance itself remains private.

---

## 8.2 CloudFront VPC Origin

CloudFront VPC Origin allows CloudFront to connect to resources inside a private VPC.

The POC uses the private EC2 instance as the origin resource.

The VPC Origin points to the EC2 resource rather than exposing the EC2 instance publicly.

---

## 8.3 VPC Origin Endpoint

The manually created POC VPC Origin previously displayed an endpoint similar to:

```text
ip-172-31-63-241.ap-south-1.compute.internal
```

The actual endpoint is AWS-managed and should not be treated as a permanent value.

---

## 8.4 VPC Origin Protocol

The intended configuration is:

```text
CloudFront -> VPC Origin
Protocol: HTTP
Port: 80
```

The public viewer connection is handled separately.

---

# 9. CloudFront Viewer Protocol

The intended CloudFront viewer behavior is:

```text
HTTP  -> HTTPS redirect
HTTPS -> Application
```

This means users are redirected from HTTP to HTTPS.

CloudFront handles the public TLS connection.

The connection from CloudFront to the private EC2 origin is configured separately.

---

# 10. CloudFront Caching

The POC CloudFront configuration disables application caching.

The purpose is to keep the POC behavior simple and similar to the target application architecture.

CloudFront forwards requests to the origin rather than serving cached application responses.

Caching can be designed separately for future production optimization.

---

# 11. CloudFront Distribution and VPC Origin Relationship

A CloudFront distribution can have multiple origins.

Each private EC2 origin used directly as a CloudFront VPC Origin generally requires its corresponding VPC Origin configuration.

A new EC2 instance does not automatically require a new CloudFront distribution.

For example:

```text
CloudFront Distribution
       |
       +---- VPC Origin 1 ---- EC2 Application 1
       |
       +---- VPC Origin 2 ---- EC2 Application 2
```

CloudFront behaviors can determine which origin receives a request.

---

# 12. NAT Gateway vs VPC Origin

These two components have different purposes.

## CloudFront VPC Origin

Used for:

```text
CloudFront
    |
    v
Private EC2
```

This is inbound application traffic from CloudFront to the private application.

## NAT Gateway

Used for:

```text
Private EC2
    |
    v
NAT Gateway
    |
    v
Internet / AWS Services
```

This is outbound traffic from the private application.

Therefore:

**NAT Gateway is not used to connect CloudFront to the EC2 instance.**

---

# 13. AWS WAF

## 13.1 Purpose

AWS WAF protects the CloudFront distribution from common web-based attacks and unwanted traffic.

The WAF is associated with CloudFront.

It is not a separate network hop.

Conceptually:

```text
Internet
   |
   v
CloudFront
   |
   +---- AWS WAF protection
   |
   v
VPC Origin
   |
   v
Private EC2
```

---

## 13.2 CloudFront WAF Region

CloudFront-scoped WAF resources are created in:

```text
us-east-1
```

This is different from the main application infrastructure region.

The POC application infrastructure is in:

```text
ap-south-1
```

The CloudFront WAF is therefore managed separately.

---

## 13.3 Managed Rules

The intended WAF configuration uses AWS Managed Rules including:

1. AWSManagedRulesCommonRuleSet
2. AWSManagedRulesKnownBadInputsRuleSet
3. AWSManagedRulesLinuxRuleSet
4. AWSManagedRulesSQLiRuleSet
5. AWSManagedRulesAmazonIpReputationList

These rules provide protection against common malicious requests and known attack patterns.

---

# 14. AWS WAF CloudFormation Stack

The WAF configuration is defined in:

```text
cloudformation/04-waf.yaml
```

The stack creates:

```text
AWS::WAFv2::WebACL
```

with:

```text
Scope: CLOUDFRONT
```

The stack exports the Web ACL ARN.

The CloudFront stack can then use the ARN when associating the WAF with the distribution.

---

# 15. CloudFront Account Verification

During CloudFormation deployment, CloudFront resource creation returned the following AWS error:

```text
Your account must be verified before you can add new CloudFront resources.
```

The failure occurred while creating:

```text
AWS::CloudFront::VpcOrigin
```

The CloudFormation template itself passed validation.

Therefore the issue was not identified as a YAML syntax problem.

The issue is an AWS account-level CloudFront resource creation restriction.

An AWS Support case was raised requesting account verification and CloudFront resource creation access.

Until the restriction is removed, the CloudFront CloudFormation stack cannot be fully deployed.

---

# 16. CloudFormation Architecture

The infrastructure is divided into multiple CloudFormation stacks.

```text
01-network
     |
     v
02-compute
     |
     v
03-cloudfront
     |
     v
04-waf

02-compute
     |
     v
05-backup
```

The dependency order is intentional.

---

# 17. CloudFormation Stack 01 – Network

File:

```text
cloudformation/01-network.yaml
```

Purpose:

- Create private subnet
- Create private route table
- Associate private subnet
- Optionally create NAT Gateway
- Create Elastic IP when NAT is enabled
- Create default NAT route

The VPC and public subnet are supplied as parameters.

---

# 18. Network Stack Parameters

Important parameters include:

```text
ExistingVpcId
ExistingPublicSubnetId
AvailabilityZone
PrivateSubnetCidr
EnableNatGateway
ProjectName
Environment
Owner
```

This allows the same template to be used with different VPCs and subnets.

---

# 19. Network Stack Outputs

The network stack exports values such as:

```text
VpcId
PrivateSubnetId
PrivateRouteTableId
NatGatewayId
```

The compute stack can consume these values using CloudFormation cross-stack references.

---

# 20. CloudFormation Stack 02 – Compute

File:

```text
cloudformation/02-compute.yaml
```

Purpose:

- Create IAM role
- Create EC2 instance profile
- Create EC2 security group
- Launch Amazon Linux EC2
- Configure SSM
- Configure CloudWatch Agent
- Create CloudWatch alarms

---

# 21. Compute Stack Dependencies

The compute stack depends on the network stack.

The private subnet is obtained from the network stack.

Conceptually:

```text
Network Stack
     |
     | PrivateSubnetId
     v
Compute Stack
     |
     v
Private EC2
```

---

# 22. EC2 IAM Role

The EC2 instance uses an IAM role.

The role includes:

```text
AmazonSSMManagedInstanceCore
CloudWatchAgentServerPolicy
```

These permissions support:

- Systems Manager Session Manager
- CloudWatch Agent
- CloudWatch metric publishing

---

# 23. Systems Manager

AWS Systems Manager is the preferred administrative access method.

The architecture does not depend on SSH access from the public Internet.

The intended administrative flow is:

```text
Administrator
     |
     v
AWS Systems Manager
     |
     v
Private EC2
```

This avoids exposing SSH port 22 publicly.

---

# 24. CloudWatch Monitoring

The EC2 instance is configured to send monitoring information to CloudWatch.

The CloudWatch Agent is configured to collect additional metrics such as:

- Memory utilization
- Root filesystem disk utilization

Metrics are collected periodically.

---

# 25. EC2 User Data

The EC2 User Data performs initial configuration.

The intended actions include:

```text
Install required packages
Install CloudWatch Agent
Configure timezone
Enable SSM
Enable CloudWatch Agent
Start required services
Configure monitoring
```

The configuration depends on outbound network access through NAT or suitable VPC endpoints.

---

# 26. CloudFormation Stack 03 – CloudFront

File:

```text
cloudformation/03-cloudfront.yaml
```

Purpose:

- Create CloudFront VPC Origin
- Create CloudFront distribution
- Connect CloudFront to private EC2
- Configure viewer protocol policy
- Configure origin protocol
- Configure caching behavior
- Optionally associate WAF

---

# 27. CloudFront Stack Dependency

The CloudFront stack depends on the compute stack.

Conceptually:

```text
Compute Stack
     |
     | EC2 ARN / private origin information
     v
CloudFront Stack
     |
     +---- VPC Origin
     |
     +---- CloudFront Distribution
```

---

# 28. CloudFormation Stack 04 – WAF

File:

```text
cloudformation/04-waf.yaml
```

Purpose:

- Create CloudFront-scoped WAF
- Enable AWS managed rule groups
- Export Web ACL ARN

The stack must be deployed in:

```text
us-east-1
```

because the WAF scope is:

```text
CLOUDFRONT
```

---

# 29. CloudFormation Stack 05 – Backup

File:

```text
cloudformation/05-backup.yaml
```

Purpose:

- Create AWS Backup vault
- Create backup IAM role
- Create backup plan
- Configure weekly backup
- Configure retention
- Select the EC2 instance

---

# 30. AWS Backup Schedule

The intended schedule is:

```text
Every Monday at 00:00 JST
```

The CloudFormation cron expression is:

```text
cron(0 15 ? * SUN *)
```

This is:

```text
15:00 UTC Sunday
```

which corresponds to:

```text
00:00 JST Monday
```

---

# 31. Backup Retention

The intended retention period is:

```text
30 days
```

The backup vault is configured with retention protection at the CloudFormation resource level using:

```text
DeletionPolicy: Retain
UpdateReplacePolicy: Retain
```

This reduces the risk of accidentally deleting the vault when the CloudFormation stack is removed.

---

# 32. Backup Scope

The POC backup configuration protects the EC2 instance selected by the backup selection.

The backup design should not automatically be interpreted as complete application-level data protection.

If the application uses additional resources such as:

- RDS
- EFS
- S3
- DynamoDB
- Application-specific storage

those resources require their own backup and recovery requirements.

---

# 33. Backup vs High Availability

AWS Backup is not the same as high availability.

Backup provides recovery capability.

High availability provides continued service availability.

For example:

```text
Backup:
EC2
 |
 v
Recovery Point
 |
 v
Restore
```

High availability would involve additional architecture such as:

```text
Load Balancer
     |
     +---- EC2 AZ-A
     |
     +---- EC2 AZ-B
```

The current POC focuses on backup and recovery rather than full high availability.

---

# 34. Security Architecture

The security model is based on minimizing public exposure.

The intended architecture is:

```text
Internet
   |
   v
CloudFront
   |
   v
WAF
   |
   v
VPC Origin
   |
   v
Private EC2
```

The EC2 instance does not have a public IP.

---

# 35. Public vs Private Resources

## Public

The public subnet is used for infrastructure that requires Internet Gateway connectivity.

In this POC:

```text
Public Subnet
    |
    +---- NAT Gateway
```

## Private

The private subnet contains the application EC2 instance.

```text
Private Subnet
    |
    +---- EC2
```

The private subnet does not assign public IP addresses to the EC2 instance.

---

# 36. Network Security Summary

| Component | Exposure |
|---|---|
| EC2 | Private |
| EC2 Public IP | None |
| SSH | Not required |
| HTTP | Restricted by Security Group |
| CloudFront | Public entry point |
| WAF | CloudFront protection |
| NAT Gateway | Outbound connectivity |
| Internet Gateway | Public subnet connectivity |

---

# 37. Resource Naming

The CloudFormation templates use a naming pattern based on:

```text
ProjectName
Environment
Resource
```

Example:

```text
IFIS-POC-network
IFIS-POC-compute
IFIS-POC-cloudfront
```

This makes resources easier to identify.

---

# 38. Tagging

Resources use tags such as:

```text
Project
Environment
Owner
```

Example:

```text
Project     = IFIS
Environment = POC
Owner       = Sohan
```

Tags should be adjusted for future client environments.

---

# 39. Repository Structure

The repository is organized as follows:

```text
sohan_IFIS_project/
|
+-- cloudformation/
|   +-- 01-network.yaml
|   +-- 02-compute.yaml
|   +-- 03-cloudfront.yaml
|   +-- 04-waf.yaml
|   +-- 05-backup.yaml
|
+-- docs/
|   +-- architecture.md
|   +-- deployment-guide.md
|   +-- validation-checklist.md
|   +-- troubleshooting.md
|
+-- config/
|
+-- scripts/
|
+-- diagrams/
|
+-- .gitignore
|
+-- README.md
```

---

# 40. CloudFormation Deployment Order

The recommended deployment order is:

```text
1. Network
      |
      v
2. Compute
      |
      +----------------+
      |                |
      v                v
3. CloudFront       5. Backup
      |
      v
4. WAF
```

Operationally, WAF can be prepared independently in `us-east-1`, but CloudFront must receive the Web ACL ARN when the distribution is configured.

---

# 41. Current POC Deployment Status

| Component | Status |
|---|---|
| VPC | Existing |
| Network Stack | Deployed |
| Private Subnet | Created |
| Private Route Table | Created |
| NAT Gateway | Created |
| Compute Stack | Deployed |
| Private EC2 | Deployed |
| Security Group | Configured |
| SSM | Configured |
| CloudWatch | Configured |
| CloudFront Template | Validated |
| CloudFront Deployment | Blocked by AWS account verification |
| WAF Template | Validated |
| WAF Deployment | Pending |
| Backup Template | Validated |
| Backup Deployment | Pending |

---

# 42. Important Current AWS IDs

The following IDs represent the current POC environment only.

They must not be hardcoded into reusable templates.

```text
VPC:
vpc-06900f62513eff63

VPC CIDR:
172.31.0.0/16

Public Subnet:
subnet-0d6445bd9f644383b

Private Subnet:
subnet-08183fb86ccc151e1

Private Subnet CIDR:
172.31.48.0/20

NAT Gateway:
nat-0702c9152a11907d3

EC2:
i-043ba12e8364ece96

CloudFront Managed Prefix List:
pl-9aa247f3
```

These values are documented for POC troubleshooting only.

---

# 43. Reusability Design

The CloudFormation templates are designed to avoid hardcoding environment-specific resources wherever practical.

For example, the network template accepts:

```text
ExistingVpcId
ExistingPublicSubnetId
PrivateSubnetCidr
AvailabilityZone
```

The compute stack receives network information from the network stack.

The backup stack receives the EC2 resource ARN from the compute stack.

This creates a reusable dependency chain.

---

# 44. Cross-Stack References

CloudFormation exports and imports are used to connect stacks.

Conceptually:

```text
Network Stack
     |
     | Export
     v
PrivateSubnetId
     |
     | Import
     v
Compute Stack
```

and:

```text
Compute Stack
     |
     | Export
     v
EC2InstanceArn
     |
     | Import
     v
Backup Stack
```

This avoids hardcoding resource IDs.

---

# 45. Multi-Account Reuse

For a future AWS account, the same CloudFormation templates can be reused.

Environment-specific values can be supplied during deployment.

Example:

```text
Account A
    |
    +-- VPC A
    +-- Private Subnet A
    +-- EC2 A

Account B
    |
    +-- VPC B
    +-- Private Subnet B
    +-- EC2 B
```

The templates remain the same while parameter values change.

---

# 46. Multi-Region Reuse

The application infrastructure can be deployed into another AWS region by supplying region-specific parameters and deploying the stacks in that region.

However, CloudFront and CloudFront-scoped WAF have special regional behavior.

The WAF CloudFront scope must remain associated with the CloudFront global service architecture.

Region-specific resources such as:

- VPC
- Subnet
- NAT Gateway
- EC2
- Security Groups

must exist in the target AWS region.

---

# 47. Production Improvements

The POC is intentionally simpler than a production architecture.

Possible production improvements include:

- Multiple Availability Zones
- Multiple private subnets
- NAT Gateway per Availability Zone
- Application Load Balancer
- Multiple EC2 instances
- Auto Scaling
- Route 53 health checks
- Stronger security group restrictions
- VPC endpoints
- Centralized logging
- GuardDuty
- Security Hub
- AWS Config
- CloudTrail
- KMS key management
- More detailed CloudWatch monitoring
- Automated disaster recovery testing

These are design considerations and are not automatically part of the current POC.

---

# 48. POC vs Production

The current POC is intended to prove the following architecture:

```text
CloudFront
    |
    v
VPC Origin
    |
    v
Private EC2
```

with:

```text
Private EC2
    |
    v
NAT Gateway
    |
    v
Outbound Connectivity
```

and:

```text
CloudFront
    |
    v
AWS WAF
```

plus:

```text
EC2
 |
 +---- CloudWatch
 |
 +---- SSM
 |
 +---- AWS Backup
```

Production architecture should be reviewed separately before implementation.

---

# 49. Validation Strategy

Validation should be performed layer by layer.

## Layer 1 – Network

Validate:

- VPC
- Private subnet
- Route table
- NAT Gateway
- Default route
- Internet connectivity

## Layer 2 – Compute

Validate:

- EC2 running
- Private IP
- No public IP
- Security Group
- SSM
- CloudWatch Agent

## Layer 3 – CloudFront

Validate:

- VPC Origin
- Distribution
- Viewer HTTPS
- Origin connectivity
- Application response

## Layer 4 – WAF

Validate:

- Web ACL
- Managed rules
- CloudFront association
- WAF metrics

## Layer 5 – Backup

Validate:

- Backup vault
- Backup plan
- Backup selection
- Recovery point
- Restore operation

---

# 50. Failure Testing

The POC should eventually test failure scenarios.

Examples include:

## NAT Gateway Failure

Expected impact:

```text
EC2 -> Internet
```

may fail.

CloudFront -> EC2 does not use NAT.

---

## EC2 Failure

Expected impact:

```text
CloudFront -> EC2
```

fails if there is no alternate origin.

This demonstrates why a production architecture may require multiple application instances.

---

## CloudFront Failure

The public application entry point becomes unavailable.

The private EC2 instance may still be running.

---

## WAF Configuration Issue

Incorrect WAF rules can block legitimate traffic.

WAF monitoring and testing should therefore be performed carefully.

---

## Backup Restore Test

A recovery point should be restored to a test environment to verify that the backup is actually usable.

A successful backup job alone does not prove that application recovery is successful.

---

# 51. Operational Access

The preferred operational access method is:

```text
AWS Console / AWS CLI
        |
        v
AWS Systems Manager
        |
        v
EC2
```

The architecture does not require exposing SSH to the Internet.

---

# 52. Cost Considerations

The following POC resources can generate ongoing AWS charges:

- NAT Gateway
- Elastic IP associated with NAT
- EC2
- EBS volume
- CloudWatch
- CloudFront
- AWS WAF
- AWS Backup
- Data transfer

For a temporary POC, unused resources should be deleted after validation.

CloudFormation templates and Git repository contents remain available even after the AWS resources are deleted.

---

# 53. Resource Cleanup

CloudFormation should be used to remove resources where possible.

Recommended cleanup order:

```text
CloudFront
    |
    v
Backup
    |
    v
Compute
    |
    v
Network
```

Before deleting resources, verify whether any retained backup vaults or other resources need to remain.

The AWS Backup vault intentionally uses retention settings to reduce accidental deletion.

---

# 54. Important Cleanup Consideration

EC2 termination protection can prevent CloudFormation from deleting the instance.

If termination protection is enabled, it may need to be disabled before deleting the compute stack.

Example:

```bash
aws ec2 modify-instance-attribute \
  --instance-id <instance-id> \
  --no-disable-api-termination \
  --region ap-south-1
```

Always verify the instance ID before running this command.

---

# 55. CloudFormation Validation

Before deploying a template, validate it.

Example:

```bash
aws cloudformation validate-template \
  --template-body file://cloudformation/01-network.yaml \
  --region ap-south-1
```

Repeat for the other templates.

---

# 56. CloudFormation Change Management

Before production deployment, use CloudFormation change sets where appropriate.

Recommended workflow:

```text
Edit Template
     |
     v
Validate Template
     |
     v
Create Change Set
     |
     v
Review Changes
     |
     v
Execute Change Set
     |
     v
Validate Resources
```

Templates should be maintained in Git.

---

# 57. Git Version Control

The infrastructure code and documentation are maintained in Git.

The repository provides:

- Version history
- Change tracking
- Documentation history
- Rollback reference
- Reusable templates
- Team collaboration

The repository should not contain:

- Passwords
- Access keys
- Private keys
- Secrets
- Sensitive credentials

---

# 58. Sensitive Information

The following should never be committed to Git:

```text
AWS Access Keys
AWS Secret Keys
Private SSH Keys
Passwords
API Tokens
Application Secrets
.env files containing credentials
```

The `.gitignore` file is configured to exclude common sensitive files.

---

# 59. Recommended Future Configuration Management

For future client environments, environment-specific configuration can be stored separately from the templates.

For example:

```text
config/
|
+-- poc-ap-south-1.json
+-- client-a-ap-south-1.json
+-- client-b-ap-northeast-1.json
```

The same CloudFormation templates can then be reused with different configuration values.

---

# 60. Architecture Principles

The POC follows these major principles:

### Private application

The application EC2 instance is not directly exposed to the Internet.

### Managed public entry point

CloudFront provides the public application entry point.

### Web protection

AWS WAF protects the CloudFront layer.

### Controlled outbound connectivity

NAT Gateway provides outbound connectivity for private resources.

### Managed administration

Systems Manager provides administrative access without requiring public SSH.

### Monitoring

CloudWatch provides infrastructure monitoring.

### Backup

AWS Backup provides scheduled recovery points.

### Infrastructure as Code

CloudFormation provides repeatable infrastructure deployment.

### Version control

Git provides infrastructure and documentation version history.

---

# 61. Final Architecture Summary

The complete logical architecture is:

```text
                         INTERNET
                            |
                            v
                       ROUTE 53
                            |
                            v
                      CLOUDFRONT
                            |
                       AWS WAF
                            |
                            v
                    CLOUDFRONT VPC
                       ORIGIN
                            |
                            v
                  +------------------+
                  |      VPC         |
                  |                  |
                  | Private Subnet   |
                  |                  |
                  |   +----------+   |
                  |   |   EC2    |   |
                  |   +----------+   |
                  |        |         |
                  |        |         |
                  |        v         |
                  |   NAT Gateway    |
                  |        |         |
                  +--------|---------+
                           |
                           v
                        INTERNET


Supporting Services:

EC2 ---------> Systems Manager
 |
 +-----------> CloudWatch
 |
 +-----------> AWS Backup
```

---

# 62. Target Deployment Model

The reusable deployment model is:

```text
Existing VPC
     |
     v
Network Stack
     |
     v
Compute Stack
     |
     +--------------------+
     |                    |
     v                    v
CloudFront Stack       Backup Stack
     |
     v
WAF Association
```

The templates are intended to be reusable across environments by changing parameters rather than modifying the core infrastructure logic.

---

# 63. Current Repository Templates

The current CloudFormation templates are:

```text
cloudformation/01-network.yaml
cloudformation/02-compute.yaml
cloudformation/03-cloudfront.yaml
cloudformation/04-waf.yaml
cloudformation/05-backup.yaml
```

Each template has a focused responsibility.

This separation makes the infrastructure easier to understand, deploy, validate, troubleshoot, and reuse.

---

# 64. Current POC Conclusion

The POC has successfully demonstrated the foundational AWS infrastructure required for the IFIS architecture.

The following areas have been implemented or prepared:

- Existing VPC integration
- Private subnet
- Private route table
- NAT Gateway
- Private EC2
- Security Group
- IAM role
- Systems Manager
- CloudWatch monitoring
- CloudFormation network stack
- CloudFormation compute stack
- CloudFront VPC Origin template
- CloudFront distribution template
- CloudFront WAF template
- AWS Backup template
- Architecture documentation

The remaining CloudFront validation depends on AWS account verification.

Once CloudFront resource creation is enabled, the remaining validation should continue with:

```text
CloudFront
    |
    v
VPC Origin
    |
    v
Private EC2
    |
    v
Application Test
```

followed by:

```text
AWS WAF
    |
    v
CloudFront
```

and:

```text
AWS Backup
    |
    v
Backup
    |
    v
Restore Test
```

---

# 65. Related Documentation

Additional project documentation should include:

```text
docs/deployment-guide.md
docs/validation-checklist.md
docs/troubleshooting.md
```

These documents should explain:

- How to deploy the stacks
- Required parameters
- Validation commands
- AWS Console validation
- Troubleshooting
- Failure scenarios
- Cleanup
- Restore procedures
- Future environment replication

---

# 66. Document Maintenance

This document should be updated whenever there is a significant architecture change.

Examples:

- New AWS service
- New subnet
- New CloudFormation stack
- New security control
- CloudFront architecture change
- WAF rule change
- Backup policy change
- Monitoring change
- Production architecture change

Environment-specific IDs should be updated only where required for POC operational reference.

Reusable CloudFormation templates should continue to avoid hardcoded environment-specific resource IDs wherever possible.

---

# 67. Document Ownership

| Item | Value |
|---|---|
| Project | IFIS |
| Environment | POC |
| Document | AWS Architecture |
| Infrastructure | AWS CloudFormation |
| Repository | sohan_IFIS_project |
| Primary Region | ap-south-1 |
| CloudFront WAF Region | us-east-1 |

---

# 68. End State

The desired end state of the POC is:

```text
User
 |
 v
Route 53
 |
 v
CloudFront
 |
 v
AWS WAF
 |
 v
CloudFront VPC Origin
 |
 v
Private EC2
 |
 +---- SSM
 |
 +---- CloudWatch
 |
 +---- AWS Backup
 |
 +---- NAT Gateway
          |
          v
       Internet
```

This architecture provides a private application server with CloudFront as the public entry point, WAF protection at the CloudFront layer, controlled outbound connectivity through NAT Gateway, centralized AWS management through Systems Manager, monitoring through CloudWatch, and scheduled backup through AWS Backup.