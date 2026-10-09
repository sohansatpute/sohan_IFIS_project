# IFIS POC – Manual AWS Deployment Guide

> **Scope and safety:** This is a manual, console-based procedure for the IFIS proof of concept (POC), not an automated deployment guide. Values and resource IDs shown below describe the existing POC and are examples only. Before using this guide in another account, region, or environment, identify and substitute the target environment's VPC, subnet, CIDR, Availability Zone, prefix list, naming, and security requirements. Do not copy POC resource IDs blindly. Review expected AWS charges before creating NAT Gateway, EC2, CloudFront, WAF, EBS, or backup resources.

## 1. Purpose

This document provides the complete manual deployment procedure for the IFIS POC AWS architecture using the AWS Management Console.

The manual deployment is intended to:

- Understand each AWS resource before automation.
- Reproduce the POC manually.
- Validate the architecture independently from CloudFormation.
- Provide a troubleshooting reference.
- Provide a deployment reference for future environments.
- Compare the manual deployment with the CloudFormation implementation.

This is a POC guide. Production deployments must be reviewed for high availability, security, monitoring, backup, disaster recovery, cost, and operational requirements.

---

# 2. Target Architecture

The target architecture is:

The default CloudFront `*.cloudfront.net` URL works without Route 53 or a custom domain. Route 53 is optional when a custom domain is needed and the domain's DNS is hosted there.

    Internet
        |
        v
    CloudFront (default URL or optional custom hostname)
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
        +---- Private Route Table
                    |
                    v
               NAT Gateway
                    |
                    v
                 Internet

The EC2 instance is also integrated with:

    Private EC2
       |
       +---- AWS Systems Manager
       |
       +---- CloudWatch
       |
       +---- AWS Backup

Important architecture points:

- EC2 is deployed in a private subnet.
- EC2 does not have a public IPv4 address.
- CloudFront reaches EC2 through a VPC Origin.
- The EC2 Security Group allows HTTP traffic from the CloudFront managed prefix list.
- NAT Gateway provides outbound connectivity from the private subnet.
- SSM Session Manager is used for administration instead of SSH.
- CloudWatch Agent collects memory and disk metrics.
- AWS WAF protects the CloudFront distribution.
- AWS Backup provides scheduled EC2 backups.
- Route 53 is optional for the POC.

---

# 3. AWS Regions

Use the following regions:

| Service | Region |
|---|---|
| VPC | ap-south-1 |
| Private EC2 | ap-south-1 |
| NAT Gateway | ap-south-1 |
| CloudFront | Global |
| CloudFront VPC Origin | ap-south-1 |
| AWS WAF for CloudFront | us-east-1 |
| AWS Backup | ap-south-1 |

Important:

A CloudFront-scoped WAF must be created in us-east-1.

Always verify the AWS Console region before creating a resource.

---

# 4. Current POC Network Details

The current POC uses the existing VPC.

| Item | Value |
|---|---|
| Region | ap-south-1 |
| VPC ID | vpc-06900f62513eff63 |
| VPC CIDR | 172.31.0.0/16 |
| Public subnet ID | subnet-0d6445bd9f644383b |
| Public subnet CIDR | 172.31.0.0/20 |
| Public subnet AZ | ap-south-1b |
| Private subnet ID | subnet-08183fb86ccc151e |
| Private subnet CIDR | 172.31.48.0/20 |
| Private subnet AZ | ap-south-1b |
| CloudFront managed prefix list | pl-9aa247f3 |
| CloudFront prefix list name | com.amazonaws.global.cloudfront.origin-facing |

These values identify the current POC environment; they are not universal defaults. Confirm each value in the target AWS account and selected region before proceeding. For another account, region, or environment, identify the equivalent resources instead of copying these IDs. The private subnet ID is shown for reference only; it will be newly assigned when you create a new private subnet.

---

# 5. Prerequisites

Before starting the manual deployment, confirm that you have:

- AWS Management Console access.
- Permission to create VPC resources.
- Permission to create NAT Gateways.
- Permission to create IAM roles and instance profiles.
- Permission to create EC2 instances.
- Permission to create Security Groups.
- Permission to create CloudFront resources.
- Permission to create CloudFront VPC Origins.
- Permission to create WAF resources.
- Permission to create AWS Backup resources.
- Permission to use Systems Manager.
- Permission to use CloudWatch.

---

Before creating billable resources, confirm the cost owner and teardown plan. A public NAT Gateway incurs hourly and data-processing charges; EC2, EBS, CloudFront, WAF, CloudWatch, and AWS Backup can also incur charges. The AWS Console region must be checked for every regional resource. CloudFront-scoped WAF resources must be created in `us-east-1`; the workload resources in this POC use `ap-south-1`.

---

# 6. Deployment Order

Deploy the architecture in this order:

1. Verify VPC.
2. Verify Internet Gateway.
3. Identify public subnet.
4. Create private subnet.
5. Create private route table.
6. Associate private subnet with private route table.
7. Allocate Elastic IP.
8. Create NAT Gateway.
9. Add NAT route to private route table.
10. Create EC2 IAM role.
11. Create EC2 instance profile.
12. Create EC2 Security Group.
13. Launch EC2.
14. Configure User Data.
15. Configure storage.
16. Configure IMDSv2.
17. Disable public IP.
18. Enable termination protection.
19. Validate EC2.
20. Validate SSM.
21. Validate CloudWatch.
22. Create CloudFront VPC Origin.
23. Create CloudFront distribution.
24. Create CloudFront WAF in us-east-1.
25. Associate WAF with CloudFront.
26. Create AWS Backup configuration.
27. Run end-to-end validation.
28. Run failure testing.
29. Clean up the POC when testing is complete.

---

# 7. Step 1 – Verify the Existing VPC

Open:

AWS Console → VPC → Your VPCs

