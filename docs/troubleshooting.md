# IFIS POC – AWS Troubleshooting Guide

## 1. Purpose and scope

Use this guide to isolate failures in CloudFormation deployment, networking, EC2 bootstrap, Systems Manager (SSM), CloudWatch, CloudFront, AWS WAF, and AWS Backup.

Troubleshoot one layer at a time. Capture the exact error and the first failing CloudFormation event before changing resources.

The public request path is:

`Client → CloudFront (with WAF inspection) → CloudFront VPC Origin → private EC2 → web application`

The instance's outbound path, when configured, is:

`Private EC2 → private route table → NAT Gateway → public subnet route → Internet Gateway`

For AWS service access, NAT may be replaced by suitable VPC endpoints where supported and configured.

> **Important:** Region IDs, resource IDs, stack names, and the CloudFront managed prefix-list ID shown below are POC-specific examples. Verify them in the target account and Region before running commands. These commands inspect resources; commands that change or delete resources are called out separately.

## 2. Start with a safe diagnostic sequence

1. Confirm the AWS identity and Region.
2. Identify the affected stack/resource and copy the exact error.
3. Read CloudFormation stack events and locate the first actual failure.
4. Check the resource's state, dependencies, route tables, security groups, and IAM permissions.
5. Make one targeted change only after the cause is understood.
6. Re-test the failing layer, then the end-to-end path.
7. Record the root cause, fix, and validation result.

Confirm the AWS identity:

```bash
aws sts get-caller-identity
```

List stacks in the main infrastructure Region:

```bash
aws cloudformation list-stacks \
  --region ap-south-1 \
  --stack-status-filter CREATE_IN_PROGRESS CREATE_FAILED CREATE_COMPLETE \
  UPDATE_IN_PROGRESS UPDATE_COMPLETE UPDATE_FAILED ROLLBACK_IN_PROGRESS \
  ROLLBACK_COMPLETE DELETE_IN_PROGRESS DELETE_FAILED
```

For CloudFront-scoped WAF, use `us-east-1` when inspecting that stack.

## 3. CloudFormation stack failed or rolled back

Inspect stack status:

```bash
aws cloudformation describe-stacks \
  --stack-name <STACK_NAME> \
  --region <REGION>
```

Inspect recent events:

```bash
aws cloudformation describe-stack-events \
  --stack-name <STACK_NAME> \
  --region <REGION>
```

Look for statuses such as `CREATE_FAILED`, `UPDATE_FAILED`, `ROLLBACK_IN_PROGRESS`, `ROLLBACK_COMPLETE`, and `DELETE_FAILED`. Find the earliest resource event that identifies the real failure; later rollback events may only be consequences.

Common causes include:
- A parameter value, Region, or resource ID is incorrect.
- A required export/import is missing.
- The deployment identity lacks permission.
- A resource already exists or conflicts with another resource.
- An AWS service/account restriction prevents resource creation.
- A dependent resource is not ready.

Validate the template before retrying:

```bash
aws cloudformation validate-template \
  --template-body file://cloudformation/<TEMPLATE_FILE>.yaml \
  --region <REGION>
```

Template validation checks basic template syntax/structure; it does not guarantee that AWS will permit creation or that all runtime dependencies are valid.

## 4. Network stack or private subnet problem

Inspect the VPC and subnet:

```bash
aws ec2 describe-vpcs \
  --vpc-ids <VPC_ID> \
  --region ap-south-1

aws ec2 describe-subnets \
  --subnet-ids <PRIVATE_SUBNET_ID> \
  --region ap-south-1
```

Confirm:
- The subnet belongs to the intended VPC.
- The CIDR does not overlap with existing VPC subnets or connected networks.
- The Availability Zone is correct.
- Public IP assignment is disabled for the private subnet.
- The intended route table is associated with the private subnet.

Inspect route-table associations and routes:

```bash
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=<PRIVATE_SUBNET_ID>" \
  --region ap-south-1
```

If no route table is returned, inspect the VPC's main route table and the explicit subnet association. Do not assume a route-table ID from an old deployment.

## 5. Private EC2 has no outbound connectivity

If the design uses a NAT Gateway, the private route table should normally have:

`0.0.0.0/0 → NAT Gateway`

Inspect the route table:

```bash
aws ec2 describe-route-tables \
  --route-table-ids <PRIVATE_ROUTE_TABLE_ID> \
  --region ap-south-1
```

Inspect NAT Gateway state:

```bash
aws ec2 describe-nat-gateways \
  --nat-gateway-ids <NAT_GATEWAY_ID> \
  --region ap-south-1
```

