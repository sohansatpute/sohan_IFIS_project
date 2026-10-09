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

The Route 53 record and a custom domain are optional. The CloudFront distribution can be tested with its default `*.cloudfront.net` domain without configuring Route 53 or a custom certificate.

    Internet
        |
        v
    CloudFront (default *.cloudfront.net URL)
        ^
        |
    Optional: Route 53 alias + custom domain
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

Recommended sequence:

1. Deploy Network.
2. Deploy Compute.
3. Deploy the WAF stack in `us-east-1`.
4. Deploy CloudFront in the workload Region and pass the WAF stack's `WebAclArn` output as `WebAclArn`.
5. Deploy Backup in the workload Region.

The CloudFront template accepts an empty `WebAclArn` for an initial distribution without WAF. If CloudFront is deployed first, update/redeploy that stack with the WAF ARN afterwards. Do not deploy a CloudFront-scoped WAF in `ap-south-1`; its stack must be in `us-east-1`.

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

These values are examples from the current POC environment only. They are not defaults to copy into a client account without verification.

Before deploying in another account or VPC, discover and validate every account-specific value, especially the VPC ID, public subnet ID, Availability Zone, non-overlapping private subnet CIDR, actual VPC CIDR, and Region-specific CloudFront origin-facing managed prefix list. Do not hardcode these resource IDs for future environments.

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

### Exact replacement map for the example commands

| Example in a command | What to use for another environment | How to obtain/choose it |
|---|---|---|
| `ap-south-1` | Your workload Region | Choose the Region approved by the client; keep the WAF stack in `us-east-1` because it has `Scope: CLOUDFRONT`. |
| `vpc-06900f62513eff63` | The client-approved VPC ID | Run the VPC discovery command in Section 11 and select the correct VPC. |
| `172.31.0.0/16` | Actual VPC CIDR | Read `Vpcs[0].CidrBlock` from Section 11; pass it to the Compute stack's `VpcCidr` parameter. |
| `subnet-0d6445bd9f644383b` | Existing public subnet ID in that VPC | Run the subnet discovery command in Section 12 and verify its route table points to an Internet Gateway. |
| `ap-south-1b` | AZ of the selected public subnet | Derive it with the `AvailabilityZone` command in Section 12. |
| `172.31.48.0/20` | Approved, non-overlapping private subnet CIDR | Review all existing VPC subnet CIDRs and associated VPC CIDRs with the network owner; choose an unused block. |
| `pl-9aa247f3` | Managed CloudFront origin-facing prefix-list ID in your Region | Run Section 13; confirm the name is `com.amazonaws.global.cloudfront.origin-facing`. |
| `IFIS-POC-network` etc. | Consistent, unique stack names for the deployment | Choose once; ensure Compute references the exact Network stack name, CloudFront and Backup reference the exact Compute stack name. IAM role names must not collide with existing roles. |
| `IFIS-POC`, `dev`, `CLIENT-OWNER` | Client project, environment, owner/tag values | Use the client's naming/tagging policy. For templates 01–03, Environment must be `dev`, `test`, `uat`, or `prod`. The WAF/Backup templates have different defaults, so pass the intended values explicitly. |
| `t3a.small`, `20` | Approved EC2 size and root volume GB | Choose based on workload and budget. The template only allows `t3.small`, `t3a.small`, `t3.medium`, or `t3a.medium`; it does not allow `t3.micro`. |
| `cron(0 15 ? * SUN *)`, `30` | Approved weekly backup schedule in UTC and retention days | Confirm the required backup window/time zone and retention policy. This cron is 15:00 UTC Sunday (20:30 Sunday IST), with 30-day retention. |

**Copy/paste rule:** The commands in Sections 14–22 are the current POC examples, not universal commands. After running the discovery steps, replace every listed POC value in each command before pressing Enter. Shell variables in Section 9–13 are there to help discover/hold values, but the existing example commands still show literal values for readability. Never paste placeholder strings such as `vpc-REPLACE...` into a deployment command.

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

