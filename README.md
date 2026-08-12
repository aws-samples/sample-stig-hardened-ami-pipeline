# Automate STIG-hardened AMI pipelines with OpenSCAP assessment evidence and organization governance

## Overview

This CloudFormation template deploys an EC2 Image Builder pipeline that produces DISA STIG-hardened Amazon Linux 2023 AMIs. The pipeline applies the AWS-managed `stig-build-linux` component with configurable hardening levels, assesses the resulting image with OpenSCAP, stores assessment evidence in Amazon S3, and publishes the latest successfully gated AMI ID to AWS Systems Manager Parameter Store for CI/CD consumption.

Key capabilities:
- **Configurable STIG hardening level:** Select High, Medium, or Low for the AWS-managed `stig-build-linux` component. This parameter changes hardening scope; it does not select OpenSCAP rules or alter the fixed high-severity release gate.
- **Three compliance baselines:** During the test stage the pipeline runs three separate OpenSCAP scans — DISA STIG, CIS Level 1 (Server), and CIS Level 2 (Server) — and produces a separate report per baseline. The build gate is based on the STIG scan; the CIS reports are informational evidence.
- **Full, transparent scans and auditable gate decisions:** Every scan evaluates its complete profile with no rules skipped, so all results (including FIPS) appear in the reports. A concise `compliance-summary.txt` is generated alongside `gate-decision.json` for automation and `gate-decision.txt` for reviewers. The decision artifacts record the discovered STIG profile, scan exit code, evidence state, blocking and manual-review rule IDs, fixed FIPS exclusions, and final reasons.
- **Date- and image-keyed report storage:** Reports are grouped under `reports/<YYYY-MM-DD>/<ami-id>/<baseline>/`, using the UTC report-generation date with the AMI ID one level below. Objects remain tagged with the AMI ID and recipe version, preserving direct evidence-to-image traceability while making builds easy to browse chronologically.
- **Encrypted image storage:** The recipe encrypts the temporary build instance's root EBS volume with a rotating customer-managed KMS key, and the distribution configuration uses the same key for the published AMI snapshots. This is EBS-layer encryption, not in-guest LUKS or filesystem encryption.
- **IMDSv2 required on the build fleet:** The Image Builder infrastructure configuration sets `HttpTokens: required`, so build and test instances accept only token-based (IMDSv2) instance-metadata requests.
- **Encrypted build notifications:** An EventBridge rule publishes `AVAILABLE` and `FAILED` states to an SNS topic protected by a separate rotating customer-managed KMS key. The key policy limits the service publisher to the stack's rule and account.
- **Optional FIPS mode:** Configure operating-system FIPS mode with a single parameter. When enabled, the pipeline runs `fips-mode-setup --enable` and reboots before assessment; when disabled, the five fixed FIPS-related high-severity rules still appear in evidence but are excluded from the release-gate decision only. This setting does not by itself establish FIPS certification or validate every cryptographic module used by a workload.
- **Optional automatic triggering:** Set `EnableAutoTrigger=true` to rebuild when Amazon Linux 2023 base-image update notifications are published by AWS. Set it to `false` to use manual or customer-provided triggering only.
- **SSM parameter output:** Publish the latest AMI ID at `/ami/<PipelineName>/latest`.
- **Custom build workflow:** Skip Image Builder InventoryCollection to avoid SSM conflicts after hardening changes firewall behavior.
- **Report retention and access logging:** Store encrypted OpenSCAP reports in a versioned S3 bucket with lifecycle cleanup and server access logging to a separate encrypted, retained bucket. The destination policy accepts delivery only from the reports bucket in the same account.
- **Organization governance:** Include an AWS Organizations Declarative Policy example for restricting EC2 launches to approved pipeline AMIs.

## Architecture

