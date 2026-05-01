# Nested Templates

This directory holds CloudFormation modules used by the stage root templates.

Current modules:

- `network.yaml` - VPC, IPv4/IPv6 CIDRs, public/app/database subnets, route tables, IGW.
- `shared-ssm.yaml` - fixed `/Wordpress/...` Parameter Store names and public endpoint values.
- `iam.yaml` - EC2 instance profiles and SSM Automation roles.
- `automation.yaml` - SSM Automation role, pre-stage AMI document, and local MariaDB to RDS migration document.
- `single-server.yaml` - stage 1 standalone EC2 with local MariaDB.
- `single-server-lt.yaml` - stage 2 standalone EC2 via launch template.
- `database.yaml` - RDS subnet group, database security group, and RDS MySQL instance.
- `app-rds.yaml` - stage 3 singleton WordPress app instance backed by RDS.

Planned modules:

- `filesystem.yaml` - EFS file system and mount targets.
- `singleton-app.yaml` - stage 4 single WordPress app instance with EFS.
- `alb-asg.yaml` - stage 5 ALB, target group, ASG, launch template, scaling policy.
- `dns-acm.yaml` - optional Route53/ACM custom domain support.

Do not add Lambda-backed custom resources here. Imperative migration work belongs in
SSM Automation documents under `ssm-documents/`.
