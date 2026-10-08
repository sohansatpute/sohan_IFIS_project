# IFIS POC – AWS Deployment Guide

## 1. Purpose

This document provides a complete step-by-step procedure for deploying the IFIS AWS architecture.

It is written so that a new team member or intern with basic AWS knowledge can follow the document and recreate the POC environment.

This guide covers two deployment methods:

1. CloudFormation deployment – recommended and reusable method.
2. AWS Console manual deployment – useful for learning, troubleshooting, and understanding what CloudFormation creates.

The CloudFormation templates are the preferred method for repeatable deployments.

---

## 2. Architecture Being Deployed

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

### Important

- CloudFront provides the public entry point.
- AWS WAF protects the CloudFront distribution.
- CloudFront VPC Origin provides the connection from CloudFront to the private EC2.
- EC2 does not require a public IP.
- NAT Gateway provides outbound connectivity from the private subnet.
- Systems Manager provides administrative access without SSH.
- CloudWatch provides monitoring and alarms.
- AWS Backup provides EC2 backup and recovery.

---

## 3. Deployment Order

Always follow this order unless a specific procedure says otherwise.

    1. Prepare AWS account
           |
           v
    2. Prepare workstation
           |
           v
    3. Verify existing VPC
           |
           v
    4. Deploy Network
           |
           v
    5. Validate Network
           |
           v
    6. Deploy Compute
           |
           v
    7. Validate Compute
           |
           v
    8. Deploy CloudFront
           |
           v
    9. Deploy WAF
           |
           v
    10. Deploy Backup
           |
           v
    11. Perform end-to-end validation
           |
           v
    12. Perform failure testing
           |
           v
    13. Clean up POC if required

---

## 4. AWS Regions

The main infrastructure is deployed in:

    ap-south-1

This is the AWS Mumbai Region.

The CloudFront-scoped WAF is deployed in:

    us-east-1

This is required because AWS WAF Web ACLs with CloudFront scope are managed from US East (N. Virginia).

Do not accidentally deploy the CloudFront WAF stack in `ap-south-1`.

---

## 5. Current POC Values

The current POC uses the following values:

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
| Application Port | TCP 80 |
| Administration | AWS Systems Manager |
| Infrastructure as Code | AWS CloudFormation |

### Important

These values are specific to the current POC.

For a new environment, do not blindly copy AWS resource IDs.

Verify the VPC, subnet, Availability Zone, and CloudFront managed prefix list for that environment.

---

## 6. Prerequisites

Before starting, confirm that the following are available.

### AWS Requirements

You need access to an AWS account with permissions to create and manage:

- VPC resources
- Subnets
- Route tables
- NAT Gateway
- Elastic IP
- EC2
- Security Groups
- IAM roles
- CloudWatch alarms
- CloudFront
- CloudFront VPC Origins
- AWS WAF
- AWS Backup
- CloudFormation

### Workstation Requirements

Install:

- Git
- Git Bash
- AWS CLI

Verify Git:

    git --version

Verify AWS CLI:

    aws --version

Verify AWS access:

    aws sts get-caller-identity

---

## 7. AWS Console Login

Open the AWS Management Console.

Sign in using the credentials provided by your organization.

Never put AWS passwords, access keys, secret keys, `.pem` files, or other secrets into the Git repository.

---

## 8. Select AWS Region

After logging in:

1. Look at the top-right corner of the AWS Console.
2. Click the Region selector.
3. Select:

    Asia Pacific (Mumbai)
    ap-south-1

Most of the POC infrastructure will be created in this region.

---

## 9. Verify AWS Account

Before creating resources, verify that you are working in the correct AWS account.

Run:

    aws sts get-caller-identity

Example response:

    {
        "UserId": "AIDA...",
        "Account": "123456789012",
        "Arn": "arn:aws:iam::123456789012:user/example"
    }

Record the Account ID.

Do not continue if the AWS account is not the expected account.

---

## 10. Create Deployment Worksheet

Create a temporary worksheet containing:

| Item | Value |
|---|---|
| AWS Account ID | |
| AWS Region | |
| VPC ID | |
| VPC CIDR | |
| Existing Public Subnet ID | |
| Existing Public Subnet CIDR | |
| Availability Zone | |
| Private Subnet ID | |
| Private Subnet CIDR | |
| Private Route Table ID | |
| NAT Gateway ID | |
| EC2 Instance ID | |
| EC2 Private IP | |
| Security Group ID | |
| VPC Origin ID | |
| CloudFront Distribution ID | |
| CloudFront Domain | |
| WAF Web ACL ARN | |
| Backup Vault | |
| Backup Plan | |
| Recovery Point ID | |

Do not store passwords, secret keys, access keys, or private keys in this worksheet.

---

## 11. Clone the Project Repository

Open Git Bash.

Clone the repository:

    git clone https://github.com/sohansatpute/sohan_IFIS_project.git

Move into the project directory:

    cd sohan_IFIS_project

Check the project files:

    ls

Expected directories include:

    cloudformation
    docs
    config
    scripts
    diagrams

---

## 12. Check CloudFormation Templates

Run:

    ls cloudformation

Expected:

    01-network.yaml
    02-compute.yaml
    03-cloudfront.yaml
    04-waf.yaml
    05-backup.yaml

The templates are deployed in dependency order.

---

## 13. Verify AWS CLI Region

Check:

    aws configure get region

If required:

    aws configure set region ap-south-1

Verify:

    aws configure get region

Expected:

    ap-south-1

---

# PART 1 – NETWORK DEPLOYMENT

## 14. Verify Existing VPC

The POC uses an existing VPC.

### AWS Console