Select the existing VPC.

Verify:

VPC ID:
vpc-06900f62513eff63

IPv4 CIDR:
172.31.0.0/16

Region:
ap-south-1

Also verify that the VPC has an Internet Gateway attached.

Go to:

AWS Console → VPC → Internet Gateways

Confirm that an Internet Gateway is attached to the VPC.

---

# 8. Step 2 – Identify the Public Subnet

Open:

AWS Console → VPC → Subnets

Identify the public subnet:

Subnet ID:
subnet-0d6445bd9f644383b

CIDR:
172.31.0.0/20

Availability Zone:
ap-south-1b

Check the subnet's associated route table.

The public subnet must have:

Destination:
0.0.0.0/0

Target:
Internet Gateway

This subnet will host the NAT Gateway.

Do not deploy the EC2 instance into this subnet.

---

# 9. Step 3 – Create the Private Subnet

Open:

AWS Console → VPC → Subnets → Create subnet

Select the existing VPC.

Enter:

Subnet name:
IFIS-POC-private-subnet

Availability Zone:
ap-south-1b

IPv4 CIDR:
172.31.48.0/20

Create the subnet.

After creation, verify the subnet CIDR and Availability Zone.

Also verify that public IPv4 address assignment is not enabled for the private subnet.

The expected private subnet is:

Subnet ID:
subnet-08183fb86ccc151e

CIDR:
172.31.48.0/20

---

# 10. Step 4 – Create the Private Route Table

Open:

AWS Console → VPC → Route Tables → Create route table

Enter:

Name:
IFIS-POC-private-route-table

VPC:
Existing IFIS VPC

Create the route table.

At this point, the route table will normally contain the local VPC route automatically.

Expected local route:

Destination:
172.31.0.0/16

Target:
local

---

# 11. Step 5 – Associate the Private Subnet

Open the new private route table.

Select:

Subnet associations → Edit subnet associations

Select:

IFIS-POC-private-subnet

Save the association.

Verify that the private subnet is associated with the private route table.

---

# 12. Step 6 – Allocate an Elastic IP

The NAT Gateway requires an Elastic IP.

Open:

AWS Console → VPC → Elastic IPs

Select:

Allocate Elastic IP address

Use:

Network Border Group:
ap-south-1

Allocate the address.

Record the allocated Elastic IP.

Do not release it while the NAT Gateway is using it.

---

# 13. Step 7 – Create the NAT Gateway

Open:

AWS Console → VPC → NAT Gateways → Create NAT Gateway

Configure:

Subnet:
Public subnet 172.31.0.0/20

Connectivity type:
Public

Elastic IP allocation ID:
Select the Elastic IP allocated in the previous step.

Name:

IFIS-POC-NAT

Create the NAT Gateway.

Wait until the NAT Gateway state becomes:

Available

Do not continue until the NAT Gateway is available.

---

# 14. Step 8 – Add the Private Route Through NAT

Open:

AWS Console → VPC → Route Tables

Select:

IFIS-POC-private-route-table

Open:

Routes → Edit routes

Add:

Destination:
0.0.0.0/0

Target:
NAT Gateway

Select the IFIS POC NAT Gateway.

Save the route.

The final private route table should contain approximately:

Destination:
172.31.0.0/16

Target:
local

Destination:
0.0.0.0/0

Target:
nat-xxxxxxxxxxxxxxxxx

This allows resources in the private subnet to initiate outbound connections.

It does not make the private EC2 publicly reachable.

---

# 15. Step 9 – Create the EC2 IAM Role

The EC2 instance needs permissions for Systems Manager and CloudWatch.

Open:

AWS Console → IAM → Roles → Create role

Select:

Trusted entity type:
AWS service

Use case:
EC2

Continue.

Attach:

AmazonSSMManagedInstanceCore

and:

CloudWatchAgentServerPolicy

Continue.

Role name:

IFIS-POC-EC2-Role

Create the role.

The role provides:

- Systems Manager Session Manager access.
- CloudWatch Agent permissions.

Do not attach AdministratorAccess to the EC2 role.

---

# 16. Step 10 – Create or Verify the Instance Profile

When an IAM role is created for EC2 through the AWS Console, AWS normally creates an instance profile associated with the role.

Open:

AWS Console → IAM → Roles

Open:

IFIS-POC-EC2-Role

Verify that the role can be used as an EC2 instance profile.

When launching EC2, select the corresponding IAM role under:

Advanced details → IAM instance profile

---

# 17. Step 11 – Identify the CloudFront Managed Prefix List

CloudFront VPC Origin traffic should be allowed through the AWS-managed CloudFront origin-facing prefix list. The prefix list ID is region-specific; look up the current ID in the workload region rather than assuming the POC ID is valid elsewhere.

Open:

AWS Console → VPC → Managed Prefix Lists

Find:

Name:
com.amazonaws.global.cloudfront.origin-facing

Current POC prefix list:

pl-9aa247f3

Verify the prefix list is the AWS-managed CloudFront origin-facing prefix list.

Do not blindly copy this ID to another account or region without verification.

---

# 18. Step 12 – Create the EC2 Security Group

Open:

AWS Console → EC2 → Security Groups → Create security group

Configure:

Security group name:
IFIS-POC-EC2-SG

Description:
Security group for private IFIS POC EC2

VPC:
Existing IFIS VPC

Inbound rules:

Rule 1:

Type:
HTTP

Protocol:
TCP

Port:
80

Source:
VPC CIDR

172.31.0.0/16

Rule 2:

Type:
HTTP

Protocol:
TCP

Port:
80

Source:
Custom

Select the AWS-managed prefix list:

com.amazonaws.global.cloudfront.origin-facing

pl-9aa247f3

Outbound rules:

Allow all outbound traffic.

Recommended outbound configuration:

Type:
All traffic

