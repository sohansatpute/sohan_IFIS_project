# IFIS POC – AWS Validation Checklist

## 1. Purpose and rules

Use this checklist after deployment to record whether each requirement has been verified. It covers networking, EC2, SSM, CloudWatch, CloudFront, WAF, AWS Backup, end-to-end behavior, failure scenarios, cost, and cleanup.

**Do not mark a check as passed because the resource merely exists.** Run the test and record evidence. Use `Not tested`, `Pass`, `Fail`, or `Blocked` as appropriate. Resource IDs, CIDRs, Regions, and managed prefix-list IDs are POC-specific; verify them before reuse in another account or Region.

## 2. Test record

- **Environment/account:** use a safe account alias or last four digits; avoid publishing full account IDs.
- **Test date/time:** `YYYY-MM-DD HH:MM TZ`
- **Tester:**
- **Primary Region:** `ap-south-1` unless intentionally changed.
- **CloudFront-scoped WAF Region:** `us-east-1`.
- **Stack names:**
- **CloudFront distribution ID/domain:**
- **Known restrictions/blockers:**

For each test, record status and evidence (command output, console state, metric timestamp, HTTP status, or backup job ID). Redact sensitive identifiers before sharing logs.

## 3. Suggested validation order

1. Confirm AWS identity, Region, and stack status.
2. Validate network, subnets, and route tables.
3. Validate EC2, storage, and security groups.
4. Validate User Data, HTTP service, SSM, and CloudWatch.
5. Validate CloudFront VPC Origin and distribution.
6. Validate WAF and its distribution association.
7. Validate backup job and restore.
8. Run end-to-end checks and approved failure scenarios.
9. Review security, cost, cleanup, and documentation.

Actual CloudFormation order depends on template dependencies and exports/imports. Confirm these before relying on a universal order. CloudFront-scoped WAF is deployed in `us-east-1`; the main network/compute resources in this POC are documented in `ap-south-1`.

## 4. Identity and Region

- [ ] **Status: Not tested** — `aws sts get-caller-identity` confirms the intended account/role.
- [ ] **Status: Not tested** — Main infrastructure Region is correct (`ap-south-1` unless changed).
- [ ] **Status: Not tested** — CloudFront-scoped WAF uses `us-east-1` and `Scope: CLOUDFRONT`.

Evidence/notes:

## 5. VPC and subnet validation

Documented POC reference values; verify them against the target account:
- Existing VPC: `vpc-06900f62513eff63`
- VPC CIDR: `172.31.0.0/16`
- Public subnet for NAT: `subnet-0d6445bd9f644383b`
- Public subnet CIDR: `172.31.0.0/20`, AZ `ap-south-1b`
- Intended private subnet CIDR: `172.31.48.0/20`, AZ `ap-south-1b`

- [ ] **Status: Not tested** — Intended VPC exists and is available.
- [ ] **Status: Not tested** — VPC and subnet CIDRs match the approved design and do not overlap.
- [ ] **Status: Not tested** — Public subnet is in the intended VPC/AZ and its route table has `0.0.0.0/0 → Internet Gateway`.
- [ ] **Status: Not tested** — Private subnet exists in the intended VPC/AZ.
- [ ] **Status: Not tested** — Private subnet auto-assign public IPv4 is disabled.
- [ ] **Status: Not tested** — Private subnet is associated with the intended private route table.
- [ ] **Status: Not tested** — Private subnet has no direct default route to the Internet Gateway.

Evidence/notes:

## 6. NAT Gateway and routing

- [ ] **Status: Not tested** — NAT Gateway is in the intended public subnet, has an Elastic IP, and state is `available`.
- [ ] **Status: Not tested** — Private route table has `0.0.0.0/0 → NAT Gateway` when NAT is enabled.
- [ ] **Status: Not tested** — Public route table has `0.0.0.0/0 → Internet Gateway`.
- [ ] **Status: Not tested** — Private EC2 can reach required outbound destinations.
- [ ] **Status: Not tested** — If NAT is disabled, the alternative required path (such as VPC endpoints) is configured and tested.

NAT enables outbound connections; it does not make private EC2 directly reachable by unsolicited inbound internet traffic. Do not remove routes during testing unless an approved test window and recovery plan exist.

Evidence/notes:

## 7. EC2 instance and storage