Open:

    AWS Console
        |
        v
    VPC
        |
        v
    Your VPCs

Find the VPC that will be used.

Verify:

- VPC ID
- IPv4 CIDR
- State = Available

Current POC:

    VPC ID:
    vpc-06900f62513eff63

    CIDR:
    172.31.0.0/16

For another AWS account, the VPC ID will normally be different.

---

## 15. Verify Internet Gateway

Open:

    AWS Console
        |
        v
    VPC
        |
        v
    Internet gateways

Find the Internet Gateway attached to the selected VPC.

Verify:

    State = Attached

Verify that it is attached to the correct VPC.

The public subnet used for the NAT Gateway must have Internet Gateway connectivity.

---

## 16. Verify Public Subnet

Open:

    AWS Console
        |
        v
    VPC
        |
        v
    Subnets

Find the public subnet that will host the NAT Gateway.

For the current POC:

    Subnet ID:
    subnet-0d6445bd9f644383b

    CIDR:
    172.31.0.0/20

    Availability Zone:
    ap-south-1b

Verify:

- Correct VPC
- Correct CIDR
- Correct Availability Zone
- Public route table associated

---

## 17. Verify Public Route Table

Open:

    AWS Console
        |
        v
    VPC
        |
        v
    Route Tables

Find the route table associated with the public subnet.

Open:

    Routes

Expected route:

    Destination:
    0.0.0.0/0

    Target:
    Internet Gateway

This allows resources in the public subnet to reach the Internet.

---

## 18. Verify Availability Zone

Open:

    VPC
        |
        v
    Subnets

Verify the Availability Zone.

Current POC:

    ap-south-1b

The private subnet and NAT public subnet are currently using this Availability Zone.

For production, a multi-AZ architecture should be considered separately.

---

## 19. CloudFormation Deployment Overview

The project contains five CloudFormation templates:

    01-network.yaml
            |
            v
    02-compute.yaml
            |
            +-------------------+
            |                   |
            v                   v
    03-cloudfront.yaml     05-backup.yaml
            |
            v
    04-waf.yaml

Logical dependency:

    Network
       |
       v
    Compute
       |
       +----> CloudFront
       |
       +----> Backup

    WAF
       |
       v
    CloudFront

---

## 20. Network Stack

The Network stack creates:

- Private subnet
- Private route table
- Private route table association
- Optional Elastic IP
- NAT Gateway
- Default route through NAT Gateway

It uses an existing VPC and existing public subnet.

The Network template does not create a new VPC.

---

## 21. Validate Network Template

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/01-network.yaml \
      --region ap-south-1

The command should return template information without a syntax error.

---

## 22. Deploy Network Stack

Example:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-network \
      --template-body file://cloudformation/01-network.yaml \
      --parameters \
        ParameterKey=ExistingVpcId,ParameterValue=<VPC_ID> \
        ParameterKey=ExistingPublicSubnetId,ParameterValue=<PUBLIC_SUBNET_ID> \
        ParameterKey=AvailabilityZone,ParameterValue=ap-south-1b \
        ParameterKey=PrivateSubnetCidr,ParameterValue=172.31.48.0/20 \
        ParameterKey=EnableNatGateway,ParameterValue=true \
        ParameterKey=ProjectName,ParameterValue=IFIS \
        ParameterKey=Environment,ParameterValue=POC \
        ParameterKey=Owner,ParameterValue=Sohan \
      --region ap-south-1

Replace:

    <VPC_ID>
    <PUBLIC_SUBNET_ID>

with the actual environment values.

---

## 23. Wait for Network Stack

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-network \
      --region ap-south-1

Then check:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-network \
      --region ap-south-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

## 24. Get Network Outputs

Run:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-network \
      --region ap-south-1 \
      --query "Stacks[0].Outputs"

Record:

- VpcId
- PrivateSubnetId
- PrivateRouteTableId
- NatGatewayId

---

## 25. Validate Private Subnet

Open:

    AWS Console
        |
        v
    VPC
        |
        v
    Subnets

Open the private subnet created by CloudFormation.

Verify:

    VPC = correct VPC
    CIDR = 172.31.48.0/20
    Availability Zone = ap-south-1b

Also verify:

    Auto-assign public IPv4 address = Disabled

The private subnet must not automatically assign public IPv4 addresses.

---

## 26. Validate Private Route Table

Open:

    AWS Console
        |
        v
    VPC
        |
        v
    Route Tables

Find the private route table.

Open:

    Routes

Expected:

    Destination       Target
    --------------------------------
    172.31.0.0/16     local
    0.0.0.0/0         NAT Gateway

The NAT Gateway should be the NAT Gateway created by the Network stack.

---

## 27. Validate NAT Gateway

Open:

    VPC
        |
        v
    NAT Gateways

Verify:

    State = Available

Verify:

    Subnet = Public Subnet

Verify that an Elastic IP is associated.

---

## 28. Network Validation Checklist

Before moving to the Compute stack, verify:

- [ ] Correct VPC
- [ ] Correct public subnet
- [ ] Public subnet has Internet Gateway route
- [ ] Private subnet exists
- [ ] Private subnet public IP assignment disabled
- [ ] Private route table exists
- [ ] Private subnet is associated with private route table
- [ ] Default route points to NAT Gateway
- [ ] NAT Gateway is Available
- [ ] NAT Gateway is in public subnet

Do not continue if private routing is incorrect.

---

# PART 2 – COMPUTE DEPLOYMENT

## 29. Compute Stack

The Compute stack creates:

- IAM role
- IAM instance profile
- Security Group
- EC2 instance
- CloudWatch alarms