![Architecture diagram showing the pipeline account, where EC2 Image Builder builds, hardens, scans, and publishes an encrypted AMI with evidence retained in Amazon S3, and the workload OU, where a declarative policy restricts instance launches to the pipeline's AMIs](ARCHITECTURE.drawio.png)

Each build moves through the following stages, matching the numbered steps in the diagram:

1. EC2 Image Builder launches a temporary build instance in your VPC with an encrypted root volume.
2. The AWS-managed `stig-build-linux` component applies STIG hardening.
3. Image Builder creates the encrypted AMI, then launches a temporary test instance from it.
4. On the test instance, the `openscap-scan` component installs OpenSCAP and runs the DISA STIG, CIS Level 1, and CIS Level 2 scans in full. Because the scanner is installed on the test instance, it is not included in the published AMI.
5. The same component computes a machine-readable gate decision (`gate-decision.json`) and a human-readable one (`gate-decision.txt`).
6. The `openscap-gate` component uploads the decision and the scan reports to the encrypted, versioned reports bucket, and records the outcome on the AMI as the `StigGateDecision` tag.
7. The gate enforces the uploaded decision: a missing profile, invalid scan output, or blocking high-severity STIG finding marks the image as failed, and the AMI is not distributed.
8. On a pass, Image Builder distributes the encrypted AMI, updates the `/ami/<PipelineName>/latest` parameter, notifies subscribers, and optionally shares the AMI with your organization.

The auto-trigger resources (the Lambda function and its SNS subscription to the Amazon Linux 2023 update topic) are created only when `EnableAutoTrigger=true`.

The diagram source is [ARCHITECTURE.drawio](ARCHITECTURE.drawio), editable at [draw.io](https://app.diagrams.net/).

## Prerequisites

1. **VPC subnet:** Network connectivity to the required AWS service endpoints (including SSM, Image Builder, and S3) and operating system package repositories through approved internet/NAT access, supported VPC endpoints, or customer-managed repositories, as appropriate.
2. **Security group:** Allows outbound HTTPS (port 443).
3. **IAM permissions:** Create IAM roles and instance profiles, customer-managed KMS keys and aliases, Image Builder resources (including optional lifecycle policies), Lambda functions, CloudWatch Logs log groups, EventBridge rules, S3 buckets, SNS topics/subscriptions, and pass roles.
4. **EC2 Image Builder:** Available in the target region.
5. **CloudFormation artifact bucket:** An existing S3 bucket in the deployment Region. The template exceeds CloudFormation's 51,200-byte direct body limit, so `aws cloudformation deploy` uploads it to this bucket and uses a template URL.
6. **Service quotas:** Sufficient Image Builder and EC2 quotas for builds.

## Parameters

CloudFormation displays these parameters in logical groups: **Pipeline and Build**, **Network**, **STIG and Compliance**, **Organization Sharing**, **Build Notifications**, and **AMI Lifecycle**. The parameter table above uses the same group order.

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `PipelineName` | `String` | No | `stig-al2023-arm64-pipeline` | S3-compatible name prefix, 3-31 lowercase letters, numbers, and hyphens. Must start and end with a letter or number. |
| `BaseImage` | `String` | No | `amazon-linux-2023-arm64` | Amazon Linux 2023 base image on `x86_64` or `arm64` (Graviton). This solution is built and tested for Amazon Linux 2023 only. |
| `InstanceType` | `String` | No | `m6g.large` | EC2 instance type. The default uses AWS Graviton. x86 base images require `m5`, `c5`, `r5`, or `t3`; arm64 base images require `m6g`, `c6g`, or `t4g`. |
| `EnableAutoTrigger` | `String` | No | `true` | Set to `true` to create the SNS/Lambda trigger that rebuilds when AWS publishes an Amazon Linux 2023 base-image update. Set to `false` to omit all trigger resources and use manual or customer-provided triggering. |
| `SubnetId` | `AWS::EC2::Subnet::Id` | Yes | | VPC subnet for the build instance. |
| `SecurityGroupId` | `AWS::EC2::SecurityGroup::Id` | Yes | | Security group for the build instance. |
| `STIGLevel` | `String` | No | `High` | Hardening scope passed to AWS-managed `stig-build-linux`: **High** (CAT I+II+III), **Medium** (CAT I+II), or **Low** (CAT I only). This does not select OpenSCAP rules or change the release-gate threshold. |
| `EnableFIPS` | `String` | No | `false` | Set to `true` to configure operating-system FIPS mode and reboot before assessment. When `false`, the five fixed FIPS-related high-severity rules remain in evidence but are excluded from the release gate. This setting is not a FIPS certification. |
| `OrganizationId` | `String` | No | *(empty)* | AWS Organization ID for AMI sharing, such as `o-abcdef1234`. Leave empty to skip organization sharing. |
| `ManagementAccountId` | `String` | No | *(empty)* | 12-digit AWS Organizations management account ID. Required when `OrganizationId` is set. |
| `NotificationEmail` | `String` | No | *(empty)* | Optional email to notify on build `AVAILABLE`/`FAILED`. Leave empty to create the SNS topic and EventBridge rule without a subscription (subscribe endpoints later). |
| `EnableAmiLifecycle` | `String` | No | `false` | Set to `true` to create an optional EC2 Image Builder lifecycle policy that acts on older AMIs so workload accounts can only use recent images. |
| `AmiLifecycleAction` | `String` | No | `DEPRECATE` | Action for AMIs older than `KeepLatestCount`: `DEPRECATE` (marks only, does not block launches), `DISABLE` (blocks new launches, reversible), or `DELETE` (deregisters, destructive). Use `DISABLE`/`DELETE` to enforce latest-only. |
| `KeepLatestCount` | `Number` | No | `2` | Number of most recent AMIs to retain unaffected (latest plus rollback headroom). |

## Deployment

Pass each non-default value from the parameter table above as a `Key=Value` entry under `--parameter-overrides`. At minimum, replace the subnet and security-group placeholders below. If you want organization-wide AMI sharing, also add `OrganizationId=<organization-id>` and `ManagementAccountId=<management-account-id>` overrides.

The template is larger than CloudFormation's 51,200-byte direct `TemplateBody` limit. Use an existing deployment-artifacts bucket in the same Region; the AWS CLI uploads the template and supplies CloudFormation with an S3 template URL. The bucket is separate from the two buckets created by this stack.

```bash
AWS_PROFILE="<aws-profile>"
AWS_REGION="<aws-region>"
DEPLOYMENT_BUCKET="<existing-cloudformation-artifact-bucket>"
STACK_NAME="stig-pipeline"

aws cloudformation deploy \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION" \
  --stack-name "$STACK_NAME" \
  --template-file template.yaml \
  --s3-bucket "$DEPLOYMENT_BUCKET" \
  --s3-prefix "$STACK_NAME" \
  --parameter-overrides \
    "SubnetId=<subnet-id>" \
    "SecurityGroupId=<security-group-id>" \
  --capabilities CAPABILITY_NAMED_IAM \
  --no-execute-changeset
```

This creates the change set without executing it. Review the generated change set in CloudFormation, then rerun the same command without the final `--no-execute-changeset` flag to deploy it. If the deployment bucket requires a customer-managed key, also provide `--kms-key-id <deployment-artifact-key-id>`. See the [AWS CLI `cloudformation deploy` reference](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/deploy.html).

> **Tip:** Deploy a separate stack for each architecture you need. Use a distinct `PipelineName` per stack, such as `stig-al2023-arm64-pipeline` and `stig-al2023-x86-pipeline`.

## Encryption and access logging

### EBS volumes and AMI snapshots

Image recipe version `1.0.0` creates a rotating customer-managed KMS key and uses it in two places:

1. `STIGImageRecipe.BlockDeviceMappings` encrypts the temporary build instance's root EBS volume before hardening begins. The template maps `/dev/xvda`, the Amazon Linux 2023 root device.
2. `DistributionConfig.AmiDistributionConfiguration.KmsKeyId` encrypts the published AMI snapshots with the same key.

The key and alias are retained if the stack is deleted because existing encrypted AMIs depend on the key. `ImageEncryptionKeyArn` exposes the exact ARN for workload-account IAM policies. Do not disable or schedule deletion of this key while any dependent AMI or snapshot is needed.

This is AWS EBS encryption at rest. It does not configure or attest to in-guest LUKS/filesystem encryption, and the OpenSCAP `encrypt_partitions` rule can still require manual assessment.

When `OrganizationId` is configured, the key policy conditionally allows principals in that organization to decrypt/re-encrypt the shared AMI and create AWS-resource grants. A workload role that launches the AMI must also have identity-policy permission on the `ImageEncryptionKeyArn`; reading the SSM parameter through `AmiParameterReaderRole` does not grant KMS access:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "UseSharedEncryptedAmi",
    "Effect": "Allow",
    "Action": [
      "kms:DescribeKey",
      "kms:CreateGrant",
      "kms:ReEncrypt*",
      "kms:Decrypt"
    ],
    "Resource": "<ImageEncryptionKeyArn>"
  }]
}
```

For services such as EC2 Auto Scaling, validate the service-linked role and KMS grant path in each workload account. See [Allow organizations and OUs to use a KMS key](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/allow-org-ou-to-use-key.html) and [share an encrypted AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/sharingamis-explicit.html).

### SNS notifications

`ImageNotificationTopic.KmsMasterKeyId` references a separate rotating customer-managed key. Its policy grants `kms:GenerateDataKey*` and `kms:Decrypt` only to `events.amazonaws.com`, constrained to the stack's build-state rule and account. The rule uses the documented Image Builder event fields: it matches `AVAILABLE`/`FAILED` in `detail.state.status`, limits events to the pipeline's image ARN prefix in `resources`, and sends the image ARN plus status to subscribers. Set `NotificationEmail` to create an email subscription, or add other subscribers later. The key ARN is available as `NotificationEncryptionKeyArn`. Image Builder service-event delivery to EventBridge is best effort, so use the SSM latest-AMI parameter and retained gate evidence as the release record; do not treat notification receipt as proof that a build passed.

### S3 reports and server access logs

The compliance reports bucket uses SSE-S3 (`AES256`), versioning, public-access blocking, TLS-only access, lifecycle expiration, and server access logging. Logs are delivered under `compliance-reports-access-logs/` in the separate `<PipelineName>-access-logs-<AccountId>` bucket. That destination is also encrypted, versioned, retained, private, and protected by TLS-only access; its delivery policy is restricted by both source account and source bucket ARN. The destination bucket is not configured to log to itself, avoiding recursive logging. S3 server access-log delivery is best effort and is not a substitute for CloudTrail data events when complete API-level audit coverage is required.

## Usage

### Automatic triggering for Amazon Linux

The template creates the Lambda trigger and SNS subscription when `EnableAutoTrigger` is `true` (the default). The subscription uses the AWS-published Amazon Linux 2023 AMI update topic:

- `arn:aws:sns:us-east-1:137112412989:amazon-linux-2023-ami-updates`

Set `EnableAutoTrigger=false` to omit the trigger log group, IAM role, Lambda function, Lambda permission, SNS subscription, and their conditional outputs. Manual pipeline execution remains available.

### Manual triggering

```bash
AWS_PROFILE="<aws-profile>"
AWS_REGION="<aws-region>"
STACK_NAME="stig-pipeline"