Find the CloudFormation templates. This guide's commands expect the five files to be together, either in `cloudformation/` or in the repository root. Run:

```bash
if [ -f cloudformation/01-network.yaml ]; then
  export TEMPLATE_DIR="cloudformation"
elif [ -f 01-network.yaml ]; then
  export TEMPLATE_DIR="."
else
  echo "ERROR: 01-network.yaml was not found. Check the repository root and template filenames."; exit 1
fi
printf 'Using template directory: %s\n' "$TEMPLATE_DIR"
ls "$TEMPLATE_DIR"/0*.yaml
```

Confirm the directory contains `01-network.yaml`, `02-compute.yaml`, `03-cloudfront.yaml`, `04-waf.yaml`, and `05-backup.yaml`. If filenames or locations differ, correct the path/filenames before running any deployment command. Do not rename the uploaded copies with `(2)` suffixes; use the actual repository filenames.

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

# 11. Discover and Verify the Existing VPC

The network template **does not create a VPC**. Select the VPC that the client has approved for this deployment.

List available VPCs in the selected Region:

```bash
aws ec2 describe-vpcs --region "$AWS_REGION" \
  --query 'Vpcs[].{VpcId:VpcId,Cidr:CidrBlock,IsDefault:IsDefault,State:State,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table
```

After choosing the intended VPC, set its ID. Replace the sample ID below with the value from the table:

```bash
export VPC_ID="vpc-REPLACE_WITH_CLIENT_VPC_ID"
aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --region "$AWS_REGION" \
  --query 'Vpcs[0].{VpcId:VpcId,Cidr:CidrBlock,State:State}' --output table
export VPC_CIDR="$(aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --region "$AWS_REGION" --query 'Vpcs[0].CidrBlock' --output text)"
echo "VPC_CIDR=$VPC_CIDR"
```

Do not continue if the VPC ID is wrong or the VPC is not available. If the VPC has multiple associated CIDR blocks, review all of them in the VPC console before selecting the private subnet CIDR.

# 12. Discover and Verify the Existing Public Subnet

The network template places the NAT Gateway in an **existing public subnet** belonging to the VPC chosen above. List that VPC's subnets:

```bash
aws ec2 describe-subnets --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[].{SubnetId:SubnetId,Cidr:CidrBlock,AZ:AvailabilityZone,PublicIpOnLaunch:MapPublicIpOnLaunch,State:State,Name:Tags[?Key==`Name`]|[0].Value}' \
  --output table
```

Choose a subnet only after confirming that its route table has `0.0.0.0/0` pointing to an Internet Gateway. `MapPublicIpOnLaunch=true` alone does not prove it is a working public subnet. In the VPC console, open **Route tables**, find the subnet's associated route table (or the VPC main route table if no explicit association exists), and verify the Internet Gateway route.

Set the selected subnet ID and discover its Availability Zone:

```bash
export PUBLIC_SUBNET_ID="subnet-REPLACE_WITH_CLIENT_PUBLIC_SUBNET_ID"
aws ec2 describe-subnets --subnet-ids "$PUBLIC_SUBNET_ID" --region "$AWS_REGION" \
  --query 'Subnets[0].{SubnetId:SubnetId,VpcId:VpcId,Cidr:CidrBlock,AZ:AvailabilityZone,State:State}' --output table
export AZ="$(aws ec2 describe-subnets --subnet-ids "$PUBLIC_SUBNET_ID" --region "$AWS_REGION" --query 'Subnets[0].AvailabilityZone' --output text)"
export PUBLIC_SUBNET_VPC="$(aws ec2 describe-subnets --subnet-ids "$PUBLIC_SUBNET_ID" --region "$AWS_REGION" --query 'Subnets[0].VpcId' --output text)"
test "$PUBLIC_SUBNET_VPC" = "$VPC_ID" || { echo "ERROR: selected subnet is not in VPC $VPC_ID"; exit 1; }
echo "AZ=$AZ"
```

