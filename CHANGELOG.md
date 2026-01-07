# Changelog

All notable changes are documented here following [Keep a Changelog](https://keepachangelog.com/) conventions.

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