- [ ] **Status: Not tested** — EC2 is in the intended private subnet, running, and passes status checks.
- [ ] **Status: Not tested** — EC2 has no public IPv4 address.
- [ ] **Status: Not tested** — Instance type matches the approved POC parameters.
- [ ] **Status: Not tested** — Root volume is encrypted, uses `gp3`, and is 20 GiB unless intentionally changed.
- [ ] **Status: Not tested** — IMDSv2 is required.
- [ ] **Status: Not tested** — Termination protection matches the approved configuration.
- [ ] **Status: Not tested** — Intended IAM role and instance profile are attached.

Evidence/notes:

## 8. Security group and access control

The source notes list CloudFront origin-facing managed prefix list `pl-9aa247f3`. Verify the correct prefix list ID for the target account/Region before relying on it.

- [ ] **Status: Not tested** — Intended security group is attached to EC2.
- [ ] **Status: Not tested** — TCP/80 ingress is restricted to approved source(s).
- [ ] **Status: Not tested** — CloudFront managed prefix list was verified for this deployment.
- [ ] **Status: Not tested** — Any VPC-CIDR HTTP rule exists only if required by the approved design.
- [ ] **Status: Not tested** — No unnecessary inbound SSH rule exists.
- [ ] **Status: Not tested** — Egress and network ACLs allow required connections and return traffic.
- [ ] **Status: Not tested** — EC2 administration does not depend on a public IP.

Do not open TCP/80 to `0.0.0.0/0` as a troubleshooting shortcut. Any temporary exception must be approved, time-limited, and removed.

Evidence/notes:

## 9. User Data and web service

- [ ] **Status: Not tested** — User Data/cloud-init completed without unresolved errors.
- [ ] **Status: Not tested** — Required packages and agents installed successfully.
- [ ] **Status: Not tested** — `amazon-ssm-agent` is running.
- [ ] **Status: Not tested** — `amazon-cloudwatch-agent` is running.
- [ ] **Status: Not tested** — HTTP service (`httpd` in the documented POC) is running.
- [ ] **Status: Not tested** — `curl -I http://localhost` or equivalent local request succeeds.
- [ ] **Status: Not tested** — Expected test page/content is returned locally.

If bootstrap downloads packages, NAT or an equivalent network path must be available during initialization.

Evidence/notes:

## 10. Systems Manager Session Manager

- [ ] **Status: Not tested** — Instance appears as a managed node in the correct Region and is `Online`.
- [ ] **Status: Not tested** — Required instance permissions are present, including `AmazonSSMManagedInstanceCore` or an approved equivalent.
- [ ] **Status: Not tested** — SSM Agent is active and required endpoints are reachable through NAT or suitable VPC endpoints.
- [ ] **Status: Not tested** — Session Manager shell opens without SSH, a public IP, or a bastion host.

Evidence/notes:

## 11. CloudWatch metrics and alarms

The source notes mention `IFIS/EC2` and `CWAgent`. Verify the actual namespace, metric names, and dimensions against the agent configuration and published metrics.

- [ ] **Status: Not tested** — CloudWatch Agent is running.
- [ ] **Status: Not tested** — Expected metric namespace is visible.
- [ ] **Status: Not tested** — Memory utilization metric is publishing.
- [ ] **Status: Not tested** — Root filesystem disk utilization metric is publishing.
- [ ] **Status: Not tested** — Metric dimensions match configured alarms.
- [ ] **Status: Not tested** — Metrics arrive at the expected 300-second interval.
- [ ] **Status: Not tested** — Required CPU, instance status, memory, and disk alarms target the intended instance/metrics.
- [ ] **Status: Not tested** — Alarm state and missing-data behavior have been reviewed.

For disk monitoring, verify the actual metric name and path/dimensions (for example, `disk_used_percent` for `/`). Do not mark an alarm passed based only on its existence.

Evidence/notes:

## 12. CloudFront VPC Origin

- [ ] **Status: Not tested** — VPC Origin exists and is deployed/ready.
- [ ] **Status: Not tested** — VPC Origin targets the intended private EC2 resource.
- [ ] **Status: Not tested** — Origin protocol/port match the template and web service.
- [ ] **Status: Not tested** — Security group permits intended origin-facing traffic.
- [ ] **Status: Not tested** — EC2 does not require a public IP.

The documented POC uses HTTP port 80 from CloudFront to the origin. Viewer HTTPS does not by itself mean CloudFront-to-origin traffic uses HTTPS.

Evidence/notes:

## 13. CloudFront distribution