Destination:
0.0.0.0/0

Create the Security Group.

Important:

Do not allow HTTP from 0.0.0.0/0.

The purpose of the Security Group is to allow the required internal/VPC traffic and CloudFront origin-facing traffic while keeping the EC2 instance private.

---

# 19. Step 13 – Launch the EC2 Instance

Open:

AWS Console → EC2 → Instances → Launch instance

Configure:

Name:

IFIS-POC-EC2

AMI:

Amazon Linux 2023

Architecture:

x86_64

Instance type:

t3.small

For a POC, a smaller instance can be used if required for cost control, but the documented project baseline is t3.small.

Key pair:

No key pair is required when using Systems Manager Session Manager.

Network settings:

VPC:
Existing IFIS VPC

Subnet:
IFIS-POC-private-subnet

CIDR:
172.31.48.0/20

Auto-assign public IP:
Disable

Security Group:
IFIS-POC-EC2-SG

IAM instance profile:

IFIS-POC-EC2-Role

---

# 20. Step 14 – Configure EC2 Storage

Under:

Configure storage

Use:

Root volume:

Device:
/

Volume type:
gp3

Size:
20 GiB

Delete on termination:
Yes

Encryption:
Enable encryption

The root volume should be encrypted.

For the POC, the default AWS-managed EBS encryption key is acceptable unless the target environment requires a customer-managed KMS key.

---

# 21. Step 15 – Configure IMDSv2

Expand:

Advanced details

Find:

Metadata version

Configure:

IMDSv2:
Required

This corresponds to the CloudFormation configuration requiring:

HttpTokens:
required

This prevents IMDSv1 from being used.

---

# 22. Step 16 – Configure Termination Protection

For the POC architecture, termination protection should be enabled to reduce accidental deletion risk.

After the instance is launched:

Open:

EC2 → Instances

Select:

IFIS-POC-EC2

Choose:

Actions → Instance settings → Change termination protection

Enable:

Termination protection

Important:

Termination protection prevents API termination of the instance and can block termination-based cleanup. The CloudFormation compute template sets `DisableApiTermination: true`; enable the equivalent EC2 setting for the manual POC after launch. Before final cleanup, disable termination protection first.

---

# 23. Step 17 – Configure EC2 User Data

This is an important part of the manual deployment.

The CloudFormation compute stack uses User Data to bootstrap the EC2 instance.

The manual deployment must use equivalent User Data so that the manually created EC2 behaves like the CloudFormation-created EC2.

During EC2 launch:

Open:

Advanced details

Find:

User data

Paste the following complete User Data:

    #!/bin/bash

    dnf update -y

    dnf install -y amazon-cloudwatch-agent jq unzip wget curl

    timedatectl set-timezone Asia/Kolkata

    systemctl enable amazon-ssm-agent
    systemctl start amazon-ssm-agent

    cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json <<'EOF'
    {
      "agent": {
        "metrics_collection_interval": 300,
        "run_as_user": "root"
      },
      "metrics": {
        "namespace": "CWAgent",
        "metrics_collected": {
          "mem": {
            "measurement": [
              "mem_used_percent"
            ],
            "metrics_collection_interval": 300
          },
          "disk": {
            "measurement": [
              "used_percent"
            ],
            "resources": [
              "/"
            ],
            "metrics_collection_interval": 300
          }
        }
      }
    }
    EOF

    systemctl enable amazon-cloudwatch-agent
    systemctl start amazon-cloudwatch-agent

Important:

The User Data is executed during the initial instance boot.

The EC2 instance requires outbound connectivity through the NAT Gateway to download packages and communicate with AWS services, unless suitable VPC endpoints and package sources are configured. If NAT connectivity is unavailable and no alternative path exists, package installation, SSM, and CloudWatch publishing may fail.

---

# 24. Step 18 – Launch the Instance

Review all EC2 settings.

Important final configuration:

| Setting | Expected Value |
|---|---|
| AMI | Amazon Linux 2023 |
| Instance type | t3.small |
| VPC | IFIS VPC |
| Subnet | Private subnet |
| Public IP | Disabled |
| Security Group | IFIS-POC-EC2-SG |
| IAM role | IFIS-POC-EC2-Role |
| Root volume | 20 GiB gp3 |
| Encryption | Enabled |
| IMDSv2 | Required |
| User Data | Configured |
| Termination protection | Enable after launch |

Launch the instance.

---

# 25. Step 19 – Verify EC2 Networking

Open:

AWS Console → EC2 → Instances

Select:

IFIS-POC-EC2

Verify:

State:
Running

Subnet:
Private subnet

Private IPv4:
Assigned

Public IPv4:
None

Public DNS:
None

Security Group:
IFIS-POC-EC2-SG

The instance must not have a public IP address.

---

# 26. Step 20 – Verify EC2 Private IP

Do not configure a fixed private IP unless the architecture specifically requires one.

The EC2 instance should receive a private IP from the private subnet.

Example:

172.31.x.x

The exact private IP can change if the instance is recreated.

Applications should normally use DNS, load balancing, or CloudFront VPC Origin rather than depending on a manually assigned EC2 private IP.

---

# 27. Step 21 – Validate Systems Manager

Open:

AWS Console → Systems Manager → Fleet Manager

or:

AWS Console → Systems Manager → Managed nodes

The EC2 instance should appear as a managed node.

Expected state:

Online

If the instance does not appear:

Check:

1. IAM role.
2. AmazonSSMManagedInstanceCore policy.
3. NAT Gateway.
4. Private route table.
5. Security Group outbound access.
6. Instance User Data.
7. SSM Agent service.
8. Instance time and DNS connectivity.

---

# 28. Step 22 – Connect Using Session Manager

Open:

AWS Console → Systems Manager → Session Manager

Select:

Start session

Select:

IFIS-POC-EC2

Start the session.

No SSH key is required.

No public IP is required.

No inbound SSH port 22 is required.

This is the preferred administration method for the POC.

---

# 29. Step 23 – Validate the User Data

Inside the Session Manager session, verify:

    systemctl status amazon-ssm-agent

The service should be active.

Verify CloudWatch Agent:

    systemctl status amazon-cloudwatch-agent

The service should be active.

Verify installed packages:

    rpm -q jq
    rpm -q unzip
    rpm -q wget
    rpm -q curl

Verify timezone:

    timedatectl

Expected timezone:

    Asia/Kolkata

Verify the CloudWatch Agent configuration:

    cat /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

The configuration should contain:

- CWAgent namespace.
- Memory metric.
- Root disk metric.
- 300-second collection interval.

---

# 30. Step 24 – Validate Outbound Connectivity

From the Session Manager session, test DNS:

    nslookup amazon.com

or:

    curl -I https://aws.amazon.com

If the request succeeds, private-subnet outbound connectivity is working.

If it fails, check:

- Private route table.
- NAT Gateway state.
- Public subnet route table.
- Internet Gateway.
- Security Group outbound rules.
- Network ACLs.
- DNS settings.

---

# 31. Step 25 – Install or Verify a Web Server

The CloudFront VPC Origin requires the EC2 instance to provide a reachable HTTP service. The CloudFormation compute template installs and starts Apache (`httpd`) and writes a test page during User Data execution. For a manual launch, the following steps reproduce that basic test service if the application is not already installed.

Example:

    dnf install -y httpd

Enable and start it:

    systemctl enable httpd
    systemctl start httpd

Verify:

    systemctl status httpd

Create a simple test page:

    echo "IFIS POC - CloudFront VPC Origin Test" > /var/www/html/index.html

Test locally from the EC2 instance:

    curl http://localhost

Expected result:

    IFIS POC - CloudFront VPC Origin Test

Important:

The EC2 Security Group must allow TCP port 80 from the required CloudFront origin-facing prefix list. The current compute template also allows HTTP from the configured VPC CIDR; review whether that broad internal access is appropriate for your target environment.

---

# 32. Step 26 – Validate HTTP From Inside the VPC

The application must be listening on:

TCP 80

Verify:

    ss -lntp | grep :80

Expected result should show a listening HTTP service.

If HTTP is not listening, CloudFront cannot reach the application even if the VPC Origin configuration is correct.

---

# 33. Step 27 – Create the CloudFront VPC Origin

Open:

AWS Console → CloudFront

Select:

VPC origins

Choose:

Create VPC origin

Select the private EC2 instance.

For the current POC:

EC2:
IFIS-POC-EC2

Configure:

Origin protocol:
HTTP only

HTTP port:
80

HTTPS port:
443

Create the VPC Origin.

Wait for the VPC Origin status to become:

Deployed

VPC Origin creation can take several minutes.

Do not continue until the VPC Origin is ready.

Important:

CloudFront VPC Origin connects CloudFront to the private resource.

The EC2 instance does not need a public IP.

---

# 34. Step 28 – Create the CloudFront Distribution

Open:

AWS Console → CloudFront → Distributions

Choose:

Create distribution

Select the VPC Origin created in the previous step.

Configure:

Origin:
IFIS-POC VPC Origin

Origin protocol:
HTTP only

Viewer protocol policy:

Redirect HTTP to HTTPS

For the IFIS POC, use settings consistent with `cloudformation/03-cloudfront.yaml`:

- **Allowed HTTP methods:** all seven methods — GET, HEAD, OPTIONS, PUT, POST, PATCH, and DELETE. The template allows these methods even though a simple static test page only needs GET and HEAD. For a real application, restrict methods to the minimum required; do not enable write methods without an application/security review.
- **Cached methods:** GET and HEAD.
- **Caching:** disabled by setting minimum, default, and maximum TTLs to `0`.
- **Query strings:** forwarded to the origin.
- **Compression:** enabled.
- **Price class:** `PriceClass_100`, which limits edge locations and may affect latency for some viewers.
- **HTTP versions:** HTTP/2 and HTTP/3.
- **IPv6:** disabled in the current template.

Default root object:

Not required if the application handles the root path.

Create the distribution.

Wait for the CloudFront distribution status to become:

Deployed

---

# 35. Step 29 – Verify the CloudFront Distribution

After deployment, record:

Distribution ID

CloudFront domain name

Example:

dxxxxxxxxxxxx.cloudfront.net

Open the CloudFront domain in a browser:

https://dxxxxxxxxxxxx.cloudfront.net

The request should reach:

CloudFront
    →
VPC Origin
    →
Private EC2
    →
HTTP service

The EC2 instance should remain private.

---

# 36. Step 30 – Create the CloudFront WAF

Important:

CloudFront-scoped WAF must be created in:

us-east-1

Change the AWS Console region to:

US East (N. Virginia)

Open:

AWS Console → WAF & Shield

Choose:

Web ACLs

Create web ACL.

Configure:

Resource type:
CloudFront distributions

Scope:
CloudFront / Global

Name:

IFIS-POC-cloudfront-waf

Default action:

Allow

Add the following AWS Managed Rules:

1. AWSManagedRulesCommonRuleSet
2. AWSManagedRulesKnownBadInputsRuleSet
3. AWSManagedRulesLinuxRuleSet
4. AWSManagedRulesSQLiRuleSet
5. AWSManagedRulesAmazonIpReputationList

Enable:

Sampled requests

CloudWatch metrics

Create the Web ACL.

---

# 37. Step 31 – Associate WAF With CloudFront

Open:

AWS Console → CloudFront

Select the IFIS POC distribution.

Open:

Security

Find:

Web Application Firewall (WAF)

Select:

IFIS-POC-cloudfront-waf

