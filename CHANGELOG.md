# Changelog

All notable changes are documented here following [Keep a Changelog](https://keepachangelog.com/) conventions.

## [May 2026]

### Added
- Added nested root templates for stages 1–5, backed by reusable `nested/` modules and packaged `build/` artifacts.
- Added SSM Automation documents for pre-stage AMI creation and local MariaDB to RDS migration.
- Stage 5 can now create an optional HTTPS custom domain using Route 53 and ACM, with ALB DNS fallback.
- Added concise runbook and architecture notes in `README.md`, `stages/`, `nested/`, `ssm-documents/`, and `legacy/`.

### Changed
- Current deployment path now uses `stages/` root templates; old monolithic templates moved under `legacy/`.
- Stage evolution now uses explicit operator-run SSM Automation followed by CloudFormation stack updates.
- Stage 5 now reuses existing RDS and EFS resources during stage 4 to stage 5 updates, then points WordPress at the ALB or custom domain URL.
- RDS instance identifiers now use the root stack name while keeping `wordpressdb` as the WordPress schema name.

### Fixed
- Stage 1 no longer tries to create empty SSM parameters for later-stage values.
- Nested Stage 3 greenfield stacks now leave `wp-content/uploads` writable on AL2023 with SELinux enabled.
- Stage 5 now handles HTTPS admin redirects, ASG rolling updates, and mixed-case ALB URL rewrites during custom-domain updates.
- Stage 5 no longer overwrites existing EFS `wp-content` when ASG instances start.

## [April 2026]

### Added
- Stage 1 can now create optional root-volume snapshots after WordPress is configured, giving later same-stack updates a data-preserving source.
- Stage 2 can now run either as a fresh stack or as a same-stack continuation from stage 1 by booting from the captured root snapshot.
- Stage 3 can now preserve both database content and uploaded media during stage 2 to stage 3 same-stack updates.
- Stages 1–4 now refresh WordPress URLs from stack metadata changes so public URL/IP changes do not strand the site on stale addresses.

### Changed
- Stages 1–3 use the newer AL2023 kernel 6.18 AMI path, updated MariaDB package choices, and native public-subnet IPv6 assignment.
- Bootstrap flow is more consistent across stages, including clearer CloudFormation signaling when instance setup succeeds or fails.
- Stage 3 now creates the application tier only after the database and optional migration path are ready.
- Stage 4 can now keep uploaded media intact while moving `wp-content` onto EFS during a stage 3 to stage 4 update.

### Fixed
- Stages 1–3 no longer rely on the older IPv6 workaround custom resource for public subnet IPv6 behavior.
- Stage 3 greenfield and migration paths now leave `wp-content/uploads` writable on AL2023 with SELinux enabled.
- Stage 4 now keeps uploads writable after the local-to-EFS cutover, including greenfield stacks and same-stack updates from stage 3.

### Removed
- Stages 1–3 removed the custom IPv6 subnet workaround in favor of native subnet properties.

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
