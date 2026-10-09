# IFIS POC – AWS Architecture

## 1. Purpose

This document describes the intended and partially implemented AWS architecture for the IFIS / W2P-Japan Proof of Concept (POC), including network design, CloudFormation responsibilities, security, monitoring, backup, validation, and operations. It is not a production specification. Distinguish resources confirmed deployed from resources whose deployment or end-to-end validation is still pending.

## 2. Project Information

| Item | POC value / note |
|---|---|
| Project | W2P-Japan / IFIS |
| Environment | POC |
| Primary infrastructure region | `ap-south-1` |
| CloudFront-scoped AWS WAF region | `us-east-1` |
| Infrastructure as Code | AWS CloudFormation |
| Repository | `sohan_IFIS_project` |

The original notes included an AWS account ID and live resource identifiers. Keep these in an access-controlled POC inventory if required; avoid publishing account IDs or live resource IDs in public documentation. All resource IDs below are POC-specific and must not be reused for another environment.

## 3. Architecture Overview

The application EC2 instance is intended to remain in a private subnet. CloudFront is the public viewer entry point, and AWS WAF is associated with the CloudFront distribution to inspect applicable requests. CloudFront reaches the private EC2 origin through a CloudFront VPC Origin.

```text
Viewer / Internet
       |
       v
Amazon CloudFront  <---- AWS WAF protection associated with distribution
       |
       v
CloudFront VPC Origin
       |
       v
Private EC2 web server
       |
       +---- AWS Systems Manager (administration)
       +---- Amazon CloudWatch (monitoring)
       +---- AWS Backup (scheduled recovery points)
       |
       +---- Private route table -> NAT Gateway -> Internet Gateway -> Internet
                                      outbound connectivity only
```

Route 53 is optional for a custom DNS name. It is not a required hop for every CloudFront request, and should only be described as deployed if configured. A distribution can be accessed through its default CloudFront domain.

**Important traffic distinction:** NAT Gateway provides outbound connectivity from the private subnet. It is not the path CloudFront uses to reach the private EC2 origin. The EC2 instance should not have a public IPv4 address.

## 4. Network Architecture

### 4.1 Existing VPC

The network stack uses an existing VPC supplied as a parameter; it does not create the VPC itself.

| Property | Current POC reference |
|---|---|
| VPC ID | `vpc-06900f62513eff63` |
| VPC CIDR | `172.31.0.0/16` |
| Region | `ap-south-1` |

Verify the VPC ID against the target AWS account before any operation. For another environment, supply that environment's VPC ID and CIDR.

### 4.2 Existing public subnet

The NAT Gateway is placed in an existing public subnet. The original POC notes list:

| Property | Current POC reference |
|---|---|
| Subnet ID | `subnet-0d6445bd9f644383b` |
| CIDR | `172.31.0.0/20` |
| Availability Zone | `ap-south-1b` |

Confirm that the subnet has a route to an Internet Gateway and that the selected Availability Zone matches the intended design.

### 4.3 Private subnet and route table

The network stack creates a private subnet and dedicated route table. The original POC notes list private subnet CIDR `172.31.48.0/20` in `ap-south-1b`. The CIDR must not overlap any existing subnet.

With NAT enabled, the expected route table is:

| Destination | Target |
|---|---|
| VPC CIDR (`172.31.0.0/16` in this POC) | `local` |
| `0.0.0.0/0` | NAT Gateway |

When NAT is disabled, do not assume a default route exists. Instance initialization, Systems Manager, and CloudWatch Agent connectivity must instead be supported by suitable VPC endpoints or another approved network path.

### 4.4 NAT Gateway

The NAT Gateway is placed in the existing public subnet and uses an Elastic IP when enabled by the network stack. It allows private-subnet resources to initiate outbound connections. It does not allow arbitrary Internet-initiated connections to the private EC2 instance. NAT Gateways incur ongoing charges.

## 5. Compute and Security

### 5.1 EC2 instance

The compute stack deploys an Amazon Linux 2023 EC2 instance into the private subnet. The original POC notes list instance ID `i-043ba12e8364ece96`; this is an inventory reference only and must not be hardcoded into reusable templates.

