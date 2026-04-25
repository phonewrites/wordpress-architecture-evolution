# Changelog

All notable changes are documented here following [Keep a Changelog](https://keepachangelog.com/) conventions.

## [April 2026]

Cumulative updates to stages 1–3 CloudFormation templates since the January 2026 baseline on `main` (diff from last `main` commit dated January 2026 to the current branch). This section documents only `stage1_SingleServer.yaml`, `stage2_SingleServerLT.yaml`, and `stage3_DbSplitFromServer.yaml`.

### Added
- Stage 1: Parameters `CfnMetadataNonce`, `EnableSnapshotStack`, `SnapshotNonce`; optional `SnapshotHelperLambda` + `Custom::WordpressRootEbsSnapshot` (root EBS snapshot, outputs `WordpressRootSnapshotId` / `WordpressRootVolumeId`); `AWS::CloudFormation::Init` on `WordpressEC2` with `wordpress_always` (timestamp log + `01_wp_sync_public_ip`); `cfn-hup` to re-run `cfn-init` on metadata changes; `WordpressRole` adds `AWSCloudFormationReadOnlyAccess`; `wp-config.php` snippet so `WP_HOME` / `WP_SITEURL` follow `HTTP_HOST`; outputs `WordpressURL` and `WordpressInstanceAutoIP`.
- Stage 2: Same snapshot and nonce parameters as stage 1; `RestoreRootSnapshotId` selects `FreshWordpressLT` (greenfield) vs `WordpressLT` restored from `Custom::RestoreRootSnapshotAmi`; `WordpressEC2FromLT` with CreationPolicy + `cfn-signal`; `cfn-init` / `cfn-hup` / `01_wp_sync_public_ip` aligned with stage 1 (IMDS public hostname when present, `wp search-replace` for stale URLs including EC2 DNS / IPv4 variants).
- Stage 3: Same-stack parameters `MigrateFromStage2LocalMariaDb`, `SourceWordpressInstanceId`, `UploadsSourceEbsSnapshotId`, `CfnMetadataNonce`, and template Rules when migration is on; `RdsLocalMariaMigrateLambda` + `Custom::LocalMariaDbToRdsMigration` (SSM on the stage 2 instance: `mysqldump` from local MariaDB, import to RDS, stop MariaDB, point `wp-config` at RDS); `WordpressUploadsSourceVolume` from optional snapshot; `WordpressAppTierLT` UserData attaches/uploads flow (poll volume, optional force-detach from a previous instance, attach `/dev/sdf`, mount first XFS partition with `nouuid`, `rsync` into `wp-content/uploads`); `cfn-init` hook `restore_uploads_from_ebs_snapshot_volume` and parameter `WpContentUploadsRestore` for stack-driven re-restore; `WordpressEC2` CreationPolicy timeout `PT45M`; instance role includes `AmazonEC2FullAccess` for the EC2 volume API calls used in that path.

### Changed
- Stages 1–3: Default AL2023 AMI SSM path to kernel 6.18; stages 1–2 MariaDB package to `mariadb1011-server`; public subnets set `AssignIpv6AddressOnCreation: true` alongside existing `Ipv6CidrBlock` suffix scheme.
- Stages 1–3: Bootstrap/UserData patterns aligned (e.g. `dnf -y --refresh upgrade`, install `aws-cfn-bootstrap` where needed; stage 3 adds `trap` + `cfn-signal` wrapper for early bootstrap failures).
- Stage 3: Launch template logical id `WordpressAppTierLT` (replaces `WordpressLaunchTemplate`); `WordpressEC2` `DependsOn` includes `RDSinstance` and `LocalMariaDbToRdsMigration` so RDS and the migration custom resource (no-op when disabled) run before the app instance create.

### Fixed
- Stages 1–3 public subnets: removed the IPv6 workaround custom resource (Lambda used `is` instead of `==` on `RequestType` and Delete handling was unreliable); use native `AssignIpv6AddressOnCreation` on public subnets instead. Stage 5 still ships the older workaround pattern on public subnets.
- Stage 3: Media uploads failed on greenfield AL2023 (`Unable to create directory … wp-content/uploads/…`) because `wp-content` stayed `httpd_sys_content_t` under SELinux; UserData now ensures `wp-content/uploads` exists, normalizes modes after optional snapshot rsync, and applies `chcon -R -t httpd_sys_rw_content_t` on `wp-content` when SELinux is Enforcing or Permissive. The `cfn-init` uploads-restore hook applies the same labeling after rsync.
- Stage 4: Same upload symptom on greenfield AL2023: UserData ensures local `wp-content/uploads`, `chcon` to `httpd_sys_rw_content_t`, and persistent `setsebool -P httpd_use_nfs 1` so httpd can write once `wp-content` is mounted on EFS (NFS). The uploads-restore `cfn-init` hook applies `chcon`, modes, and `httpd_use_nfs` after rsync; **`migrate_wp_content_to_efs`** applies directory/file modes and `httpd_use_nfs` after the live EFS mount (including when `wp-content` was already an EFS mountpoint).

### Removed
- Stages 1–3: `IPv6WorkaroundLambda`, `IPv6WorkaroundRole`, and per-subnet `Custom::SubnetModify` helpers on public subnets A/B/C.

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
