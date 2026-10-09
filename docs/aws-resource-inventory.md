# IFIS AWS Resource Inventory and Cleanup Reference

**Project:** IFIS\
**Website:** https://ifis.sohansatpute.in\
**Purpose:** Record where project resources are deployed, how they
relate to one another, and what to check before cleanup or redeployment.

> **Important:** This is a working inventory based on project
> documentation and template details available when this file was
> prepared. It is not a live inventory of the AWS account. Confirm
> resource IDs, stack names, statuses, and regions in the AWS Console or
> CLI before deleting anything. Do not delete shared resources.

## 1. Region reference

  -------------------------------------------------------------------------
  Service/resource scope  Region to check         Notes
  ----------------------- ----------------------- -------------------------
  Main infrastructure     `ap-south-1` (Mumbai)   Confirm the actual
  (VPC, subnets, NAT                              deployment region and
  Gateway, EC2,                                   stack locations.
  CloudWatch, AWS Backup)                         

  CloudFront distribution Global service          CloudFront is a global
                                                  service; check it from
                                                  the CloudFront console.

  CloudFront-scoped AWS   `us-east-1` (N.         CloudFront-scope WAF
  WAF Web ACL             Virginia)               resources must be managed
                                                  in `us-east-1`.

  ACM certificate used by `us-east-1`             CloudFront viewer
  CloudFront, if any                              certificates must be in
                                                  `us-east-1`. The project
                                                  notes indicate a default
                                                  CloudFront certificate,
                                                  so confirm whether a
                                                  custom certificate
                                                  exists.

  Route 53 hosted zone /  Global service          Check the hosted zone and
  DNS record, if used                             the record for
                                                  `ifis.sohansatpute.in`;
                                                  confirm ownership before
                                                  changing it.
  -------------------------------------------------------------------------

## 2. CloudFormation stack inventory

The project documentation describes the following logical stack groups.
**Use the actual stack names shown in the AWS account**; names below are
stack roles, not a guarantee of the deployed stack name.

  -------------------------------------------------------------------------
  Stack purpose     Expected region   Resources /        Before deleting
                                      responsibilities   
  ----------------- ----------------- ------------------ ------------------
  Network           `ap-south-1`      Private subnet,    Confirm whether
                                      route table and    the VPC/public
                                      association,       subnet are shared.
                                      optional NAT       Do not delete
                                      Gateway and        shared networking.
                                      Elastic IP; uses   Check NAT and
                                      an existing VPC    route
                                      and public subnet  dependencies.

  Compute           `ap-south-1`      Private EC2        Check termination
                                      instance, security protection,
                                      group, IAM         backups,
                                      role/instance      application data,
                                      profile,           and whether
                                      CloudWatch alarms  CloudFront/VPC
                                      and related        networking depends
                                      instance           on this instance.
                                      configuration      

  CloudFront        Global service    CloudFront         Distribution
                                      distribution and   changes/deletion
                                      VPC Origin to the  may take time.
                                      private EC2 origin Check DNS and
                                                         origin
                                                         dependencies
                                                         first.

  WAF               `us-east-1`       CloudFront-scope   Confirm Web ACL
                                      Web ACL and        association with
                                      managed rules      the distribution.
                                                         Review
                                                         logging/metrics
                                                         and any other
                                                         distributions
                                                         using it.

  Backup            `ap-south-1`      AWS Backup plan,   Check recovery
                                      selection/role and points, retention,
                                      backup vault       and vault deletion
                                                         policy. A vault
                                                         may be retained
                                                         when its stack is
                                                         deleted.
  -------------------------------------------------------------------------

## 3. Known environment/network reference values

