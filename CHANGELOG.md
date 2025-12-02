# Changelog

All notable changes are documented here following [Keep a Changelog](https://keepachangelog.com/) conventions.

## [December 2025]
### Changed - Stage 1: Single Server
- Upgraded Amazon Linux 2 to **Amazon Linux 2023 (kernel 6.12)** 🎉
- Modernized AWS CLI calls: replaced `sed` parsing with `--output text` flag
- Fixed region from hardcoded `us-east-1` to dynamic `${AWS::Region}`
- Secured WordPress download: `http://` → `https://`
- Added PHP extensions: `mbstring`, `xml`, `gd`, `opcache`, `intl`, `zip`
- Removed unnecessary `sudo` calls and redundant `php` package

### Fixed - Stage 1: Single Server
- Corrected MariaDB package to `mariadb105-server` for AL2023 compatibility
- Fixed cfn-signal reliability

---

*See commit log for more details or changes to other architecture stages.*