The EC2 instance is launched into the private subnet created by the Network stack.

---

## 30. Validate Compute Template

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/02-compute.yaml \
      --region ap-south-1

The template should pass validation.

---

## 31. Deploy Compute Stack

Run:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-compute \
      --template-body file://cloudformation/02-compute.yaml \
      --parameters \
        ParameterKey=NetworkStackName,ParameterValue=IFIS-POC-network \
        ParameterKey=InstanceType,ParameterValue=t3a.small \
        ParameterKey=InstanceName,ParameterValue=IFIS-POC-EC2 \
        ParameterKey=RootVolumeSize,ParameterValue=20 \
        ParameterKey=CloudFrontOriginFacingPrefixListId,ParameterValue=pl-9aa247f3 \
        ParameterKey=ProjectName,ParameterValue=IFIS \
        ParameterKey=Environment,ParameterValue=POC \
        ParameterKey=Owner,ParameterValue=Sohan \
      --region ap-south-1

Important:

The template already has a default Amazon Linux 2023 AMI parameter.

Normally, do not manually provide the AMI parameter.

---

## 32. Important Git Bash AMI Note

When using Git Bash on Windows, a parameter beginning with:

    /aws/service/...

can sometimes be interpreted as a Windows filesystem path.

It can incorrectly become something similar to:

    C:/Users/...

This can cause CloudFormation errors.

For this project, the recommended approach is to allow the Compute template to use its default AMI parameter unless a different AMI is specifically required.

---

## 33. Wait for Compute Stack

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-compute \
      --region ap-south-1

Then:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-compute \
      --region ap-south-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

## 34. Get Compute Outputs

Run:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-compute \
      --region ap-south-1 \
      --query "Stacks[0].Outputs"

Record:

- EC2 Instance ID
- Private IP
- Security Group ID
- Instance Profile
- Role ARN
- Private DNS
- EC2 ARN

---

## 35. Validate EC2

Open:

    AWS Console
        |
        v
    EC2
        |
        v
    Instances

Find the EC2 instance created by the Compute stack.

Verify:

    Instance state = Running

---

## 36. Verify EC2 Operating System

Open the EC2 instance.

Verify that it is using Amazon Linux 2023.

Do not permanently document a specific AMI ID as a reusable value.

AMI IDs can change over time and differ between AWS Regions.

---

## 37. Verify EC2 Subnet

Open the EC2 instance details.

Verify:

    Subnet ID = Private Subnet

The instance must be inside the private subnet.

---

## 38. Verify EC2 Public IP

In the EC2 instance details, find:

    Public IPv4 address

Expected:

    None

This is an important security validation.

The EC2 instance must remain private.

---

## 39. Verify EC2 Private IP

Locate:

    Private IPv4 address

Record the value.

Important:

Do not treat the private IP as a permanent identifier.

A new EC2 deployment may receive a different private IP.

---

## 40. Verify Security Group

Open the Security Group attached to the EC2 instance.

Verify inbound rules.

Expected CloudFront rule:

    Type:
    HTTP

    Protocol:
    TCP

    Port:
    80

    Source:
    CloudFront origin-facing managed prefix list

Current POC:

    pl-9aa247f3

The Security Group may also contain:

    TCP 80
    Source:
    172.31.0.0/16

according to the current POC template.

---

## 41. Verify Security Group Outbound Rules

Open:

    Security Group
        |
        v
    Outbound rules

The POC allows outbound traffic.

Expected:

    All traffic
    Destination:
    0.0.0.0/0

---

## 42. Verify IAM Role

Open:

    AWS Console
        |
        v
    IAM
        |
        v
    Roles

Find the role created by the Compute stack.

Verify that it contains:

    AmazonSSMManagedInstanceCore

and:

    CloudWatchAgentServerPolicy

---

## 43. Verify Systems Manager

Open:

    AWS Console
        |
        v
    Systems Manager
        |
        v
    Managed nodes

The EC2 instance should appear as a managed node.

It may take a few minutes after instance launch.

---

## 44. Verify Session Manager

Open:

    Systems Manager
        |
        v
    Session Manager
        |
        v
    Start session

Select the EC2 instance.

Start a session.

This confirms that the EC2 instance can be administered through Systems Manager without requiring SSH.

---

## 45. Verify CloudWatch

Open:

    AWS Console
        |
        v
    CloudWatch

Check:

    Metrics

and:

    Alarms

The Compute stack creates CloudWatch alarms.

Verify that the expected alarms exist.

---

## 46. Verify Application on EC2

Connect to the instance using Session Manager.

Run:

    sudo ss -lntp

Verify that the application/web server is listening on:

    TCP 80

Then:

    curl http://localhost

The expected application response should be returned.

If `curl http://localhost` fails, fix the application before troubleshooting CloudFront.

---

## 47. Compute Validation Checklist

Before moving to CloudFront:

- [ ] EC2 is Running
- [ ] Amazon Linux 2023 is installed
- [ ] EC2 is in private subnet
- [ ] EC2 has no public IPv4 address
- [ ] Security Group is correct
- [ ] IAM role is attached
- [ ] SSM managed node is available
- [ ] Session Manager works
- [ ] CloudWatch Agent is running
- [ ] CloudWatch alarms exist
- [ ] Application listens on TCP 80
- [ ] curl http://localhost works

---

# PART 3 – CLOUDFRONT DEPLOYMENT

## 48. CloudFront Prerequisite

Before deploying CloudFront, verify whether the AWS account is allowed to create CloudFront resources.

New or unverified AWS accounts can sometimes have CloudFront resource creation restrictions.