These values were recorded in project documentation and must be verified
against the live account before use.

  ----------------------------------------------------------------------------
  Item                    Recorded value               Verification required
  ----------------------- ---------------------------- -----------------------
  Primary AWS region      `ap-south-1`                 Confirm in the
                                                       console/CLI and for
                                                       each stack.

  Existing VPC ID         `vpc-06900f62513eff63`       Confirm the VPC belongs
                                                       to this project and is
                                                       not shared.

  VPC CIDR                `172.31.0.0/16`              Confirm in VPC details.

  Existing public subnet  `subnet-0d6445bd9f644383b`   Confirm subnet and
  ID                                                   route table.

  Public subnet CIDR      `172.31.0.0/20`              Confirm in subnet
                                                       details.

  Public subnet           `ap-south-1b`                Confirm in subnet
  Availability Zone                                    details.

  Intended private subnet `172.31.48.0/20`             Confirm the actual
  CIDR                                                 created subnet CIDR and
                                                       avoid overlap.

  CloudFront origin       `pl-9aa247f3` (recorded in   Verify the managed
  access prefix list      prior project notes)         prefix list ID in the
                                                       deployed security group
                                                       and current region. Do
                                                       not rely on this value
                                                       without checking.

  Website hostname        `ifis.sohansatpute.in`       Confirm DNS record and
                                                       CloudFront distribution
                                                       target.
  ----------------------------------------------------------------------------

## 4. Resource-by-resource live inventory

Complete this table from the AWS Console or CLI before any cleanup. Keep
IDs and ARNs in a secure project record; do not record credentials,
access keys, tokens, or secrets.

  ---------------------------------------------------------------------------------------------------
  Resource             Actual resource ID / name    Region/scope   CloudFormation     Status / notes
                                                                   stack              
  -------------------- ---------------------------- -------------- ------------------ ---------------
  Network stack        `FILL IN`                    `ap-south-1`   `FILL IN`          

  Compute stack        `FILL IN`                    `ap-south-1`   `FILL IN`          

  CloudFront           `FILL IN`                    Global         `FILL IN`          
  stack/distribution                                                                  
  ID                                                                                  

  WAF stack/Web ACL    `FILL IN`                    `us-east-1`    `FILL IN`          
  name or ARN                                                                         

  Backup stack/plan ID `FILL IN`                    `ap-south-1`   `FILL IN`          

  VPC                  `vpc-06900f62513eff63`       `ap-south-1`   Existing/shared;   Do not delete
                       (verify)                                    verify             until ownership
                                                                                      is confirmed

  Public subnet        `subnet-0d6445bd9f644383b`   `ap-south-1`   Existing/shared;   Do not delete
                       (verify)                                    verify             until ownership
                                                                                      is confirmed

  Private subnet       `FILL IN`                    `ap-south-1`   Network stack      

  NAT Gateway /        `FILL IN`                    `ap-south-1`   Network stack, if  Check costs and
  Elastic IP                                                       created            dependencies

  EC2 instance ID      `FILL IN`                    `ap-south-1`   Compute stack      

  EC2 security group   `FILL IN`                    `ap-south-1`   Compute stack      
  ID                                                                                  

  IAM role / instance  `FILL IN`                    `ap-south-1`   Compute stack      Confirm whether
  profile                                                                             names are
                                                                                      shared

  CloudWatch alarms /  `FILL IN`                    `ap-south-1`   Compute stack or   Check whether
  log groups                                                       other              retained or
                                                                                      shared

  AWS Backup vault /   `FILL IN`                    `ap-south-1`   Backup stack or    Check retention
  recovery points                                                  retained           and recovery
                                                                                      requirements

  DNS record / hosted  `FILL IN`                    Global         Usually managed    Do not remove
  zone                                                             separately         unless
                                                                                      intentionally
                                                                                      retiring the
                                                                                      hostname

  ACM certificate, if  `FILL IN`                    `us-east-1`    Verify             Do not delete
  any                                               for CloudFront                    if another
                                                                                      distribution
                                                                                      uses it
  ---------------------------------------------------------------------------------------------------

## 5. Safe pre-cleanup checklist

Before deleting or recreating anything:

-   [ ] Confirm the business owner approves taking the website offline.
-   [ ] Verify the current website and record the working CloudFront
    distribution ID and hostname.
-   [ ] List all CloudFormation stacks, their regions, statuses, and
    stack outputs.
