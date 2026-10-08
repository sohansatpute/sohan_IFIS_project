# IFIS POC – AWS Validation Checklist

## 1. Purpose

This checklist is used to validate the IFIS AWS POC after deployment.

The objective is to confirm that:

- The network architecture is correct.
- The private subnet is actually private.
- NAT Gateway provides outbound connectivity.
- The EC2 instance has no public IP.
- EC2 is reachable through AWS Systems Manager Session Manager.
- CloudWatch monitoring is working.
- CloudFront can reach the private EC2 through VPC Origin.
- AWS WAF is associated with CloudFront.
- AWS Backup protects the EC2 instance.
- Backup restore can be performed successfully.
- The complete architecture works end to end.

This checklist can be reused when deploying the architecture into another AWS account or Region.


## 2. Deployment Order

Validate the infrastructure in this order:

1. Network
2. Compute
3. CloudFront VPC Origin
4. CloudFront Distribution
5. WAF
6. AWS Backup
7. End-to-end connectivity
8. Monitoring
9. Failure scenarios
10. Backup restore


## 3. AWS Region Validation

### Primary Region

Expected:

- Region: `ap-south-1`
- Location: Mumbai

### CloudFront WAF Region

Expected:

- Region: `us-east-1`

Important:

AWS WAF for a CloudFront distribution must use:

`Scope = CLOUDFRONT`

and must be created in:

`us-east-1`


## 4. VPC Validation

### Check

Confirm the existing VPC is:

- VPC ID: `vpc-06900f62513eff63`
- CIDR: `172.31.0.0/16`

### Expected Result

- VPC exists.
- CIDR is correct.
- VPC is available.


## 5. Public Subnet Validation

### Check

Confirm the subnet used for the NAT Gateway is:

- Subnet ID: `subnet-0d6445bd9f644383b`
- CIDR: `172.31.0.0/20`
- Availability Zone: `ap-south-1b`

### Expected Result

- Subnet is associated with the public route table.
- Route table contains:
  - `0.0.0.0/0`
  - Target: Internet Gateway
- NAT Gateway can be created in this subnet.


## 6. Private Subnet Validation

### Check

Confirm the private subnet created by the POC is:

- Subnet CIDR: `172.31.48.0/20`
- Availability Zone: `ap-south-1b`
- Public IP assignment: Disabled

### Expected Result

- Subnet does not have a direct route to the Internet Gateway.
- Default route points to NAT Gateway.
- Resources launched in this subnet do not receive public IPv4 addresses automatically.


## 7. Private Route Table Validation

### Check

Confirm the private route table is associated with the private subnet.

Expected route:

`0.0.0.0/0 → NAT Gateway`

### Expected Result

The private subnet has outbound internet access through NAT Gateway.

There must not be:

`0.0.0.0/0 → Internet Gateway`

for the private subnet.


## 8. NAT Gateway Validation

### Check

Confirm the NAT Gateway is:

- In the public subnet.
- Associated with an Elastic IP.
- State: `Available`.

### Expected Result

Private EC2 can initiate outbound connections through the NAT Gateway.

NAT Gateway does not allow unsolicited inbound internet connections to the private EC2.


## 9. EC2 Validation

### Check

Confirm:

- Instance state: `Running`
- Instance is in the private subnet.
- Public IPv4 address: None.
- Private IPv4 address: Present.
- Instance type matches the POC configuration.
- Root volume is encrypted.
- Root volume type is gp3.
- Root volume size is 20 GB unless changed intentionally.
- IMDSv2 is required.
- Termination protection is enabled when appropriate.

### Expected Result

EC2 is not directly accessible from the public internet.


## 10. EC2 Security Group Validation

### Inbound Rules

Expected HTTP rule:

- Protocol: TCP
- Port: 80
- Source: CloudFront managed prefix list

CloudFront managed prefix list for the current POC:

`pl-9aa247f3`

Expected VPC rule:

- Protocol: TCP
- Port: 80
- Source: `172.31.0.0/16`

### Outbound Rules

Expected:

- All traffic allowed outbound.

### Expected Result

HTTP access is limited to the intended sources.

The EC2 instance does not require an inbound SSH rule for administration.


## 11. CloudFront Managed Prefix List Validation

Confirm the AWS managed prefix list:

`com.amazonaws.global.cloudfront.origin-facing`

is used by the EC2 security group.

Current POC prefix list:

`pl-9aa247f3`

### Expected Result

CloudFront-origin traffic can reach the EC2 security group.

Do not hardcode this prefix list ID when reusing the architecture in another Region/account without verifying the correct managed prefix list.


