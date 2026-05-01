# Stage Architecture

The files in this directory are deployable root CloudFormation templates. Package
them from the repository root before uploading them to CloudFormation.

## Stage 1: Single Server

`stage1.yaml` creates the baseline lab:

- VPC, public subnets, IAM, SSM documents, and shared SSM parameters.
- One EC2 instance running Apache, WordPress, and local MariaDB.
- WordPress files and database rows live on the instance.

Use this stage to see the simplest architecture and to create content before the
first update.

## Stage 2: Launch Template

`stage2.yaml` keeps the single-server architecture but launches EC2 through a
launch template.

Difference from stage 1:

- Compute becomes easier to replace from an AMI.
- The stage 1 instance can be imaged with `Wordpress-PreStageSnapshot`, then the
  same stack can update to stage 2 with `SourceAmiId`.

## Stage 3: Database Split

`stage3.yaml` introduces RDS MySQL.

Greenfield mode:

- Use `Stage3Mode=cutover` and leave `SourceAmiId` empty.
- WordPress starts fresh on EC2 and stores data in RDS.

Same-stack update mode:

1. Update from stage 2 to `Stage3Mode=prepare`.
2. Run `Wordpress-MigrateLocalMariaDbToRds`.
3. Image the still-running stage 2 instance.
4. Update again with `Stage3Mode=cutover` and the new `SourceAmiId`.

The AMI keeps local files such as `wp-content/uploads`; RDS keeps database rows.

## Stage 4: File Split

`stage4.yaml` introduces EFS for `wp-content`.

Difference from stage 3:

- RDS remains the database.
- EFS becomes the shared WordPress content store.
- The single app instance gets a stable Elastic IP URL.

For same-stack updates, image the stage 3 instance first and pass that AMI as
`SourceAmiId`. Stage 4 copies local `wp-content` from the AMI into EFS.

## Stage 5: Load Balanced App Tier

`stage5.yaml` replaces the single app instance with an ALB and Auto Scaling Group.

Difference from stage 4:

- RDS remains durable database state.
- EFS remains durable media/content state.
- EC2 instances become replaceable app nodes behind the ALB.
- Optional Route 53 + ACM parameters can publish `https://subdomain.example.com`.

Stage 4 to 5 does not need another AMI because the durable state has already
moved to RDS and EFS.
