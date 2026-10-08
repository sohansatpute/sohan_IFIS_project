# IFIS POC – AWS Troubleshooting Guide

## 1. Purpose

This document provides troubleshooting steps for the IFIS AWS POC.

Use it when a deployment, connectivity test, monitoring test, CloudFront test, WAF test, or backup test fails.

The troubleshooting approach should follow the architecture path:

Client

→ CloudFront

→ WAF

→ VPC Origin

→ Private EC2

→ Application

and separately:

Private EC2

→ NAT Gateway

→ Internet / AWS services


## 2. Basic Troubleshooting Method

When something fails:

1. Identify which layer is failing.
2. Check CloudFormation stack status.
3. Check resource status.
4. Check route tables.
5. Check security groups.
6. Check IAM permissions.
7. Check EC2 system/service status.
8. Check CloudWatch/Systems Manager.
9. Check CloudFront/WAF configuration.
10. Re-test from the failing layer outward.


## 3. CloudFormation Stack Failed

### Check stack status

Run:

`aws cloudformation describe-stacks --stack-name <STACK_NAME> --region <REGION>`

### Check stack events

Run:

`aws cloudformation describe-stack-events --stack-name <STACK_NAME> --region <REGION>`

Look for:

- `CREATE_FAILED`
- `UPDATE_FAILED`
- `ROLLBACK_IN_PROGRESS`
- `ROLLBACK_COMPLETE`
- `DELETE_FAILED`

The first resource showing the actual failure is usually the most important event to investigate.


## 4. Network Stack Failure

### Check VPC

Run:

`aws ec2 describe-vpcs --vpc-ids <VPC_ID> --region ap-south-1`

Confirm the VPC exists.

### Check private subnet

Run:

`aws ec2 describe-subnets --subnet-ids <PRIVATE_SUBNET_ID> --region ap-south-1`

Confirm:

- Correct CIDR.
- Correct Availability Zone.
- Correct VPC.
- Public IP assignment disabled.


## 5. Private Subnet Has No Internet Access

Check the route table.

Run:

`aws ec2 describe-route-tables --route-table-ids <PRIVATE_ROUTE_TABLE_ID> --region ap-south-1`

Expected:

`0.0.0.0/0 → NAT Gateway`

If the route is missing, the private subnet cannot reach the internet through NAT.

### Check NAT Gateway

Run:

`aws ec2 describe-nat-gateways --nat-gateway-ids <NAT_GATEWAY_ID> --region ap-south-1`

Expected state:

`available`

If the NAT Gateway is:

`pending`

wait for it to become available.

If it is:

`failed`

check the public subnet, Elastic IP, and Internet Gateway configuration.


## 6. NAT Gateway Is Available but EC2 Has No Outbound Access

Check all of the following:

1. EC2 is in the correct private subnet.
2. Private subnet is associated with the private route table.
3. Route table has `0.0.0.0/0 → NAT Gateway`.
4. NAT Gateway is in a public subnet.
5. Public subnet route table has `0.0.0.0/0 → Internet Gateway`.
6. NAT Gateway has an Elastic IP.
7. Security group allows outbound traffic.
8. Network ACL does not block outbound/return traffic.


## 7. EC2 Has a Public IP Unexpectedly

Check the subnet configuration and EC2 network interface.

Expected POC configuration:

`AssociatePublicIpAddress = false`

The private subnet should also have public IP assignment disabled.

If a public IP exists unexpectedly, investigate how the instance was launched.


## 8. EC2 Is Running but SSM Shows Offline

Check:

- EC2 IAM role.
- `AmazonSSMManagedInstanceCore`.
- Outbound network connectivity.
- SSM Agent status.

From Session Manager or another valid access method:

`sudo systemctl status amazon-ssm-agent`

Start it if required:

`sudo systemctl start amazon-ssm-agent`

Enable it:

`sudo systemctl enable amazon-ssm-agent`

The instance requires connectivity to AWS Systems Manager endpoints.

This can be provided through:

- NAT Gateway
- Appropriate VPC endpoints


## 9. Session Manager Cannot Connect

Check:

1. EC2 is running.
2. SSM Agent is running.
3. IAM instance profile is attached.
4. Instance has outbound connectivity.
5. Region is correct.
6. Systems Manager shows the instance as a managed node.

Do not immediately add SSH access just to solve an SSM problem. First identify why SSM is unavailable.


## 10. CloudWatch Agent Not Running

Check:

`sudo systemctl status amazon-cloudwatch-agent`

Start:

`sudo systemctl start amazon-cloudwatch-agent`

Enable:

`sudo systemctl enable amazon-cloudwatch-agent`

Check the configuration file:

`sudo cat /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`

Check CloudWatch Agent logs:

`sudo tail -n 100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`

Common causes:

- No outbound connectivity.
- Invalid configuration.
- IAM permissions missing.
- Agent installation failed.
- Service failed to start.


## 11. User Data Did Not Complete

Check cloud-init output:

`sudo tail -n 200 /var/log/cloud-init-output.log`

Also check:

`sudo tail -n 200 /var/log/cloud-init.log`

Look for:

- Package installation failures.
- Repository/network failures.
- Permission errors.
- Service startup errors.

User Data depends on network connectivity when packages are downloaded.


## 12. HTTP Service Is Not Working

Check:

`sudo systemctl status httpd`

Start:

`sudo systemctl start httpd`

Enable:

`sudo systemctl enable httpd`

Test locally:

`curl http://localhost`

If localhost works but CloudFront does not, investigate the network path rather than the web server first.


## 13. Security Group Blocks CloudFront

Expected inbound HTTP rule:

- Port: 80
- Protocol: TCP
- Source: CloudFront managed prefix list

Current POC managed prefix list:

`pl-9aa247f3`

Confirm the security group is attached to the correct EC2 instance.

Check security groups:

`aws ec2 describe-security-groups --group-ids <SG_ID> --region ap-south-1`


## 14. CloudFront Cannot Reach EC2

Troubleshoot in this order:

1. EC2 is running.
2. HTTP service is running.
3. `curl http://localhost` works.
4. Security group allows CloudFront prefix list.
5. VPC Origin exists.
6. VPC Origin status is deployed.
7. VPC Origin targets the correct EC2 resource.
8. CloudFront origin configuration is correct.
9. CloudFront distribution is deployed.
10. WAF is not blocking the request.


## 15. CloudFront VPC Origin Creation Fails

If CloudFormation returns:

`Your account must be verified before you can add new CloudFront resources.`

This is an AWS account-level CloudFront restriction.

It is not necessarily a CloudFormation YAML syntax problem.

Recommended action:

1. Open AWS Support.
2. Create/support the existing case.
3. Provide the exact CloudFront error.
4. Request account verification/unblocking for CloudFront resource creation.
5. Wait for AWS confirmation.
6. Retry CloudFront resource creation after the account is enabled.

Do not repeatedly modify a working CloudFormation template solely because of this error.


## 16. CloudFront Distribution Is Not Deploying

Check:

- Distribution status.
- Origin configuration.
- VPC Origin status.
- WAF ARN if configured.
- CloudFront service errors.
- AWS account CloudFront restrictions.

CloudFront changes can take time to deploy.

Do not repeatedly delete and recreate the distribution unless necessary.


## 17. CloudFront Returns 403

Possible causes:

### WAF

Check WAF sampled requests and rules.

A WAF managed rule may be blocking the request.

### Origin

Check whether the origin is reachable.

### Security Group

Confirm CloudFront managed prefix-list access.

### CloudFront Configuration

Check:

- Distribution enabled.
- Correct origin.
- Correct behavior.
- Viewer protocol policy.
- Allowed HTTP methods.

Determine whether the 403 is generated by CloudFront/WAF or by the origin application.


## 18. CloudFront Returns 502 / 503

Possible causes:

- EC2 is stopped.
- HTTP service is stopped.
- VPC Origin is unavailable.
- Security group blocks CloudFront.
- Origin port is incorrect.
- Application is not listening on port 80.
- Origin resource is unhealthy.

Test on EC2:

`curl http://localhost`

Then inspect the CloudFront origin configuration.


## 19. WAF Stack Fails

Important:

CloudFront-scoped WAF must be created in:

`us-east-1`

The Web ACL must use:

`Scope: CLOUDFRONT`

If created in another Region, the CloudFormation deployment will not produce the intended CloudFront-scoped Web ACL.


## 20. WAF Is Created but CloudFront Is Not Protected

Check the CloudFront distribution Web ACL configuration.

Confirm the Web ACL ARN is associated with the distribution.

The architecture should be:

Client

→ CloudFront

→ WAF inspection

→ VPC Origin

→ EC2


## 21. AWS Backup Stack Fails

Check the compute stack first.

The Backup stack imports:

`<ComputeStackName>-EC2InstanceArn`

Therefore the compute stack must already exist and expose the expected export.

Check CloudFormation exports:

`aws cloudformation list-exports --region ap-south-1`

Confirm the EC2 ARN export exists.


## 22. Backup Job Does Not Start

Check:

- Backup plan exists.
- Backup selection exists.
- EC2 instance exists.
- Backup IAM role exists.
- Backup vault exists.
- Backup schedule is correct.
- AWS Backup service has the required permissions.

Check the AWS Backup console for job status and failure details.


## 23. Backup Restore Fails

Check:

- Backup recovery point is available.
- Recovery point belongs to the expected EC2 resource.
- IAM permissions are correct.
- Target subnet exists.
- Security group exists.
- Required network connectivity is available.

Document the exact restore error before changing infrastructure.


## 24. CloudFormation Import/Export Error

If a stack reports that an export cannot be found:

Check:

`aws cloudformation list-exports --region ap-south-1`

The dependent stack may be expecting an export from another stack.

Deployment order must be respected:

`Network → Compute → CloudFront → WAF/Backup`

depending on the chosen integration method.


## 25. Parameter Error with AWS CLI and Git Bash

When passing Systems Manager Parameter Store paths through Git Bash, values beginning with `/` can be interpreted unexpectedly.

For example:

`/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`

may be transformed by Git Bash into a Windows-style path.

If the CloudFormation template already contains the correct default AMI parameter, omit the `AmiId` override.

This was encountered during the IFIS POC deployment.


## 26. EC2 Termination Protection Blocks Stack Deletion

If CloudFormation cannot terminate the EC2 instance because termination protection is enabled, disable termination protection first.

Example:

`aws ec2 modify-instance-attribute --instance-id <INSTANCE_ID> --no-disable-api-termination --region ap-south-1`

Then retry CloudFormation stack deletion.

Only do this during intentional cleanup/testing.


## 27. CloudFormation Stack Is Stuck in ROLLBACK

Check:

`aws cloudformation describe-stack-events --stack-name <STACK_NAME> --region <REGION>`

Wait for:

`ROLLBACK_COMPLETE`

before deleting the stack if required.

Then:

`aws cloudformation delete-stack --stack-name <STACK_NAME> --region <REGION>`


## 28. CloudFront Account Verification Case

Current POC issue:

CloudFormation attempted to create:

`AWS::CloudFront::VpcOrigin`

AWS returned:

`403 AccessDenied`

with:

`Your account must be verified before you can add new CloudFront resources.`

Action taken:

- AWS Support case raised.
- AWS Support contacted.
- Written response provided.
- Waiting for CloudFront resource creation to be enabled.

Do not treat this as a template validation failure.


## 29. Useful AWS CLI Commands

### Current AWS identity

`aws sts get-caller-identity`

### List CloudFormation stacks

`aws cloudformation list-stacks --region ap-south-1`

### Stack events

`aws cloudformation describe-stack-events --stack-name <STACK_NAME> --region ap-south-1`

### EC2 instances

`aws ec2 describe-instances --region ap-south-1`

### Subnets

`aws ec2 describe-subnets --region ap-south-1`

### Route tables

`aws ec2 describe-route-tables --region ap-south-1`

### NAT Gateways

`aws ec2 describe-nat-gateways --region ap-south-1`

### Security groups

`aws ec2 describe-security-groups --region ap-south-1`

### CloudFormation exports

`aws cloudformation list-exports --region ap-south-1`


## 30. Troubleshooting Record

For every significant issue, record:

### Date

YYYY-MM-DD

### Component

Example:

CloudFront / EC2 / NAT / WAF / Backup

### Symptom

Describe what failed.

### Error

Copy the exact AWS error message.

### Root Cause

Document the confirmed cause.

### Resolution

Document the action taken.

### Validation

Document how the fix was tested.

### Status

- Open
- Resolved
- Known Limitation


## 31. Current Known Limitation

### CloudFront Resource Creation

The POC CloudFormation CloudFront stack previously failed because AWS required account verification before allowing new CloudFront resources.

AWS Support case has been raised.

Status:

`Pending AWS Support`

This limitation should be removed from the final documentation once CloudFront resource creation has been successfully tested.


## 32. Final Troubleshooting Principle

Do not immediately change multiple resources when troubleshooting.

Use the architecture path and isolate the failing layer:

`Client`

→ `CloudFront`

→ `WAF`

→ `VPC Origin`

→ `Security Group`

→ `Private EC2`

→ `Application`

For outbound EC2 problems:

`EC2`

→ `Private Route Table`

→ `NAT Gateway`

→ `Public Subnet`

→ `Internet Gateway`

For AWS service connectivity:

`EC2`

→ `NAT Gateway or VPC Endpoint`

→ `AWS Service`

Identify the failing layer first, then make the smallest required change.