Save the configuration.

Verify that the CloudFront distribution shows the WAF association.

Important:

WAF is not a separate network hop.

It is associated with and evaluated at the CloudFront layer.

---

# 38. Step 32 – Validate WAF

Open:

AWS Console → WAF & Shield → Web ACLs

Select:

IFIS-POC-cloudfront-waf

Review:

- Web ACL status.
- Managed rules.
- Sampled requests.
- CloudWatch metrics.

The Web ACL template uses **Default action: Allow** and the listed AWS Managed Rule groups with their normal blocking behavior (it does not set the rules to Count mode). A request can therefore be blocked by a managed rule even while the default action is Allow. Review sampled requests and metrics, test legitimate application traffic, and consider a staged Count-mode evaluation in a non-production environment before enforcing managed rules on a production application. WAF usage may incur additional charges.

---

# 39. Step 33 – Create AWS Backup

AWS Backup protects the EC2 instance independently of CloudFront.

Open:

AWS Console → AWS Backup

Create a Backup vault.

Name:

IFIS-POC-backup-vault

Create the vault.

Because the CloudFormation implementation uses Retain behavior for the backup vault, remember that deleting the stack may not delete the vault automatically.

---

# 40. Step 34 – Create AWS Backup Plan

Open:

AWS Backup → Backup plans

Choose:

Create backup plan

Create a new plan.

Plan name:

IFIS-POC-backup-plan

Create a backup rule.

Rule name:

WeeklyEC2Backup

Backup vault:

IFIS-POC-backup-vault

Schedule:

Weekly

The current template default is `cron(0 15 ? * SUN *)`, meaning 15:00 UTC every Sunday. This corresponds to Sunday 20:30 India Standard Time (IST). AWS Backup cron expressions are evaluated in UTC; confirm the required local schedule before changing it. (For reference, 15:00 UTC Sunday is Monday 00:00 in Japan Standard Time.)

Retention:

30 days

Configure the start and completion windows according to the approved backup requirements.

Create the backup plan.

---

# 41. Step 35 – Assign the EC2 Instance to the Backup Plan

Select:

Assign resources

Choose:

Resource type:
EC2

Select:

IFIS-POC-EC2

Alternatively, use resource tags if the environment is designed around tag-based backup selection.

Verify that the EC2 instance is included in the backup selection.

---

# 42. Step 36 – Validate AWS Backup

Open:

AWS Backup → Protected resources

Verify the EC2 instance appears.

Check:

Backup plan:
IFIS-POC-backup-plan

Backup vault:
IFIS-POC-backup-vault

The configured schedule is `cron(0 15 ? * SUN *)`, which means 15:00 UTC every Sunday (20:30 Sunday in India Standard Time). Confirm that this is the intended local time before using it outside the POC. Retention is 30 days in the documented baseline.

A scheduled backup should eventually create a recovery point. For immediate POC validation, perform an on-demand backup if required; this may create additional AWS charges.

---

# 43. Step 37 – Validate CloudWatch

Open:

AWS Console → CloudWatch

Check:

Metrics

The CloudWatch Agent should publish metrics under the `CWAgent` namespace. Expected metrics include `mem_used_percent` and `disk_used_percent`; the root disk resource is `/`. The collection interval is 300 seconds.

Also review the CloudWatch alarms created by the compute template: high CPU, system status check failure, instance status check failure, high memory usage, and high root-disk usage. Open **CloudWatch → Alarms** and confirm the alarms exist and transition out of `INSUFFICIENT_DATA` after metrics arrive.

**Important implementation check:** the current compute template defines memory and disk alarms with an `InstanceId` metric dimension, but its CloudWatch Agent configuration does not explicitly configure that dimension. Confirm that the published `CWAgent` metrics have dimensions matching the alarm definitions. If the dimensions do not match, the memory/disk alarms may remain in `INSUFFICIENT_DATA`; correct and validate the template/agent configuration before relying on those alarms.

CloudWatch Agent logs can be checked on the EC2 instance if metrics do not appear.

---

# 44. Step 38 – Validate the Complete Request Path

Perform an end-to-end test:

Client
    →
CloudFront
    →
WAF
    →
VPC Origin
    →
Private EC2
    →
HTTP service

Open the CloudFront domain:

https://<cloudfront-domain>

Verify the expected application or test page is returned.

Confirm:

- CloudFront distribution is deployed.
- WAF is associated.
- VPC Origin is deployed.
- EC2 is running.
- EC2 has no public IP.
- HTTP service is running.
- Security Group allows CloudFront origin-facing traffic.
- Private subnet route goes through NAT.
- EC2 remains private.

---

# 45. Step 39 – Verify EC2 Cannot Be Accessed Directly From the Internet

The EC2 instance must not have a public IP.

Verify:

EC2 → Instance → Networking

Public IPv4 address:

None

Public DNS:

None

This is an important security validation.

The intended public entry point is CloudFront.

---

# 46. Step 40 – Test Direct Private IP Access

The EC2 private IP should not be directly reachable from the public Internet.

Do not expose the private EC2 address publicly.

The expected access pattern is:

Internet client
    →
CloudFront
    →
VPC Origin
    →
EC2

not:

Internet client
    →
EC2

---

# 47. Step 41 – Test SSM Access

Start a Session Manager session.

Verify that administrative access continues to work without:

- Public IP.
- SSH.
- Port 22.
- Bastion host.

This confirms that the private EC2 management design is working.

---

# 48. Step 42 – Failure Test – Stop EC2

For POC testing, stop the EC2 instance.

Expected result:

CloudFront cannot successfully serve the application while the origin EC2 is unavailable.

This demonstrates that the current single-EC2 POC does not provide application high availability.

Start the EC2 instance again.

Verify:

- EC2 becomes running.
- SSM returns.
- HTTP service starts.
- CloudFront can reach the origin again.

