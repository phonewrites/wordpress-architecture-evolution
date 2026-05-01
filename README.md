# WordPress App AWS Architecture Evolution
CloudFormation (IaC) version of the **[manual demo project](https://github.com/acantril/learn-cantrill-io-labs/tree/master/aws-elastic-wordpress-evolution)** from one of Adrian Cantrill's courses.

Follow these instructions from the repository root. This is the directory where you run
`aws cloudformation package`, create stacks, and update stacks.

The current path has been smoke-tested for greenfield stages 1 through 5 and for
same-stack evolution from stage 1 through stage 5.

## 1. (Not so) Foolish Assumptions

- You are deploying in one AWS account and one region.
- You run only one active stack from this repo per account and region because the
  templates use fixed SSM Parameter Store names under `/Wordpress/...`.
- You have AWS CLI access with permissions for CloudFormation, EC2, SSM, IAM,
  RDS, EFS, ELBv2, Route 53, ACM, and S3.
- You can create or use an S3 bucket in the same region for packaged nested
  templates.
- You understand that the stack creates billable resources.
- Optional custom domain: the domain already has a public Route 53 hosted zone in
  the same account.

Set shell variables first:
```bash
export AWS_PROFILE=STAGING
export AWS_REGION=us-east-1
export STACK_NAME=wp-evolution
export ARTIFACT_BUCKET=YOUR_UNIQUE_CFN_ARTIFACT_BUCKET
```


## 2. Create Prerequisites

Create the artifact bucket if you do not already have one:
```bash
aws s3 mb "s3://$ARTIFACT_BUCKET" --region "$AWS_REGION" --profile "$AWS_PROFILE"
```

Create the database passwords in SSM Parameter Store:
```bash
aws ssm put-parameter --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --name "/Wordpress/DBPassword" \
  --description "Password for the DB user" \
  --type "SecureString" \
  --value "REPLACE_ME_DB_PASSWORD" \
  --overwrite

aws ssm put-parameter --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --name "/Wordpress/DBRootPassword" \
  --description "Password for the DB root user – used for self-managed admin" \
  --type "SecureString" \
  --value "REPLACE_ME_DB_ROOT_PASSWORD" \
  --overwrite
```

For `/Wordpress/DBPassword`, avoid `/`, `@`, `"`, and spaces because RDS rejects
those characters for the master password. For both passwords, avoid newlines and
single quotes.


## 3. Package Templates

Run packaging from this directory:

```bash
for STAGE in 1 2 3 4 5; do
  aws cloudformation package \
    --region "$AWS_REGION" \
    --profile "$AWS_PROFILE" \
    --template-file "stages/stage${STAGE}.yaml" \
    --s3-bucket "$ARTIFACT_BUCKET" \
    --output-template-file "build/stage${STAGE}.packaged.yaml"
done
```
Use the files in `build/` for CloudFormation create/update operations.

## Choose one path:

- **[Evolve one stack](#4a-evolve-one-stack)**: start at stage 1 and update the same
  stack through later stages. This follows the original manual demo flow.
- **[Deploy a greenfield stage](#4b-deploy-a-greenfield-stage)**: create a fresh stack
  directly at a chosen stage.

## 4a. Evolve One Stack

Start at stage 1, then update the same stack one stage at a time. You can stop
after stages 1, 2, 3, or 4 if that architecture is the one you want to inspect,
or continue to stage 5.

After every create/update, open the `WordpressURL` output and verify public posts, `/wp-admin`, and media uploads.

Define helper functions for the CLI path:

```bash
get_output () {
  aws cloudformation describe-stacks \
    --region "$AWS_REGION" --profile "$AWS_PROFILE" \
    --stack-name "$STACK_NAME" \
    --query "Stacks[0].Outputs[?OutputKey=='$1'].OutputValue | [0]" \
    --output text
}

wait_automation () {
  while true; do
    STATUS=$(aws ssm get-automation-execution \
      --region "$AWS_REGION" --profile "$AWS_PROFILE" \
      --automation-execution-id "$1" \
      --query 'AutomationExecution.AutomationExecutionStatus' \
      --output text)
    echo "$1 $STATUS"
    case "$STATUS" in
      Success) break ;;
      Failed|Cancelled|TimedOut) exit 1 ;;
      *) sleep 30 ;;
    esac
  done
}
```

### Create Stage 1

Console:
1. Open CloudFormation -> **Create stack** -> **With new resources**.
2. Upload `build/stage1.packaged.yaml`.
3. Enter `STACK_NAME`.
4. Acknowledge IAM and nested stack capabilities.
5. Create the stack and wait for `CREATE_COMPLETE`.

CLI:
```bash
aws cloudformation create-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage1.packaged.yaml \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

aws cloudformation wait stack-create-complete \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"
```

Open `WordpressURL`, finish WordPress setup, and add test content. You can stop
here for the single-server architecture.

### Update Stage 1 to Stage 2

Console:
1. Open Systems Manager -> **Automation** -> **Execute automation**.
2. Choose `Wordpress-PreStageSnapshot`.
3. Enter:
   - `AutomationAssumeRole`: `AutomationRoleArn` stack output
   - `InstanceId`: `WordpressInstanceId` stack output
   - `SourceStage`: `stage1`
   - `SourceStackName`: your stack name
   - `NoReboot`: `false`
4. Run it and copy output `createImage.ImageId`.
5. Open CloudFormation -> your stack -> **Update**.
6. Choose **Replace current template** and upload `build/stage2.packaged.yaml`.
7. Set `SourceAmiId` to the AMI ID from the automation output.
8. Set `CfnMetadataNonce` to `2`, acknowledge capabilities, and update.

CLI:
```bash
AUTOMATION_ROLE_ARN=$(get_output AutomationRoleArn)
INSTANCE_ID=$(get_output WordpressInstanceId)

EXECUTION_ID=$(aws ssm start-automation-execution \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --document-name Wordpress-PreStageSnapshot \
  --parameters "AutomationAssumeRole=$AUTOMATION_ROLE_ARN,InstanceId=$INSTANCE_ID,SourceStage=stage1,SourceStackName=$STACK_NAME,NoReboot=false" \
  --query AutomationExecutionId --output text)

wait_automation "$EXECUTION_ID"

STAGE1_AMI=$(aws ssm get-automation-execution \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --automation-execution-id "$EXECUTION_ID" \
  --query 'AutomationExecution.Outputs."createImage.ImageId"[0]' \
  --output text)

aws cloudformation update-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage2.packaged.yaml \
  --parameters \
    ParameterKey=SourceAmiId,ParameterValue="$STAGE1_AMI" \
    ParameterKey=CfnMetadataNonce,ParameterValue=2 \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

aws cloudformation wait stack-update-complete \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"
```

You can stop here for the launch-template single-server architecture.

### Update Stage 2 to Stage 3

First add RDS while the stage 2 app still serves traffic.

Console:
1. Open CloudFormation -> your stack -> **Update**.
2. Upload `build/stage3.packaged.yaml`.
3. Set:
   - `Stage3Mode`: `prepare`
   - `SourceAmiId`: the stage 1 AMI used for stage 2
   - `CfnMetadataNonce`: `3`
4. Update and wait for `UPDATE_COMPLETE`.
5. Open Systems Manager -> **Automation** -> **Execute automation**.
6. Choose `Wordpress-MigrateLocalMariaDbToRds`.
7. Enter:
   - `AutomationAssumeRole`: `AutomationRoleArn` output
   - `SourceInstanceId`: current `WordpressInstanceId` output
   - `RdsEndpoint`: `RdsEndpoint` output
   - `RdsPort`: `3306`
8. Run it and wait for `Success`.
9. Run `Wordpress-PreStageSnapshot` against the same current instance with
   `SourceStage=stage2`; copy output `createImage.ImageId`.
10. Update the stack again with `build/stage3.packaged.yaml`.
11. Set `Stage3Mode=cutover`, `SourceAmiId=<stage2 AMI>`, and
    `CfnMetadataNonce=4`.

CLI:
```bash
aws cloudformation update-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage3.packaged.yaml \
  --parameters \
    ParameterKey=Stage3Mode,ParameterValue=prepare \
    ParameterKey=SourceAmiId,ParameterValue="$STAGE1_AMI" \
    ParameterKey=CfnMetadataNonce,ParameterValue=3 \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

aws cloudformation wait stack-update-complete \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"

INSTANCE_ID=$(get_output WordpressInstanceId)
RDS_ENDPOINT=$(get_output RdsEndpoint)
AUTOMATION_ROLE_ARN=$(get_output AutomationRoleArn)

MIGRATION_ID=$(aws ssm start-automation-execution \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --document-name Wordpress-MigrateLocalMariaDbToRds \
  --parameters "AutomationAssumeRole=$AUTOMATION_ROLE_ARN,SourceInstanceId=$INSTANCE_ID,RdsEndpoint=$RDS_ENDPOINT,RdsPort=3306" \
  --query AutomationExecutionId --output text)

wait_automation "$MIGRATION_ID"

EXECUTION_ID=$(aws ssm start-automation-execution \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --document-name Wordpress-PreStageSnapshot \
  --parameters "AutomationAssumeRole=$AUTOMATION_ROLE_ARN,InstanceId=$INSTANCE_ID,SourceStage=stage2,SourceStackName=$STACK_NAME,NoReboot=false" \
  --query AutomationExecutionId --output text)

wait_automation "$EXECUTION_ID"

STAGE2_AMI=$(aws ssm get-automation-execution \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --automation-execution-id "$EXECUTION_ID" \
  --query 'AutomationExecution.Outputs."createImage.ImageId"[0]' \
  --output text)

aws cloudformation update-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage3.packaged.yaml \
  --parameters \
    ParameterKey=Stage3Mode,ParameterValue=cutover \
    ParameterKey=SourceAmiId,ParameterValue="$STAGE2_AMI" \
    ParameterKey=CfnMetadataNonce,ParameterValue=4 \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

aws cloudformation wait stack-update-complete \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"
```

You can stop here for the RDS-backed app architecture.

### Update Stage 3 to Stage 4

Console:
1. Run Systems Manager Automation document `Wordpress-PreStageSnapshot`.
2. Enter:
   - `AutomationAssumeRole`: `AutomationRoleArn` output
   - `InstanceId`: current `WordpressInstanceId` output
   - `SourceStage`: `stage3`
   - `SourceStackName`: your stack name
   - `NoReboot`: `false`
3. Copy output `createImage.ImageId`.
4. Open CloudFormation -> your stack -> **Update**.
5. Upload `build/stage4.packaged.yaml`.
6. Set `SourceAmiId` to the stage 3 AMI and `CfnMetadataNonce` to `5`.
7. Acknowledge capabilities and update.

CLI:
```bash
INSTANCE_ID=$(get_output WordpressInstanceId)
AUTOMATION_ROLE_ARN=$(get_output AutomationRoleArn)

EXECUTION_ID=$(aws ssm start-automation-execution \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --document-name Wordpress-PreStageSnapshot \
  --parameters "AutomationAssumeRole=$AUTOMATION_ROLE_ARN,InstanceId=$INSTANCE_ID,SourceStage=stage3,SourceStackName=$STACK_NAME,NoReboot=false" \
  --query AutomationExecutionId --output text)

wait_automation "$EXECUTION_ID"

STAGE3_AMI=$(aws ssm get-automation-execution \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --automation-execution-id "$EXECUTION_ID" \
  --query 'AutomationExecution.Outputs."createImage.ImageId"[0]' \
  --output text)

aws cloudformation update-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage4.packaged.yaml \
  --parameters \
    ParameterKey=SourceAmiId,ParameterValue="$STAGE3_AMI" \
    ParameterKey=CfnMetadataNonce,ParameterValue=5 \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

aws cloudformation wait stack-update-complete \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"
```

You can stop here for the RDS + EFS architecture.

### Update Stage 4 to Stage 5

Console:
1. Open CloudFormation -> your stack -> **Update**.
2. Upload `build/stage5.packaged.yaml`.
3. Leave the custom domain parameters blank to use the ALB DNS name, or enter all
   three custom domain parameters:
   - `HostedZoneName`, for example `example.com`
   - `HostedZoneId`, for example `Z1234567890ABC`
   - `CustomSubdomainName`, for example `wordpress`
4. Acknowledge capabilities and update.

CLI (without a custom domain):
```bash
aws cloudformation update-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage5.packaged.yaml \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

aws cloudformation wait stack-update-complete \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"
```

CLI (with a custom HTTPS URL):
```bash
aws cloudformation update-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage5.packaged.yaml \
  --parameters \
    ParameterKey=HostedZoneName,ParameterValue=example.com \
    ParameterKey=HostedZoneId,ParameterValue=Z1234567890ABC \
    ParameterKey=CustomSubdomainName,ParameterValue=wordpress \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

aws cloudformation wait stack-update-complete \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"
```

Check outputs at any stage:

```bash
aws cloudformation describe-stacks \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' \
  --output table
```

## 4b. Deploy a Greenfield Stage

Greenfield means you create a new stack directly at one stage. Delete any
existing stack from this repo first, unless you have changed the fixed
`/Wordpress/...` SSM names.

For stage 1, use [Create Stage 1](#create-stage-1).

Console path for stages 2 through 5:

1. Open CloudFormation -> **Create stack** -> **With new resources**.
2. Upload the matching `build/stageN.packaged.yaml`.
3. Enter a stack name.
4. Set parameters for:
   - Stage 2: leave `SourceAmiId` empty.
   - Stage 3: set `Stage3Mode=cutover` and leave `SourceAmiId` empty.
   - Stage 4: leave `SourceAmiId` empty.
   - Stage 5: leave custom domain fields blank for the ALB DNS name, or enter
     `HostedZoneName`, `HostedZoneId`, and `CustomSubdomainName` together.
5. Acknowledge IAM and nested stack capabilities.
6. Create the stack, then open the `WordpressURL` output.

CLI path for stages 2 through 5:

```bash
# Stage 2: single EC2 instance through a launch template.
aws cloudformation create-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage2.packaged.yaml \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

# Stage 3: EC2 app tier plus RDS.
aws cloudformation create-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage3.packaged.yaml \
  --parameters ParameterKey=Stage3Mode,ParameterValue=cutover \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

# Stage 4: EC2 app tier plus RDS and EFS.
aws cloudformation create-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage4.packaged.yaml \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND

# Stage 5: ALB plus Auto Scaling Group, RDS, and EFS.
aws cloudformation create-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage5.packaged.yaml \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

For stage 5 with a custom HTTPS URL, use this command instead:
```bash
aws cloudformation create-stack --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --template-body file://build/stage5.packaged.yaml \
  --parameters \
    ParameterKey=HostedZoneName,ParameterValue=example.com \
    ParameterKey=HostedZoneId,ParameterValue=Z1234567890ABC \
    ParameterKey=CustomSubdomainName,ParameterValue=wordpress \
  --capabilities CAPABILITY_NAMED_IAM CAPABILITY_AUTO_EXPAND
```

Wait and read outputs:

```bash
aws cloudformation wait stack-create-complete \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"

aws cloudformation describe-stacks \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME" \
  --query 'Stacks[0].Outputs[].[OutputKey,OutputValue]' \
  --output table
```

## 5. Cleanup

Delete the stack:

```bash
aws cloudformation delete-stack \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"

aws cloudformation wait stack-delete-complete \
  --region "$AWS_REGION" --profile "$AWS_PROFILE" \
  --stack-name "$STACK_NAME"
```

Optional cleanup:

- Delete old AMIs and snapshots created by `Wordpress-PreStageSnapshot`.
- Delete `/Wordpress/DBPassword` and `/Wordpress/DBRootPassword` if you are done.
- For custom domains, ACM validation CNAMEs may remain for renewal/replacement.
  Remove stale records from Route 53 only when no active certificate uses that
  domain.

## How The Pieces Fit Together

- [stages/](stages/README.md): what each stage adds and how the architecture evolves.
- [nested/](nested/README.md): how the nested CloudFormation modules fit together.
- [ssm-documents/](ssm-documents/README.md): why SSM Automation owns migration steps.

## Legacy Stacks

The older monolithic templates are in [legacy/](legacy/). They are not the
current path, not fully tested in this final workflow, and not maintained.