PIPELINE_ARN="$(
  aws cloudformation describe-stacks \
    --stack-name "$STACK_NAME" \
    --profile "$AWS_PROFILE" \
    --region "$AWS_REGION" \
    --query "Stacks[0].Outputs[?OutputKey=='PipelineArn'].OutputValue | [0]" \
    --output text
)"

aws imagebuilder start-image-pipeline-execution \
  --image-pipeline-arn "$PIPELINE_ARN" \
  --profile "$AWS_PROFILE" \
  --region "$AWS_REGION"
```

### Retrieving the latest AMI ID

After each successful build, the AMI ID is published to SSM Parameter Store:

```bash
aws ssm get-parameter \
  --name /ami/stig-al2023-arm64-pipeline/latest \
  --query Parameter.Value \
  --output text
```

In CloudFormation, reference it dynamically:

```yaml
ImageId: '{{resolve:ssm:/ami/stig-al2023-arm64-pipeline/latest}}'
```

Each pipeline maintains its own parameter based on `PipelineName`:

| Variant | PipelineName | SSM Parameter |
|---------|-------------|---------------|
| Amazon Linux 2023 arm64 | `stig-al2023-arm64-pipeline` | `/ami/stig-al2023-arm64-pipeline/latest` |
| Amazon Linux 2023 x86_64 | `stig-al2023-x86-pipeline` | `/ami/stig-al2023-x86-pipeline/latest` |

### Cross-account AMI ID consumption

Workload accounts in other AWS accounts need a way to discover the latest hardened AMI ID that the pipeline produces. This solution uses the **assume-role + `GetParameter`** pattern, which is the best fit for its architecture:

- The hardened AMI is shared to the **whole organization** through launch permissions (`LaunchPermissionConfiguration.OrganizationArns`). One AMI, owned by the pipeline account, is launchable org-wide — there are no per-account AMI copies.
- The latest AMI ID is published to a **single central parameter** `/ami/<PipelineName>/latest` in the pipeline account.
- When `OrganizationId` is set, the template creates an `AmiParameterReaderRole` that any principal in the organization (gated by `aws:PrincipalOrgID`) can assume to read that parameter with least-privilege, read-only access.

This keeps a single source of truth and needs no advanced-tier parameters. Each workload still requires explicit authorization: its identity must be allowed to assume the reader role, and its AMI-launch path must be allowed to use `ImageEncryptionKeyArn`; service-managed launch paths may also need a KMS grant. This pattern is a good fit when consumers are CI/CD pipelines that call AWS APIs.

**Role ARN pattern:** `arn:aws:iam::<pipeline-account-id>:role/<PipelineName>-ami-reader`

Workload teams need:
- Pipeline account ID
- Pipeline name
- Region where the pipeline is deployed
- `ImageEncryptionKeyArn` from the stack outputs for encrypted-AMI launch permissions
- An AWS CLI source profile (or equivalent workload identity) whose principal is allowed to call `sts:AssumeRole` on the generated reader role

The role is created only when `OrganizationId` is configured. Its `aws:PrincipalOrgID` trust condition limits eligible callers to the configured organization, but trust alone does **not** grant a workload principal permission to assume it. Add a least-privilege policy like this to the workload pipeline role or user:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::<pipeline-account-id>:role/<PipelineName>-ami-reader"
  }]
}
```

