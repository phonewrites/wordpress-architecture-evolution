# Stage Root Templates

This directory will hold the deployable root templates:

- `stage1.yaml` - standalone EC2 with local MariaDB.
- `stage2.yaml` - standalone EC2 launched through a launch template.
- `stage3.yaml` - singleton EC2 app tier backed by RDS.
- `stage4.yaml`
- `stage5.yaml`

Each stage root template should be valid for greenfield deployment. Same-stack
evolution is still explicit: run the relevant SSM Automation document, then update
the same CloudFormation stack to the next stage template.

Nested stack `TemplateURL` values may point at local files in source form during
development. Release artifacts should include packaged root templates whose child
stack URLs point at S3, so beginners can upload them through the CloudFormation
console.