Choose a private CIDR that does not overlap any existing VPC subnet or associated VPC CIDR. List existing subnet CIDRs and check them with the network administrator before using a value such as `172.31.48.0/20` (that value is only the POC example):

```bash
aws ec2 describe-subnets --region "$AWS_REGION" \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[].{SubnetId:SubnetId,Cidr:CidrBlock,AZ:AvailabilityZone}' --output table
export PRIVATE_SUBNET_CIDR="REPLACE_WITH_APPROVED_NON_OVERLAPPING_CIDR"
```

The template does not perform a complete overlap check for you. If NAT is disabled, verify that the private instance can reach required AWS services through VPC endpoints or another approved egress route before deploying.

---

# 13. Discover the CloudFront Origin-Facing Managed Prefix List

The Compute security group references the AWS-managed CloudFront origin-facing prefix list. Its ID can vary by Region; do not blindly reuse the POC ID.

```bash
aws ec2 describe-managed-prefix-lists --region "$AWS_REGION" \
  --filters "Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing" \
  --query 'PrefixLists[].{Name:PrefixListName,Id:PrefixListId,State:State}' --output table
export CLOUDFRONT_PREFIX_LIST_ID="$(aws ec2 describe-managed-prefix-lists --region "$AWS_REGION" \
  --filters "Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing" \
  --query 'PrefixLists[0].PrefixListId' --output text)"
echo "CLOUDFRONT_PREFIX_LIST_ID=$CLOUDFRONT_PREFIX_LIST_ID"
```

Confirm the returned name is `com.amazonaws.global.cloudfront.origin-facing` and the ID begins with `pl-`. If no entry is returned, stop and investigate the selected Region/permissions rather than copying the POC ID.

---

# 14. Network Stack

> **Important before deployment:** The `create-stack` commands below are fresh-deployment examples and contain current POC names/values. Follow Sections 9–13 to discover your account's values and replace each corresponding parameter. Do not run `create-stack` again for a stack that already exists; use a reviewed CloudFormation update/deploy procedure instead. NAT Gateway, CloudFront, WAF, EBS, and backups can incur charges.

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
      --template-body file://$TEMPLATE_DIR/01-network.yaml \
      --region ap-south-1

The command should complete successfully.

---

## 14.3 Deploy Network

Example:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-network \
      --template-body file://$TEMPLATE_DIR/01-network.yaml \
      --parameters \
        ParameterKey=ExistingVpcId,ParameterValue=vpc-06900f62513eff63 \
        ParameterKey=ExistingPublicSubnetId,ParameterValue=subnet-0d6445bd9f644383b \
        ParameterKey=AvailabilityZone,ParameterValue=ap-south-1b \
        ParameterKey=PrivateSubnetCidr,ParameterValue=172.31.48.0/20 \
        ParameterKey=EnableNatGateway,ParameterValue=true \
        ParameterKey=ProjectName,ParameterValue=IFIS-POC \
        ParameterKey=Environment,ParameterValue=dev \
        ParameterKey=Owner,ParameterValue=CLIENT-OWNER \
      --region ap-south-1

For another environment, replace the values as described in Sections 9–13: `ExistingVpcId` = the selected VPC ID, `ExistingPublicSubnetId` = the selected public subnet ID, `AvailabilityZone` = that subnet's AZ, and `PrivateSubnetCidr` = your approved non-overlapping private CIDR. Also update stack name, Region, ProjectName, Environment, and Owner. These CloudFormation commands are examples for a fresh POC stack, not safe to rerun unchanged against an existing stack.

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
      --template-body file://$TEMPLATE_DIR/02-compute.yaml \
      --region ap-south-1

---

## 16.3 Deploy Compute

