# IFIS AWS Infrastructure POC

## 1. Purpose

This repository contains a reusable AWS infrastructure implementation based on the IFIS / W2P-Japan architecture.

The objective is to reproduce and validate the target architecture using AWS CloudFormation. The infrastructure is organized into separate stacks to support repeatable deployment, validation, and future environment setup.

**Project type:** Proof of Concept (POC)

## 2. AWS Region

The primary POC region is:

* **Region:** `ap-south-1` (Mumbai)

The AWS region must be selected consistently during deployment. CloudFront-scoped AWS WAF resources must be created in `us-east-1`, as required by AWS.

## 3. Existing POC Network

The current POC uses the following existing VPC:

| Property | Value                   |
| -------- | ----------------------- |
| VPC ID   | `vpc-06900f62513eff63a` |
| VPC CIDR | `172.31.0.0/16`         |
| Region   | `ap-south-1`            |

These values describe the current POC environment only. Before deploying into another AWS account or environment, identify the target VPC, public subnet, Availability Zone, and available private subnet CIDR. Do not assume the POC VPC ID is reusable.

## 4. Architecture Overview

The solution uses Amazon CloudFront, AWS WAF, a private EC2 instance, and AWS Backup.

```text
Users / Internet
       |
       v
Amazon CloudFront
       |
       v
AWS WAF
       |
       v
CloudFront VPC Origin
       |
       v
Private EC2 Web Server
       |
       v
NAT Gateway (outbound connectivity, when enabled)
       |
       v
Internet
```

AWS WAF is associated with the CloudFront distribution to inspect and filter applicable viewer requests. The private EC2 instance hosts the web application or test page and is reached through the CloudFront VPC Origin.

The NAT Gateway provides outbound connectivity for resources in the private subnet when configured and enabled. It is not the inbound path for normal CloudFront-to-EC2 web traffic.

AWS Backup provides scheduled backup and recovery for the EC2 resources configured in the backup plan.

## 5. CloudFormation Stacks

The infrastructure is organized into five CloudFormation templates:

| Deployment order | Stack component | Purpose                                                                                                                           |
| ---------------: | --------------- | --------------------------------------------------------------------------------------------------------------------------------- |
|                1 | Network         | Creates the private subnet and related routing resources in the existing VPC; NAT resources depend on the selected configuration. |
|                2 | Compute         | Deploys the private EC2 instance, IAM resources, web server setup, and monitoring configuration.                                  |
|                3 | CloudFront      | Creates the CloudFront distribution and VPC Origin for the private web server.                                                    |
|                4 | WAF             | Configures AWS WAF managed rule groups for CloudFront.                                                                            |
|                5 | Backup          | Configures the AWS Backup vault, plan, selection, and required IAM role.                                                          |

Review each template's parameters and prerequisites before deployment. Stack names, parameter values, and outputs should be recorded so dependent stacks can reference the correct resources.

## 6. Repository Documentation

Refer to the following guides for detailed instructions:

* [CloudFormation Deployment Guide](docs/cloudformation-deployment-guide.md) — deployment sequence, parameters, commands, and stack validation.
* [Manual Deployment Guide](docs/manual-deployment-guide.md) — console-based setup and validation procedures.
* [Architecture](docs/architecture.md) — architecture components and traffic flow.
* [Troubleshooting](docs/troubleshooting.md) — common deployment and validation issues.
* [Validation Checklist](docs/validation-checklist.md) — checks to perform before considering the POC validated.

Confirm that these files exist at the listed paths in the repository.

## 7. Prerequisites

Before deploying, ensure that you have:

* Access to the intended AWS account and the required permissions to create and manage the resources.
* AWS CLI installed and configured, if using command-line deployment.
* Access to the five CloudFormation templates.
* Identified the target VPC, public subnet, Availability Zone, and non-overlapping private subnet CIDR.
* Reviewed IAM permissions, security-group rules, NAT Gateway costs, backup retention, and CloudFront/WAF settings.
* Checked the deployment guide for required CloudFormation capabilities and region-specific resources.

## 8. Validation

After deployment, validate the following:

* All five CloudFormation stacks reach the expected successful status.
* The EC2 instance is running in the private subnet and has no public IP address.
* The web server responds through the CloudFront distribution over HTTPS.
* AWS WAF is associated with the distribution and its configured managed rules are enabled as intended.
* CloudWatch monitoring and alarms receive the expected metrics.
* AWS Backup has the expected plan, selection, schedule, and retention settings.

A successful stack deployment alone does not prove that every application, security, monitoring, and recovery requirement has been validated. Record test results and any known limitations separately.

## 9. Security and Cost Considerations

* Keep the EC2 instance private and restrict inbound traffic to the sources required by the design.
* Review the CloudFront allowed HTTP methods and origin protocol settings before using the configuration beyond the POC.
* Confirm that the private subnet has the outbound connectivity or VPC endpoints required for instance initialization, Systems Manager, and CloudWatch.
* AWS WAF managed rules may block legitimate requests; test their behavior before production use.
* NAT Gateways, CloudFront, EC2, CloudWatch, and retained backup recovery points may incur charges.
* Review backup-vault retention and resource deletion behavior before cleaning up the environment.

## 10. Project Status

**Current status:** Preparing the repository and deployment framework.

The POC should be considered validated only after the planned deployment, connectivity, security, monitoring, backup, and cleanup checks have been completed and documented.
