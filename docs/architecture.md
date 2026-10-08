# IFIS POC – AWS Architecture

## 1. Document Purpose

This document describes the AWS architecture implemented for the IFIS POC (Proof of Concept).

The purpose of this POC is to reproduce the core architecture of the W2P-Japan / IFIS production environment in a separate AWS account/environment so that the design can be:

- Understood by the project team
- Tested safely without affecting production
- Validated through hands-on deployment
- Documented for future handover
- Reused for future client environments
- Reused across AWS accounts and regions with different parameters

The POC is designed using AWS CloudFormation wherever practical so that the infrastructure can be deployed consistently and repeatedly.

---

## 2. Project Overview

### Project

W2P-Japan / IFIS

### Environment

POC

### Primary AWS Region

```text
ap-south-1

Mumbai region.

CloudFront / WAF Region

CloudFront-scoped WAF resources are created in:

us-east-1

This is required because AWS WAF resources with CloudFront scope are managed from the US East (N. Virginia) region.

3. High-Level Architecture

The target architecture is:

                         Internet Users
                               |
                               v
                         Route 53 / DNS
                               |
                               v
                         Amazon CloudFront
                               |
                               v
                        AWS WAF Web ACL
                               |
                               v
                       CloudFront VPC Origin
                               |
                               v
                    +-----------------------+
                    |       AWS VPC         |
                    |                       |
                    |   Private Subnet      |
                    |                       |
                    |   +---------------+   |
                    |   | Private EC2   |   |
                    |   | Web Server     |   |
                    |   +---------------+   |
                    |          |            |
                    +----------|------------+
                               |
                               v
                         NAT Gateway
                               |
                               v
                           Internet

AWS Backup and CloudWatch operate separately as supporting services:

                    +----------------+
                    |   AWS Backup   |
                    +-------+--------+
                            |
                            v
                      Private EC2

                    +----------------+
                    |   CloudWatch   |
                    +-------+--------+
                            |
                            v
                  EC2 Metrics / Alarms

                    +----------------+
                    | SSM Session    |
                    | Manager        |
                    +-------+--------+
                            |
                            v
                      Private EC2
4. AWS Services Used

The POC uses the following AWS services:

Service	Purpose
Amazon VPC	Network isolation
Amazon EC2	Private application/web server
Internet Gateway	Internet connectivity for public subnet
NAT Gateway	Outbound internet access from private subnet
Amazon Route Tables	Network routing
Security Groups	Instance-level network security
AWS CloudFront	Public entry point / CDN
CloudFront VPC Origin	Connect CloudFront to private EC2
AWS WAF	Web application protection
AWS Systems Manager	Secure EC2 administration
Amazon CloudWatch	Monitoring and alarms
AWS Backup	EC2 backup and recovery
AWS CloudFormation	Infrastructure as Code
AWS IAM	Roles and permissions
5. Network Architecture
5.1 Existing VPC

The POC uses the existing default VPC.

VPC ID:
vpc-06900f62513eff63

CIDR:

172.31.0.0/16

The CloudFormation network stack does not create a new VPC.

Instead, the existing VPC is passed as a parameter.

This makes the CloudFormation template reusable because a future environment can provide a different VPC ID.

5.2 Public Subnet Used for NAT Gateway

The POC uses an existing public subnet for the NAT Gateway.

Subnet ID:
subnet-0d6445bd9f644383b

CIDR:

172.31.0.0/20

Availability Zone:

ap-south-1b

The subnet is used because the NAT Gateway requires a public subnet with internet access through an Internet Gateway.

5.3 Private Subnet

A new private subnet is created by the CloudFormation network stack.

CIDR:

172.31.48.0/20

Availability Zone:

ap-south-1b

Public IP assignment is disabled.

The EC2 instance is deployed into this subnet.

The subnet is considered private because it does not have a direct route to the Internet Gateway.

5.4 Private Route Table

A dedicated route table is created for the private subnet.

The expected routing is:

Destination             Target
-----------------------------------------
172.31.0.0/16           local
0.0.0.0/0               NAT Gateway

The local route allows communication inside the VPC.

The NAT Gateway route allows resources in the private subnet to initiate outbound connections to the internet.

5.5 NAT Gateway

The POC creates a NAT Gateway in the existing public subnet.

Example:

NAT Gateway:
nat-0702c9152a11907d3

The NAT Gateway provides outbound connectivity for the private EC2 instance.

Typical outbound communication includes:

AWS Systems Manager communication
CloudWatch Agent communication
Operating system package downloads
External internet connections initiated by the server

The NAT Gateway does not provide inbound internet access to the EC2 instance.

This distinction is important.

Internet
   |
   X
   |
Private EC2

Inbound internet traffic does not directly reach the private EC2.

For outbound traffic:

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
5.6 Network Stack

The network infrastructure is managed by:

cloudformation/01-network.yaml

CloudFormation stack:

IFIS-POC-network

The stack creates:

Private subnet
Private route table
Route table association
NAT Gateway
Elastic IP for NAT Gateway
Default route through NAT Gateway

NAT creation is controlled through a CloudFormation parameter.

This allows the same template to be used in environments where NAT is required or not required.

6. Compute Architecture
6.1 EC2 Instance

The application/web server is deployed as a private EC2 instance.

Current POC instance:

i-043ba12e8364ece96

The instance is deployed inside the private subnet.

The EC2 instance does not have a public IPv4 address.

This is intentional.

6.2 Operating System

The EC2 instance uses:

Amazon Linux 2023

The AMI is selected through the CloudFormation template.

The template uses the AWS Systems Manager public parameter for the latest Amazon Linux 2023 AMI.

This avoids hardcoding an AMI ID.

6.3 Instance Type

The POC compute template uses:

t3a.small

as the default instance type.

The instance type is a CloudFormation parameter and can therefore be changed for another environment.

6.4 Root Volume

The EC2 root volume is:

20 GB
gp3
Encrypted

The root volume configuration is controlled by CloudFormation parameters.

6.5 Instance Metadata

The EC2 instance requires:

IMDSv2

This improves the security of instance metadata access.

6.6 Termination Protection

Termination protection is enabled on the EC2 instance by the CloudFormation template.

This is intended to reduce the chance of accidental termination.

During POC cleanup, termination protection must be disabled before the EC2 instance can be deleted.

7. EC2 IAM Role

The EC2 instance uses an IAM role and instance profile.

The role provides permissions required for:

AWS Systems Manager Session Manager
CloudWatch Agent

The main managed policies are:

AmazonSSMManagedInstanceCore
CloudWatchAgentServerPolicy

No access keys are stored on the EC2 instance.

This is preferred over creating and storing long-term IAM credentials.

8. Systems Manager Session Manager

The POC uses AWS Systems Manager Session Manager for EC2 administration.

The intended administrative access model is:

Administrator
     |
     v
AWS Systems Manager
     |
     v
Private EC2

SSH from the public internet is not required.

The EC2 instance therefore does not need:

Public IP
Public SSH access
SSH security group rule

This provides a more secure administrative model.

9. CloudWatch Monitoring

The EC2 instance is configured to use the CloudWatch Agent.

The bootstrap configuration installs and starts the CloudWatch Agent.

The POC collects additional system metrics such as:

Memory usage
Root disk usage

Metrics are sent periodically to CloudWatch.

CloudWatch alarms are also created by the compute CloudFormation stack.

These alarms can be configured for thresholds such as:

CPU utilization
Memory utilization
Disk utilization

The thresholds are CloudFormation parameters and can be changed for different environments.

10. EC2 Security Group

The EC2 security group controls access to the private instance.

The expected inbound HTTP rules are:

TCP 80
Source: VPC CIDR

and:

TCP 80
Source: CloudFront origin-facing managed prefix list

The VPC CIDR for the POC is:

172.31.0.0/16

The CloudFront managed prefix list used in the POC is:

pl-9aa247f3

Name:

com.amazonaws.global.cloudfront.origin-facing

Outbound traffic is allowed.

The important design principle is:

Internet
   |
   v
CloudFront
   |
   v
VPC Origin
   |
   v
Security Group
   |
   v
Private EC2

The EC2 instance itself does not need to be publicly accessible.

11. CloudFront Architecture
11.1 Purpose

Amazon CloudFront provides the public entry point to the application.

The intended request path is:

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

This removes the requirement for a public IP address on the EC2 instance.

11.2 CloudFront VPC Origin

CloudFront VPC Origin is used to connect CloudFront to the private EC2 instance.

The VPC Origin points to the EC2 private resource.

The origin protocol is:

HTTP

Origin port:

80

The CloudFront distribution can therefore forward HTTP requests from CloudFront to the private EC2 instance.

11.3 VPC Origin and NAT Are Different

This distinction is important.

CloudFront VPC Origin

Provides the inbound application path:

CloudFront
     |
     v
VPC Origin
     |
     v
Private EC2
NAT Gateway

Provides outbound connectivity:

Private EC2
     |
     v
NAT Gateway
     |
     v
Internet

NAT Gateway is not used to connect CloudFront to the EC2 instance.

12. CloudFront Viewer Protocol

The intended CloudFront viewer behavior is:

HTTP  -> HTTPS redirect
HTTPS -> Application

This means users are redirected from HTTP to HTTPS.

CloudFront handles the public TLS connection.

The connection from CloudFront to the private EC2 origin is configured separately.

13. CloudFront Caching

The POC CloudFront configuration disables caching for the application.

This is intended to keep the POC behavior simple and similar to the target architecture.

CloudFront forwards requests to the origin rather than serving cached application responses.

Caching can be designed separately for future production optimization.

14. CloudFront Distribution and VPC Origin Relationship

A CloudFront distribution can have multiple origins.

Each private EC2 origin used directly as a CloudFront VPC Origin generally requires its corresponding VPC Origin configuration.

A new EC2 instance does not automatically require a new CloudFront distribution.

For example:

                 CloudFront Distribution
                         |
             +-----------+-----------+
             |                       |
             v                       v
        VPC Origin 1            VPC Origin 2
             |                       |
             v                       v
          EC2-1                   EC2-2

Different CloudFront behaviors can route different URL paths to different origins.

15. AWS WAF
15.1 Purpose

AWS WAF protects the CloudFront distribution from common web-based attacks.

The WAF Web ACL is configured with AWS Managed Rules.

15.2 WAF Scope

The WAF Web ACL uses:

Scope: CLOUDFRONT

CloudFront-scoped WAF resources are created in:

us-east-1

This is different from the main application infrastructure region.

The application infrastructure is deployed in:

ap-south-1
15.3 Managed Rules

The POC WAF configuration contains the following AWS Managed Rule Groups:

AWSManagedRulesCommonRuleSet
AWSManagedRulesKnownBadInputsRuleSet
AWSManagedRulesLinuxRuleSet
AWSManagedRulesSQLiRuleSet
AWSManagedRulesAmazonIpReputationList

These rules provide protection against common malicious request patterns and known bad sources.

15.4 WAF Request Flow

WAF is associated with CloudFront.

It is not a separate network hop.

The logical request flow is:

Internet User
     |
     v
CloudFront
     |
     +---- AWS WAF inspection
     |
     v
VPC Origin
     |
     v
Private EC2
15.5 WAF CloudFormation Stack

WAF is managed using:

cloudformation/04-waf.yaml

The WAF stack is intentionally separate from the CloudFront stack.

This allows the WAF to be deployed in the required region and then supplied to the CloudFront stack.

16. AWS Backup
16.1 Purpose

AWS Backup is used to protect the EC2 instance.

The backup design provides a recovery point that can be used for restoration.

Backup is separate from high availability.

A backup does not automatically provide failover.

16.2 Backup Schedule

The planned schedule is:

Weekly
Monday 00:00 JST

The CloudFormation template uses:

cron(0 15 ? * SUN *)

This is:

15:00 UTC Sunday

which corresponds to:

00:00 Monday JST
16.3 Retention

The planned backup retention period is:

30 days
16.4 Backup Vault

A dedicated backup vault is created:

IFIS-POC-backup-vault

The CloudFormation template uses:

DeletionPolicy: Retain
UpdateReplacePolicy: Retain

This is intentional because deleting the CloudFormation stack should not automatically remove the backup vault and its recovery points.

16.5 Backup Selection

The backup selection obtains the EC2 instance ARN from the compute CloudFormation stack.

This creates a dependency between:

IFIS-POC-compute
        |
        v
IFIS-POC-backup

The compute stack must therefore exist before the backup stack can be deployed.

17. CloudFormation Architecture

The infrastructure is separated into multiple CloudFormation stacks.

The planned deployment order is:

01 Network
    |
    v
02 Compute
    |
    v
03 CloudFront
    |
    v
04 WAF
    |
    v
05 Backup

However, WAF is technically created in us-east-1, while the other application resources are in ap-south-1.

18. CloudFormation Stack 01 – Network

Template:

cloudformation/01-network.yaml

Stack:

IFIS-POC-network

Responsibilities:

Use existing VPC
Use existing public subnet
Create private subnet
Create private route table
Associate private subnet with route table
Create NAT Gateway when enabled
Create NAT EIP
Create default route through NAT Gateway
Export network values

Important outputs include:

VpcId
PrivateSubnetId
PrivateRouteTableId
NatGatewayId
19. CloudFormation Stack 02 – Compute

Template:

cloudformation/02-compute.yaml

Stack:

IFIS-POC-compute

Responsibilities:

Create IAM role
Create instance profile
Create EC2 security group
Create private EC2 instance
Install required agents
Configure CloudWatch monitoring
Configure SSM access
Create CloudWatch alarms

Important outputs include:

EC2InstanceId
EC2PrivateIp
EC2SecurityGroupId
EC2InstanceProfile
EC2RoleArn
EC2PrivateDnsName
EC2InstanceArn
20. CloudFormation Stack 03 – CloudFront

Template:

cloudformation/03-cloudfront.yaml

Stack:

IFIS-POC-cloudfront

Responsibilities:

Create CloudFront VPC Origin
Connect CloudFront to private EC2
Create CloudFront distribution
Configure viewer protocol
Configure origin
Configure caching behavior
Optionally attach WAF

Important outputs include:

VpcOriginId
DistributionId
DistributionDomainName
21. CloudFormation Stack 04 – WAF

Template:

cloudformation/04-waf.yaml

Stack:

IFIS-POC-waf

Region:

us-east-1

Responsibilities:

Create CloudFront-scoped WAF Web ACL
Enable AWS Managed Rules
Export WAF ARN
Export WAF ID
Export WAF name

Important output:

WebAclArn

The WAF ARN is then provided to the CloudFront stack.

22. CloudFormation Stack 05 – Backup

Template:

cloudformation/05-backup.yaml

Stack:

IFIS-POC-backup

Responsibilities:

Create AWS Backup vault
Create AWS Backup IAM role
Create backup plan
Configure weekly schedule
Configure retention
Select EC2 instance for backup
23. Cross-Stack Dependencies

The stacks are designed to communicate using CloudFormation Outputs and Exports.

Example:

Network Stack
      |
      | PrivateSubnetId
      v
Compute Stack
      |
      | EC2InstanceArn
      v
Backup Stack

For CloudFront:

Compute Stack
      |
      | EC2PrivateDnsName
      | EC2InstanceArn
      v
CloudFront Stack

This avoids hardcoding resource IDs inside templates.

24. Reusability

The primary design objective of the POC is reusability.

The templates should not depend on fixed production IDs.

For example, the network template accepts:

ExistingVpcId
ExistingPublicSubnetId
PrivateSubnetCidr
AvailabilityZone
EnableNatGateway

The compute template accepts parameters such as:

NetworkStackName
AmiId
InstanceType
RootVolumeSize
CloudFrontOriginFacingPrefixListId

This allows the same templates to be used for another environment.

For example:

Environment A
VPC: vpc-AAAA
Subnet: subnet-AAAA

Environment B
VPC: vpc-BBBB
Subnet: subnet-BBBB

The CloudFormation templates do not need to be rewritten.

Only the parameters change.

25. Repository Structure

The project repository is organized as follows:

sohan_IFIS_project/
│
├── cloudformation/
│   ├── 01-network.yaml
│   ├── 02-compute.yaml
│   ├── 03-cloudfront.yaml
│   ├── 04-waf.yaml
│   └── 05-backup.yaml
│
├── docs/
│   ├── architecture.md
│   ├── deployment-guide.md
│   ├── validation-checklist.md
│   └── troubleshooting.md
│
├── config/
│
├── scripts/
│
├── diagrams/
│   └── IFIS POC AWS Architecture Diagram.png
│
├── .gitignore
└── README.md
26. Security Design

The main security principles used in the POC are:

Private EC2

The EC2 instance does not have a public IP address.

Session Manager

Administrative access is performed using AWS Systems Manager Session Manager.

Security Group

Inbound HTTP access is restricted to:

VPC sources
CloudFront origin-facing managed prefix list
IMDSv2

EC2 metadata access requires IMDSv2.

Encryption

The EC2 root EBS volume is encrypted.

IAM Roles

AWS services use IAM roles instead of storing long-term credentials on the EC2 instance.

WAF

CloudFront is protected by AWS WAF Managed Rules.

Backup

EC2 recovery points are stored in AWS Backup.

27. POC Versus Production

The current environment is a POC.

It is not intended to be a production-ready highly available architecture in its current form.

Important production considerations include:

Multi-AZ

The POC currently uses one Availability Zone.

A production environment should consider multiple Availability Zones.

NAT Gateway

The POC uses one NAT Gateway.

A highly available production architecture may use multiple NAT Gateways, normally one per Availability Zone, depending on the design and cost requirements.

EC2 High Availability

The POC uses a single EC2 instance.

Production may require:

Multiple EC2 instances
Auto Scaling Group
Load Balancer
Multi-AZ architecture

depending on application requirements.

Backup

The POC has a backup plan, but backup alone does not provide automatic failover.

Production DR requirements should define:

Recovery Point Objective (RPO)
Recovery Time Objective (RTO)
Backup retention
Cross-region backup requirements
Restore procedures
Recovery testing
Monitoring

Production monitoring should include appropriate:

CloudWatch metrics
CloudWatch alarms
Logs
Dashboards
Notifications
Operational alerts
28. Current POC Status

The following components have been implemented and validated in the POC:

Network CloudFormation
        |
        v
Private Subnet
        |
        v
NAT Gateway
        |
        v
Private EC2
        |
        +---- SSM
        |
        +---- CloudWatch

The current EC2 instance is:

i-043ba12e8364ece96

The EC2 instance is running in the private subnet and does not have a public IP.

The Network and Compute CloudFormation stacks have successfully been deployed.

29. CloudFront Account Verification Dependency

During the POC deployment, CloudFormation successfully validated the CloudFront template.

However, creation of the CloudFront VPC Origin returned an AWS account-level restriction:

Your account must be verified before you can add new CloudFront resources.

The AWS Support case was raised to request verification/unblocking of CloudFront resource creation.

This is an AWS account-level restriction and is not a CloudFormation template syntax error.

The CloudFront template should therefore not be unnecessarily rewritten unless AWS Support identifies a template or configuration issue.

Once the account is verified, the CloudFront stack can be deployed and tested again.

30. Architecture Diagram

The project contains an architecture diagram:

diagrams/IFIS POC AWS Architecture Diagram.png

The diagram illustrates:

Route 53 / DNS
CloudFront
AWS WAF
CloudFront VPC Origin
VPC
Public subnet
NAT Gateway
Private subnet
Private EC2
Security Group
AWS Backup
CloudWatch
Systems Manager

The diagram is intended to communicate the logical architecture.

Resource IDs and subnet CIDRs should not be treated as permanent identifiers for future environments.

The CloudFormation templates are the reusable implementation.

31. Important Design Principle

The architecture should be understood as two separate traffic directions.

Incoming application traffic
Internet
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
Outgoing EC2 traffic
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
Internet / AWS Services

The two paths serve different purposes.

CloudFront VPC Origin provides the inbound application path.

NAT Gateway provides outbound connectivity from the private subnet.

32. Operational Access Model

The preferred administrative model is:

Administrator
 |
 v
AWS Console / AWS CLI
 |
 v
AWS Systems Manager
 |
 v
Session Manager
 |
 v
Private EC2

The architecture does not require direct public SSH access to the EC2 instance.

33. Deployment Philosophy

The infrastructure should be deployed in controlled stages.

Recommended order:

1. Network
2. Compute
3. CloudFront
4. WAF
5. Backup
6. Validation
7. Failure testing
8. Documentation

Each layer should be validated before moving to the next layer.

This makes troubleshooting easier because a failure can be isolated to a specific stack.

34. Validation Philosophy

The POC should not be considered successful simply because CloudFormation reports:

CREATE_COMPLETE

Each layer must also be functionally tested.

Examples:

Network

Verify:

Private subnet exists
Route table association is correct
NAT Gateway is available
Private EC2 has outbound connectivity
Compute

Verify:

EC2 is running
No public IP
Correct subnet
Correct security group
SSM is online
CloudWatch Agent is running
CloudFront

Verify:

Distribution is deployed
VPC Origin is deployed
CloudFront can reach the private EC2
HTTP redirects to HTTPS
Application response is received
WAF

Verify:

WAF is associated with CloudFront
Managed rules are enabled
WAF metrics are available
Backup

Verify:

Backup plan exists
EC2 is selected
Backup job completes
Recovery point is created
Restore procedure can be tested
35. Future Replication

The architecture is intended to be reusable for future client environments.

A future deployment should generally require:

Identify target AWS account.
Identify target AWS region.
Identify existing VPC.
Identify public subnet for NAT.
Select private subnet CIDR.
Deploy network stack.
Deploy compute stack.
Deploy CloudFront.
Deploy CloudFront-scoped WAF in us-east-1.
Deploy backup.
Validate all components.
Document environment-specific values.

The CloudFormation templates should remain largely unchanged.

Environment-specific values should be supplied through parameters or configuration.

36. Multi-Account and Multi-Region Considerations

For future environments, the same CloudFormation templates can be reused in different AWS accounts and regions.

However, the following values may differ:

AWS Account ID
Region
VPC ID
Subnet ID
Availability Zone
VPC CIDR
Private subnet CIDR
AMI
CloudFront managed prefix list
IAM naming requirements
CloudFront account restrictions
Existing networking

These values should therefore not be assumed to be globally identical.

37. Change Management

CloudFormation templates are stored in Git.

The repository provides:

Version history
Change tracking
Rollback reference
Reusable templates
Documentation
Team collaboration

Infrastructure changes should be made to the CloudFormation templates rather than manually modifying resources whenever practical.

Manual AWS Console changes may cause infrastructure drift.

38. Cost Awareness

The POC is intentionally designed to be cost-conscious.

Resources that may incur ongoing charges include infrastructure such as:

NAT Gateway
EC2
EBS
CloudWatch usage
CloudFront
AWS Backup
Data transfer

When the POC is not actively being tested, unnecessary resources should be stopped or deleted where appropriate.

Special attention should be given to the NAT Gateway because it is a continuously provisioned networking component.

Before deleting resources, ensure that any required logs, screenshots, test evidence, or documentation have been collected.

39. Cleanup Strategy

The POC resources are managed through CloudFormation.

Where possible, cleanup should be performed by deleting the corresponding CloudFormation stack rather than manually deleting individual resources.

Recommended dependency-aware cleanup order is approximately:

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

Some resources, such as the AWS Backup vault, intentionally use retention policies and may remain after stack deletion.

EC2 termination protection may also need to be disabled before the compute stack can be deleted.

40. Final Architecture Summary

The IFIS POC architecture demonstrates a secure private-EC2 application design using AWS managed services.

The main application path is:

User
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

The private EC2 instance receives no direct public internet access.

For outbound connectivity:

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

For administration:

Administrator
 |
 v
AWS Systems Manager Session Manager
 |
 v
Private EC2

For monitoring:

EC2
 |
 v
CloudWatch Agent
 |
 v
CloudWatch

For protection and recovery:

CloudFront
 |
 v
AWS WAF

EC2
 |
 v
AWS Backup

The architecture is implemented using reusable CloudFormation templates and is intended to provide a foundation that can be replicated across future AWS environments.

41. Related Documentation

The following documents provide additional operational information:

docs/deployment-guide.md
docs/validation-checklist.md
docs/troubleshooting.md