# IFIS POC – Manual AWS Console Deployment Guide

## 1. Purpose

This document describes how to manually deploy the IFIS AWS POC architecture using the AWS Management Console.

This guide is separate from the CloudFormation deployment guide.

Manual deployment is primarily intended for:

- AWS learning
- Architecture understanding
- Troubleshooting
- Demonstration
- Validation
- Emergency/manual deployment scenarios

For repeatable project deployments, the CloudFormation deployment method should be preferred.

---

# 2. Architecture

The target architecture is:

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
    CloudFront VPC Origin
        |
        v
    Private EC2
        |
        v
    NAT Gateway
        |
        v
    Internet

Supporting services:

    Private EC2
        |
        +---- AWS Systems Manager
        |
        +---- CloudWatch
        |
        +---- AWS Backup

---

# 3. Important Traffic Flows

## 3.1 Application Traffic

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
        |
        v
    Application

---

## 3.2 EC2 Outbound Traffic

    Private EC2
        |
        v
    Private Route Table
        |
        v
    NAT Gateway
        |
        v
    Internet

The NAT Gateway is used for outbound connectivity.

It is not the path used by CloudFront to reach the EC2 instance.

---

## 3.3 Administration

    Administrator
        |
        v
    AWS Systems Manager
        |
        v
    Private EC2

The EC2 instance does not require a public IP for normal administration.

---

# 4. Current POC Values

| Item | Current POC Value |
|---|---|
| AWS Region | ap-south-1 |
| WAF Region | us-east-1 |
| VPC ID | vpc-06900f62513eff63 |
| VPC CIDR | 172.31.0.0/16 |
| NAT Public Subnet | subnet-0d6445bd9f644383b |
| NAT Public Subnet CIDR | 172.31.0.0/20 |
| Availability Zone | ap-south-1b |
| Private Subnet CIDR | 172.31.48.0/20 |
| CloudFront Prefix List | pl-9aa247f3 |
| EC2 OS | Amazon Linux 2023 |
| EC2 Type | t3a.small |
| Application Port | TCP 80 |

### Important

These values are for the current POC only.

For another environment:

- Verify the VPC.
- Verify the public subnet.
- Verify the Availability Zone.
- Select an unused private subnet CIDR.
- Verify the CloudFront managed prefix list.
- Do not copy old AWS resource IDs blindly.

---

# 5. Prerequisites

You need access to:

- AWS Management Console
- VPC
- EC2
- IAM
- Systems Manager
- CloudWatch
- CloudFront
- AWS WAF
- AWS Backup

The AWS account must have sufficient permissions to create the required resources.

---

# 6. Select AWS Region

Sign in to the AWS Console.

For the main infrastructure select:

    Asia Pacific (Mumbai)
    ap-south-1

The CloudFront WAF will be configured separately in:

    US East (N. Virginia)
    us-east-1

---

# 7. Verify AWS Account

Before creating resources, verify that you are in the correct AWS account.

Check the account information in the AWS Console.

Do not create resources until the correct AWS account has been confirmed.

---

# PART 1 – NETWORK

# 8. Verify Existing VPC

Open:

    AWS Console
        |
        v
    VPC
        |
        v
    Your VPCs

Find the VPC to be used.

Current POC:

    VPC ID:
    vpc-06900f62513eff63

    CIDR:
    172.31.0.0/16

Verify:

- VPC state = Available
- Correct CIDR
- Correct AWS account
- Correct AWS Region

---

# 9. Verify Internet Gateway

Open:

    VPC
        |
        v
    Internet gateways

Find the Internet Gateway attached to the selected VPC.

Verify:

    State = Attached

The public subnet used for the NAT Gateway must have Internet Gateway connectivity.

---

# 10. Verify Public Subnets

Open:

    VPC
        |
        v
    Subnets

Current POC public subnets include:

    subnet-0d6445bd9f644383b
    172.31.0.0/20
    ap-south-1b

The NAT Gateway will be placed in this public subnet.

Verify that the subnet has a public route table.

---

# 11. Verify Public Route Table

Open:

    VPC
        |
        v
    Route Tables