If CloudFormation returns:

    Your account must be verified before you can add new CloudFront resources.

this may be an AWS account-level restriction.

Do not immediately change the CloudFormation template.

Contact AWS Support and request CloudFront resource creation to be enabled.

---

## 49. Validate CloudFront Template

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/03-cloudfront.yaml \
      --region ap-south-1

The template should pass validation.

---

## 50. Deploy CloudFront Stack

If CloudFront resource creation is available:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-cloudfront \
      --template-body file://cloudformation/03-cloudfront.yaml \
      --parameters \
        ParameterKey=ComputeStackName,ParameterValue=IFIS-POC-compute \
        ParameterKey=ProjectName,ParameterValue=IFIS \
        ParameterKey=Environment,ParameterValue=POC \
        ParameterKey=Owner,ParameterValue=Sohan \
      --region ap-south-1

---

## 51. Wait for CloudFront Stack

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-cloudfront \
      --region ap-south-1

Then:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-cloudfront \
      --region ap-south-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

## 52. If CloudFront Stack Fails

Run:

    aws cloudformation describe-stack-events \
      --stack-name IFIS-POC-cloudfront \
      --region ap-south-1

Find the first:

    CREATE_FAILED

The first failure is normally more useful than later rollback messages.

---

## 53. Validate VPC Origin

Open:

    AWS Console
        |
        v
    CloudFront
        |
        v
    VPC origins

Verify that the VPC Origin exists.

Expected configuration:

    Origin resource = EC2
    HTTP port = 80
    HTTPS port = 443
    Origin protocol = HTTP only

The VPC Origin should eventually show:

    Deployed

VPC Origin creation can take several minutes.

---

## 54. Validate CloudFront Distribution

Open:

    AWS Console
        |
        v
    CloudFront
        |
        v
    Distributions

Open the POC distribution.

Verify:

    Status = Enabled

If the distribution is still deploying, wait until deployment is complete.

---

## 55. Verify Viewer Protocol Policy

Open the CloudFront distribution behavior.

The target configuration is:

    HTTP -> HTTPS Redirect

Expected user flow:

    HTTP request
        |
        v
    CloudFront
        |
        v
    HTTPS

---

## 56. Get CloudFront Domain

From the CloudFront distribution, copy the distribution domain.

It will look similar to:

    xxxxxxxxxxxx.cloudfront.net

Record it in the deployment worksheet.

---

## 57. Test CloudFront

Open:

    https://<CLOUDFRONT_DOMAIN>

Verify that the application responds.

Expected result:

    Application response

If the application does not respond, troubleshoot in this order:

    1. EC2 application
    2. Security Group
    3. VPC Origin
    4. CloudFront

---

## 58. CloudFront Security Group Requirement

The EC2 Security Group must allow CloudFront origin-facing traffic.

The required AWS managed prefix list is:

    com.amazonaws.global.cloudfront.origin-facing

Current POC:

    pl-9aa247f3

For a new environment, verify the correct managed prefix list rather than blindly copying the current POC ID.

---

# PART 4 – WAF DEPLOYMENT

## 59. WAF Deployment

The WAF template uses:

    Scope = CLOUDFRONT

Therefore the CloudFormation stack must be deployed in:

    us-east-1

---

## 60. Validate WAF Template

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/04-waf.yaml \
      --region us-east-1

The template should pass validation.

---

## 61. Deploy WAF

Run:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-waf \
      --template-body file://cloudformation/04-waf.yaml \
      --region us-east-1

---

## 62. Wait for WAF

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-waf \
      --region us-east-1

Then:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-waf \
      --region us-east-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

## 63. Get WAF Outputs

Run:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-waf \
      --region us-east-1 \
      --query "Stacks[0].Outputs"

Record:

- WebAclArn
- WebAclId
- WebAclName

---

## 64. WAF Console Validation

Open:

    AWS Console
        |
        v
    WAF & Shield
        |
        v
    Web ACLs

Make sure the console region is:

    US East (N. Virginia)
    us-east-1

Open the Web ACL.

Verify:

    Scope = CloudFront
    Default action = Allow

---

## 65. Verify WAF Managed Rules

The POC WAF contains:

    AWSManagedRulesCommonRuleSet

    AWSManagedRulesKnownBadInputsRuleSet

    AWSManagedRulesLinuxRuleSet

    AWSManagedRulesSQLiRuleSet

    AWSManagedRulesAmazonIpReputationList

Verify all five rule groups exist.

---

## 66. Associate WAF with CloudFront

The WAF Web ACL must be associated with the CloudFront distribution.

The target architecture is:

    Internet
        |
        v
    CloudFront
        |
        +---- AWS WAF
        |
        v
    VPC Origin
        |
        v
    Private EC2

WAF is associated with CloudFront.

WAF is not installed on the EC2 instance.

---

## 67. Verify WAF Association

Open:

    CloudFront
        |
        v
    Distributions
        |
        v
    <POC Distribution>

Verify that the correct Web ACL is associated.

Then open WAF and verify the CloudFront distribution association.

---

# PART 5 – AWS BACKUP DEPLOYMENT

## 68. Backup Deployment

AWS Backup is deployed in:

    ap-south-1

The Backup stack protects the EC2 instance created by the Compute stack.

---

## 69. Validate Backup Template

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/05-backup.yaml \
      --region ap-south-1

---

## 70. Deploy Backup

