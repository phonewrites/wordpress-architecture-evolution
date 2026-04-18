# Changelog

All notable changes are documented here following [Keep a Changelog](https://keepachangelog.com/) conventions.

## [April 2026]

### Added
- **Stage 1**: Optional root EBS snapshot path (`EnableSnapshotStack`, `SnapshotNonce`, Lambda + `Custom::WordpressRootEbsSnapshot`); `CfnMetadataNonce` + `AWS::CloudFormation::Init` + **cfn-init** / **cfn-hup** for stack-driven on-instance refresh; `WordpressInstanceProfile` / `WordpressRole` include **AWSCloudFormationReadOnlyAccess** for those helpers
- **Stage 1**: Outputs **`WordpressURL`** (HTTP + `PublicDnsName`) and **`WordpressInstanceAutoIP`** (`PublicIp`) so stack updates tolerate a stopped instance better than IP-only URLs
- **Stage 2**: Optional **`RestoreRootSnapshotId`** (`snap-…`) with **`Custom::RestoreRootSnapshotAmi`** + conditional restore Launch Template so the same stack can move from Stage 1 while keeping the root volume contents; **`FreshWordpressLT`** is created only when **`RestoreRootSnapshotId`** is empty (standalone Stage 2)
- **Stage 2**: Same optional snapshot stack parameters as Stage 1; **`WordpressEC2FromLT`** + **CreationPolicy** + **cfn-signal**; public subnets use **native dual-stack IPv6** like Stage 1 (no IPv6 workaround Lambda on Stage 2)
- **Stages 3–5**: **IMDSv2-only** instance metadata on the WordPress Launch Template (`MetadataOptions` / `HttpTokens: required`)
- **Stage 3**: **`WordpressInstanceAutoIP`** output and **`WordpressURL`** wording aligned with Stages 1–2

### Changed
- **Stage 1–2**: Default AL2023 AMI SSM path to **kernel 6.18** (`al2023-ami-kernel-6.18-x86_64`); MariaDB package to **`mariadb1011-server`**; bootstrap, browser-hostname **`wp-config`** handling, and post-install signals aligned between Stage 1 and Stage 2 fresh-install paths
- **Stage 2**: Launch Template–backed single instance (`WordpressEC2FromLT`, `FreshWordpressLT` / `WordpressLT` split) replacing the earlier single-LT layout; UserData and Init paths aligned with Stage 1 for easier maintenance
- **Stages 3–5**: IPv6 subnet helper Lambda + custom resources **still present** (not removed vs ~3 months ago—only **Stages 1–2** dropped that pattern); Lambda now uses **`==`** for **`RequestType`** comparisons (Python) and passes a physical resource id on Delete responses

### Fixed
- IPv6 workaround Lambda **Delete** handling in Stages **3–5** (avoid incorrect **`is`** comparison on request type)

### Removed
- **Stages 1–2 only**: IPv6 subnet helper Lambda + custom resources on public subnets (replaced by **`AssignIpv6AddressOnCreation`** and **`Ipv6CidrBlock`** on each public subnet). **Stages 3–5** still use the Lambda + custom-resource pattern from earlier revisions.

## [January 2026]

### Added
- Elastic IP resource in Stage 4 for WordPress URL persistence across instance restarts/replacements
- Enhanced auto-scaling strategy in Stage 5 with dual-metric approach (CPU credits + CPU utilization)
- CloudWatch alarms for CPU credit monitoring (t2.micro burst capacity management)
- Output URLs for all stages for easy access after deployment
- Comprehensive inline documentation across all CloudFormation templates
- Amazon EFS utilities (`amazon-efs-utils`) package in Stages 4-5

### Changed
- **All Stages**: Upgraded Amazon Linux 2 to **Amazon Linux 2023** (kernel 6.12) 🎉
- **All Stages**: Modernized AWS CLI parameter parsing - replaced `sed` manipulation with `--output text` flag
- **All Stages**: Region references changed from hardcoded `us-east-1` to dynamic `${AWS::Region}`
- **All Stages**: WordPress download protocol changed from `http://` to `https://` for security
- **All Stages**: VPC CIDR block changed from `10.16.0.0/16` to `10.125.0.0/16`
- **All Stages**: SSM parameter naming simplified from `/A4L/Wordpress/*` to `/Wordpress/*`
- **All Stages**: Enhanced PHP extensions - added `php-mbstring`, `php-xml`, `php-gd`, `php-opcache`, `php-intl`, `php-zip`
- **All Stages**: AMI selection automated via SSM Parameter Store reference
- **Stage 1-2**: MariaDB package corrected to `mariadb105-server` for AL2023 compatibility
- **Stage 5**: Auto-scaling thresholds optimized for t2.micro free-tier instances
- **Stage 5**: CloudWatch alarm evaluation periods tuned for responsive scaling (5-7 minute windows)
- **Stage 5**: Scaling policies use "M of N" datapoint thresholds for stable decision-making

### Fixed
- CloudFormation signal (`cfn-signal`) reliability improved with proper error handling
- WordPress installation sequencing - now installs before EFS mount to prevent permission issues
- MySQL syntax errors in database configuration scripts
- IAM instance profile attachment for EC2 instances

### Removed
- Unnecessary `sudo` calls in UserData scripts (already running as root)
- Redundant `php` package specification (covered by specific PHP modules)

---

*See commit log for granular changes to specific stages or configuration details.*