Find the route table associated with the public subnet.

Open:

    Routes

Expected:

    Destination:
    0.0.0.0/0

    Target:
    Internet Gateway

This route allows the NAT Gateway's public subnet to reach the Internet.

---

# 12. Create Private Subnet

Open:

    AWS Console
        |
        v
    VPC
        |
        v
    Subnets

Click:

    Create subnet

Select the existing VPC.

Enter:

    Subnet name:
    IFIS-POC-private-subnet

Select:

    Availability Zone:
    ap-south-1b

Enter:

    IPv4 subnet CIDR block:
    172.31.48.0/20

Click:

    Create subnet

---

# 13. Disable Public IPv4 Assignment

Open the new private subnet.

Choose:

    Actions
        |
        v
    Edit subnet settings

Find:

    Enable auto-assign public IPv4 address

Make sure it is disabled.

Save the configuration.

The private subnet must not automatically assign public IPv4 addresses.

---

# 14. Create Private Route Table

Open:

    VPC
        |
        v
    Route Tables

Click:

    Create route table

Enter:

    Name:
    IFIS-POC-private-rt

Select the correct VPC.

Click:

    Create route table

---

# 15. Associate Private Subnet

Open the newly created private route table.

Select:

    Subnet associations

Click:

    Edit subnet associations

Select:

    IFIS-POC-private-subnet

Save.

---

# 16. Create NAT Gateway

Open:

    VPC
        |
        v
    NAT Gateways

Click:

    Create NAT gateway

Select:

    Subnet:
    <PUBLIC_SUBNET_ID>

Set:

    Connectivity type:
    Public

Allocate an Elastic IP.

Create the NAT Gateway.

Wait until:

    State = Available

---

# 17. Add NAT Route

Open the private route table.

Go to:

    Routes

Click:

    Edit routes

Add:

    Destination:
    0.0.0.0/0

Target:

    NAT Gateway

Select the newly created NAT Gateway.

Save.

---

# 18. Validate Network

Verify:

- [ ] Private subnet exists
- [ ] Private subnet uses correct CIDR
- [ ] Private subnet is in correct AZ
- [ ] Public IP assignment is disabled
- [ ] Private route table exists
- [ ] Private subnet is associated
- [ ] Default route points to NAT Gateway
- [ ] NAT Gateway is Available
- [ ] NAT Gateway is in public subnet
- [ ] Public subnet has Internet Gateway route

---

# PART 2 – SECURITY GROUP

# 19. Create Security Group

Open:

    EC2
        |
        v
    Security Groups

Click:

    Create security group

Enter:

    Security group name:
    IFIS-POC-EC2-SG

Select the correct VPC.

---

# 20. Add Inbound HTTP Rule

Add:

    Type:
    HTTP

    Protocol:
    TCP

    Port:
    80

For the current POC, allow traffic from:

    VPC CIDR:
    172.31.0.0/16

---

# 21. Add CloudFront Origin-Facing Rule

Add another HTTP rule:

    Type:
    HTTP

    Protocol:
    TCP

    Port:
    80

For the source, select the AWS-managed prefix list:

    com.amazonaws.global.cloudfront.origin-facing

Current POC:

    pl-9aa247f3

For another environment, verify the correct managed prefix list.

---

# 22. Configure Outbound Rules

The POC allows outbound traffic.

Expected:

    All traffic
    Destination:
    0.0.0.0/0

Create the Security Group.

---

# PART 3 – IAM AND EC2

# 23. Create IAM Role

Open:

    AWS Console
        |
        v
    IAM
        |
        v
    Roles

Click:

    Create role

Select trusted entity:

    AWS service

Select:

    EC2

Attach:

    AmazonSSMManagedInstanceCore

and:

    CloudWatchAgentServerPolicy

Use an appropriate role name, for example:

    IFIS-POC-EC2-Role

Create the role.

---

# 24. Launch EC2

Open:

    AWS Console
        |
        v
    EC2
        |
        v
    Instances

Click:

    Launch instance

Set:

    Name:
    IFIS-POC-EC2

---

# 25. Select Amazon Linux 2023

Select:

    Amazon Linux 2023

