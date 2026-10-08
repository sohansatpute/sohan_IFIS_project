# IFIS POC – CloudFormation Deployment Guide

## 1. Purpose

This document provides the step-by-step procedure for deploying the IFIS AWS POC architecture using AWS CloudFormation.

CloudFormation is the primary and recommended deployment method because it provides:

- Repeatable deployments
- Infrastructure as Code
- Version control
- Parameterization
- Consistent configuration
- Easier review
- Easier replication across environments

This guide is intended to be reusable for future AWS accounts, environments, and regions.

---

## 2. Architecture

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

### Important traffic flows

CloudFront to EC2:

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

EC2 outbound:

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

These are two different traffic paths.

The NAT Gateway is not used for CloudFront-to-EC2 traffic.

---

## 3. CloudFormation Stacks

The project contains the following CloudFormation templates:

    cloudformation/
    |
    +-- 01-network.yaml
    |
    +-- 02-compute.yaml
    |
    +-- 03-cloudfront.yaml
    |
    +-- 04-waf.yaml
    |
    +-- 05-backup.yaml

### Stack responsibilities

| Template | Purpose | Region |
|---|---|---|
| 01-network.yaml | Private subnet, route table, NAT Gateway | ap-south-1 |
| 02-compute.yaml | IAM, Security Group, EC2, CloudWatch | ap-south-1 |
| 03-cloudfront.yaml | VPC Origin and CloudFront distribution | ap-south-1 |
| 04-waf.yaml | CloudFront WAF | us-east-1 |
| 05-backup.yaml | AWS Backup | ap-south-1 |

---

## 4. Deployment Dependency

The deployment dependency is:

    Existing VPC
        |
        v
    01-network
        |
        v
    02-compute
        |
        +------------------+
        |                  |
        v                  v
    03-cloudfront       05-backup
        |
        v
    04-waf

The actual CloudFront/WAF deployment can be arranged so that the WAF is created first and then associated with CloudFront, or CloudFront can be created first and the WAF association completed afterwards.

For a clean fully integrated deployment, creating the WAF before the final CloudFront configuration is preferred.

---

## 5. AWS Regions

Main infrastructure:

    ap-south-1

CloudFront-scoped WAF:

    us-east-1

The WAF template uses:

    Scope: CLOUDFRONT

Therefore the WAF CloudFormation stack must be deployed in `us-east-1`.

---

## 6. Current POC Values

The current POC uses:

| Item | Value |
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

These values are for the current POC.

Do not hardcode these resource IDs for future environments.

Future deployments should supply:

- VPC ID
- Public Subnet ID
- Availability Zone
- Private Subnet CIDR
- CloudFront managed prefix list
- Environment name
- Project name
- Owner

through CloudFormation parameters or deployment configuration.

---

# 7. Prerequisites

## 7.1 AWS Access

The AWS account must have permissions to create and manage:

- CloudFormation
- VPC
- EC2
- IAM
- CloudWatch
- CloudFront
- AWS WAF
- AWS Backup

---

## 7.2 Workstation

Install:

- Git
- Git Bash
- AWS CLI

Verify Git:

    git --version

Verify AWS CLI:

    aws --version

Verify AWS identity:

    aws sts get-caller-identity

---

# 8. Prepare Repository

Clone the repository:

    git clone https://github.com/sohansatpute/sohan_IFIS_project.git

Move into the repository:

    cd sohan_IFIS_project

Check files:

    ls

Check CloudFormation templates:

    ls cloudformation

Expected:

    01-network.yaml
    02-compute.yaml
    03-cloudfront.yaml
    04-waf.yaml
    05-backup.yaml

---

# 9. Verify AWS Region

For the main infrastructure:

    aws configure set region ap-south-1

Verify:

    aws configure get region

Expected:

    ap-south-1

Do not deploy the CloudFront WAF template using this region.

The WAF stack must use:

    us-east-1

---

# 10. Verify AWS Account

Run:

    aws sts get-caller-identity

Verify the Account ID.

Make sure the correct AWS account is being used before creating resources.

---

# 11. Verify Existing VPC

The Network template uses an existing VPC.

Verify the VPC:

    AWS Console
        |
        v
    VPC
        |
        v
    Your VPCs

Current POC:

    VPC:
    vpc-06900f62513eff63

    CIDR:
    172.31.0.0/16

For another environment, replace the VPC ID with the correct environment VPC.

---

# 12. Verify Existing Public Subnet

The NAT Gateway is created inside an existing public subnet.

Current POC:

    Subnet:
    subnet-0d6445bd9f644383b

    CIDR:
    172.31.0.0/20

    Availability Zone:
    ap-south-1b

Verify that the public subnet has a route:

    0.0.0.0/0 -> Internet Gateway

---

# 13. Verify CloudFront Managed Prefix List

