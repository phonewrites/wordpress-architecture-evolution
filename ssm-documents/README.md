# SSM Automation Documents

These documents hold the imperative steps that should not be modeled as
CloudFormation resources. CloudFormation creates the documents through
`nested/automation.yaml`; the operator runs them between stage updates.

## Documents

- `Wordpress-PreStageSnapshot`: creates an AMI from the current WordPress EC2
  instance and tags the AMI and backing snapshot with the source stage, stack, and
  instance ID.
- `Wordpress-MigrateLocalMariaDbToRds`: dumps local MariaDB from the stage 2
  instance and imports it into the stage 3 RDS database.

## Where They Fit

- Stage 1 -> 2: run `Wordpress-PreStageSnapshot` against the stage 1 instance,
  then update to stage 2 with `SourceAmiId`.
- Stage 2 -> 3: update to stage 3 `prepare`, run
  `Wordpress-MigrateLocalMariaDbToRds`, run `Wordpress-PreStageSnapshot` against
  the same stage 2 instance, then update to stage 3 `cutover`.
- Stage 3 -> 4: run `Wordpress-PreStageSnapshot` against the stage 3 instance,
  then update to stage 4 with `SourceAmiId`.
- Stage 4 -> 5: no SSM document is needed because state already lives in RDS and
  EFS.