Use the current supported Amazon Linux 2023 AMI available in the selected AWS Region.

Do not copy an AMI ID from another Region.

---

# 26. Select Instance Type

Current POC:

    t3a.small

The production instance type should follow the approved project sizing.

---

# 27. Key Pair

The preferred administration method is Systems Manager.

SSH is not required for normal administration.

If the environment requires a key pair:

- Follow the organization's key management process.
- Store the private key securely.
- Do not commit it to Git.
- Do not upload it to GitHub.

---

# 28. Configure EC2 Network

Under Network settings:

Select:

    VPC:
    <VPC_ID>

Select:

    Subnet:
    <PRIVATE_SUBNET_ID>

Set:

    Auto-assign Public IP:
    Disable

Select:

    Security Group:
    IFIS-POC-EC2-SG

---

# 29. Configure Storage

Configure the root volume:

    Size:
    20 GB

    Volume type:
    gp3

    Encryption:
    Enabled

---

# 30. Configure IAM Role

Under advanced or IAM settings, select:

    IFIS-POC-EC2-Role

The EC2 instance must receive the IAM role containing:

    AmazonSSMManagedInstanceCore

and:

    CloudWatchAgentServerPolicy

---

# 31. Launch EC2

Before clicking Launch, verify:

- Correct VPC
- Correct private subnet
- Public IP disabled
- Correct Security Group
- Correct IAM role
- Encrypted root volume

Click:

    Launch instance

---

# 32. Validate EC2

Open the instance.

Verify:

    State:
    Running

Verify:

    Subnet:
    Private subnet

Verify:

    Public IPv4:
    None

Verify:

    Private IPv4:
    Present

Record the private IP.

Do not treat the private IP as permanent.

A replacement EC2 instance can receive a different private IP.

---

# PART 4 – SYSTEMS MANAGER

# 33. Verify SSM Managed Node

Open:

    AWS Console
        |
        v
    Systems Manager
        |
        v
    Managed nodes

Wait for the EC2 instance to appear.

---

# 34. Start Session Manager

Open:

    Systems Manager
        |
        v
    Session Manager
        |
        v
    Start session

Select:

    IFIS-POC-EC2

Start the session.

This confirms administrative access without SSH.

---

# 35. Validate Application

Inside the Session Manager session, run:

    sudo ss -lntp

Verify that TCP port 80 is listening.

Then:

    curl http://localhost

The application should return its expected response.

---

# PART 5 – CLOUDWATCH

# 36. Verify CloudWatch

Open:

    AWS Console
        |
        v
    CloudWatch

Check:

    Metrics

and:

    Alarms

Verify that the expected EC2 monitoring and alarms exist.

---

# PART 6 – CLOUDFRONT VPC ORIGIN

# 37. CloudFront Account Verification

Before creating a VPC Origin, verify that the AWS account is allowed to create CloudFront resources.

If you see:

    Your account must be verified before you can add new CloudFront resources.

this is an account-level CloudFront restriction.

Contact AWS Support and request CloudFront resource creation to be enabled.

Do not assume that the VPC Origin configuration is wrong.

---

# 38. Create VPC Origin

Open:

    AWS Console
        |
        v
    CloudFront

Find the VPC Origins section.

Choose:

    Create VPC origin

Select the EC2 resource.

Configure:

    HTTP port:
    80

    HTTPS port:
    443

    Origin protocol:
    HTTP only

Create the VPC Origin.

---

# 39. Wait for VPC Origin

Wait until the VPC Origin status becomes:

    Deployed

VPC Origin creation can take several minutes.

---

# 40. Validate VPC Origin

Verify:

- Correct EC2 resource
- HTTP port = 80
- HTTPS port = 443
- Origin protocol = HTTP only
- Status = Deployed

Record the VPC Origin ID.

---

# PART 7 – WAF

# 41. Open WAF

Open:

    AWS Console
        |
        v
    WAF & Shield

Switch the AWS Region to:

    US East (N. Virginia)
    us-east-1

CloudFront-scoped WAF is managed from this Region.

---

# 42. Create Web ACL

Select:

    Web ACLs