---

# 49. Step 43 – Failure Test – Remove NAT Route

For controlled testing only, temporarily remove:

0.0.0.0/0 → NAT Gateway

from the private route table.

Expected effect:

- EC2 loses normal outbound Internet connectivity.
- Package downloads may fail.
- SSM connectivity may eventually fail if no VPC endpoints exist.
- CloudWatch Agent connectivity may fail.
- CloudFront-to-EC2 traffic may still be possible because CloudFront VPC Origin traffic is inbound to the private resource and does not require NAT.

Restore the NAT route after testing.

---

# 50. Step 44 – Failure Test – NAT Gateway Removal (Destructive)

For controlled testing only, **delete** the NAT Gateway if you need to test this failure mode. AWS NAT Gateways do not have a stop/start action. Deleting and recreating one can change its ID and incur charges; avoid this destructive test unless you have approval and a recovery plan. A safer first test is to temporarily remove the private route to the NAT Gateway, as described in the previous section.

Expected result:

Private EC2 loses outbound connectivity.

This demonstrates the dependency:

Private EC2
    →
Private Route Table
    →
NAT Gateway
    →
Internet Gateway
    →
Internet

If you deleted the NAT Gateway, recreate it, associate an Elastic IP, and verify both the public-subnet route to the Internet Gateway and the private-subnet route to the new NAT Gateway. If you only removed the private route, restore that route after testing.

NAT Gateway creation and usage may incur AWS charges.

---

# 51. Step 45 – Failure Test – Stop HTTP Service

Inside the SSM session:

    systemctl stop httpd

Test the CloudFront URL.

Expected result:

The application response fails because the origin web service is unavailable.

Start the service again:

    systemctl start httpd

Test CloudFront again.

---

# 52. Step 46 – Backup Restore Test

If a recovery point has been created, perform a restore test.

Open:

AWS Backup → Backup vaults

Select:

IFIS-POC-backup-vault

Select a recovery point.

Choose:

Restore

Restore the EC2 instance to a test configuration.

Do not overwrite the working POC instance unless specifically required.

Validate:

- EC2 is restored.
- Networking is correct.
- IAM role is correct.
- Security Group is correct.
- Application files are present.
- Application starts.
- SSM works.
- CloudWatch works.

Record the restore result.

A backup that has never been restored should not be treated as a fully validated recovery process.

---

# 53. Step 47 – Route 53 Configuration

Route 53 is optional for the POC.

If a DNS name is required:

Open:

AWS Console → Route 53

Create or use the appropriate hosted zone.

Create an alias record pointing to the CloudFront distribution.

Example:

Application domain:

app.example.com

Target:

CloudFront distribution

Do not create a Route 53 record unless a domain is available and DNS testing is required.

---

## 53.1 Optional Custom Domain and HTTPS

The manual POC can use the default CloudFront URL and does not require a domain. If a custom hostname is required, configure it after the CloudFront distribution exists:

1. Request a public ACM certificate for the hostname in **`us-east-1`**.
2. Add the DNS validation CNAME supplied by ACM and wait until the certificate status is **Issued**. Keep the validation CNAME for certificate renewal.
3. In CloudFront distribution settings, add the hostname under **Alternate domain name (CNAME)** and select the issued ACM certificate from `us-east-1`.
4. Save and wait for the distribution status to become **Deployed**.
5. In the domain's authoritative DNS service, create an A Alias record in Route 53 (or the equivalent supported alias with the DNS provider) pointing to the CloudFront distribution. The current CloudFormation template has IPv6 disabled, so do not add an AAAA alias unless IPv6 is enabled.
6. Verify DNS resolution and test `https://<hostname>` in a browser. Confirm that the certificate matches the hostname.

The ACM validation CNAME and the website alias record are different records. These settings are manual and are not managed by the current `03-cloudfront.yaml` template.

---

# 54. Step 48 – Final Validation Checklist

Before declaring the manual deployment successful, verify all of the following.

## Network

- VPC exists.
- VPC CIDR is correct.
- Internet Gateway is attached.
- Public subnet exists.
- Private subnet exists.
- Private subnet has public IP assignment disabled.
- Private route table exists.
- Private subnet is associated with private route table.
- NAT Gateway is available.
- NAT Gateway is in public subnet.
- Private route table has 0.0.0.0/0 → NAT Gateway.
- Public subnet route table has 0.0.0.0/0 → Internet Gateway.

## EC2

- Amazon Linux 2023 is used.
- EC2 is in private subnet.
- EC2 has no public IP.
- EC2 uses correct Security Group.
- EC2 uses IAM role.
- Root volume is gp3.
- Root volume is encrypted.
- Root volume size is 20 GiB for the documented POC.
- IMDSv2 is required.
- Termination protection is enabled.
- User Data completed successfully.
- HTTP service is running.

## Security

- Port 80 is not open to 0.0.0.0/0.
- CloudFront managed prefix list is allowed.
- VPC CIDR access is configured as required.
- Outbound traffic is controlled according to the POC design.
- No public IP is assigned to EC2.
- SSH is not required.
- SSM is used for administration.

## Systems Manager

- EC2 appears as a managed node.
- Session Manager connection works.
- SSM Agent is running.

## CloudWatch

- CloudWatch Agent is running.
- CWAgent namespace exists.
- Memory metrics are available.
- Root disk metrics are available.
- Metrics are collected every 300 seconds.

## CloudFront

- VPC Origin exists.
- VPC Origin status is deployed.
- Origin points to private EC2.
- HTTP port 80 is configured.
- CloudFront distribution is deployed.
- Viewer redirects HTTP to HTTPS.
- CloudFront domain is reachable.

## WAF