- [ ] **Status: Not tested** — Distribution exists, is enabled, and status is `Deployed`.
- [ ] **Status: Not tested** — Origin points to the intended VPC Origin.
- [ ] **Status: Not tested** — Viewer protocol policy redirects HTTP to HTTPS, as intended.
- [ ] **Status: Not tested** — Allowed methods match the approved configuration.
- [ ] **Status: Not tested** — Caching behavior matches the template (documented POC uses zero TTLs).
- [ ] **Status: Not tested** — HTTP versions and IPv6 settings match the intended design.
- [ ] **Status: Not tested** — Default CloudFront domain is reachable over HTTPS.
- [ ] **Status: Not tested** — Expected page/content is returned through CloudFront.

Evidence/notes:

## 14. End-to-end request test

Test `https://<cloudfront-domain>`.

- [ ] **Status: Not tested** — HTTPS viewer request succeeds.
- [ ] **Status: Not tested** — HTTP redirects to HTTPS if configured to do so.
- [ ] **Status: Not tested** — Expected application/test page is returned.
- [ ] **Status: Not tested** — Request traverses the intended distribution and private origin.
- [ ] **Status: Not tested** — Errors are traced to CloudFront, WAF, origin/network, or application based on evidence.

Expected path: `Client → CloudFront (WAF association) → VPC Origin → private EC2 → HTTP service`

Evidence/notes:

## 15. AWS WAF

CloudFront-scoped WAF should use `Scope: CLOUDFRONT` in `us-east-1`.

- [ ] **Status: Not tested** — Web ACL exists in `us-east-1` with scope `CLOUDFRONT`.
- [ ] **Status: Not tested** — Web ACL is associated with the intended distribution.
- [ ] **Status: Not tested** — Association appears in the deployed distribution configuration.
- [ ] **Status: Not tested** — Default action and managed-rule behavior match the approved POC design.
- [ ] **Status: Not tested** — Expected managed rule groups are present:
  - `AWSManagedRulesCommonRuleSet`
  - `AWSManagedRulesKnownBadInputsRuleSet`
  - `AWSManagedRulesLinuxRuleSet`
  - `AWSManagedRulesSQLiRuleSet`
  - `AWSManagedRulesAmazonIpReputationList`
- [ ] **Status: Not tested** — Sampled requests and CloudWatch metrics are visible where enabled.
- [ ] **Status: Not tested** — Full WAF logging is separately configured if required.

A Web ACL existing in AWS does not prove it protects the distribution. Sampled requests/metrics do not by themselves prove full request logging is configured.

Evidence/notes:

## 16. AWS Backup plan and job

Documented schedule: `cron(0 15 ? * SUN *)` = 15:00 UTC Sunday (20:30 Sunday IST; 00:00 Monday JST). Confirm this is the intended schedule.

- [ ] **Status: Not tested** — Backup vault and plan/rule exist.
- [ ] **Status: Not tested** — EC2 is included in the backup selection.
- [ ] **Status: Not tested** — Backup IAM role and permissions are correct.
- [ ] **Status: Not tested** — Schedule matches approved requirements.
- [ ] **Status: Not tested** — Retention is 30 days unless intentionally changed.
- [ ] **Status: Not tested** — Backup job completed successfully.
- [ ] **Status: Not tested** — Recovery point exists for the intended EC2 resource.
- [ ] **Status: Not tested** — Vault retention and any `Retain` deletion policy implications are understood.

Evidence/notes:

## 17. Backup restore test

A recovery point existing is not proof that restore works.

- [ ] **Status: Not tested** — Recovery point is available.
- [ ] **Status: Not tested** — Restore job completes successfully.
- [ ] **Status: Not tested** — Restored resource launches with appropriate network/security settings.
- [ ] **Status: Not tested** — SSM works on the restored instance where configured.
- [ ] **Status: Not tested** — Restored application/service passes its health check.
- [ ] **Status: Not tested** — Restored test resources are documented and cleaned up when no longer needed.

Evidence/notes:

## 18. Controlled failure scenarios

Run only in an approved test window, with recovery steps ready. Do not attempt to stop a NAT Gateway; if testing the NAT path, use an approved reversible route change or a separate test environment.

### A. NAT/outbound path
- [ ] **Status: Not tested** — Approved test demonstrates expected outbound impact when no alternative endpoint exists.
- [ ] **Status: Not tested** — Original route is restored and outbound access re-verified.