Click:

    Create web ACL

Enter an appropriate name, for example:

    IFIS-POC-cloudfront-waf

Set scope:

    CloudFront

Set default action:

    Allow

---

# 43. Add AWS Managed Rules

Add:

    AWSManagedRulesCommonRuleSet

Add:

    AWSManagedRulesKnownBadInputsRuleSet

Add:

    AWSManagedRulesLinuxRuleSet

Add:

    AWSManagedRulesSQLiRuleSet

Add:

    AWSManagedRulesAmazonIpReputationList

Review the configuration.

Create the Web ACL.

---

# 44. Record WAF ARN

Open the Web ACL.

Record:

    Web ACL ARN

This ARN will be required when configuring CloudFront.

---

# PART 8 – CLOUDFRONT DISTRIBUTION

# 45. Create CloudFront Distribution

Open:

    AWS Console
        |
        v
    CloudFront
        |
        v
    Distributions

Click:

    Create distribution

Select the VPC Origin created earlier.

---

# 46. Configure Viewer Protocol

Configure:

    HTTP -> HTTPS Redirect

The target behavior is:

    HTTP request
        |
        v
    CloudFront
        |
        v
    HTTPS

---

# 47. Configure Origin

Use the VPC Origin.

Target:

    HTTP
    Port 80

Verify that the origin points to the intended private EC2 resource.

---

# 48. Configure WAF

During CloudFront distribution configuration, select the WAF Web ACL created earlier.

Select:

    IFIS-POC-cloudfront-waf

Verify that the scope is:

    CloudFront

---

# 49. Configure Distribution

Configure the remaining CloudFront settings according to the approved project architecture.

The POC target includes:

- HTTPS
- HTTP to HTTPS redirect
- HTTP/2
- HTTP/3
- IPv6 disabled
- Caching disabled for the application POC
- Required HTTP methods

Create the distribution.

---

# 50. Wait for CloudFront Deployment

CloudFront deployment can take several minutes.

Wait until the distribution status shows:

    Enabled

and deployment is complete.

---

# 51. Record CloudFront Domain

Open the CloudFront distribution.

Copy the distribution domain.

It will look similar to:

    xxxxxxxxxxxx.cloudfront.net

Record it in the deployment worksheet.

---

# 52. Test CloudFront

Open:

    https://<CLOUDFRONT_DOMAIN>

Verify that the application responds.

Expected:

    Application response

---

# 53. CloudFront Troubleshooting

If CloudFront cannot reach the EC2 instance, check in this order:

1. EC2 is Running.
2. Application is listening on TCP 80.
3. Security Group allows HTTP from the CloudFront managed prefix list.
4. VPC Origin status is Deployed.
5. VPC Origin points to the correct EC2.
6. CloudFront distribution is Enabled.
7. WAF is not blocking valid traffic.

---

# PART 9 – AWS BACKUP

# 54. Open AWS Backup

Switch back to:

    ap-south-1

Open:

    AWS Backup

---

# 55. Create Backup Vault

Open:

    Backup vaults

Click:

    Create backup vault

Name:

    IFIS-POC-backup-vault

Create the vault.

---

# 56. Create Backup Plan

Open:

    Backup plans

Click:

    Create backup plan

Create a weekly backup schedule.

The target POC schedule is:

    Monday 00:00 JST

Verify the required UTC time before configuring the schedule.

---

# 57. Configure Backup Retention

Set:

    Retention:
    30 days

Verify the lifecycle settings.

---

# 58. Create Backup Selection

Select the EC2 instance:

    IFIS-POC-EC2

Create the backup selection.

Use the appropriate AWS Backup service role.

---

# 59. Validate Backup Plan

Verify:

- Backup plan exists
- Backup rule exists
- EC2 is selected
- Schedule is correct
- Retention is correct

---

# 60. Perform On-Demand Backup

For the POC, do not wait for the scheduled backup.

Start an on-demand backup.

Select the EC2 instance.

Choose the backup vault:

    IFIS-POC-backup-vault

Start the backup job.

---

# 61. Verify Backup Job

Open:

    AWS Backup
        |
        v
    Jobs