The compute template is documented as configuring an encrypted gp3 root volume, requiring IMDSv2, enabling EC2 termination protection, creating an IAM role and instance profile, and using user data to install/configure the web server, Systems Manager, and CloudWatch Agent. Confirm the current template values before changing a live stack.

### 5.2 Security group

The POC design describes inbound TCP port 80 from the VPC CIDR and from the CloudFront origin-facing managed prefix list. The original notes list prefix-list ID `pl-9aa247f3`; verify it in the target region/account rather than copying it blindly.

The original design permits outbound traffic. Review whether broad egress is necessary for the target environment. Do not expose SSH (TCP 22) to the public Internet; use Systems Manager where configured and operational.

### 5.3 Systems Manager

Systems Manager is the intended administrative path and avoids requiring public SSH. Successful use depends on the instance role, SSM Agent, DNS, and network access to required Systems Manager endpoints through NAT or suitable VPC endpoints.

## 6. CloudFront and VPC Origin

The CloudFront stack is intended to create a VPC Origin for the private EC2 resource and a distribution that forwards viewer requests to that origin.

The documented template behavior includes:
- Redirecting viewer HTTP requests to HTTPS.
- Using the default CloudFront certificate unless custom-domain configuration is added separately.
- Using HTTP on port 80 from CloudFront to the private origin (`origin protocol policy: http-only`).
- Disabling caching through zero TTL settings.
- Forwarding query strings and enabling compression.
- Allowing the HTTP methods configured in the template; review these methods before production because write methods expand the origin's request surface.

**TLS boundary:** Viewer-to-CloudFront traffic uses HTTPS after redirection, but CloudFront-to-origin traffic is HTTP in the described template. That origin leg is not encrypted. Evaluate whether origin HTTPS is required.

CloudFront VPC Origin creation was previously reported to fail because of an account-verification restriction. Treat this as a blocker until a successful stack event confirms the restriction has been lifted and the origin/distribution deploy. Template validation alone does not prove AWS resource creation will succeed.

## 7. AWS WAF

The WAF stack creates an `AWS::WAFv2::WebACL` with `Scope: CLOUDFRONT`. CloudFront-scoped WAF resources must be created in `us-east-1`, even when the application VPC and EC2 instance are in `ap-south-1`.

The documented managed rule groups are:
- `AWSManagedRulesCommonRuleSet`
- `AWSManagedRulesKnownBadInputsRuleSet`
- `AWSManagedRulesLinuxRuleSet`
- `AWSManagedRulesSQLiRuleSet`
- `AWSManagedRulesAmazonIpReputationList`

The described configuration uses a default Allow action and managed rules to evaluate and block requests according to the rules. Metrics and sampled requests are described as enabled. Do not assume WAF request logging is configured unless a logging destination is separately set up and verified.

Managed rules can block legitimate requests. Validate them in a non-production environment and consider Count mode during initial evaluation where appropriate.

### WAF and CloudFront dependency

The WAF can be created independently, but the distribution must be configured with the correct Web ACL ARN for protection to be active. Follow the templates' actual parameter/export mechanism; do not assume the stacks are connected merely because both exist.

## 8. CloudFormation Stack Responsibilities and Dependencies

The repository's five templates are intended to have these responsibilities:

| Template | Responsibility | Main dependency |
|---|---|---|
| `cloudformation/01-network.yaml` | Private subnet, route table and association, optional NAT Gateway/EIP and route | Existing VPC and public subnet parameters |
| `cloudformation/02-compute.yaml` | EC2, security group, IAM role/profile, instance configuration, CloudWatch alarms | Network outputs, including private subnet |
| `cloudformation/03-cloudfront.yaml` | CloudFront VPC Origin and distribution; optional Web ACL association | Compute outputs and, if associated at deployment, WAF ARN |
| `cloudformation/04-waf.yaml` | CloudFront-scoped WAF Web ACL and managed rules | Deploy in `us-east-1`; distribution association needs its ARN |
| `cloudformation/05-backup.yaml` | Backup vault, plan, role, selection and retention | Compute instance/resource ARN |

