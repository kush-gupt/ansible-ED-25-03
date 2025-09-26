# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2025-09-26

### Added
- Initial release of CISA ED 25-03 Ansible collection
- `version_assessment` role for ASA version analysis and vulnerability detection
- `collect_artifacts` role for gathering device information and logs
- `core_dump` role for emergency core dump procedures (with safety checks)
- `hunt_syslog_checks` role for analyzing syslog files for compromise indicators
- `patch_checks` role for verifying firmware update status
- Comprehensive playbooks for ED 25-03 compliance workflow
- Support for CVE-2025-20333 and CVE-2025-20362 vulnerability assessment
- Automated device type detection (ASA, ASAv, FTD, ASA-SM)
- JSON reporting with consolidated findings and recommendations

### Security
- Core dump functionality requires explicit acknowledgment and confirmation
- All operations designed with safety-first approach
- Comprehensive input validation and error handling

[0.1.0]: https://github.com/kush-gupt/ansible-ED-25-03/releases/tag/v0.1.0