Verify:

    Status = Completed

Record:

- Backup job ID
- Resource
- Start time
- Completion time
- Recovery point ID

---

# 62. Verify Recovery Point

Open:

    AWS Backup
        |
        v
    Backup vaults
        |
        v
    IFIS-POC-backup-vault

Verify that the recovery point exists.

---

# PART 10 – RESTORE TEST

# 63. Restore Backup

Select the recovery point.

Choose:

    Restore

Restore the resource into a test environment.

Do not overwrite production resources.

---

# 64. Validate Restored EC2

After restoration, verify:

- EC2 exists
- EC2 is Running
- Network configuration
- Security Group
- IAM role
- Application
- Management access

---

# 65. Validate Application After Restore

Connect through Systems Manager if available.

Run:

    curl http://localhost

Verify that the application responds.

Record the restore result.

---

# PART 11 – END-TO-END VALIDATION

# 66. Application Test

Test:

    Browser
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
        |
        v
    Application

Open:

    https://<CLOUDFRONT_DOMAIN>

Expected:

    Application response

---

# 67. Verify EC2 Remains Private

Return to:

    EC2
        |
        v
    Instances

Verify:

    Public IPv4 address:
    None

This confirms that the application is being served through CloudFront without giving the EC2 instance a public IP.

---

# 68. Test EC2 Outbound Connectivity

Start a Session Manager session.

Run:

    curl -I https://aws.amazon.com

Expected:

    Successful HTTP response

Expected path:

    EC2
        |
        v
    Private Route Table
        |
        v
    NAT Gateway
        |
        v
    Internet

---

# 69. Validate SSM

Verify:

    Systems Manager
        |
        v
    Managed nodes

The EC2 instance should be available.

Start another Session Manager session if required.

---

# 70. Validate CloudWatch

Verify:

    CloudWatch
        |
        v
    Metrics

and:

    CloudWatch
        |
        v
    Alarms

Verify that expected metrics and alarms are available.

---

# PART 12 – FAILURE TESTING

# 71. EC2 Failure Test

For a POC, stop the EC2 instance.

Expected:

    CloudFront
        |
        v
    VPC Origin
        |
        X
    EC2

The application should become unavailable.

Start the EC2 instance again.

---

# 72. NAT Failure Concept

NAT Gateway failure affects outbound traffic from the private subnet.

It does not represent the CloudFront-to-EC2 traffic path.

Therefore:

    CloudFront -> VPC Origin -> EC2

is independent of:

    EC2 -> NAT -> Internet

---

# 73. WAF Test

Do not introduce aggressive blocking rules in a shared environment.

Monitor WAF activity through:

    WAF
        |
        v
    CloudWatch metrics

The objective is to confirm that legitimate CloudFront traffic remains functional.

---

# PART 13 – TROUBLESHOOTING

# 74. Private EC2 Has No Internet Access

Check:

1. Private subnet route table.
2. Default route.
3. NAT Gateway state.
4. Public subnet route table.
5. Internet Gateway.
6. Security Group outbound rules.
7. Network ACL.
8. DNS configuration.

Expected path:

    EC2
        |
        v
    Private Route Table
        |
        v
    NAT Gateway
        |
        v
    Public Subnet
        |
        v
    Internet Gateway
        |
        v
    Internet

---

# 75. EC2 Does Not Appear in SSM

Check:

1. IAM role.
2. AmazonSSMManagedInstanceCore.
3. SSM Agent.
4. DNS.
5. Security Group outbound rules.
6. NAT Gateway.
7. Internet connectivity.

---

# 76. CloudFront Cannot Reach EC2

Check:

1. EC2 state.
2. Application port 80.
3. Security Group.
4. CloudFront managed prefix list.
5. VPC Origin.
6. VPC Origin status.
7. CloudFront distribution.
8. WAF.

---

# 77. CloudFront Account Verification Error

If the console or API reports:

    Your account must be verified before you can add new CloudFront resources.

This indicates an account-level CloudFront restriction.

Contact AWS Support.

Provide:

- AWS Account ID
- Exact error
- Resource type
- CloudFront VPC Origin requirement
- POC/business purpose