## 12. IAM Role Validation

EC2 instance role should include:

- `AmazonSSMManagedInstanceCore`
- `CloudWatchAgentServerPolicy`

### Expected Result

The EC2 instance can:

- Register with Systems Manager.
- Start a Session Manager session.
- Send CloudWatch metrics.


## 13. Systems Manager Validation

Open:

AWS Console → Systems Manager → Fleet Manager / Managed Nodes

### Expected Result

The EC2 instance appears as:

`Online`

### Session Test

Start a Session Manager session.

### Expected Result

A shell session opens without using:

- SSH
- Public IP
- Bastion host
- Key pair


## 14. EC2 User Data Validation

Confirm the User Data completed successfully.

Expected operations include:

- `dnf update -y`
- CloudWatch Agent installation
- jq installation
- unzip installation
- wget installation
- curl installation
- Timezone configuration
- SSM Agent enable/start
- CloudWatch Agent configuration
- CloudWatch Agent enable/start

### Expected Result

Both services are running:

`amazon-ssm-agent`

`amazon-cloudwatch-agent`

The EC2 instance must have outbound connectivity through NAT Gateway or equivalent VPC endpoints while performing the initial installation.


## 15. CloudWatch Validation

Confirm CloudWatch Agent is running.

Expected custom namespace:

`IFIS/EC2`

Expected metrics include:

- Memory utilization
- Root filesystem disk utilization

Expected collection interval:

`300 seconds`

### Expected Result

Metrics appear in CloudWatch.


## 16. Web Server Validation

Install a simple HTTP server for the POC if required.

Example:

`sudo dnf install -y httpd`

Start it:

`sudo systemctl enable --now httpd`

Create a simple test page:

`echo "IFIS POC EC2 Web Server" | sudo tee /var/www/html/index.html`

Test locally from the EC2 instance:

`curl http://localhost`

### Expected Result

The EC2 instance returns the test page.


## 17. CloudFront VPC Origin Validation

Confirm the VPC Origin:

- Exists.
- Status is deployed/available.
- Targets the intended EC2 resource.
- Uses HTTP port 80.
- Uses the EC2 private resource rather than a public IP.

### Expected Result

CloudFront is able to use the private EC2 as an origin.

The EC2 does not need a public IP.


## 18. CloudFront Distribution Validation

Confirm:

- Distribution is enabled.
- VPC Origin is configured.
- Viewer protocol redirects HTTP to HTTPS.
- HTTP/2 is enabled.
- HTTP/3 is enabled where configured.
- IPv6 setting matches the intended design.
- Caching behavior matches the POC configuration.
- Origin points to the VPC Origin.

### Expected Result

Opening the CloudFront distribution domain returns the EC2 web page.


## 19. CloudFront End-to-End Test

Test:

`https://<cloudfront-domain>`

### Expected Flow

Client

→ CloudFront

→ WAF

→ CloudFront VPC Origin

→ Private EC2

→ HTTP port 80


### Expected Result

The browser displays:

`IFIS POC EC2 Web Server`


## 20. WAF Validation

Confirm the Web ACL:

- Scope: `CLOUDFRONT`
- Region: `us-east-1`
- Associated with the CloudFront distribution.
- Default action: Allow.
- AWS Managed Rules are enabled.

Expected managed rule groups:

- AWSManagedRulesCommonRuleSet
- AWSManagedRulesKnownBadInputsRuleSet
- AWSManagedRulesLinuxRuleSet
- AWSManagedRulesSQLiRuleSet
- AWSManagedRulesAmazonIpReputationList

### Expected Result

CloudFront requests are inspected by AWS WAF before reaching the origin.


## 21. WAF Logging / Metrics Validation

Confirm:

- Sampled requests are enabled.
- CloudWatch metrics are enabled.
- WAF metrics are visible.

### Expected Result

WAF activity can be monitored.


## 22. AWS Backup Validation

Confirm:

- Backup vault exists.
- Backup plan exists.
- EC2 instance is selected.
- Backup IAM role exists.
- Weekly schedule is configured.
- Retention is 30 days unless intentionally changed.

Current schedule:

`cron(0 15 ? * SUN *)`

This corresponds to:

`15:00 UTC Sunday`

which is:

`00:00 Monday JST`


## 23. Backup Job Validation

Wait for the scheduled backup or initiate an on-demand backup for testing.

### Expected Result

Backup job status:

`Completed`

A recovery point appears in the backup vault.


## 24. Backup Restore Validation

Perform a restore test when practical.

### Expected Result

AWS Backup can restore the EC2 resource.

Validate:

- Instance is created successfully.
- Network configuration is appropriate.
- Security group is appropriate.
- Instance can start.
- SSM connectivity works if the required network/IAM configuration is present.

Document the restore result.


## 25. Failure Test – NAT Gateway

Test scenario:

Temporarily remove or disable the private subnet's NAT route.

### Expected Result

Private EC2 loses outbound internet/AWS service connectivity unless equivalent VPC endpoints are configured.

Potential impact:

- SSM connectivity can fail.
- CloudWatch Agent communication can fail.
- Package downloads can fail.

Restore the NAT route after testing.


## 26. Failure Test – EC2 HTTP Service

Stop the HTTP service.

Example:

`sudo systemctl stop httpd`

### Expected Result

CloudFront cannot successfully retrieve the web content from the origin.

Start the service again:

`sudo systemctl start httpd`

### Expected Result

CloudFront access recovers after the origin becomes healthy.


## 27. Failure Test – Security Group

Temporarily remove the CloudFront managed prefix-list HTTP rule.

### Expected Result

CloudFront should no longer be able to reach the EC2 origin.

Restore the rule after testing.


## 28. Failure Test – EC2 Instance

Stop the EC2 instance.

### Expected Result

CloudFront origin requests fail while the instance is stopped.

Start the instance again.

### Expected Result

After EC2 and the web service become healthy, CloudFront access should recover.


## 29. Security Validation

Confirm:

- EC2 has no public IP.
- No inbound SSH rule is required.
- Administration is through SSM.
- Root volume is encrypted.
- IMDSv2 is required.
- Security group allows only required inbound traffic.
- CloudFront is the public entry point.
- WAF protects CloudFront.
- Private subnet has no direct Internet Gateway route.


## 30. Cost Validation

Before leaving the POC running, review:

- NAT Gateway hourly cost.
- NAT data processing.
- EC2 instance cost.
- EBS volume cost.
- CloudFront usage.
- WAF usage.
- AWS Backup storage and backup activity.
- CloudWatch usage.

Delete resources when testing is complete if they are not required.


## 31. Cleanup Validation

After the POC is complete:

### CloudFormation

Delete stacks in dependency order:

1. Backup
2. WAF
3. CloudFront
4. Compute
5. Network

Note:

CloudFront/WAF deletion may require additional time.

### Manual Resources

Check for:

- Elastic IPs
- NAT Gateways
- VPC Origins
- CloudFront distributions
- Backup vaults
- IAM roles
- Security groups
- EC2 instances
- EBS volumes

Do not delete resources manually if they are still managed by CloudFormation unless intentional.


## 32. Final Acceptance Checklist

### Network

- [ ] Existing VPC identified.
- [ ] Public subnet identified.
- [ ] Private subnet created.
- [ ] Private route table created.
- [ ] NAT Gateway available.
- [ ] Private subnet has NAT default route.
- [ ] No direct Internet Gateway route from private subnet.

### Compute

- [ ] EC2 running.
- [ ] EC2 in private subnet.
- [ ] No public IP.
- [ ] Encrypted gp3 root volume.
- [ ] IMDSv2 required.
- [ ] Termination protection configured as required.
- [ ] SSM role attached.
- [ ] CloudWatch role attached.

### Security

- [ ] Security group configured.
- [ ] CloudFront managed prefix list configured.
- [ ] No unnecessary SSH access.
- [ ] WAF created in us-east-1.
- [ ] WAF associated with CloudFront.

### CloudFront

- [ ] VPC Origin created.
- [ ] VPC Origin deployed.
- [ ] CloudFront distribution created.
- [ ] CloudFront reaches private EC2.
- [ ] HTTPS viewer access works.

### Monitoring

- [ ] CloudWatch Agent running.
- [ ] Memory metrics available.
- [ ] Disk metrics available.
- [ ] EC2/CloudWatch alarms configured where required.

### Backup

- [ ] Backup vault created.
- [ ] Backup plan created.
- [ ] EC2 selected.
- [ ] Backup job completed.
- [ ] Recovery point exists.
- [ ] Restore test completed where practical.

### Documentation

- [ ] Architecture documented.
- [ ] Manual deployment documented.
- [ ] CloudFormation deployment documented.
- [ ] Validation checklist completed.
- [ ] Troubleshooting guide updated.
- [ ] Git repository updated.
- [ ] Final AWS resource inventory documented.

## 33. Validation Result

Overall POC status:

`PENDING`

Update this value after testing:

`PASS`

or

`PASS WITH KNOWN LIMITATIONS`

or

`FAILED`

Record any limitations or failed tests in:

`docs/troubleshooting.md`