The expected ready state is `available`. If the NAT Gateway is still `pending`, wait and recheck. If it is `failed`, inspect the public subnet, Elastic IP allocation, and Internet Gateway route.

Check all of the following:
1. EC2 is in the intended private subnet.
2. The subnet is associated with the expected route table.
3. The private route table's default route targets the correct NAT Gateway.
4. The NAT Gateway is in a public subnet and has an Elastic IP.
5. The public subnet route table has `0.0.0.0/0 → Internet Gateway`.
6. Security-group egress and network ACL rules permit the traffic and return path.
7. DNS resolution works if the failure involves hostnames.

A NAT Gateway is for outbound connectivity; it does not make the private EC2 instance directly reachable from the internet. If NAT is intentionally disabled, bootstrap, SSM, and monitoring need another supported path, such as the required VPC endpoints and endpoint security rules.

## 6. EC2 is running but SSM shows Offline

Check:
- The expected IAM instance profile is attached.
- The role includes the required Systems Manager permissions, such as `AmazonSSMManagedInstanceCore`.
- SSM Agent is installed and running.
- The instance can reach the Systems Manager endpoints through NAT or suitable VPC endpoints.
- The instance and console/CLI are using the intended Region.

From an existing valid shell/session on the instance:

```bash
sudo systemctl status amazon-ssm-agent
sudo systemctl start amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
```

Use `start` or `enable` only if appropriate; capture service output first when diagnosing.

Do not add SSH ingress as the first response to an SSM issue. Identify the IAM, agent, DNS, endpoint, or outbound-network problem first.

## 7. CloudWatch Agent or custom metrics are missing

Check the service:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

Inspect the configuration and logs:

```bash
sudo cat /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
sudo tail -n 100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

Common causes:
- Agent installation or startup failed during User Data.
- The configuration is invalid.
- The instance role lacks required CloudWatch permissions.
- Outbound connectivity or DNS is unavailable.
- The metric namespace, metric name, or dimensions do not match the alarm configuration.

For memory and disk alarms, verify that the expected custom metrics actually appear in CloudWatch and that the alarm dimensions match the published metrics. For disk metrics, check the configured path (for example `/`) and the metric name (commonly `disk_used_percent`); do not assume the alarm is correct until the metric is visible.

## 8. EC2 User Data did not complete

Inspect cloud-init logs from a valid shell/session:

```bash
sudo tail -n 200 /var/log/cloud-init-output.log
sudo tail -n 200 /var/log/cloud-init.log
```

Look for:
- Package repository/download failures.
- DNS or outbound-network failures.
- Permission errors.
- Shell syntax or command errors.
- Service startup failures.

The POC bootstrap installs packages and configures the web server, CloudWatch Agent, and SSM-related components. Package installation depends on network access to the required repositories/endpoints. Fix the underlying error before rerunning bootstrap; repeated commands may not be safe if the script is not idempotent.

## 9. HTTP service is not working on EC2

Check and test the web service locally:

```bash
sudo systemctl status httpd
curl -I http://localhost
```

If it is stopped, review its logs and then start it if appropriate:

```bash
sudo systemctl start httpd
sudo systemctl enable httpd
```

If localhost fails, troubleshoot the service, configuration, files, and logs before investigating CloudFront. If localhost works but CloudFront fails, continue with the origin, VPC Origin, security group, distribution, and WAF checks below.

## 10. Security group or CloudFront origin connectivity failure

The documented POC expects inbound HTTP on TCP port 80 from the CloudFront origin-facing managed prefix list. The POC has used `pl-9aa247f3`, but verify the correct prefix list for the target Region/account and confirm that the rule is attached to the security group used by the EC2 instance.

Inspect the security group:

```bash
aws ec2 describe-security-groups \
  --group-ids <SECURITY_GROUP_ID> \
  --region ap-south-1
```

Check:
- The EC2 instance uses the expected security group.
- Inbound TCP/80 permits the intended CloudFront origin-facing source.
- Outbound rules allow the required traffic.
- Network ACLs permit both the request and return traffic.
- The origin is configured for the port on which the application listens.

Do not open port 80 to `0.0.0.0/0` as a troubleshooting shortcut unless that exposure is explicitly approved and temporary; restore the intended restricted rule immediately if an exception is authorized.

## 11. CloudFront cannot reach the private EC2 origin

Check in this order:

1. EC2 is running and has passed its status checks.
2. The web service is running and `curl -I http://localhost` succeeds.
3. The EC2 security group allows the expected CloudFront managed prefix list on the origin port.
4. The VPC Origin exists and has reached its deployed/ready state.
5. The VPC Origin targets the intended private resource.
6. The distribution's origin configuration points to that VPC Origin.
7. The distribution is enabled and deployed.
8. The origin protocol/port matches the template and web service.
9. WAF is not blocking the request.