Do not modify the architecture simply because the account has not yet been verified.

---

# 78. WAF Problems

If legitimate traffic is blocked:

Check:

1. Web ACL association.
2. Managed rule groups.
3. WAF sampled requests.
4. CloudWatch metrics.
5. Rule action.
6. CloudFront distribution.

Do not disable all security controls without understanding the rule causing the block.

---

# 79. Backup Failure

Check:

1. Backup plan.
2. Backup selection.
3. EC2 resource.
4. Backup IAM role.
5. Backup job status.
6. Backup vault.

A backup plan existing does not prove that backups are working.

A completed backup job and successful restore test are stronger validation.

---

# PART 14 – MANUAL RESOURCE INVENTORY

# 80. Record Resource IDs

After manual deployment, record:

    AWS Account ID:

    Region:

    VPC ID:

    Public Subnet ID:

    Private Subnet ID:

    Private Route Table ID:

    NAT Gateway ID:

    Elastic IP:

    Security Group ID:

    IAM Role:

    EC2 Instance ID:

    EC2 Private IP:

    VPC Origin ID:

    CloudFront Distribution ID:

    CloudFront Domain:

    WAF Web ACL ARN:

    Backup Vault:

    Backup Plan:

    Recovery Point ID:

Do not record passwords, access keys, secret keys, or private keys.

---

# PART 15 – MANUAL CLEANUP

# 81. Cleanup Order

Recommended manual cleanup order:

    CloudFront
        |
        v
    WAF
        |
        v
    Backup
        |
        v
    EC2
        |
        v
    Security Group
        |
        v
    NAT Gateway
        |
        v
    Elastic IP
        |
        v
    Private Route Table
        |
        v
    Private Subnet

Do not delete the existing VPC if it is shared with other resources.

---

# 82. Delete CloudFront Distribution

Open:

    CloudFront
        |
        v
    Distributions

Disable the distribution.

Wait until the distribution is fully disabled.

Then delete it.

---

# 83. Delete VPC Origin

After the CloudFront distribution no longer uses the VPC Origin, remove the VPC Origin if required.

Verify that no other CloudFront distribution depends on it.

---

# 84. Delete WAF

Open:

    WAF & Shield

Select:

    us-east-1

Remove the association with CloudFront if required.

Delete the Web ACL.

---

# 85. Delete Backup

Open:

    AWS Backup

Review:

- Backup plans
- Backup jobs
- Recovery points
- Backup vault

Important:

Recovery points may remain according to the retention configuration.

Do not delete backup data until the required retention and testing requirements are satisfied.

---

# 86. Terminate EC2

Open:

    EC2
        |
        v
    Instances

Select the POC instance.

Terminate it.

Verify:

    Instance state = Terminated

---

# 87. Delete Security Group

After the EC2 instance has been removed and no other resources use the Security Group:

Open:

    EC2
        |
        v
    Security Groups

Delete:

    IFIS-POC-EC2-SG

---

# 88. Delete NAT Gateway

Open:

    VPC
        |
        v
    NAT Gateways

Delete:

    IFIS-POC NAT Gateway

Wait until the NAT Gateway is deleted.

---

# 89. Release Elastic IP

Open:

    VPC
        |
        v
    Elastic IPs

Release the Elastic IP associated with the POC NAT Gateway.

This is important because unused Elastic IPs may incur charges depending on AWS pricing and account conditions.

---

# 90. Delete Private Route Table

Open:

    VPC
        |
        v
    Route Tables

Delete:

    IFIS-POC-private-rt

Verify that it is not associated with any subnet before deletion.

---

# 91. Delete Private Subnet

Open:

    VPC
        |
        v
    Subnets

Delete:

    IFIS-POC-private-subnet

Verify that the subnet is no longer required.

---

# PART 16 – MANUAL VALIDATION CHECKLIST

# 92. Network

- [ ] Correct VPC
- [ ] Correct public subnet
- [ ] Internet Gateway attached
- [ ] Public route table verified
- [ ] Private subnet created
- [ ] Public IP assignment disabled
- [ ] Private route table created
- [ ] Private subnet associated
- [ ] NAT Gateway created
- [ ] NAT Gateway Available
- [ ] NAT route configured

