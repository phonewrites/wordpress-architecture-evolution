# WordPress App AWS Architecture Evolution

CloudFormation (IaC) version of the **[manual demo project](https://github.com/acantril/learn-cantrill-io-labs/tree/master/aws-elastic-wordpress-evolution)** from one of Adrian Cantrill's courses.

## Architecture Stages

This project demonstrates a step-by-step evolution of WordPress architecture on AWS:

1. **Stage 1**: Single EC2 instance (Apache + WordPress + MariaDB)
2. **Stage 2**: Single EC2 instance with Launch Template
3. **Stage 3**: EC2 instance + separate RDS MySQL database
4. **Stage 4**: EC2 instance + RDS + EFS shared storage
5. **Stage 5**: Auto-scaling with ALB + ASG + RDS + EFS

Each stage builds upon the previous one, introducing new AWS services and architectural patterns to improve scalability, reliability, and maintainability.

> ⚠️ Each stage is a **self-contained, independent CloudFormation stack**. They represent different architectural phases, not an in-place upgrade path. Deploying a new stage template to an existing stack will replace resources (like EC2 instances), destroying your WordPress application setup. Deploy each stage as a fresh stack to demonstrate that specific architectural pattern.

## Prerequisites

- AWS Account with appropriate permissions
- AWS CLI configured
- Basic understanding of CloudFormation, EC2, VPC, and RDS

## Required SSM Parameters

> ⚠️ SSM SecureString parameters cannot be created directly in CloudFormation templates and must be created manually before deployment.

All stages require SSM SecureString parameters for database credentials:
- ❌ Avoid using `/`, `@`, `"`, or spaces in password values
- Parameters are imported in UserData scripts using `get-parameters --with-decryption`
- RDS instances reference parameters using CloudFormation dynamic references

**For Stages 1-2** (Local MariaDB on EC2):
```bash
aws ssm put-parameter --name "/Wordpress/DBPassword" \
  --description "Wordpress Database Password" \
  --type "SecureString" \
  --value "YOUR_DB_PASSWORD"

aws ssm put-parameter --name "/Wordpress/DBRootPassword" \
  --description "Wordpress Database Root Password" \
  --type "SecureString" \
  --value "YOUR_ROOT_PASSWORD"
```

**For Stages 3-5** (RDS MySQL):
```bash
aws ssm put-parameter --name "/Wordpress/DBPassword" \
  --description "Wordpress Database Password" \
  --type "SecureString" \
  --value "YOUR_DB_PASSWORD"
```
<hr style="border-top:6px solid #333; margin:24px 0;" />
<br />
<br />

# Deployment Instructions

> 💡 **Deployment Strategy**: Each stage is a complete, standalone stack with its own VPC and all required resources. To explore the architectural evolution:
> 1. Deploy a stage, configure WordPress, and explore the architecture
> 2. Delete the stack when done: `aws cloudformation delete-stack --stack-name <stack-name>`
> 3. Deploy the next stage with a different stack name to see the next architectural phase

### Stage 1: Single Server (All-in-One)

**Architecture**: EC2 instance running Apache, WordPress, and MariaDB locally.

**Deploy**:
```bash
aws cloudformation create-stack --stack-name wordpress-stage1 \
  --template-body file://stage1_SingleServer.yaml \
  --capabilities CAPABILITY_IAM
```

**Access**: After deployment, get the WordPress URL from stack outputs:
```bash
aws cloudformation describe-stacks --stack-name wordpress-stage1 \
  --query 'Stacks[0].Outputs[?OutputKey==`WordpressURL`].OutputValue' --output text
```

### Stage 2: Single Server with Launch Template

**Architecture**: Same as Stage 1 but uses Launch Template for easier future updates.

**Deploy**:
```bash
aws cloudformation create-stack --stack-name wordpress-stage2 \
  --template-body file://stage2_SingleServerLT.yaml \
  --capabilities CAPABILITY_IAM
```

### Stage 3: Separate Database (RDS)

**Architecture**: EC2 instance for WordPress + separate RDS MySQL database.

**Benefits**: Better scalability, automated backups, easier database management.

**Deploy**:
```bash
aws cloudformation create-stack --stack-name wordpress-stage3 \
  --template-body file://stage3_DbSplitFromServer.yaml \
  --capabilities CAPABILITY_IAM
```

### Stage 4: Shared Storage (EFS)

**Architecture**: EC2 instance + RDS + EFS for shared wp-content directory.

**Benefits**: Enables horizontal scaling in Stage 5. Static IP via Elastic IP ensures WordPress URL persistence.

**Deploy**:
```bash
aws cloudformation create-stack --stack-name wordpress-stage4 \
  --template-body file://stage4_FsSplitFromServer.yaml \
  --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND
```

### Stage 5: Auto-Scaling Architecture

**Architecture**: ALB + Auto Scaling Group + RDS + EFS for fully scalable deployment.

**Benefits**: Automatic scaling based on demand, high availability across multiple AZs.

**Deploy**:
```bash
aws cloudformation create-stack --stack-name wordpress-stage5 \
  --template-body file://stage5_AlbAsg.yaml \
  --capabilities CAPABILITY_IAM
```

**Access**: Use ALB DNS name from outputs for consistent access to WordPress installation.
<br />
<br />

### Auto-Scaling Strategy used in Stage 5

Stage 5 implements a **hybrid auto-scaling approach** optimized for the free-tier t2.micro instances with CPU credit management.

### Scaling Policies

1. **Scale-Out (Add Instances)**:
   - **CPU Credits Low**: When any instance drops to 3 or fewer CPU credits
   - **High CPU Utilization**: When average CPU exceeds 50% for 5 minutes
   
2. **Scale-In (Remove Instances)**:
   - **Low CPU Utilization**: When average CPU stays below 15% for 7 minutes

### CloudWatch Alarms

- **WpLowCreditAlarm**: Uses `Minimum` statistic to detect *any* instance in critical state
- **WpHighCpuAlarm**: Uses `Average` statistic across all instances
- **WpLowCpuAlarm**: Uses `Average` statistic for scale-in decisions

### t2.micro Considerations

- **CPU Credits**: 144 max, earns 6/hour, consumes at variable rate based on usage
- **Baseline Performance**: 10% CPU when credits exhausted
- **Credit Monitoring**: Essential to prevent performance degradation
- **Responsive Scaling**: 5-7 minute evaluation windows for quick response

This hybrid approach ensures WordPress remains responsive during traffic spikes while efficiently managing costs during low-traffic periods.
<br />
<br />

## Network Architecture

All stages use a VPC with:
- **CIDR**: 10.125.0.0/16
- **Public Subnets**: 3 subnets across 3 Availability Zones (for high availability)
- **Database Subnets** (Stages 3-5): 3 private subnets for RDS
- **App Subnets** (Stages 4-5): 3 subnets for EFS mount targets
- **IPv4 and IPv6**: Dual-stack support enabled
- **Internet Gateway**: For public internet access

### Subnet Tiers

- **Public/App Tier**: Hosts EC2 instances (and ALB in Stage 5)
- **Database Tier**: RDS MySQL instances
- **EFS Tier**: Elastic File System mount targets

## Security Groups

Security groups follow the principle of least privilege:
- **wordpress-sg**: Allows HTTP (port 80) from anywhere; used by EC2/ALB
- **database-sg**: Allows MySQL (port 3306) only from WordPress security group
- **efs-sg**: Allows NFS (port 2049) only from WordPress security group
- **alb-sg** (Stage 5): Allows HTTP (port 80) from anywhere


### IPv6 Configuration

CloudFormation doesn't natively support the `AssignIpv6AddressOnCreation` subnet attribute. The templates include a Lambda-backed custom resource to enable this feature automatically.

## Cleanup

To delete a deployed stage and all its resources:
```bash
aws cloudformation delete-stack --stack-name [STACK-NAME]
```

### 👉 SSM SecureString parameters must be deleted manually:

```bash
aws ssm delete-parameter --name "/Wordpress/DBPassword"
aws ssm delete-parameter --name "/Wordpress/DBRootPassword"
```

## References
- Original manual lab instructions: [acantril/learn-cantrill-io-labs](https://github.com/acantril/learn-cantrill-io-labs/tree/master/aws-elastic-wordpress-evolution)
- AWS SSM Parameter Store guide: [Using SecureString Parameters in CloudFormation](https://aws.amazon.com/blogs/mt/using-aws-systems-manager-parameter-store-secure-string-parameters-in-aws-cloudformation-templates/)

---

## Major Changes from Original Project

This repository is a complete Infrastructure as Code (IaC) re-implementation of Adrian Cantrill's manual WordPress evolution lab, providing fully automated CloudFormation templates instead of manual console instructions.

- Self-contained CloudFormation templates with embedded VPC infrastructure for each stage (original used separate VPC template + manual resource creation)
- One-command deployment per stage instead of manual AWS Console configuration of EC2, RDS, EFS, ALB, and ASG
- Region-agnostic design with dynamic `${AWS::Region}` references instead of hardcoded `us-east-1`
- Upgraded from Amazon Linux 2 to Amazon Linux 2023 with enhanced PHP extensions and corrected package names
- Improved AWS CLI parsing using `--output text` flag instead of `sed` string manipulation
- Secured WordPress download using `https://` instead of `http://`
- Automated AMI selection via SSM Parameter Store
- Simplified SSM parameter naming from `/A4L/Wordpress/*` to `/Wordpress/*`
- Added Elastic IP in Stage 4 for WordPress URL persistence across instance restarts
- Implemented hybrid auto-scaling in Stage 5 optimized for t2.micro free-tier instances with dual-metric approach (CPU credits + utilization)
- CloudFormation signals (`cfn-signal`) for deployment validation and comprehensive inline documentation
- Output URLs provided for all stages for easy access after deployment