The described POC template uses HTTPS from viewer to CloudFront but HTTP on port 80 from CloudFront to the origin. Do not assume origin-side HTTPS is configured unless the template and origin actually establish it.

## 12. CloudFront VPC Origin creation fails because of account verification

A previously observed POC error was:

`Your account must be verified before you can add new CloudFront resources.`

This message points to an AWS account/service restriction and is not, by itself, evidence of a YAML syntax error.

If this error recurs:
1. Capture the full CloudFormation event and AWS error.
2. Check whether an AWS Support case is already open.
3. Provide the exact error and affected resource type (`AWS::CloudFront::VpcOrigin`) to AWS Support.
4. Wait for AWS to confirm that resource creation is enabled.
5. Retry only after the account/service restriction is resolved.

This was a historical POC issue. Verify the current AWS Support case/status before describing it as still pending or resolved.

## 13. CloudFront distribution deployment is delayed or fails

Check:
- Distribution status and whether it is enabled.
- Distribution configuration and origin ID.
- VPC Origin state.
- Web ACL ARN/association if WAF is intended.
- CloudFormation events and service error details.
- Whether account-level CloudFront restrictions remain.

CloudFront configuration changes can take time to propagate. Avoid repeatedly deleting and recreating resources without evidence that recreation is necessary.

## 14. CloudFront returns HTTP 403

Possible sources include WAF, CloudFront behavior/configuration, origin authorization, or the application itself.

1. Inspect WAF sampled requests and rule metrics for a matching block.
2. Check the distribution's Web ACL association.
3. Confirm the distribution is enabled and the request uses the expected hostname/path.
4. Check allowed methods and viewer protocol policy.
5. Verify origin reachability and security-group rules.
6. Compare the response and logs to determine whether the 403 came from CloudFront/WAF or the origin.

If a managed WAF rule is responsible, inspect the matching rule and request before changing rule actions. Do not disable the entire Web ACL merely to make the test pass.

## 15. CloudFront returns HTTP 502 or 503

Potential causes include:
- EC2 is stopped or unhealthy.
- The web service is stopped or not listening on the configured port.
- The VPC Origin is unavailable or not deployed.
- The security group or network ACL blocks the origin request.
- The origin target, port, or protocol is incorrect.
- A service-side or distribution deployment issue exists.

Start with local service checks and then verify VPC Origin and distribution configuration. Record the timestamp, request path, HTTP status, and any request/error IDs shown in the response.

## 16. WAF stack fails or does not protect CloudFront

A CloudFront-scoped Web ACL must use `Scope: CLOUDFRONT` and be deployed through CloudFormation in `us-east-1`.

Check:
- The stack is deployed in `us-east-1`.
- The template sets `Scope: CLOUDFRONT`.
- The stack has the expected Web ACL output/ARN.
- The CloudFront distribution is associated with that ARN.
- The association is deployed and the WAF metrics/sampled requests are being reviewed.

A Web ACL existing in the account does not prove it is associated with the intended distribution. The documented template enables sampled requests/metrics but does not, by itself, establish that full WAF request logging has been configured.

## 17. AWS Backup stack or job fails

The documented Backup stack depends on the compute stack's exported EC2 ARN (`<ComputeStackName>-Ec2Arn`). Confirm that the expected export exists in the correct Region:

```bash
aws cloudformation list-exports --region ap-south-1
```

Check:
- The compute stack is complete and has the expected export.
- The Backup stack uses the correct export name and Region.
- The backup plan, selection, vault, and IAM role exist.
- The selected EC2 resource is the intended instance.
- The backup job's status and failure message in AWS Backup.

For restore failures, confirm that the recovery point is available and inspect the exact restore error, IAM permissions, target subnet, security group, and required network connectivity. Do not infer that a backup is recoverable solely because a recovery point exists; perform a restore test where practical.

## 18. CloudFormation export/import error

If CloudFormation says an export cannot be found, inspect the exports in the Region where the dependent stack is deployed:

```bash
aws cloudformation list-exports --region <REGION>
```

Confirm:
- The producing stack completed successfully.
- The output export name exactly matches the import.
- The dependent stack is deployed in the same Region as the export.
- The stack deployment order respects the dependency.