---

# 93. Security

- [ ] Security Group created
- [ ] HTTP port 80 configured
- [ ] VPC source configured if required
- [ ] CloudFront managed prefix list configured
- [ ] Outbound traffic configured
- [ ] EC2 has no public IP

---

# 94. Compute

- [ ] Amazon Linux 2023
- [ ] Correct instance type
- [ ] Private subnet
- [ ] Public IP disabled
- [ ] IAM role attached
- [ ] Encrypted root volume
- [ ] EC2 Running
- [ ] SSM working
- [ ] Session Manager working
- [ ] Application listening on port 80
- [ ] curl localhost works

---

# 95. CloudFront

- [ ] CloudFront account verified
- [ ] VPC Origin created
- [ ] VPC Origin Deployed
- [ ] EC2 selected as origin
- [ ] HTTP port 80
- [ ] CloudFront distribution created
- [ ] HTTP redirects to HTTPS
- [ ] Distribution Enabled
- [ ] CloudFront domain recorded
- [ ] Application accessible

---

# 96. WAF

- [ ] WAF created in us-east-1
- [ ] Scope = CloudFront
- [ ] Default action = Allow
- [ ] Common Rule Set configured
- [ ] Known Bad Inputs configured
- [ ] Linux Rule Set configured
- [ ] SQLi Rule Set configured
- [ ] Amazon IP Reputation List configured
- [ ] WAF associated with CloudFront

---

# 97. Backup

- [ ] Backup vault created
- [ ] Backup plan created
- [ ] EC2 selected
- [ ] Weekly schedule configured
- [ ] Retention configured
- [ ] On-demand backup completed
- [ ] Recovery point exists
- [ ] Restore test completed

---

# 98. Final End-to-End Validation

Verify:

    Browser
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
        |
        v
    Application

Separately verify:

    Private EC2
        |
        v
    NAT Gateway
        |
        v
    Internet

And:

    Private EC2
        |
        +---- SSM
        |
        +---- CloudWatch
        |
        +---- AWS Backup

All paths should be validated independently.

---

# 99. Difference Between Manual and CloudFormation Deployment

| Area | Manual | CloudFormation |
|---|---|---|
| Creation | Console clicks | YAML |
| Repeatability | Low | High |
| Version control | Limited | Yes |
| Parameters | Manual | CloudFormation parameters |
| Dependencies | Human-managed | Template-managed |
| Replication | More effort | Easier |
| Troubleshooting | Good for learning | Good for infrastructure consistency |
| Recommended for production | Usually not | Yes |
| Recommended for learning | Yes | Yes |

---

# 100. When to Use Manual Deployment

Use the manual process when:

- Learning AWS
- Understanding how a service works
- Troubleshooting CloudFormation
- Testing a resource configuration
- Demonstrating the architecture
- Investigating account restrictions

---

# 101. When to Use CloudFormation

Use CloudFormation when:

- Deploying repeatable environments
- Deploying client infrastructure
- Maintaining infrastructure as code
- Replicating across AWS accounts
- Replicating across Regions
- Reviewing infrastructure changes
- Maintaining configuration consistency

---

# 102. Important Reusability Rule

Manual deployment should not become the permanent source of truth for the architecture.

Once the correct manual configuration is understood and validated, the equivalent configuration should be represented in CloudFormation wherever practical.

The CloudFormation templates should remain the primary reusable infrastructure definition.

---

# 103. Related Documentation

Architecture:

    docs/architecture.md

CloudFormation deployment:

    docs/cloudformation-deployment-guide.md

Validation:

    docs/validation-checklist.md

Troubleshooting:

    docs/troubleshooting.md

CloudFormation templates:

    cloudformation/

---

# 104. Document Ownership

Project:

    IFIS

Environment:

    POC

Primary Region:

    ap-south-1

CloudFront WAF Region:

    us-east-1

Owner:

    Sohan

Repository:

    sohan_IFIS_project

This document should be updated whenever the manual AWS Console procedure changes.