Run:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-compute \
      --template-body file://$TEMPLATE_DIR/02-compute.yaml \
      --parameters \
        ParameterKey=NetworkStackName,ParameterValue=IFIS-POC-network \
        ParameterKey=VpcCidr,ParameterValue=172.31.0.0/16 \
        ParameterKey=InstanceType,ParameterValue=t3a.small \
        ParameterKey=InstanceName,ParameterValue=IFIS-POC-EC2 \
        ParameterKey=RootVolumeSize,ParameterValue=20 \
        ParameterKey=CloudFrontOriginFacingPrefixListId,ParameterValue=pl-9aa247f3 \
        ParameterKey=ProjectName,ParameterValue=IFIS-POC \
        ParameterKey=Environment,ParameterValue=dev \
        ParameterKey=Owner,ParameterValue=CLIENT-OWNER \
      --region ap-south-1 \
      --capabilities CAPABILITY_NAMED_IAM

IMPORTANT: Replace `VpcCidr` with the actual VPC CIDR discovered in Section 11, and replace the network stack name/prefix-list ID if your names or Region differ. The command above shows the POC example values.

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
      --template-body file://$TEMPLATE_DIR/04-waf.yaml \
      --region us-east-1

---

## 19.3 Deploy WAF

Run:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-waf \
      --template-body file://$TEMPLATE_DIR/04-waf.yaml \
      --parameters \
        ParameterKey=ProjectName,ParameterValue=IFIS-POC \
        ParameterKey=Environment,ParameterValue=dev \
        ParameterKey=Owner,ParameterValue=CLIENT-OWNER \
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
      --template-body file://$TEMPLATE_DIR/03-cloudfront.yaml \
      --region ap-south-1

---

## 20.2 Deploy CloudFront

If the account is verified and CloudFront resource creation is allowed:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-cloudfront \
      --template-body file://$TEMPLATE_DIR/03-cloudfront.yaml \
      --parameters \
        ParameterKey=ComputeStackName,ParameterValue=IFIS-POC-compute \
        ParameterKey=WebAclArn,ParameterValue=REPLACE_WITH_WEB_ACL_ARN \
        ParameterKey=ProjectName,ParameterValue=IFIS-POC \
        ParameterKey=Environment,ParameterValue=dev \
        ParameterKey=Owner,ParameterValue=CLIENT-OWNER \
      --region ap-south-1

Get the exact ARN from the deployed WAF stack (use the same stack name you supplied above):

```bash
aws cloudformation describe-stacks \
  --stack-name IFIS-POC-waf --region us-east-1 \
  --query "Stacks[0].Outputs[?OutputKey=='WebAclArn'].OutputValue | [0]" \
  --output text
```

Copy that returned ARN and replace `REPLACE_WITH_WEB_ACL_ARN` in the CloudFront command. Do not include the angle brackets. If the command returns `None`, the WAF stack did not return the expected output or the stack name/Region is wrong. If you use a different WAF stack name, replace `IFIS-POC-waf` in this lookup too.

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

## 21.1 Optional Custom Domain (Manual Configuration)

The CloudFront template currently uses the default CloudFront certificate and does not define `Aliases` or a custom ACM certificate parameter. Leave the template unchanged for the default `https://<distribution-domain>.cloudfront.net` URL.

If a client wants a custom domain, complete these steps manually after the distribution exists:

1. Choose the hostname, for example `www.example.com`, and confirm control of its DNS zone.
2. In **ACM in `us-east-1`**, request a public certificate for the exact hostname (and any required additional names).
3. Complete ACM DNS validation by adding the CNAME record ACM provides. Keep this validation CNAME in DNS so ACM can renew the certificate.
4. In CloudFront, open the distribution and edit its settings. Add the hostname under **Alternate domain name (CNAME)** and select the issued ACM certificate from `us-east-1`. Save the change and wait until the distribution status is **Deployed**.
5. In the authoritative DNS service, create a Route 53 **A – Alias** record (or the equivalent supported alias at the DNS provider) pointing the hostname to the CloudFront distribution. Create an AAAA Alias only if IPv6 is enabled on the distribution; the current template sets IPv6 to disabled.
6. Verify DNS resolution and open `https://<hostname>` in a browser. Confirm the certificate hostname and HTTPS response.