For local testing or a runner that supports shared AWS configuration, define a named assume-role profile. The AWS CLI obtains and refreshes temporary credentials through the source profile; do not copy credentials into commands or logs.

```ini
# ~/.aws/config
[profile workload-source]
region = us-east-1

[profile hardened-ami-reader]
source_profile = workload-source
role_arn = arn:aws:iam::<pipeline-account-id>:role/<PipelineName>-ami-reader
role_session_name = latest-ami-lookup
region = us-east-1
```

Use the assumed-role profile for both the identity check and Parameter Store read. The parameter is read in the **pipeline account** and **pipeline Region**:

```bash
set -euo pipefail

READER_PROFILE=hardened-ami-reader
PIPELINE_REGION=us-east-1
PIPELINE_ACCOUNT_ID=<pipeline-account-id>
PARAMETER_NAME=/ami/<PipelineName>/latest

ACTUAL_ACCOUNT_ID=$(aws sts get-caller-identity \
  --profile "$READER_PROFILE" \
  --query Account \
  --output text)

if [ "$ACTUAL_ACCOUNT_ID" != "$PIPELINE_ACCOUNT_ID" ]; then
  echo "Expected pipeline account $PIPELINE_ACCOUNT_ID; got $ACTUAL_ACCOUNT_ID" >&2
  exit 1
fi

AMI_ID=$(aws ssm get-parameter \
  --profile "$READER_PROFILE" \
  --region "$PIPELINE_REGION" \
  --name "$PARAMETER_NAME" \
  --query Parameter.Value \
  --output text)

case "$AMI_ID" in
  ami-*) printf 'Latest gated AMI: %s\n' "$AMI_ID" ;;
  *) echo "Parameter did not return an AMI ID" >&2; exit 1 ;;
esac
```

In hosted CI/CD, configure the same role chain through the runner's AWS credential provider rather than storing long-lived credentials. The resulting `$AMI_ID` can be passed to CloudFormation, Terraform, or an EC2 API call. A CloudFormation `{{resolve:ssm:...}}` dynamic reference cannot perform this cross-account role assumption; use an API step like the one above or distribute a local parameter to each workload account.

#### Cross-account lookup troubleshooting