- WAF is created in us-east-1.
- Scope is CLOUDFRONT.
- Web ACL is associated with CloudFront.
- Common Rule Set is enabled.
- Known Bad Inputs Rule Set is enabled.
- Linux Rule Set is enabled.
- SQLi Rule Set is enabled.
- Amazon IP Reputation List is enabled.

## Backup

- Backup vault exists.
- Backup plan exists.
- EC2 is assigned to the plan.
- Weekly schedule is configured.
- Retention is 30 days.
- Recovery point is created or scheduled.
- Restore testing is completed if required.

---

# 55. Troubleshooting

## Problem: EC2 does not appear in Systems Manager

Check:

1. EC2 IAM role.
2. AmazonSSMManagedInstanceCore policy.
3. SSM Agent.
4. NAT Gateway.
5. Private route table.
6. Security Group outbound access.
7. DNS.
8. EC2 User Data.

Inside Session Manager is not possible if SSM itself is unavailable, so use EC2 console status checks and CloudWatch/log information where available.

---

## Problem: User Data did not run correctly

Check:

EC2 → Actions → Monitor and troubleshoot → Get system log

Also check:

    /var/log/cloud-init-output.log

and:

    /var/log/cloud-init.log

Verify the User Data script syntax.

---

## Problem: CloudWatch metrics are missing

Check:

    systemctl status amazon-cloudwatch-agent

Check configuration:

    cat /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

Check CloudWatch Agent logs.

Verify:

- IAM policy.
- NAT connectivity.
- DNS.
- Agent configuration.
- Agent service status.

---

## Problem: CloudFront cannot reach EC2

Check all of the following:

1. EC2 is running.
2. HTTP service is running.
3. Port 80 is listening.
4. Security Group allows the CloudFront managed prefix list.
5. VPC Origin is deployed.
6. VPC Origin points to the correct EC2.
7. EC2 private DNS/private endpoint is correct.
8. Network ACLs are not blocking traffic.
9. CloudFront distribution is deployed.

Test locally:

    curl http://localhost

---

## Problem: CloudFront returns an origin error

Check:

- EC2 HTTP service.
- Security Group.
- VPC Origin.
- CloudFront origin protocol.
- Port 80.
- EC2 private connectivity.
- Application logs.

Do not immediately make the EC2 public to troubleshoot the issue.

---

## Problem: NAT Gateway does not provide Internet access

Check:

1. NAT Gateway state is Available.
2. NAT Gateway is in a public subnet.
3. Public subnet route table has 0.0.0.0/0 → Internet Gateway.
4. Private route table has 0.0.0.0/0 → NAT Gateway.
5. EC2 Security Group allows outbound traffic.
6. Network ACLs permit traffic.
7. DNS resolution is enabled.

---

## Problem: WAF cannot be created

Verify that the AWS Console region is:

us-east-1

CloudFront WAF uses the CLOUDFRONT scope.

---

## Problem: Backup does not run

Check:

- EC2 is assigned to the backup plan.
- Backup role exists.
- Backup vault exists.
- Backup schedule is correct.
- Backup service permissions are correct.
- Backup job status.
- Backup job start/completion windows.

---

# 56. Cost Considerations

The following resources can generate ongoing or usage-based AWS charges:

- NAT Gateway.
- Elastic IP associated with NAT Gateway.
- EC2.
- EBS.
- CloudWatch metrics/logs.
- CloudFront.
- WAF.
- AWS Backup storage.
- Backup recovery points.
- Data transfer.

For a temporary POC, delete unused resources after testing.

NAT Gateway is especially important to clean up because it can generate charges even when the EC2 instance is idle.

---

# 57. Cleanup Procedure

Before deleting resources, understand the dependencies.

Recommended cleanup order:

1. Remove the custom-domain DNS alias if it should no longer point to this distribution; keep the ACM validation CNAME only if the certificate will remain in use.
2. Disable or remove CloudFront WAF association.
3. Delete CloudFront distribution.
4. Wait until CloudFront distribution is disabled/deleted.
5. Delete CloudFront VPC Origin.
6. Delete AWS Backup plan/selection.
7. Delete backup recovery points if no longer required.
8. Delete backup vault only if retention is no longer required.
9. Stop EC2.
10. Disable termination protection.
11. Terminate EC2.
12. Delete EC2 Security Group.
13. Delete IAM instance profile/role if no longer required.
14. Delete the NAT Gateway (there is no stop/start operation for a NAT Gateway). Wait until deletion completes.
15. Release the Elastic IP only after the NAT Gateway no longer uses it and only if it is not needed elsewhere.
16. Delete the private route table after removing its NAT route and subnet association as required.
17. Delete the private subnet after all resources and interfaces in it have been removed.

Do not delete the existing VPC if it is shared with other workloads.

---

# 58. Important Cleanup Note for AWS Backup

The CloudFormation implementation uses:

DeletionPolicy:
Retain

for the backup vault.

Therefore, the backup vault may remain after stack deletion.

Manual deployment should also treat backup recovery points as retained data until the required retention period or cleanup decision has been completed.

Do not delete recovery points that are still required for testing, compliance, or recovery.

---

# 59. Manual Deployment vs CloudFormation

The manual deployment and CloudFormation deployment should create functionally equivalent resources.

| Manual Resource | CloudFormation Stack |
|---|---|
| Private subnet | 01-network.yaml |
| Private route table | 01-network.yaml |
| NAT Gateway | 01-network.yaml |
| IAM role | 02-compute.yaml |
| Instance profile | 02-compute.yaml |
| EC2 Security Group | 02-compute.yaml |
| EC2 | 02-compute.yaml |
| User Data | 02-compute.yaml |
| CloudFront VPC Origin | 03-cloudfront.yaml |
| CloudFront Distribution | 03-cloudfront.yaml |
| WAF | 04-waf.yaml |
| AWS Backup | 05-backup.yaml |

This mapping is useful when troubleshooting.