-   [ ] Review each stack's **Resources** tab and identify resources not
    managed by CloudFormation.
-   [ ] Review stack dependencies, exports/imports, and references
    between stacks.
-   [ ] Confirm whether the VPC and public subnet are shared; never
    delete shared networking as routine cleanup.
-   [ ] Check EC2 termination protection and whether important data
    exists on the instance.
-   [ ] Check AWS Backup job status, recovery points, vault retention,
    and restore requirements.
-   [ ] Record CloudFront distribution ID, WAF Web ACL association, DNS
    target, and any certificate details.
-   [ ] Check NAT Gateway and Elastic IP resources so they are not
    unintentionally left behind or deleted while needed.
-   [ ] Check for retained resources, manually created resources, and
    resources with deletion policies.
-   [ ] Save the current templates, parameter files, stack outputs, and
    deployment commands.
-   [ ] Plan the stack deletion order from the actual dependency graph;
    do not rely on a generic order alone.
-   [ ] After cleanup, check each relevant region and the global
    CloudFront console for leftovers and ongoing charges.

## 6. Redeployment and post-deployment checks

After a planned redeployment:

-   [ ] Deploy network resources before resources that depend on them.
-   [ ] Deploy compute and confirm EC2 health, Systems Manager access,
    and web-server health.
-   [ ] Deploy/update CloudFront and wait until the distribution is
    deployed.
-   [ ] Deploy/update the WAF in `us-east-1` and verify it is associated
    with the intended distribution.
-   [ ] Verify DNS and HTTPS for `https://ifis.sohansatpute.in`.
-   [ ] Confirm CloudWatch alarms/metrics and AWS Backup configuration.
-   [ ] Record new IDs, ARNs, stack names, regions, outputs, and
    validation evidence in this document.
-   [ ] Mark the setup as validated only after the checks have actually
    passed.

## 7. Useful AWS CLI inventory commands

Run these from a terminal with the correct AWS profile and permissions.
They list resources; they do not delete anything.

List CloudFormation stacks in the primary region:

``` bash
aws cloudformation list-stacks \
  --region ap-south-1 \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE UPDATE_ROLLBACK_COMPLETE
```

List stacks in the CloudFront/WAF region:

``` bash
aws cloudformation list-stacks \
  --region us-east-1 \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE UPDATE_ROLLBACK_COMPLETE
```

Describe a known stack:

``` bash
aws cloudformation describe-stacks \
  --region ap-south-1 \
  --stack-name YOUR_STACK_NAME
```

List resources in a stack:

``` bash
aws cloudformation list-stack-resources \
  --region ap-south-1 \
  --stack-name YOUR_STACK_NAME
```

List EC2 instances in the primary region:

``` bash
aws ec2 describe-instances \
  --region ap-south-1 \
  --query "Reservations[].Instances[].[InstanceId,State.Name,VpcId,SubnetId,PrivateIpAddress]" \
  --output table
```

List NAT Gateways in the primary region:

``` bash
aws ec2 describe-nat-gateways \
  --region ap-south-1 \
  --query "NatGateways[].[NatGatewayId,State,VpcId,SubnetId]" \
  --output table
```

List CloudFront distributions (global):

``` bash
aws cloudfront list-distributions \
  --query "DistributionList.Items[].[Id,DomainName,Status,Enabled,Comment]" \
  --output table
```

List CloudFront-scope WAF Web ACLs in `us-east-1`:

``` bash
aws wafv2 list-web-acls \
  --scope CLOUDFRONT \
  --region us-east-1
```

> CLI commands require a configured AWS CLI identity and suitable
> permissions. Review output before taking action. Avoid placing
> credentials or secrets in this document.

## 8. Current status

**Documentation status:** Working inventory template prepared.\
**Live inventory status:** Pending verification in the AWS account.\
**Deletion status:** No resources should be deleted based on this
document alone. Verify actual resource IDs, ownership, dependencies,
retention policies, and approval first.

**Last reviewed:** `FILL IN DATE`\
**Reviewed by:** `FILL IN NAME`
