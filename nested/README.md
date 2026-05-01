# Nested Templates

The stage roots in `stages/` compose these nested templates. The root stack owns
the user-facing workflow; the nested stacks keep infrastructure layers stable as
the application architecture evolves.

## Shared Layers

- `network.yaml`: VPC, IPv4/IPv6 networking, public subnets, app subnets, database
  subnets, route tables, and internet gateway.
- `iam.yaml`: EC2 instance profile and SSM Automation role.
- `automation.yaml`: creates the SSM Automation documents used during updates.
- `shared-ssm.yaml`: writes fixed `/Wordpress/...` SSM parameters used by app
  bootstrap scripts.

These layers appear in most stages and keep names/outputs consistent.

## Application Layers

- `single-server.yaml`: stage 1 EC2 with Apache, WordPress, and local MariaDB.
- `single-server-lt.yaml`: stage 2 EC2 using a launch template and optional
  `SourceAmiId`.
- `app-rds.yaml`: stage 3 single app instance backed by RDS.
- `app-efs.yaml`: stage 4 single app instance backed by RDS and EFS.
- `alb-asg.yaml`: stage 5 ALB, target group, launch template, Auto Scaling Group,
  scaling alarms, optional ACM certificate, and optional Route 53 alias.

## Durable State Layers

- `database.yaml`: RDS MySQL, DB subnet group, database security group, and
  `/Wordpress/DBEndpoint` integration.
- `content-storage.yaml`: EFS file system, mount targets, and security group for
  shared `wp-content`.

## Integration Rules

- Package root templates before deployment so local nested paths become S3
  `TemplateURL` values.
- Keep durable nested stacks stable across updates: RDS and EFS should survive app
  tier replacement.
- Keep imperative data movement in SSM Automation, not CloudFormation custom
  resources.
- Do not add new Lambda-backed custom resources for this lab path.
