# SSM Automation Documents

This directory holds CloudFormation templates that create `AWS::SSM::Document`
resources with `DocumentType: Automation`.

Current runbooks:

- `pre-stage-snapshot.yaml` - create an operator-controlled pre-upgrade snapshot or AMI.
- `migrate-local-mariadb-to-rds.yaml` - dump local MariaDB and import into RDS.

Planned runbooks:

- `copy-local-wp-content-to-efs.yaml` - copy WordPress content files to EFS.
- `rewrite-wordpress-url.yaml` - update `home`, `siteurl`, and embedded content URLs.
- `release-stage4-instance.yaml` - release singleton instance termination protection before stage 5.
- `validate-wordpress-state.yaml` - smoke-check posts, media paths, DB endpoint, and wp-content writability.

Runbooks are intentionally invoked by the operator in this iteration, preferably
from the Systems Manager Automation console for beginner-facing walkthroughs:

1. Start SSM Automation.
2. Confirm it succeeds.
3. Update the CloudFormation stack to the next stage.

CloudFormation-triggered lifecycle automation is reserved for a later iteration.