Verify these paths against the actual repository tree before relying on them.

Recommended operational sequence:
1. Deploy or verify the network stack in the application region.
2. Deploy or verify the compute stack in the application region.
3. Create the WAF stack in `us-east-1` when its ARN is needed for CloudFront configuration.
4. Deploy the CloudFront stack with the correct compute-origin information and Web ACL ARN if supported by its template.
5. Deploy the backup stack in the region containing the protected EC2 resource.
6. Validate origin response, WAF association, monitoring, backup jobs, and restore.

The exact order of WAF and CloudFront depends on how the templates pass the Web ACL ARN. Review template parameters and exports before deployment.

## 9. Monitoring and Initialization

The compute template's described user data installs required packages, configures the web server, enables Systems Manager, and configures the CloudWatch Agent. The agent is intended to publish memory and root filesystem utilization metrics. CloudWatch alarms are described for CPU, instance status checks, memory, and disk.

Initialization depends on package repositories and AWS service endpoints being reachable. If NAT is disabled, provide the required VPC endpoints or another approved path. Confirm that agent metrics use the same metric names and dimensions expected by alarms; an alarm's existence does not prove its metric is being published.

## 10. AWS Backup

The backup template is described as creating a backup vault, IAM role, plan, and selection for the EC2 resource, with a 30-day retention period.

The schedule recorded in the POC notes is:

```text
cron(0 15 ? * SUN *)
```

AWS Backup evaluates this schedule in UTC: 15:00 UTC Sunday, 00:00 JST Monday, and 20:30 IST Sunday. Confirm the current template's schedule before relying on these times.

The vault is documented with `DeletionPolicy: Retain` and `UpdateReplacePolicy: Retain`. A retained vault and recovery points may remain after stack deletion and continue to incur charges. A successful backup job does not prove the application can be recovered; perform a restore test and record the outcome.

The described selection covers the configured EC2 resource only. Other data stores or application dependencies need their own backup and recovery requirements.

## 11. Current POC Status

The following is a historical status snapshot from the original notes, not a live AWS check. Update it after checking current stack events and resources.

| Component | Status reported in original notes |
|---|---|
| Existing VPC | Existing |
| Network stack / private subnet / route table / NAT | Reported deployed |
| Compute stack / private EC2 / security group | Reported deployed |
| Systems Manager / CloudWatch | Reported configured; operational validation should be confirmed |
| CloudFront template | Reported validated |
| CloudFront deployment | Reported blocked by AWS account verification |
| WAF template | Reported validated; deployment pending |
| Backup template | Reported validated; deployment pending |

A template passing `validate-template` checks structure; it does not prove resources can be created or that the resulting architecture works end to end. Update each status based on current evidence.

## 12. Validation Plan

Validate each layer separately and record evidence.

### Network
- Confirm VPC, subnet CIDRs, Availability Zones, route table association, and route targets.
- Confirm NAT Gateway state and outbound connectivity when NAT is enabled.
- Confirm that disabling NAT does not leave initialization or service traffic without a route.

### Compute
- Confirm EC2 is running in the intended private subnet and has no public IP.
- Confirm security group rules match intended sources and ports.
- Confirm Systems Manager access works without public SSH.
- Confirm CloudWatch Agent metrics and alarms receive the expected names and dimensions.

### CloudFront and WAF
- Confirm VPC Origin and distribution creation completed successfully.
- Confirm viewer HTTP redirects to HTTPS and the application responds through the distribution.
- Confirm origin protocol matches the documented TLS boundary.
- Confirm the Web ACL is associated with the distribution and managed rules produce expected metrics.
- Test benign and intentionally blocked requests in a controlled environment.

### Backup
- Confirm backup plan, selection, schedule, retention, and vault.
- Confirm a backup job completes and a recovery point is created.
- Perform a restore test in an appropriate test environment and document the outcome.

## 13. Failure Testing