- **`AccessDenied` from `AssumeRole`:** Confirm `OrganizationId` was configured, the reader role exists, the caller belongs to that organization, and the workload principal has an identity policy allowing `sts:AssumeRole` on the exact role ARN.
- **`ParameterNotFound`:** Confirm the pipeline name and Region. The parameter is created only after a successful image distribution, so a new pipeline with no successful build has no latest-AMI value yet.
- **Identity check returns the workload account:** The assume-role profile was not used or its role chain is misconfigured. Stop before reading the parameter.
- **AMI cannot be launched after a successful lookup:** Parameter access, AMI launch permission, and KMS key use are separate. Confirm organization sharing is configured, retrieve `ImageEncryptionKeyArn` from the stack outputs, and verify that the workload launch principal has `kms:DescribeKey`, `kms:CreateGrant`, `kms:ReEncrypt*`, and `kms:Decrypt` on that exact key. For Auto Scaling or another AWS service, also validate its service-linked-role grant path.

#### Alternative patterns (and why this one)

- **Direct parameter distribution into each account.** Image Builder can write the AMI-ID parameter directly into each consumer account using the `AmiAccountId` field of `SsmParameterConfigurations`, so each account reads a **local** parameter. This is worth adding only if consumers must resolve the AMI **declaratively with no API step** — for example an Auto Scaling launch template using `resolve:ssm:/ami/<PipelineName>/latest`, or a CloudFormation `{{resolve:ssm:...}}` dynamic reference (which is same-account only). The cost is real: every consumer account must create `EC2ImageBuilderDistributionCrossAccountRole` and be listed as a distribution target, you get one AMI copy per account instead of one shared image, and encrypted AMIs need cross-account KMS grants.

- **AWS RAM parameter sharing — not usable here.** RAM sharing of a Parameter Store parameter requires the parameter to be in the **advanced tier**, but Image Builder's `SsmParameterConfiguration` exposes no tier setting (`amiAccountId`, `parameterName`, `dataType` only), so it creates the parameter in the standard tier. Making RAM work would require flipping the account-wide default parameter tier to Advanced (broad blast radius and per-parameter cost) or manually promoting the parameter after every recreate, and even then CloudFormation `{{resolve:ssm:...}}` dynamic references still do not support shared parameters. For those reasons RAM sharing is not used.

#### References