The Compute stack requires the CloudFront origin-facing managed prefix list.

Current POC:

    pl-9aa247f3

The prefix list name is:

    com.amazonaws.global.cloudfront.origin-facing

For future environments, verify the correct managed prefix list instead of blindly copying the current ID.

---

# 14. Network Stack

## 14.1 What It Creates

The Network stack creates:

- Private subnet
- Private route table
- Route table association
- Elastic IP for NAT
- NAT Gateway
- NAT default route

It uses:

- Existing VPC
- Existing public subnet

It does not create a new VPC.

---

## 14.2 Validate Template

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/01-network.yaml \
      --region ap-south-1

The command should complete successfully.

---

## 14.3 Deploy Network

Example:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-network \
      --template-body file://cloudformation/01-network.yaml \
      --parameters \
        ParameterKey=ExistingVpcId,ParameterValue=vpc-06900f62513eff63 \
        ParameterKey=ExistingPublicSubnetId,ParameterValue=subnet-0d6445bd9f644383b \
        ParameterKey=AvailabilityZone,ParameterValue=ap-south-1b \
        ParameterKey=PrivateSubnetCidr,ParameterValue=172.31.48.0/20 \
        ParameterKey=EnableNatGateway,ParameterValue=true \
        ParameterKey=ProjectName,ParameterValue=IFIS \
        ParameterKey=Environment,ParameterValue=POC \
        ParameterKey=Owner,ParameterValue=Sohan \
      --region ap-south-1

For another environment, replace the environment-specific values.

---

## 14.4 Wait for Network

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-network \
      --region ap-south-1

Check status:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-network \
      --region ap-south-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

## 14.5 Get Network Outputs

Run:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-network \
      --region ap-south-1 \
      --query "Stacks[0].Outputs"

Record:

- VPC ID
- Private Subnet ID
- Private Route Table ID
- NAT Gateway ID

---

# 15. Validate Network

## 15.1 Private Subnet

Verify:

- Correct VPC
- Correct CIDR
- Correct Availability Zone
- Public IP assignment disabled

Current POC:

    CIDR:
    172.31.48.0/20

---

## 15.2 Private Route Table

Expected routes:

    Destination       Target
    --------------------------------
    172.31.0.0/16     local
    0.0.0.0/0         NAT Gateway

---

## 15.3 NAT Gateway

Verify:

    State = Available

Verify:

- NAT Gateway is in public subnet
- Elastic IP is associated
- Public subnet has Internet Gateway route

---

# 16. Compute Stack

## 16.1 What It Creates

The Compute stack creates:

- IAM Role
- Instance Profile
- Security Group
- EC2 instance
- CloudWatch alarms

The EC2 instance is launched in the private subnet created by the Network stack.

---

## 16.2 Validate Template

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/02-compute.yaml \
      --region ap-south-1

---

## 16.3 Deploy Compute

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

---

## 16.4 Important AMI Parameter Note

The Compute template already has a default Amazon Linux 2023 AMI parameter.

When using Git Bash on Windows, passing a value beginning with:

    /aws/service/...

can sometimes be interpreted as a Windows filesystem path.

This can produce an incorrect value such as:

    C:/Users/...

Therefore, use the template default unless a different AMI is specifically required.

---

## 16.5 Wait for Compute

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-compute \
      --region ap-south-1

Check:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-compute \
      --region ap-south-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

## 16.6 Get Compute Outputs

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

# 17. Validate Compute

Verify:

- EC2 state = Running
- EC2 is in private subnet
- Public IPv4 address = None
- Security Group is correct
- IAM role is attached
- SSM is working
- CloudWatch is working

---

## 17.1 Verify SSM

Open:

    Systems Manager
        |
        v
    Managed nodes

The EC2 instance should appear.

---

## 17.2 Verify Session Manager

Open:

    Systems Manager
        |
        v
    Session Manager
        |
        v
    Start session

Start a session with the EC2 instance.

---

## 17.3 Verify Application

From Session Manager:

    sudo ss -lntp

Verify TCP port 80 is listening.

Then:

    curl http://localhost

The application should respond.

---

# 18. CloudFront Account Verification

Before creating CloudFront resources, verify that the AWS account is permitted to create CloudFront resources.

If CloudFormation returns:

    Your account must be verified before you can add new CloudFront resources.

this is an AWS account-level restriction.

Do not immediately modify the CloudFormation template.

Contact AWS Support and request CloudFront resource creation to be enabled.

---

# 19. WAF Stack

## 19.1 Important Region

The WAF template uses:

    Scope: CLOUDFRONT

Therefore deploy it in:

    us-east-1

---

## 19.2 Validate WAF

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/04-waf.yaml \
      --region us-east-1

---

## 19.3 Deploy WAF