Run:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-backup \
      --template-body file://cloudformation/05-backup.yaml \
      --parameters \
        ParameterKey=ComputeStackName,ParameterValue=IFIS-POC-compute \
        ParameterKey=BackupScheduleCron,ParameterValue='cron(0 15 ? * SUN *)' \
        ParameterKey=RetentionDays,ParameterValue=30 \
        ParameterKey=StartWindowMinutes,ParameterValue=60 \
        ParameterKey=CompletionWindowMinutes,ParameterValue=180 \
        ParameterKey=ProjectName,ParameterValue=IFIS \
        ParameterKey=Environment,ParameterValue=POC \
        ParameterKey=Owner,ParameterValue=Sohan \
      --region ap-south-1 \
      --capabilities CAPABILITY_NAMED_IAM

---

## 71. Wait for Backup Stack

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-backup \
      --region ap-south-1

Then:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-backup \
      --region ap-south-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

## 72. Backup Schedule

The POC schedule is:

    cron(0 15 ? * SUN *)

This is:

    Sunday 15:00 UTC

The original project documentation describes this as:

    Monday 00:00 JST

For a different project/environment, verify the required timezone and schedule before deployment.

---

## 73. Verify Backup Vault

Open:

    AWS Console
        |
        v
    AWS Backup
        |
        v
    Backup vaults

Find:

    IFIS-POC-backup-vault

Verify that it exists.

---

## 74. Verify Backup Plan

Open:

    AWS Backup
        |
        v
    Backup plans

Find:

    IFIS-POC-backup-plan

Verify that:

    WeeklyEC2Backup

exists.

---

## 75. Verify Backup Selection

The backup selection should include the EC2 instance created by the Compute stack.

Verify that the correct EC2 resource is selected.

---

## 76. Verify Backup Retention

The POC retention is:

    30 days

Verify the backup lifecycle configuration.

---

## 77. Perform Backup Test

For the POC, perform an on-demand backup instead of waiting for the scheduled backup.

Open:

    AWS Backup
        |
        v
    Protected resources

or use the Backup console to initiate an on-demand backup.

Wait for the backup job to complete successfully.

---

## 78. Verify Recovery Point

Open:

    AWS Backup
        |
        v
    Backup vaults

Open:

    IFIS-POC-backup-vault

Verify that a recovery point exists.

Record:

- Recovery Point ID
- Creation Time
- Resource
- Status

---

## 79. Restore Test

A backup should be tested by restoring it into a test environment.

Do not restore over a production workload during the POC.

Expected flow:

    Recovery Point
          |
          v
    Restore Job
          |
          v
    Restored EC2
          |
          v
    Application Validation

Record the restore result.

---

# PART 6 – END-TO-END VALIDATION

## 80. End-to-End Validation

After all services are deployed, perform an end-to-end test.

Expected traffic flow:

    Browser
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
    Private EC2
       |
       v
    Application

The application should return the expected response.

---

## 81. Verify EC2 Remains Private

After CloudFront is working, return to EC2.

Verify:

    Public IPv4 address = None

This confirms that CloudFront is reaching the private origin without requiring a public EC2 IP.

---

## 82. Test EC2 Outbound Connectivity

Connect through Session Manager.

Run:

    curl -I https://aws.amazon.com

A successful HTTP response confirms outbound connectivity.

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

If this fails, investigate:

- Route table
- NAT Gateway
- Public subnet
- Internet Gateway
- Security Group
- Network ACL
- DNS

---

## 83. Test Application Locally

From Session Manager:

    curl http://localhost

Expected:

    Application response

If this fails, fix the application first.

Do not start troubleshooting CloudFront until the application works locally.

---

# PART 7 – FAILURE TESTING

## 84. Failure Test – EC2

For a POC, stop the EC2 instance.

Expected result:

    CloudFront
        |
        v
    VPC Origin
        |
        v
    EC2 unavailable

The application should become unavailable.

Start the EC2 instance again after testing.

---

## 85. Failure Test – NAT Gateway

Do not intentionally delete the NAT Gateway unless the POC is specifically testing NAT failure.

If NAT becomes unavailable:

    EC2
     |
     X
    NAT Gateway

outbound Internet connectivity may fail.

Important:

CloudFront-to-EC2 traffic does not use the NAT Gateway.

NAT is for outbound traffic from the private subnet.

---

## 86. Failure Test – WAF

Do not immediately create blocking WAF rules in a shared environment.

WAF testing should be performed carefully.

Monitor WAF activity through:

    WAF
      |
      v
    CloudWatch metrics

---

## 87. Failure Test – Backup

Perform a restore test.

The important question is not only:

    Was the backup created?

but:

    Can the backup actually be restored and used?

---

# PART 8 – TROUBLESHOOTING

## 88. CloudFormation Troubleshooting

If a CloudFormation deployment fails, check stack events.

Example:

    aws cloudformation describe-stack-events \
      --stack-name IFIS-POC-network \
      --region ap-south-1

Change the stack name as required.

Look for:

    CREATE_FAILED

The first resource failure is normally the most useful event.

---

## 89. Common CloudFormation Statuses

### CREATE_IN_PROGRESS

The stack is being created.

### CREATE_COMPLETE

The stack was created successfully.

### CREATE_FAILED

A resource failed during creation.

### ROLLBACK_IN_PROGRESS

CloudFormation is removing resources created during the failed deployment.

### ROLLBACK_COMPLETE

The failed stack has completed rollback.

### DELETE_IN_PROGRESS

The stack is being deleted.

### DELETE_COMPLETE

The stack has been deleted successfully.

---

## 90. CloudFront Account Verification Error

If you receive:

    Your account must be verified before you can add new CloudFront resources.

check the AWS account status.

This can be an account-level CloudFront restriction.

The correct action is to contact AWS Support and request CloudFront resource creation to be enabled.

Do not assume that the CloudFormation template is incorrect.

---