- [Tracking the latest server images in EC2 Image Builder pipelines](https://aws.amazon.com/blogs/compute/tracking-the-latest-server-images-in-amazon-ec2-image-builder-pipelines/) — publishing the AMI ID to an SSM parameter for consumers.
- [Reference AMIs using Systems Manager parameters](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-systems-manager-parameter-to-find-AMI.html) — the "latest AMI" pointer pattern.
- [SsmParameterConfiguration API reference](https://docs.aws.amazon.com/imagebuilder/latest/APIReference/API_SsmParameterConfiguration.html) — the `amiAccountId` field for per-account parameter distribution (and the absence of a tier field).
- [Set up cross-account AMI distribution with Image Builder](https://docs.aws.amazon.com/imagebuilder/latest/userguide/cross-account-dist.html) — cross-account distribution role and the `ssm:PutParameter` permission.
- [Working with shared parameters in Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-store-shared-parameters.html) — the advanced-tier prerequisite for RAM sharing.
- [Specifying a default parameter tier](https://docs.aws.amazon.com/systems-manager/latest/userguide/ps-default-tier.html) — how the standard/advanced tier is chosen.

## OpenSCAP assessment and release-gate verification

The pipeline includes two custom components that run during the Image Builder test stage: `<PipelineName>-openscap-scan` performs the assessments and computes the release-gate decision, and `<PipelineName>-openscap-gate` uploads the evidence, records the outcome on the AMI, and enforces the decision. They are split because a component's inline document is limited to 16,000 characters; the scan component leaves its artifacts in `/tmp` on the test instance and the recipe runs the gate component immediately after it.

- **Baselines (three separate scans/reports):**
  - STIG — `xccdf_org.ssgproject.content_profile_stig`
  - CIS Level 1 (Server) — resolved from the content (for example, `..._profile_cis_server_l1`)
  - CIS Level 2 (Server) — resolved from the content (for example, `..._profile_cis`)
  - Profiles are discovered from the installed data stream at scan time. The exact STIG profile ID is recorded in both gate-decision artifacts. A missing STIG profile fails closed because there is no gate evidence; unavailable CIS profiles are skipped with an informational note.
- **Content:** Auto-detected from `/usr/share/xml/scap/ssg/content` for the selected OS.
- **Scan scope:** Each profile is evaluated in full. No rules are skipped, so every report is a complete record of the evaluated results.
- **`STIGLevel` boundary:** `STIGLevel` configures the hardening scope passed to the AWS-managed `stig-build-linux` component. It does not select individual OpenSCAP rules, change the discovered `_profile_stig` assessment profile, or change the fixed release-gate threshold.
- **Gate evaluation order:** in the scan component, `SummarizeResults` computes `gate-decision.json` and `gate-decision.txt`; then in the gate component, `UploadReports` preserves both decision artifacts and the supporting scan evidence, `TagImageWithGateOutcome` records the decision on the AMI itself, and only then does `ValidateOpenSCAPResult` enforce the precomputed decision. Failed gate evidence is therefore retained.
- **Evidence handling is fail-closed:** the build fails if the AMI under test cannot be identified from IMDSv2 (after three attempts), if a decision artifact is missing, if any evidence upload fails, or if the gate outcome cannot be recorded on the AMI. An image is never published with its evidence absent, filed under a placeholder identifier, or left unlabelled. Every artifact is attempted before the step fails, so a single upload failure does not discard the remaining evidence. Object *tagging* failures warn without failing the build, because the tags carry traceability metadata rather than the evidence itself.
- **Gate fail criteria:** The decision is `FAIL` if the STIG profile was not discovered, the scan exit code is missing/invalid/unexpected, the STIG XML is missing, malformed, or has no rule results, or a non-excluded high-severity STIG-profile result is `fail`, `error`, or `unknown`. A high-severity `notchecked` result also blocks unless its rule result contains OpenSCAP's exact informational message `No candidate or applicable check found.`; that deterministic no-automated-check case is recorded in `manual_review_findings` instead of being treated as an automated failure. OpenSCAP exit code `1` fails as a scan error; exit code `2` does not independently fail because it indicates findings that the gate evaluates from XML.
- **Gate pass criteria:** The discovered STIG profile and valid non-empty results are recorded, the OpenSCAP exit code is `0` or `2`, and there are zero blocking high-severity findings after the fixed FIPS and no-automated-check classifications. High-severity `pass`, `fixed`, `notapplicable`, and `notselected` do not block. A no-automated-check `notchecked` finding requires documented manual assessment and does not establish control satisfaction. CAT II/III findings and CIS L1/L2 results remain informational.
- **FIPS handling:** FIPS controls are always scanned and shown in evidence. When `EnableFIPS=false`, only these fixed high-severity rule IDs are excluded from the gate: `enable_fips_mode`, `enable_dracut_fips_module`, `sysctl_crypto_fips_enabled`, `configure_crypto_policy`, and `harden_sshd_ciphers_openssh_conf_crypto_policy` (each with the standard `xccdf_org.ssgproject.content_rule_` prefix). When `EnableFIPS=true`, they are gated like other high-severity results.

### Release-gate decision artifacts

- `gate-decision.json` is the machine-readable contract. Schema version `1.0` records `decision`, the discovered profile and presence flag, FIPS mode, scan exit code, result-file state and rule count, the fixed blocking-status list, blocking rule IDs/results, no-automated-check `manual_review_findings`, excluded FIPS rule IDs/results, and reasons.
- `gate-decision.txt` presents the same decision for human review, including blocking, manual-review, and excluded rule lists.

The final validation step reads only the precomputed JSON decision; it does not independently reparse XML. This makes the uploaded decision and the enforced decision the same artifact.

### Release-gate outcome tags on the AMI

Image Builder creates the output AMI at the end of the build stage, before the test stage runs the assessment. A gate `FAIL` therefore blocks distribution, organization sharing, and the SSM parameter update, but the AMI itself still exists in the pipeline account. To make that image unambiguous, the pipeline tags the AMI with the gate outcome before the gate is enforced:

| Tag | Value |
|-----|-------|
| `StigGateDecision` | `PASS` or `FAIL` |
| `StigGateProfile` | the discovered STIG profile ID, or `unavailable` |
| `StigGateEvaluatedAt` | UTC timestamp of the evaluation |
| `StigGateRecipeVersion` | recipe version that produced the image |

The build fails if the tag cannot be written, so a published image is always labelled. The instance role can write only these four tag keys, on images only.

Because EC2 [Allowed AMIs criteria](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-allowed-amis.html) match on image provider, image name, Marketplace product code, creation/deprecation dates, and watermarks — **not** on tags — this tag cannot be used as Allowed AMIs criteria. Use it for operator triage, for AWS Config rules, or in an SCP that denies launches of images not tagged `StigGateDecision=PASS` within the pipeline account:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyLaunchOfImagesThatFailedTheGate",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*::image/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:ResourceTag/StigGateDecision": "PASS"
        }
      }
    }
  ]
}
```

Scope any such policy to accounts that only ever launch pipeline images; an account that also launches AWS-provided or Marketplace images would be blocked by the condition above, because those images do not carry the tag.

**Report storage:** The release decision is computed first, all available artifacts are uploaded next, and the precomputed decision is enforced only after upload. Objects are grouped by the UTC report-generation date and then keyed to the built AMI:

```text
s3://<PipelineName>-compliance-reports-<AccountId>/reports/<YYYY-MM-DD>/<ami-id>/
├── gate-decision.json
├── gate-decision.txt
├── compliance-summary.txt
├── stig/
│   ├── stig-report.html
│   ├── stig-results.xml
│   ├── stig-summary.txt
│   └── stig-exit-code.txt
├── cis-l1/
│   ├── cis-l1-report.html
│   ├── cis-l1-results.xml
│   ├── cis-l1-summary.txt
│   └── cis-l1-exit-code.txt
└── cis-l2/
    ├── cis-l2-report.html
    ├── cis-l2-results.xml
    ├── cis-l2-summary.txt
    └── cis-l2-exit-code.txt