Run:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-waf \
      --template-body file://cloudformation/04-waf.yaml \
      --region us-east-1

---

## 19.4 Wait for WAF

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-waf \
      --region us-east-1

Check:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-waf \
      --region us-east-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

## 19.5 Get WAF ARN

Run:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-waf \
      --region us-east-1 \
      --query "Stacks[0].Outputs"

Record:

    WebAclArn

This ARN is used for the CloudFront distribution.

---

# 20. CloudFront Stack

## 20.1 Validate Template

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/03-cloudfront.yaml \
      --region ap-south-1

---

## 20.2 Deploy CloudFront

If the account is verified and CloudFront resource creation is allowed:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-cloudfront \
      --template-body file://cloudformation/03-cloudfront.yaml \
      --parameters \
        ParameterKey=ComputeStackName,ParameterValue=IFIS-POC-compute \
        ParameterKey=WebAclArn,ParameterValue=<WEB_ACL_ARN> \
        ParameterKey=ProjectName,ParameterValue=IFIS \
        ParameterKey=Environment,ParameterValue=POC \
        ParameterKey=Owner,ParameterValue=Sohan \
      --region ap-south-1

Replace:

    <WEB_ACL_ARN>

with the ARN returned by the WAF stack.

---

## 20.3 Wait for CloudFront

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-cloudfront \
      --region ap-south-1

Check:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-cloudfront \
      --region ap-south-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

## 20.4 Get CloudFront Outputs

Run:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-cloudfront \
      --region ap-south-1 \
      --query "Stacks[0].Outputs"

Record:

- VPC Origin ID
- CloudFront Distribution ID
- CloudFront Domain

---

# 21. Validate CloudFront

Verify:

- VPC Origin exists
- VPC Origin status = Deployed
- Origin points to EC2
- HTTP port = 80
- Distribution exists
- Distribution status = Enabled
- Viewer protocol redirects HTTP to HTTPS
- WAF is associated
- CloudFront can reach EC2

---

# 22. Backup Stack

## 22.1 What It Creates

The Backup stack creates:

- Backup Vault
- Backup IAM Role
- Backup Plan
- Backup Selection

The selection imports the EC2 ARN from the Compute stack.

---

## 22.2 Validate Template

Run:

    aws cloudformation validate-template \
      --template-body file://cloudformation/05-backup.yaml \
      --region ap-south-1

---

## 22.3 Deploy Backup

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

## 22.4 Wait for Backup

Run:

    aws cloudformation wait stack-create-complete \
      --stack-name IFIS-POC-backup \
      --region ap-south-1

Check:

    aws cloudformation describe-stacks \
      --stack-name IFIS-POC-backup \
      --region ap-south-1 \
      --query "Stacks[0].StackStatus"

Expected:

    CREATE_COMPLETE

---

# 23. Backup Validation

Verify:

- Backup Vault exists
- Backup Plan exists
- EC2 is selected
- Schedule is correct
- Retention is 30 days
- Backup job completes successfully
- Recovery point exists

---

## 23.1 Backup Schedule

The current schedule is:

    cron(0 15 ? * SUN *)

This means:

    Sunday 15:00 UTC

The original project documentation maps this to:

    Monday 00:00 JST

For another environment, verify the required timezone and schedule.

---

# 24. End-to-End Validation

## 24.1 Application Path

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

The application should respond.

---

## 24.2 EC2 Outbound Path

From Session Manager:

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

# 25. Failure Testing

## 25.1 EC2 Failure

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

## 25.2 NAT Failure

NAT Gateway failure affects outbound traffic from the private subnet.

It does not represent the CloudFront-to-EC2 traffic path.

---

## 25.3 Backup Restore Test

Perform a restore test from a recovery point.

Restore into a test environment.

Validate:

- EC2
- Network
- Application
- Administration

Record the result.

---

# 26. Troubleshooting

## 26.1 CloudFormation Failure

Check stack events:

    aws cloudformation describe-stack-events \
      --stack-name IFIS-POC-network \
      --region ap-south-1

Replace the stack name when required.

Look for:

    CREATE_FAILED

The first failure is normally the most useful event.

---

## 26.2 CloudFront Account Verification Failure

Error:

    Your account must be verified before you can add new CloudFront resources.

Action:

- Contact AWS Support.
- Request CloudFront resource creation to be enabled.
- Include the exact AWS error.
- Do not assume the CloudFormation template is invalid.

---

## 26.3 EC2 SSM Failure

Check:

1. IAM role
2. SSM Agent
3. DNS
4. Security Group outbound rules
5. Route table
6. NAT Gateway
7. AWS service connectivity

---

## 26.4 CloudFront Failure

Check:

1. EC2 is Running
2. Application listens on port 80
3. Security Group allows CloudFront origin-facing prefix list
4. VPC Origin is Deployed
5. VPC Origin points to the correct EC2
6. CloudFront distribution is Enabled
7. WAF is not blocking valid traffic

---

## 26.5 Backup Failure

Check:

1. Backup plan
2. Backup selection
3. EC2 resource
4. Backup IAM role
5. Backup job events
6. Backup vault

---

# 27. Cleanup

POC resources may be deleted after testing to control AWS cost.

Recommended deletion order:

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

The WAF can be deleted separately from:

    us-east-1

---

## 27.1 CloudFront Stack

Delete:

    aws cloudformation delete-stack \
      --stack-name IFIS-POC-cloudfront \
      --region ap-south-1

Wait:

    aws cloudformation wait stack-delete-complete \
      --stack-name IFIS-POC-cloudfront \
      --region ap-south-1

---

## 27.2 WAF Stack

Delete:

    aws cloudformation delete-stack \
      --stack-name IFIS-POC-waf \
      --region us-east-1

Wait:

    aws cloudformation wait stack-delete-complete \
      --stack-name IFIS-POC-waf \
      --region us-east-1

---

## 27.3 Backup Stack

Delete:

    aws cloudformation delete-stack \
      --stack-name IFIS-POC-backup \
      --region ap-south-1

Important:

The Backup Vault has:

    DeletionPolicy: Retain

Therefore deleting the CloudFormation stack may leave the Backup Vault behind.

The retained vault must be reviewed separately if the objective is complete POC cleanup.

---

## 27.4 Compute Stack

Before deleting the Compute stack, remember that EC2 termination protection may prevent deletion.

If required, disable termination protection:

    aws ec2 modify-instance-attribute \
      --instance-id <INSTANCE_ID> \
      --no-disable-api-termination \
      --region ap-south-1

Then:

    aws cloudformation delete-stack \
      --stack-name IFIS-POC-compute \
      --region ap-south-1

Wait:

    aws cloudformation wait stack-delete-complete \
      --stack-name IFIS-POC-compute \
      --region ap-south-1

---

## 27.5 Network Stack

Delete:

    aws cloudformation delete-stack \
      --stack-name IFIS-POC-network \
      --region ap-south-1

Wait:

    aws cloudformation wait stack-delete-complete \
      --stack-name IFIS-POC-network \
      --region ap-south-1

Verify that:

- NAT Gateway is deleted
- Elastic IP is released
- Private subnet is deleted
- Private route table is deleted

---

# 28. Reusable Deployment

For another environment:

1. Identify AWS account.
2. Identify AWS Region.
3. Identify existing VPC.
4. Identify public subnet.
5. Select private subnet CIDR.
6. Verify Availability Zone.
7. Verify CloudFront managed prefix list.
8. Deploy Network.
9. Deploy Compute.
10. Deploy WAF.
11. Deploy CloudFront.
12. Deploy Backup.
13. Validate all services.
14. Record environment-specific outputs.

Do not modify the templates simply because the VPC ID or subnet ID changes.

Pass environment-specific values as parameters.

---

# 29. Final CloudFormation Validation Checklist

## Network

- [ ] Network template validated
- [ ] Network stack CREATE_COMPLETE
- [ ] Private subnet created
- [ ] Private route table created
- [ ] NAT Gateway Available
- [ ] NAT route correct

## Compute

- [ ] Compute template validated
- [ ] Compute stack CREATE_COMPLETE
- [ ] EC2 Running
- [ ] EC2 private
- [ ] No public IP
- [ ] Security Group correct
- [ ] IAM role attached
- [ ] SSM working
- [ ] CloudWatch working
- [ ] Application working

## WAF

- [ ] WAF template validated
- [ ] WAF deployed in us-east-1
- [ ] Web ACL created
- [ ] Managed rules configured
- [ ] Web ACL ARN recorded
- [ ] WAF associated with CloudFront

## CloudFront

- [ ] CloudFront template validated
- [ ] VPC Origin created
- [ ] VPC Origin Deployed
- [ ] Distribution created
- [ ] Distribution Enabled
- [ ] HTTPS works
- [ ] Application reachable

## Backup

- [ ] Backup template validated
- [ ] Backup stack CREATE_COMPLETE
- [ ] Backup Vault exists
- [ ] Backup Plan exists
- [ ] EC2 selected
- [ ] Backup job completed
- [ ] Recovery point exists
- [ ] Restore tested

---

# 30. Related Documentation

Architecture:

    docs/architecture.md

Manual deployment:

    docs/manual-deployment-guide.md

Validation:

    docs/validation-checklist.md

Troubleshooting:

    docs/troubleshooting.md

CloudFormation templates:

    cloudformation/

---

# 31. Document Ownership

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

This document should be updated whenever the CloudFormation architecture or deployment process changes.