## 91. EC2 SSM Troubleshooting

If the EC2 instance is running but does not appear in Systems Manager:

Check in this order:

    1. IAM role
    2. SSM Agent
    3. DNS
    4. Security Group outbound access
    5. Route table
    6. NAT Gateway
    7. Internet Gateway

The EC2 instance requires connectivity to Systems Manager endpoints.

---

## 92. CloudFront Troubleshooting

If CloudFront cannot reach EC2:

Check:

    1. EC2 is Running
    2. Application listens on port 80
    3. Security Group allows CloudFront origin-facing traffic
    4. VPC Origin is Deployed
    5. VPC Origin targets correct EC2
    6. CloudFront distribution is Enabled
    7. WAF is not blocking legitimate traffic

---

## 93. Backup Troubleshooting

If the backup fails:

Check:

    1. Backup plan
    2. Backup selection
    3. EC2 resource
    4. Backup IAM role
    5. Backup job events
    6. Backup vault

A successful CloudFormation deployment does not prove that a backup job has successfully completed.

The backup job itself must be tested.

---

# PART 9 – MANUAL AWS CONSOLE DEPLOYMENT

## 94. Manual Deployment Method

CloudFormation is the recommended deployment method.

However, manual deployment through the AWS Console is useful for:

- Learning AWS
- Understanding the architecture
- Troubleshooting
- Demonstrating the architecture
- Emergency manual deployment
- Comparing AWS Console resources with CloudFormation resources

---

## 95. Manual Method – Create Private Subnet

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

Select the correct VPC.

Enter subnet name:

    IFIS-POC-private-subnet

Select Availability Zone:

    ap-south-1b

Enter CIDR:

    172.31.48.0/20

Click:

    Create subnet

---

## 96. Manual Method – Disable Public IP Assignment

Open the private subnet.

Choose:

    Actions
        |
        v
    Edit subnet settings

Find:

    Enable auto-assign public IPv4 address

Make sure it is disabled.

Save.

---

## 97. Manual Method – Create Private Route Table

Open:

    VPC
        |
        v
    Route Tables

Click:

    Create route table

Name:

    IFIS-POC-private-rt

Select the correct VPC.

Click:

    Create route table

---

## 98. Manual Method – Associate Private Subnet

Open the new route table.

Go to:

    Subnet associations

Click:

    Edit subnet associations

Select the private subnet.

Click:

    Save associations

---

## 99. Manual Method – Create NAT Gateway

Open:

    VPC
        |
        v
    NAT Gateways

Click:

    Create NAT gateway

Select the existing public subnet.

Set:

    Connectivity type:
    Public

Allocate or select an Elastic IP.

Create the NAT Gateway.

Wait until:

    State = Available

---

## 100. Manual Method – Add NAT Route

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

Select the NAT Gateway.

Save changes.

---

## 101. Manual Method – Create Security Group

Open:

    EC2
        |
        v
    Security Groups

Click:

    Create security group

Name:

    IFIS-POC-EC2-SG

Select the correct VPC.

Add inbound HTTP:

    Type:
    HTTP

    Protocol:
    TCP

    Port:
    80

Add the required source.

For the current POC:

    VPC CIDR:
    172.31.0.0/16

Also allow the CloudFront origin-facing managed prefix list:

    pl-9aa247f3

Configure outbound traffic according to the approved architecture.

Create the Security Group.

---

## 102. Manual Method – Launch EC2

Open:

    EC2
        |
        v
    Instances

Click:

    Launch instance

Enter:

    Name:
    IFIS-POC-EC2

---

## 103. Manual Method – Select AMI

Select:

    Amazon Linux 2023

Use the current supported Amazon Linux 2023 AMI for the target region.

Do not copy an old AMI ID from another region.

---

## 104. Manual Method – Instance Type

For the current POC:

    t3a.small

The production environment should use the approved instance type specified by project requirements.

---

## 105. Manual Method – Key Pair

The POC uses Systems Manager as the preferred administration method.

SSH is not required for normal administration.

If a key pair is required by the environment:

1. Follow the organization's key management process.
2. Store the private key securely.
3. Never commit the private key to Git.
4. Never upload the private key to GitHub.

---

## 106. Manual Method – Network Settings

Under Network settings:

Select:

    VPC:
    <POC_VPC_ID>

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

## 107. Manual Method – Storage

Configure the root volume:

    20 GB
    gp3
    Encrypted

Enable encryption.

---

## 108. Manual Method – IAM Role

Attach the EC2 IAM role containing:

    AmazonSSMManagedInstanceCore

and:

    CloudWatchAgentServerPolicy

---

## 109. Manual Method – Launch EC2

Before launching, verify:

- Correct VPC
- Correct private subnet
- Public IP disabled
- Correct Security Group
- Correct IAM role
- Encrypted root volume

Click:

    Launch instance

---

## 110. Manual Method – Validate EC2

After launch:

1. Open EC2.
2. Select the instance.
3. Verify state = Running.
4. Verify subnet.
5. Verify private IP.
6. Verify public IPv4 is empty.
7. Verify Security Group.
8. Verify IAM role.

---

## 111. Manual Method – Verify SSM

Open:

    Systems Manager
        |
        v
    Managed nodes

Wait for the EC2 instance to appear.

If it does not appear:

1. Verify IAM role.
2. Verify SSM Agent.
3. Verify outbound connectivity.
4. Verify DNS.
5. Verify NAT Gateway or VPC endpoints.

---

## 112. Manual Method – Create CloudFront VPC Origin

Open:

    AWS Console
        |
        v
    CloudFront

Open the VPC Origins area.

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