The expected order depends on the selected integration method. For the POC, network and compute must exist before dependent resources; CloudFront-scoped WAF is in `us-east-1`, while the main infrastructure is in `ap-south-1`. Confirm the actual template dependencies before relying on a single universal order.

## 19. AWS CLI and Git Bash parameter/path issues

Git Bash may transform arguments that begin with `/` into Windows-style paths. This can affect Systems Manager Parameter Store paths such as:

`/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`

If the template already supplies the intended AMI parameter default, omit the `AmiId` override rather than passing a path unnecessarily. If an override is required, use a quoting/argument approach appropriate to the installed AWS CLI and Git Bash version, then inspect the CloudFormation change set/events to confirm the actual parameter value.

## 20. EC2 termination protection blocks deletion

Termination protection is enabled in the documented POC compute configuration. Disable it only during intentional cleanup/testing and only after confirming the exact instance ID.

Inspect the instance first:

```bash
aws ec2 describe-instances \
  --instance-ids <INSTANCE_ID> \
  --region ap-south-1
```

If authorized to disable protection:

```bash
aws ec2 modify-instance-attribute \
  --instance-id <INSTANCE_ID> \
  --disable-api-termination \
  --region ap-south-1
```

Then retry the intended CloudFormation operation. This command changes the instance configuration; do not run it as a diagnostic-only step.

## 21. Stack is stuck in rollback or deletion

Inspect events:

```bash
aws cloudformation describe-stack-events \
  --stack-name <STACK_NAME> \
  --region <REGION>
```

Wait for the stack to reach a stable terminal state before deciding what to do next. `ROLLBACK_COMPLETE` stacks often need to be deleted before a fresh create using the same stack name. A `DELETE_FAILED` stack needs the underlying deletion blocker identified first.

Do not delete resources manually if CloudFormation still manages them, unless you understand and intentionally accept the resulting drift and recovery implications.

## 22. Useful diagnostic commands

Use the appropriate Region and exact resource identifiers.

```bash
# Confirm caller identity
aws sts get-caller-identity

# List stacks
aws cloudformation list-stacks --region ap-south-1

# Stack events
aws cloudformation describe-stack-events \
  --stack-name <STACK_NAME> --region <REGION>

# EC2 instances
aws ec2 describe-instances --region ap-south-1

# Subnets
aws ec2 describe-subnets --region ap-south-1

# Route tables
aws ec2 describe-route-tables --region ap-south-1

# NAT Gateways
aws ec2 describe-nat-gateways --region ap-south-1

# Security groups
aws ec2 describe-security-groups --region ap-south-1

# CloudFormation exports
aws cloudformation list-exports --region ap-south-1
```

For CloudFront-scoped WAF stack commands, specify `--region us-east-1`.

## 23. Troubleshooting record template

For each significant issue, record:

- **Date/time (with time zone):** `YYYY-MM-DD HH:MM TZ`
- **Component:** Network / EC2 / SSM / CloudWatch / CloudFront / WAF / Backup
- **Environment and stack:** stack name, Region, and relevant resource ID
- **Symptom:** what failed and how it was observed
- **Exact error:** copy the full error/event; redact account-sensitive details before sharing
- **Root cause:** confirmed cause, or `Not yet confirmed`
- **Resolution:** change made and who approved it
- **Validation:** exact retest and result
- **Status:** Open / Resolved / Known Limitation

## 24. Known limitation tracking

A previous POC deployment encountered an account-verification restriction while creating `AWS::CloudFront::VpcOrigin`. A support case was reported in the source notes, but this guide does not verify its current status.

Before delivery, update this section based on the latest evidence:
- **Open:** AWS still blocks resource creation; record the current case status.
- **Resolved:** CloudFront VPC Origin creation and distribution deployment have been successfully tested.
- **Not reproduced:** a new deployment no longer encounters the error, but the previous incident has not been formally closed.

Do not describe the CloudFront path as fully validated until it has been tested successfully in the intended account.

## 25. Final troubleshooting principle

Avoid changing multiple resources at once. Isolate the failing layer and make the smallest justified change.

For inbound application traffic:

`Client → CloudFront/WAF → VPC Origin → security group/network path → private EC2 → application`

For outbound instance traffic:

`EC2 → private route table → NAT Gateway or VPC endpoint → destination service`

After the fix, repeat the failing test and the end-to-end test, and record the result in `docs/validation-checklist.md`. Keep the POC status as `PENDING` until the required validation has actually been completed.