Important distinctions:

- The ACM validation CNAME proves certificate-domain control; it is not the website alias record.
- Route 53 is optional if using the default CloudFront URL, and Route 53 is not mandatory if the domain's authoritative DNS is hosted elsewhere.
- DNS propagation and CloudFront deployment can take time. Do not treat a saved configuration as ready until CloudFront reports **Deployed** and HTTPS is verified.
- These are manual console steps; they are not created by the current `03-cloudfront.yaml` template. A later CloudFormation change would be needed to manage aliases/certificate association as code.

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
      --template-body file://$TEMPLATE_DIR/05-backup.yaml \
      --region ap-south-1

---

## 22.3 Deploy Backup

Run:

    aws cloudformation create-stack \
      --stack-name IFIS-POC-backup \
      --template-body file://$TEMPLATE_DIR/05-backup.yaml \
      --parameters \
        ParameterKey=ComputeStackName,ParameterValue=IFIS-POC-compute \
        ParameterKey=BackupScheduleCron,ParameterValue='cron(0 15 ? * SUN *)' \
        ParameterKey=RetentionDays,ParameterValue=30 \
        ParameterKey=StartWindowMinutes,ParameterValue=60 \
        ParameterKey=CompletionWindowMinutes,ParameterValue=180 \
        ParameterKey=ProjectName,ParameterValue=IFIS-POC \
        ParameterKey=Environment,ParameterValue=dev \
        ParameterKey=Owner,ParameterValue=CLIENT-OWNER \
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

For India Standard Time (IST, UTC+05:30), this is Sunday 20:30 IST. It is Monday 00:00 in Japan Standard Time (JST, UTC+09:00), so do not copy the JST conversion for an India-based schedule.

AWS Backup interprets this cron expression in UTC. Confirm the desired local weekday and time with the client, then convert that time to UTC before changing `BackupScheduleCron`.

A successful CloudFormation stack deployment does not prove that a backup has completed. Check the AWS Backup job status and confirm that a recovery point exists. Perform a restore test in an isolated environment before relying on backups.

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

Get the distribution domain from the CloudFront stack:

```bash
aws cloudformation describe-stacks \
  --stack-name IFIS-POC-cloudfront --region ap-south-1 \
  --query "Stacks[0].Outputs[?OutputKey=='DistributionDomainName'].OutputValue | [0]" \
  --output text
```

Open `https://` followed by the returned domain name. Replace the stack name/Region in the lookup if yours differs.

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

POC resources may be deleted after testing to control AWS cost. NAT Gateway, EC2/EBS, CloudFront, WAF, CloudWatch usage, data transfer, and AWS Backup storage/recovery points can incur charges. Review the account's current pricing before deployment.

Before cleanup, remove any manually added custom-domain CloudFront alias and DNS record if they should no longer point to this distribution. Preserve the ACM validation CNAME only if the certificate/domain will remain in use.

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

Therefore deleting the CloudFormation stack may leave the Backup Vault behind. Recovery points may also remain and continue to incur storage charges.

The retained vault must be reviewed separately if the objective is complete POC cleanup. Confirm retention/compliance requirements before deleting recovery points, and delete the vault only after it is empty and no longer required.

---

## 27.4 Compute Stack

Before deleting the Compute stack, remember that EC2 termination protection may prevent deletion.

If required, disable termination protection:

First get the EC2 instance ID from the Compute stack:

```bash
aws cloudformation describe-stacks \
  --stack-name IFIS-POC-compute --region ap-south-1 \
  --query "Stacks[0].Outputs[?OutputKey=='Ec2InstanceId'].OutputValue | [0]" \
  --output text
```

Then copy the returned ID into the following command (remove the angle brackets):

```bash
aws ec2 modify-instance-attribute \
  --instance-id i-REPLACE_WITH_RETURNED_INSTANCE_ID \
  --no-disable-api-termination \
  --region ap-south-1
```

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