```

The `<YYYY-MM-DD>` folder is the UTC date on which the Image Builder test generated the reports. Placing `<ami-id>` directly beneath that date ties every artifact to the specific AMI that was tested, while providing a chronological browsing level. This design preserves the scan output for troubleshooting even when the AMI is not distributed.

> **Note:** A `PASS` release-gate decision is not a claim of 100% STIG compliance. The DISA STIG profile includes CAT II/III controls that the AWS `stig-build-linux` component may not fully satisfy, and this gate applies only the documented high-severity status policy after fixed FIPS exclusions and explicit no-automated-check manual-review classification. Review all findings and track remaining remediation in the customer's risk process.

## Organization governance with Allowed AMIs

To help restrict workload accounts to approved pipeline AMIs, use both:

1. **AMI sharing:** Set `OrganizationId` so Image Builder shares successful AMIs with accounts in the organization.
2. **Declarative policy:** Use `allowed-amis-policy.json` to configure EC2 Allowed AMIs criteria in workload accounts.

**What this policy does and does not enforce.** Allowed AMIs criteria match on image provider and image name, so the policy restricts launches to images produced by this pipeline. It cannot evaluate the release-gate outcome, because [Allowed AMIs criteria do not support tags](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-allowed-amis.html). A gate-failed AMI carries the same name pattern as an approved one, so the name match alone does not distinguish it. What keeps rejected images out of workload use is the combination of: no organization sharing and no SSM parameter update on `FAIL` (an unshared AMI cannot be launched from another account), consumption through the SSM parameter, which only ever points at an image that passed, and the `StigGateDecision` tag for triage and for tag-based controls inside the pipeline account.

Before applying the policy, replace both placeholders in `allowed-amis-policy.json`:

- `<PIPELINE_ACCOUNT_ID>` with the AWS account ID where the pipeline runs.
- `<PIPELINE_NAME>` with the deployed `PipelineName` value.

Illustrative OU structure:

```
Organization Root
├── Security/Tooling OU          ← Pipeline account (evaluate source-image policy separately)
│   └── <pipeline-account-id>
└── Workloads OU                 ← Attach the Declarative Policy here
    ├── Team A account
    ├── Team B account
    └── ...
```

From the management account, create and attach the policy to the workload OU:

```bash
aws organizations create-policy \
  --name "Allowed-AMIs-STIG-Pipeline" \
  --description "Restricts workload accounts to approved STIG-hardened images" \
  --type DECLARATIVE_POLICY_EC2 \
  --content file://allowed-amis-policy.json \
  --profile "<management-profile>"

aws organizations attach-policy \
  --policy-id <policy-id> \
  --target-id <workloads-ou-id> \
  --profile "<management-profile>"
```

Verify in a workload account:

```bash
aws ec2 get-allowed-images-settings \
  --region us-east-1 \
  --profile "<workload-profile>"