### B. HTTP service
- [ ] **Status: Not tested** — If approved, stopping the service causes the expected origin/request failure.
- [ ] **Status: Not tested** — Service is restarted and local plus CloudFront access recovers.

### C. Security group
- [ ] **Status: Not tested** — If approved, removing the CloudFront origin-facing rule causes expected origin failure.
- [ ] **Status: Not tested** — Original rule is restored and end-to-end access recovers.

### D. EC2 instance
- [ ] **Status: Not tested** — If approved, stopping EC2 produces the expected origin failure.
- [ ] **Status: Not tested** — EC2 is started, status checks pass, service is healthy, and CloudFront access recovers.

Evidence, approval, test window, and rollback notes:

## 19. Security acceptance

- [ ] **Status: Not tested** — EC2 has no public IP.
- [ ] **Status: Not tested** — No unnecessary SSH ingress exists; administration uses SSM.
- [ ] **Status: Not tested** — Root volume is encrypted and IMDSv2 is required.
- [ ] **Status: Not tested** — Security groups permit only approved inbound sources/ports.
- [ ] **Status: Not tested** — Private subnet has no direct default route to an Internet Gateway.
- [ ] **Status: Not tested** — WAF association and intended rules have been verified.
- [ ] **Status: Not tested** — No temporary test rules/access remain.

Evidence/notes:

## 20. Cost and cleanup

Review expected charges for NAT Gateway and data processing, EC2/EBS, CloudFront, WAF, AWS Backup, CloudWatch, and Elastic IPs.

- [ ] **Status: Not tested** — Cost-impacting resources are identified.
- [ ] **Status: Not tested** — Required resources are intentionally retained; unused resources have a cleanup plan.
- [ ] **Status: Not tested** — Stack dependencies and deletion order are reviewed.
- [ ] **Status: Not tested** — Retained backup vaults/recovery points are accounted for.
- [ ] **Status: Not tested** — Termination protection and deletion blockers are understood.
- [ ] **Status: Not tested** — No resources were unintentionally removed from CloudFormation management.

The source checklist suggests deletion order: Backup, WAF, CloudFront, Compute, Network. Confirm actual dependencies before deletion. Do not delete a shared/existing VPC unless explicitly intended and approved.

Evidence/notes:

## 21. Optional custom-domain validation

Skip if using only the default `*.cloudfront.net` domain.

- [ ] **Status: Not tested** — ACM public certificate is `Issued` in `us-east-1`.
- [ ] **Status: Not tested** — ACM DNS validation CNAME is present and retained for renewal.
- [ ] **Status: Not tested** — CloudFront alternate domain name and certificate match the hostname.
- [ ] **Status: Not tested** — Distribution is `Deployed` after changes.
- [ ] **Status: Not tested** — DNS alias points to the correct distribution.
- [ ] **Status: Not tested** — HTTPS loads and certificate matches the hostname.
- [ ] **Status: Not tested** — AAAA alias is added only if IPv6 is enabled.

Evidence/notes:

## 22. Documentation and handover

- [ ] **Status: Not tested** — Architecture document reflects the deployed design.
- [ ] **Status: Not tested** — CloudFormation guide reflects actual parameters and dependencies.
- [ ] **Status: Not tested** — Manual deployment guide reflects actual settings.
- [ ] **Status: Not tested** — Troubleshooting guide reflects current evidence and known issues.
- [ ] **Status: Not tested** — Resource inventory and Regions are recorded securely.
- [ ] **Status: Not tested** — Limitations are documented and not described as resolved without evidence.
- [ ] **Status: Not tested** — Repository changes were reviewed and committed as intended.

Evidence/notes:

## 23. Final acceptance

Select one only after reviewing the evidence:

- [ ] **PASS** — All required acceptance tests passed.
- [ ] **PASS WITH KNOWN LIMITATIONS** — Required core tests passed; remaining limitations are documented and accepted.
- [ ] **FAILED** — One or more required tests failed.
- [ ] **PENDING / BLOCKED** — Required tests are incomplete or an external blocker prevents validation.

**Overall result: `PENDING` until evidence supports a different status.**

**Open failures/limitations:**
1.
2.
3.

**Approved by:**
**Date:**
**Evidence location:**

Record unresolved errors in `docs/troubleshooting.md`. Do not mark CloudFront or end-to-end validation as passed until the distribution is deployed and the request path has actually succeeded.
