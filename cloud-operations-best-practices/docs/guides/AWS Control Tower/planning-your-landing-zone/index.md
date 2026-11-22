---
sidebar_position: 1
---
# Planning and implementing your landing zone

## Enable regions to suit your business requirements

### Choose your most commonly used region as your Home Region

Although Control Tower can govern multiple regions, it must be enabled from a single, home region. Identify the region where you expect to run most of your workloads and designate this as your Control Tower Home Region. If you are using an existing instance of AWS Identity Center, your home region must be the same region in which AWS Identity Center is configured. 

The Control Tower home region houses key configuration items for your Landing Zone. The AWS Organization is created there, IAM Identity Center is enabled there, alongside S3 buckets for Cloudtrail data storage. AWS Config in the Audit account is also configured to aggregate findings into the home region.  


### Deny unused regions, govern all allowed regions

Control Tower provides the ability to deny use of most AWS regions and enable only the subset for your business needs. This reduces your attack surface, reduces the likelihood of workloads generating unnecessary cost and simplifies your governance and observability requirements.  

The [global region deny control](https://docs.aws.amazon.com/controltower/latest/userguide/region-deny.html) can be set when you create or update your landing zone. This works in conjunction with the Control Tower governed region list, i.e. if the region isn’t enabled for governance, it will be denied. To further restrict region usage for specific Organizational Unit (OU), you can also implement the [OU region deny control](https://docs.aws.amazon.com/controltower/latest/controlreference/ou-region-deny.html). Both of these controls are implemented using [Service Control Policies (SCP)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html).  If a region is not denied, users can deploy resources to it, subject to IAM permissions. Ensure there are no resources in use in a region before denying it to avoid impact to your workloads.

The Control Tower Home Region is governed by default and cannot be ungoverned.

The Control Tower region-deny SCPs include exceptions that Control Tower requires to function. 

## Use AWS Identity Center to simplify access control

It is an AWS best practice to avoid use of IAM Users and instead to require [identity federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#bp-users-federation-idp) for granting human access to AWS resources. This mitigates much of the risk of credential compromise as you no longer need to use long-lived AWS credentials. For centralized access management we recommend you use [AWS IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/getting-started.html) to manage access to your accounts and permissions within those accounts. 

Identity Center can be enabled in a single region and be available to users globally. If Identity Center is not enabled for your Organization, Control Tower will enable it for you in your Control Tower Home Region. If Identity Center is already enabled, it must be enabled in your Control Tower home region or the [pre-flight checks](https://docs.aws.amazon.com/controltower/latest/userguide/getting-started-prereqs.html) will fail.

AWS Identity Center supports Permission Sets, which can be assigned to accounts in your AWS Organization and serve as a template for the creation of IAM roles in those accounts. When you associate an Identity Center user or group with a particular permission set in a particular account, it allows that user or group to assume the Permission Set defined role in that account. If you allow Control Tower to manage Identity Center, it will create some [preconfigured groups and permission sets](https://docs.aws.amazon.com/controltower/latest/userguide/sso-groups.html) and assign these to accounts to give you a foundation for user access. 


### Integrate your corporate identity provider

Identity Center can be used to manage users and groups but if you have an existing corporate identity provider, you should [connect it to Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/tutorials.html) to maintain a single source of truth for your identities. 

If you are using federated users and you want to make use of the default group and permission set configuration Control Tower sets up in Identity Center, you can create groups with the same names in your upstream provider and sync them to Identity Center. You can then assign users to these groups in identity provider to give them access to your enrolled accounts

### Work towards least privilege access 

The default Permission Sets Control Tower creates are designed for common use cases like **AdministratorAccess** and **DeveloperAccess**. For production workloads, particularly those involving sensitive data or other situations where security and compliance are critical concerns, best practice dictates you reduce permissions to the least access necessary. This can be achieved by using custom permission sets to specifically grant the required permissions and / or by applying service control policies to deny unnecessary access. [AWS IAM Access Analyzer](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html) can help to identify necessary permissions, remove unused permissions and write least privilege policies.


### Enable a Delegated Administrator Account

Control Tower enables Identity Center in the Organization management account. It is a best practice to minimize the need for anyone to have access to the management account as it controls the rest of your AWS Organization and cannot be constrained by preventative controls (SCP) to the same degree as member accounts. For this reason you should [enable a delegated administrator account for Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/delegated-admin.html). 

Permission sets deployed to the management account cannot be managed from the delegated administrator account, we recommend creating dedicated permission sets for the management account (for example MA_Administrator) which are only assumable by a highly restricted set of users.

### Apply additional constraints on Control Tower managed roles

Control Tower creates [various roles](https://docs.aws.amazon.com/controltower/latest/userguide/roles-how.html) in member accounts which can be assumed by AWS services. 

To protect against the [cross-service confused deputy](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html) problem you can define a [Resource Control Policy (RCP)](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps_examples.html) to prevent identities from outside of your AWS Organization tricking services into assuming roles on their behalf. 

You can also add Conditions to the Control Tower roles [to further restrict access](https://docs.aws.amazon.com/controltower/latest/userguide/conditions-for-role-trust.html), but be aware that any changes to these roles may be overwritten on landing zone updates.


## Protect your data with AWS Backup

The Control Tower [AWS Backup integration](https://docs.aws.amazon.com/controltower/latest/userguide/backup.html/) can help you set up a best practice backup solution with a backup vault in each member account, a central vault in a shared account and some standard backup policies (hourly, weekly, daily, monthly). Backup can be enabled at the OU level and individual resources can be tagged to target them for the relevant backup schedule. 

You can deploy additional backup plans to accounts as required, using your Control Tower customization method of choice ([AFC](https://docs.aws.amazon.com/controltower/latest/userguide/af-customization-page.html), [CfCT](https://docs.aws.amazon.com/controltower/latest/userguide/cfct-overview.html), [AFT](https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html), [StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html)). These can reuse the [aws-controltower-BackupRole](https://docs.aws.amazon.com/controltower/latest/userguide/backup-resources.html) role or you can create new roles as needed. 

If you have an existing backup solution you can opt out of this integration.


## Decide where your Config and CloudTrail logs are delivered

By default, AWS Control Tower creates and manages the Amazon S3 buckets that receive your AWS Config and AWS CloudTrail logs in the accounts you designate for those integrations. On Landing Zone 4.0, AWS Config moves to its own dedicated managed bucket. On upgrade, AWS CloudTrail keeps using its existing bucket; a separate CloudTrail bucket is created only when CloudTrail is first enabled on 4.0. On Landing Zone 4.0 or later, you can also bring your own S3 bucket for either integration instead of a managed one. The setup steps, bucket prerequisites, required bucket policies, and protective SCPs are documented in [Bring your own Amazon S3 bucket for log delivery](https://docs.aws.amazon.com/controltower/latest/userguide/custom-logging-bucket.html). Use this section to decide whether you should.

### Start with the managed buckets

For most landing zones, keep the Control Tower-managed buckets. Control Tower provisions them, covers them with its controls and SCPs, and keeps their policies correct across landing zone updates. The controls that protect these buckets attach to the OU that holds the integration accounts, so that protection applies there. They already support a customer-managed KMS key and a configurable retention period, so needing your own encryption key or retention schedule is not on its own a reason to bring your own bucket.

### When to bring your own bucket

Consider a custom bucket when you have a specific requirement the managed bucket cannot meet:

- A compliance or operational requirement to keep logs in a single, centralized bucket that your retention schedule, access reviews, and audit tooling are already built around. Validate that consolidating Config and CloudTrail into one bucket still meets your security and compliance posture before you do it.
- An existing central logging bucket that your SIEM or log lake already reads, where you want delivery to land there directly rather than repointing those consumers.
- A need to own the bucket's lifecycle, replication, or Object Lock configuration directly, outside Control Tower's management.

If your only goal is to get a copy of the logs into an existing bucket, S3 replication from the managed bucket is an option, but it carries a constraint worth knowing up front. The [mandatory Control Tower controls](https://docs.aws.amazon.com/controltower/latest/controlreference/mandatory-controls.html) that protect the managed buckets (`CTS3PV7` for the Config bucket and `CTS3PV8` for the CloudTrail bucket) deny `s3:PutReplicationConfiguration`, so you cannot add a replication rule the normal way. Configuring replication requires assuming the `AWSControlTowerExecution` role, which those controls exempt, as a break-glass step that bypasses the guardrail and should be used sparingly. If you need logs in a bucket you own without that trade-off, bring your own bucket so the integration delivers to it directly.

Whichever bucket you use, Control Tower delivers logs on the same standard key path, under your organization ID (`<org-id>/AWSLogs/...`). You choose the bucket, its owning account, and its KMS key, and that consistent layout keeps your existing log queries, Athena tables, and downstream tooling working without changes.

### What you take on with a custom bucket

- Control Tower controls do not cover a custom bucket. You own its bucket policy, encryption, S3 Block Public Access, lifecycle, replication, and ongoing protection. Apply the recommended SCPs from the docs to guard it.
- You maintain the delivery permissions over time. Removing them silently stops Config or CloudTrail log delivery.
- Switching the Config custom bucket later puts the affected accounts' baselines into source drift, which you clear by re-registering the OU or resetting the Config baseline. Switching only the CloudTrail bucket does not cause this drift.

### Replace every placeholder in the bucket policy

The required bucket policy template uses placeholder tokens for your bucket name and organization ID, and the organization ID appears twice in each delivery statement: as the Amazon S3 key prefix in the `PutObject` `Resource` and as the value of the `aws:SourceOrgID` condition. If you replace it in only one of the two, delivery is denied and the landing zone update fails with the delivery resource failing to stabilize, for example `AWS::Config::DeliveryChannel` returning `NotStabilized`. Replace every placeholder in both the Config and CloudTrail bucket policies, and if the bucket uses SSE-KMS, update the key policy so both AWS Config and AWS CloudTrail can use the key.

## Expand your AWS Organization Structure to suit business requirements

### Follow AWS Organizations multi-account best practices

In general, follow the AWS Organizations best practices relating to [multi-account strategy](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html) and design of your Organizational Units (OUs) when using Control Tower. Keep it simple - start with the OUs you need to support your differential governance, security and policy requirements and avoid deep nesting - Control Tower supports a maximum of five levels of nesting.  


### Do not modify or delete the Control Tower Security OU

One of the few limitations Control Tower enforces on your Organization is that you cannot create additional accounts or OUs under the Security OU and you can’t move or delete the Control Tower created accounts (log archive, audit) accounts without breaking your Control Tower environment.  


### Do not delete all OUs to leave only the Security OU

Control Tower expects to have at least two OUs, one of which must be the security OU. You can delete the Sandbox OU that is created when you enable Control Tower but only if you have at least one other OU in your Organization. 