```

Evaluate source-image policy separately for the pipeline account. Image Builder needs access to AWS-managed parent images to build new hardened AMIs, so any policy applied there must allow those source images.

### Ensuring workload accounts use only the latest AMI

The Allowed AMIs policy restricts launches to your pipeline's AMIs, but it matches a name pattern, so it still allows older builds. To ensure only the most recent image is usable, combine three layers:

1. **Consume through SSM:** Have workload deployments resolve `/ami/<PipelineName>/latest` (for example `{{resolve:ssm:/ami/<PipelineName>/latest}}` in CloudFormation, or an SSM data source in Terraform, or a launch template that reads the parameter). A new successful build updates this parameter automatically, so new deployments pick up the latest approved image.
2. **Retire older AMIs (lifecycle policy):** Set `EnableAmiLifecycle=true` to create an EC2 Image Builder lifecycle policy that keeps `KeepLatestCount` recent AMIs and applies `AmiLifecycleAction` to older ones. Because AMIs are shared to the organization through launch permissions (they stay in the pipeline account), a `DISABLE` or `DELETE` action on an older AMI immediately removes the ability for any account to launch it, leaving only the recent images. `DEPRECATE` only marks images and does not block launches.
3. **Allowed AMIs guardrail:** Keep the Declarative Policy attached to workload OUs so nothing outside the pipeline's AMIs can launch at all.

Recommended: `KeepLatestCount=2` (latest plus one for rollback) with `AmiLifecycleAction=DISABLE` for reversible enforcement, moving to `DELETE` once you are comfortable. The lifecycle policy runs on an AWS-managed schedule, so retirement of older AMIs is not instantaneous.

## Outputs

| Output | Description |
|--------|-------------|
| `PipelineArn` | Pipeline ARN, exported as `<StackName>-PipelineArn`. |
| `RoleArn` | Build instance IAM role ARN, exported as `<StackName>-RoleArn`. |
| `PipelineExecutionInfo` | CLI guidance for triggering the pipeline and finding build output. |
| `InstanceProfileArn` | Instance profile ARN. |
| `OpenSCAPScanComponentArn` | OpenSCAP assessment component ARN. |
| `OpenSCAPGateComponentArn` | Release-gate component ARN. |
| `STIGImageRecipeArn` | Image recipe ARN. |
| `InfrastructureConfigArn` | Image Builder infrastructure configuration ARN. |
| `DistributionConfigArn` | Image Builder distribution configuration ARN. |
| `BuildWorkflowArn` | Custom build workflow ARN. |
| `PipelineExecutionRoleArn` | Pipeline execution role ARN. |
| `PipelineTriggerRoleArn` | Lambda trigger role ARN, only when `EnableAutoTrigger=true` and the selected Amazon Linux base image has an AWS update topic. |
| `PipelineTriggerFunctionArn` | Lambda trigger function ARN under the same automatic-trigger condition. |
| `AmiUpdateSubscriptionArn` | Amazon Linux update-topic SNS subscription ARN under the same automatic-trigger condition. |
| `LatestAmiParameterName` | SSM parameter path for the latest AMI ID. |
| `ImageEncryptionKeyArn` | Customer-managed KMS key ARN for encrypted build volumes and published AMI snapshots; workload launch roles use this ARN in KMS permissions. |
| `NotificationEncryptionKeyArn` | Customer-managed KMS key ARN for encrypted SNS build-state notifications. |
| `ComplianceReportsBucketName` | S3 bucket for OpenSCAP compliance reports. |
| `AccessLogsBucketName` | S3 destination bucket for compliance-reports server access logs. |
| `ImageNotificationTopicArn` | Encrypted SNS topic for `AVAILABLE`/`FAILED` image build notifications. |
| `AmiLifecyclePolicyArn` | Image Builder lifecycle policy ARN, only when AMI lifecycle management is enabled. |
| `AmiParameterReaderRoleArn` | Cross-account role ARN, only when organization sharing is enabled. |

## Custom build workflow

The pipeline uses a custom Image Builder build workflow that skips `InventoryCollection`. STIG hardening can modify firewall and SSM behavior in ways that interfere with the default inventory collection step. The custom workflow includes:

1. LaunchBuildInstance
2. ApplyBuildComponents
3. RunSanitizeScript
4. CreateOutputAMI
5. TerminateBuildInstance

## Persisting OpenSCAP reports

The template creates two S3 buckets:

- `<PipelineName>-compliance-reports-<AccountId>` for OpenSCAP reports.
- `<PipelineName>-access-logs-<AccountId>` for server access logs from the reports bucket.

The reports bucket has AES-256 server-side encryption, public access blocking, versioning, TLS enforcement, 90-day lifecycle expiration for reports, and `LoggingConfiguration` targeting the access-logs bucket. The destination bucket has AES-256 encryption, public access blocking, versioning, TLS enforcement, 365-day lifecycle expiration, retained deletion policy, and a log-delivery policy constrained by source account and reports-bucket ARN. `AccessLogsBucketName` identifies the destination in stack outputs. Delivery is best effort and can be delayed.

## Known limitations

- **FIPS mode:** FIPS is disabled by default and is not enabled by the `stig-build-linux` component. Set `EnableFIPS=true` to turn it on; the pipeline then runs `fips-mode-setup --enable` and reboots the build instance (via the AWSTOE `Reboot` action, which Image Builder supports during the build stage) so FIPS activates before the AMI is captured. This setting is not a FIPS certification.
- **Single-region distribution:** This template distributes the AMI in the stack region. To distribute to additional regions, extend the distribution configuration with a customer-managed KMS key in each destination Region or deploy separate regional stacks.
- **Encrypted-AMI key dependency:** Published AMIs and their snapshots depend on the retained `ImageEncryptionKey`. Organization sharing also requires workload launch principals—and service-linked roles or grants used by services such as Auto Scaling—to be authorized for that key. Disabling or deleting the key makes dependent images unusable.
- **EBS versus in-guest encryption:** EBS encryption protects the AWS storage layer. It does not configure LUKS or prove that an operating-system partition is encrypted.
- **S3 access-log semantics:** Server access logging is best effort and can be delayed. Use CloudTrail data events if the customer requires complete API-level object-access auditing.
- **Image test timeout:** The test stage timeout is 60 minutes.
- **Extending to other distributions:** This solution is built and tested for Amazon Linux 2023. The pipeline pattern can be adapted to other distributions, but each has its own package names, SCAP content availability, and STIG profile coverage; validate the scanner installation and gate behavior for that distribution before relying on it.
- **Allowed AMIs policy scope:** Apply the workload restriction to the intended workload OUs/accounts. Evaluate source-image policy separately for the pipeline account and ensure any criteria applied there allow the AWS-managed parent images required by Image Builder.
- **A gate-failed AMI still exists in the pipeline account:** Image Builder creates the AMI at the end of the build stage, before the test stage assesses it, and a failed build does not delete it. The image is not shared to the organization and the SSM parameter is not updated, so it is not consumable elsewhere, and it is tagged `StigGateDecision=FAIL`. Delete rejected images as part of the customer's cleanup process, or add a tag-based launch restriction inside the pipeline account. Allowed AMIs criteria cannot filter on the tag.

## Disclaimer

This is sample code, for non-production usage. You should work with your security and legal teams to meet your organizational security, regulatory and compliance requirements before deployment.

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This sample code is made available under the MIT-0 license. See the [LICENSE](LICENSE) file.