## 113. Manual Method – Wait for VPC Origin

Wait until the VPC Origin status becomes:

    Deployed

VPC Origin creation may take several minutes.

---

## 114. Manual Method – Create CloudFront Distribution

Open:

    CloudFront
        |
        v
    Distributions

Click:

    Create distribution

Select the VPC Origin.

Configure viewer protocol:

    Redirect HTTP to HTTPS

Configure origin:

    HTTP
    Port 80

Configure caching according to the target POC architecture.

---

## 115. Manual Method – WAF

During CloudFront creation, the WAF section allows a Web ACL to be associated.

If the WAF has already been created, select the correct Web ACL.

If it has not yet been created, deploy WAF separately and associate it afterwards.

---

## 116. Manual Method – Create WAF

Open:

    AWS Console
        |
        v
    WAF & Shield

Select region:

    US East (N. Virginia)
    us-east-1

Choose:

    Web ACLs
        |
        v
    Create web ACL

Select CloudFront/global scope.

Configure:

    Default action:
    Allow

Add the approved AWS Managed Rules.

---

## 117. Manual Method – Add WAF Rules

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

Review and create the Web ACL.

---

## 118. Manual Method – Associate WAF

Open the Web ACL.

Associate it with the CloudFront distribution.

Verify that the CloudFront distribution shows the correct Web ACL.

---

## 119. Manual Method – Create Backup Vault

Open:

    AWS Console
        |
        v
    AWS Backup
        |
        v
    Backup vaults

Create:

    IFIS-POC-backup-vault

---

## 120. Manual Method – Create Backup Plan

Open:

    AWS Backup
        |
        v
    Backup plans

Choose:

    Create backup plan

Create a weekly schedule.

The target POC schedule is:

    Monday 00:00 JST

Verify the correct UTC schedule before creating the plan.

---

## 121. Manual Method – Backup Selection

Select the EC2 instance.

Create the backup selection.

Use the approved AWS Backup IAM service role.

---

## 122. Manual Method – Backup Retention

Set:

    30 days

Verify the lifecycle configuration.

---

## 123. Manual Method – Test Backup

Start an on-demand backup.

Wait for the backup job to complete.

Open the backup vault.

Verify that a recovery point exists.

---

## 124. Manual Method – Restore

Select the recovery point.

Choose:

    Restore

Restore into a test environment.

Do not overwrite production resources.

After restoration:

1. Verify the restored EC2 instance.
2. Verify networking.
3. Verify application.
4. Verify management access.
5. Record the result.

---

# PART 10 – CLOUD FORMATION VS MANUAL

## 125. CloudFormation vs Manual Deployment

| Area | CloudFormation | Manual |
|---|---|---|
| Network | Template | VPC Console |
| Private subnet | Template | VPC Console |
| Route table | Template | VPC Console |
| NAT Gateway | Template | VPC Console |
| Security Group | Template | EC2/VPC Console |
| EC2 | Template | EC2 Console |
| IAM | Template | IAM Console |
| SSM | Template | Systems Manager |
| CloudWatch | Template | CloudWatch |
| CloudFront | Template | CloudFront |
| VPC Origin | Template | CloudFront |
| WAF | Template | WAF |
| Backup | Template | AWS Backup |

---

## 126. Recommended Deployment Method

For normal project deployment:

    Use CloudFormation

For learning, troubleshooting, and demonstrations:

    Use the AWS Console

CloudFormation should be the primary repeatable deployment mechanism.

---

## 127. Why CloudFormation Is Preferred

CloudFormation provides:

- Repeatability
- Version control
- Parameterization
- Dependency management
- Consistent configuration
- Easier review
- Easier replication
- Reduced manual configuration errors

---

## 128. Manual Deployment Risks

Manual deployment can introduce:

- Incorrect subnet selection
- Incorrect route table association
- Incorrect Security Group rules
- Missing IAM permissions
- Wrong AWS Region
- Incorrect CloudFront configuration
- Incorrect WAF association
- Missing Backup configuration
- Configuration drift

Therefore, manual deployment should mainly be used for learning and troubleshooting.

---

# PART 11 – REUSABILITY AND FUTURE ENVIRONMENTS

## 129. Resource Dependency Summary

The resource dependencies are:

    Existing VPC
        |
        +---- Existing Public Subnet
                  |
                  v
            NAT Gateway
                  |
                  v
            Private Route Table
                  |
                  v
            Private Subnet
                  |
                  v
                 EC2
                  |
            +-----+-----+
            |           |
            v           v
          SSM       CloudWatch
            |
            v
       Administration

    EC2
     |
     v
    CloudFront VPC Origin
     |
     v
    CloudFront
     |
     v
    WAF

    EC2
     |
     v
    AWS Backup

---

## 130. Important Architecture Rule

The NAT Gateway and CloudFront VPC Origin serve different purposes.

NAT Gateway:

    Private EC2
         |
         v
    NAT Gateway
         |
         v
    Internet

This is outbound traffic.

CloudFront VPC Origin:

    CloudFront
         |
         v
    VPC Origin
         |
         v
    Private EC2

This is inbound application traffic.

Do not confuse these two paths.

---

## 131. Important Security Rule

The EC2 instance should remain private.

The intended design is:

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

Do not give the EC2 instance a public IP simply to make CloudFront work.

---

## 132. Important Administration Rule

The preferred administrative path is:

    Administrator
          |
          v
    AWS Systems Manager
          |
          v
    Private EC2

SSH should not be required for normal POC administration.

---

## 133. Important Reusability Rule

Do not hardcode production AWS resource IDs inside reusable templates.