If a manual deployment works but the CloudFormation deployment fails, compare the resource configurations.

If both fail, investigate the underlying AWS architecture or account configuration.

---

# 60. Important Difference Between Manual and CloudFormation Deployment

Manual deployment requires the engineer to configure each resource through the AWS Console.

CloudFormation deployment defines the desired configuration in YAML and creates the resources automatically.

Manual deployment:

    Engineer
        |
        v
    AWS Console
        |
        v
    Individual AWS Resources

CloudFormation deployment:

    YAML Templates
        |
        v
    CloudFormation
        |
        v
    AWS Resources

The architecture should remain the same.

---

# 61. POC Limitations

The current POC has some intentional limitations.

## Single EC2

Only one EC2 instance is used.

This means there is no application-level high availability.

If the EC2 instance fails, the CloudFront origin becomes unavailable.

## Single NAT Gateway

The POC uses one NAT Gateway.

A production multi-AZ architecture may require additional NAT Gateways and routing.

## Existing VPC

The POC uses an existing VPC rather than creating a completely new production network.

## No Application Load Balancer

CloudFront connects directly to the private EC2 through VPC Origin.

A production architecture may use an ALB depending on application requirements.

## No Multi-AZ EC2

The POC EC2 is deployed in one Availability Zone.

## Backup Is Not High Availability

AWS Backup provides recovery capability.

It does not provide automatic application failover.

---

# 62. Production Considerations

Before using this architecture in production, evaluate:

- Multiple Availability Zones.
- Multiple EC2 instances.
- Auto Scaling.
- Application Load Balancer.
- Multiple NAT Gateways.
- VPC endpoints for AWS services.
- Centralized logging.
- CloudWatch alarms.
- SNS notifications.
- WAF logging.
- CloudFront logging.
- Route 53 health checks where required.
- AWS Backup vault protection.
- Cross-account backup.
- Cross-region disaster recovery.
- KMS key management.
- IAM least privilege.
- Security Group hardening.
- Network ACL requirements.
- Patch management.
- SSM automation.
- Incident response.
- Disaster recovery testing.
- Cost monitoring.

---

# 63. Reusing This Manual Design in Another Environment

When deploying the architecture into another AWS account or region, do not copy resource IDs.

Collect the following from the target environment:

- AWS account ID.
- Region.
- VPC ID.
- VPC CIDR.
- Public subnet ID.
- Public subnet CIDR.
- Availability Zone.
- Private subnet CIDR.
- CloudFront managed prefix list ID.
- EC2 AMI.
- Instance type.
- IAM requirements.
- Backup requirements.
- Domain name if Route 53 is required.

Then recreate the architecture using the target environment's resources.

---

# 64. Reusing the Architecture Across Accounts

For another AWS account:

1. Verify account access.
2. Select target region.
3. Identify target VPC.
4. Identify public subnet.
5. Create private subnet.
6. Create NAT Gateway.
7. Create IAM role.
8. Create Security Group.
9. Launch EC2.
10. Configure User Data.
11. Configure CloudFront VPC Origin.
12. Configure CloudFront.
13. Create CloudFront WAF in us-east-1.
14. Configure Backup.
15. Validate the complete architecture.

Do not reuse:

- EC2 IDs.
- VPC IDs.
- Subnet IDs.
- Route table IDs.
- NAT Gateway IDs.
- Elastic IP allocation IDs.
- Security Group IDs.
- IAM role ARNs.
- CloudFront distribution IDs.
- WAF ARNs.

These are account/environment-specific.

---

# 65. Reusing the Architecture Across Regions

For another region:

1. Confirm CloudFront VPC Origin supports the target region/resource configuration.
2. Identify a suitable VPC.
3. Identify a public subnet.
4. Create a private subnet.
5. Create NAT Gateway.
6. Configure routing.
7. Create IAM role.
8. Launch EC2.
9. Configure User Data.
10. Verify SSM.
11. Verify CloudWatch.
12. Create VPC Origin.
13. Configure CloudFront.
14. Configure CloudFront WAF in us-east-1.
15. Configure Backup in the workload region.
16. Test the complete path.

Do not assume the CloudFront managed prefix list ID will be identical in every environment.

Verify it in the target environment.

---

# 66. Final Architecture Validation

The POC is considered manually deployed successfully when all of the following are true:

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
       +---- SSM works
       |
       +---- CloudWatch works
       |
       +---- HTTP service works
       |
       +---- No public IP
       |
       +---- NAT outbound connectivity
       |
       +---- AWS Backup configured

The most important end-to-end test is:

    Client
       |
       v
    CloudFront HTTPS URL
       |
       v
    WAF
       |
       v
    VPC Origin
       |
       v
    Private EC2 HTTP service

At the same time:

    Private EC2
       |
       +---- SSM
       |
       +---- CloudWatch
       |
       +---- NAT
       |
       +---- AWS Backup

must operate as expected.

---

# 67. Related Documentation

See the following documents in this repository:

- architecture.md
- cloudformation-deployment-guide.md
- validation-checklist.md
- troubleshooting.md

CloudFormation templates (paths relative to the repository root):

- cloudformation/01-network.yaml
- cloudformation/02-compute.yaml
- cloudformation/03-cloudfront.yaml
- cloudformation/04-waf.yaml
- cloudformation/05-backup.yaml

---

# 68. Document Maintenance

Update this document when:

- AWS console workflow changes.
- Architecture changes.
- EC2 configuration changes.
- Security Group rules change.
- CloudFront configuration changes.
- WAF rules change.
- Backup policy changes.
- New regions are supported.
- New AWS accounts are onboarded.
- Production requirements differ from the POC.

The manual guide should remain aligned with the CloudFormation implementation.

When a resource is changed in CloudFormation, review the corresponding manual deployment section and update it if required.