- **NAT Gateway or route issue:** Private-instance outbound connectivity may fail; CloudFront-to-EC2 traffic does not use NAT. Restore any modified route after testing.
- **EC2 failure:** Requests may fail if there is no alternate healthy origin or instance.
- **CloudFront/VPC Origin failure:** The public entry point or origin path may be unavailable even if EC2 remains running.
- **WAF misconfiguration:** Legitimate requests may be blocked; monitor metrics and maintain a rollback plan.
- **Backup restore failure:** A recovery point may not meet recovery objectives; investigate the restore workflow and data dependencies.

Do not describe a NAT Gateway as something that can simply be stopped like an EC2 instance. If testing changes routes or removes resources, document restoration steps.

## 14. Security, Availability, and Cost

This POC is not a multi-AZ high-availability design. Production review may consider multiple Availability Zones, redundant application instances, load balancing, NAT design per Availability Zone, VPC endpoints, centralized logging, CloudTrail, GuardDuty, Security Hub, AWS Config, KMS key strategy, least-privilege security groups, and automated disaster-recovery tests.

Potential costs include EC2, EBS, NAT Gateway, Elastic IP, data transfer, CloudFront, AWS WAF, CloudWatch, and AWS Backup recovery points. Confirm current resources and delete only those safe to remove after validation. Do not delete the existing/shared VPC as part of POC cleanup.

## 15. Cleanup Considerations

Use CloudFormation to remove resources where practical, respecting stack dependencies. Before deleting:
- Remove or update CloudFront resources that depend on the origin or Web ACL.
- Check whether backup vaults or recovery points are retained and must be preserved.
- Check whether EC2 termination protection is enabled; it may need to be disabled before stack deletion.
- Verify each resource ID and region before a destructive command.
- Preserve shared VPC and public subnet resources unless their owner explicitly approves deletion.

For an EC2 instance with termination protection enabled, disable it only after verifying the instance ID:

```bash
aws ec2 modify-instance-attribute \
  --instance-id <instance-id> \
  --no-disable-api-termination \
  --region ap-south-1
```

This changes the instance setting; it does not delete the instance.

## 16. Reusability and Configuration

Templates should accept environment-specific values through parameters or configuration files rather than hardcoding live resource IDs. Typical network parameters include existing VPC ID, public subnet ID, Availability Zone, private subnet CIDR, and whether NAT should be enabled.

For each new account or region:
- Verify the target VPC and subnet design.
- Check CIDR overlap and route requirements.
- Confirm required permissions and CloudFormation capabilities.
- Deploy regional resources in the correct region.
- Create CloudFront-scoped WAF resources in `us-east-1`.
- Verify exports, imports, and Web ACL ARN handoffs.
- Review costs, tags, and security controls.

## 17. Repository Documentation

Keep these links aligned with the actual repository files:

- [README](../README.md)
- [CloudFormation Deployment Guide](cloudformation-deployment-guide.md)
- [Manual Deployment Guide](manual-deployment-guide.md)
- [Troubleshooting](troubleshooting.md)
- [Validation Checklist](validation-checklist.md)

The five templates are expected under `cloudformation/`: `01-network.yaml`, `02-compute.yaml`, `03-cloudfront.yaml`, `04-waf.yaml`, and `05-backup.yaml`. Verify the repository tree and update this section if files are located elsewhere.

## 18. Document Maintenance

Update this document when the network layout, templates, security controls, CloudFront configuration, WAF rules, backup schedule, monitoring, or deployment process changes. Keep POC-specific IDs and historical deployment status clearly labeled. Never commit credentials, access keys, private keys, passwords, tokens, or configuration files containing secrets.

## 19. Architecture Summary

The intended design keeps the application EC2 instance private, exposes the application through CloudFront and a CloudFront VPC Origin, applies AWS WAF at the CloudFront distribution, uses NAT for private-subnet outbound connectivity when enabled, uses Systems Manager for administration, CloudWatch for monitoring, and AWS Backup for scheduled recovery points.

Consider the design fully validated only after any CloudFront account restriction is resolved, the end-to-end application path is tested, WAF association is confirmed, monitoring is verified, and a backup restore test succeeds.