Do not hardcode values such as:

    vpc-xxxxxxxx
    subnet-xxxxxxxx
    i-xxxxxxxx
    nat-xxxxxxxx

Instead use:

- CloudFormation parameters
- Stack outputs
- Cross-stack references
- Configuration files
- Deployment scripts

This allows the same templates to be reused in:

- Different AWS accounts
- Different VPCs
- Different AWS Regions
- Different environments
- Different client deployments

---

## 134. New Environment Deployment Process

For a new environment:

    1. Identify AWS account
            |
            v
    2. Identify AWS Region
            |
            v
    3. Identify existing VPC
            |
            v
    4. Identify public subnet
            |
            v
    5. Select private subnet CIDR
            |
            v
    6. Verify CloudFront prefix list
            |
            v
    7. Deploy Network stack
            |
            v
    8. Deploy Compute stack
            |
            v
    9. Deploy CloudFront
            |
            v
    10. Deploy WAF
            |
            v
    11. Deploy Backup
            |
            v
    12. Validate

---

# PART 12 – FINAL VALIDATION

## 135. Final Deployment Checklist

### AWS Account

- [ ] Correct AWS account
- [ ] Correct AWS Region
- [ ] Required IAM permissions

### Network

- [ ] Correct VPC
- [ ] Public subnet verified
- [ ] Internet Gateway verified
- [ ] Private subnet created
- [ ] Private subnet public IP assignment disabled
- [ ] Private route table created
- [ ] Private subnet associated with route table
- [ ] NAT Gateway created
- [ ] NAT Gateway Available
- [ ] Default route points to NAT Gateway

### Compute

- [ ] EC2 created
- [ ] EC2 Running
- [ ] Amazon Linux 2023
- [ ] EC2 in private subnet
- [ ] No public IP
- [ ] Security Group correct
- [ ] IAM role attached
- [ ] SSM working
- [ ] Session Manager working
- [ ] CloudWatch working
- [ ] Application listening on TCP 80
- [ ] curl localhost works

### CloudFront

- [ ] VPC Origin created
- [ ] VPC Origin Deployed
- [ ] CloudFront distribution created
- [ ] CloudFront distribution Enabled
- [ ] HTTP redirects to HTTPS
- [ ] CloudFront reaches EC2

### WAF

- [ ] WAF Web ACL created
- [ ] Scope = CloudFront
- [ ] Managed rules configured
- [ ] WAF associated with CloudFront

### Backup

- [ ] Backup vault created
- [ ] Backup plan created
- [ ] EC2 selected
- [ ] Schedule verified
- [ ] Retention verified
- [ ] Backup job completed
- [ ] Recovery point exists
- [ ] Restore test completed

---

## 136. Deployment Record

After deployment, update the deployment worksheet:

    AWS Account ID:
    AWS Region:

    VPC ID:
    VPC CIDR:

    Public Subnet ID:
    Public Subnet CIDR:

    Private Subnet ID:
    Private Subnet CIDR:

    Private Route Table ID:

    NAT Gateway ID:

    EC2 Instance ID:
    EC2 Private IP:
    EC2 Security Group ID:

    VPC Origin ID:

    CloudFront Distribution ID:
    CloudFront Domain:

    WAF Web ACL ARN:

    Backup Vault:
    Backup Plan:
    Recovery Point:

Do not store credentials or secrets in this document.

---

## 137. Final End-to-End Architecture Test

Perform the following test:

    Browser
        |
        v
    CloudFront HTTPS
        |
        v
    AWS WAF
        |
        v
    VPC Origin
        |
        v
    Private EC2
        |
        v
    Application

Then separately test:

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

And:

    Private EC2
        |
        +---- Systems Manager
        |
        +---- CloudWatch
        |
        +---- AWS Backup

All three paths must be validated independently.

---

# PART 13 – DOCUMENTATION AFTER DEPLOYMENT

## 138. Documentation After Deployment

After completing deployment, update:

    docs/validation-checklist.md

with the actual validation results.

If a problem occurs, document it in:

    docs/troubleshooting.md

The troubleshooting document should contain:

- Problem
- Symptoms
- Root cause
- How it was identified
- Resolution
- Validation after resolution

This allows future team members to solve the same problem without repeating the investigation.

---

## 139. Related Documentation

### Architecture

    docs/architecture.md

Explains:

- What are we building?
- Why are we building it this way?
- What does each AWS service do?

### Deployment Guide

    docs/deployment-guide.md

Explains:

- How do we create it?
- How do we deploy it using CloudFormation?
- How do we create it manually through the AWS Console?

### Validation Checklist

    docs/validation-checklist.md

Will explain:

- How do we prove that everything is working?

### Troubleshooting

    docs/troubleshooting.md

Will explain:

- What do we do when something fails?

---

# 140. End of Deployment Guide

This document is intended to provide a repeatable deployment procedure for the IFIS POC.

The recommended process is:

    Read Architecture
           |
           v
    Complete Deployment Worksheet
           |
           v
    Verify AWS Account
           |
           v
    Verify VPC
           |
           v
    Deploy Network
           |
           v
    Validate Network
           |
           v
    Deploy Compute
           |
           v
    Validate Compute
           |
           v
    Deploy CloudFront
           |
           v
    Deploy WAF
           |
           v
    Deploy Backup
           |
           v
    Perform End-to-End Validation
           |
           v
    Perform Failure / Restore Testing
           |
           v
    Record Results

The same CloudFormation templates should be reused for future environments wherever possible.

Environment-specific values such as VPC IDs, subnet IDs, AWS account IDs, Availability Zones, and resource IDs must be supplied through parameters or deployment configuration rather than hardcoded into reusable templates.