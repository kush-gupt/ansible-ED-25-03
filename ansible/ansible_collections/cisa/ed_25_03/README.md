### CISA ED 25-03: Core Dump & Hunt Automation (Cisco ASA)

This Ansible collection implements scripted assistance for key steps from CISA's Supplemental Direction ED 25-03 for Cisco ASA devices: artifact collection, optional core dump collection (heavily gated), patch-related checks, and syslog hunt helpers.

Reference: [CISA ED 25-03: Core Dump & Hunt Instructions](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)

Important: Follow CISA's guidance exactly and in order. Deviations may trigger anti-forensics and destroy evidence. This automation aims to issue exact CLI commands without tab completion or autocomplete.

Prerequisites
    - Ansible 2.15+
    - Collections: `cisco.asa`, `ansible.netcommon`
    - Connectivity to Cisco ASA via SSH using `network_cli`

Inventory example (hosts.ini)
```
[asa]
asa1 ansible_host=10.0.0.10

[all:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=cisco.asa.asa
ansible_user=asa_admin
ansible_ssh_pass=REDACTED
ansible_become=no
```

Roles
    - `collect_artifacts`: Runs initial commands (show checkheaps twice with wait, show tech-support detail, implant grep), saves outputs locally, and reports indicators.
    - `core_dump`: Prepares and triggers a core dump per ED 25-03 (requires explicit, multiple acks). WARNING: causes immediate reload/outage.
    - `patch_checks`: Looks for `firmwareupdate.log` on disk0: and captures content when present.
    - `hunt_syslog_checks`: Simple local log checks for ASA message IDs to help detect tampering and activity timeline.
    - `version_assessment`: Parses ASA version and flags likely vulnerability to CVE-2025-20333 and train safety (>=9.20).
    - `report`: Aggregates results into `ed25_report.json` with advisory next steps aligned to ED 25-03.

Playbooks (located in `playbooks/`)
    - `collect_artifacts.yml`
    - `core_dump.yml`
    - `patch_checks.yml`
    - `hunt_syslog_checks.yml`
    - `assess_and_report.yml`

Usage:
```bash
# Run a specific playbook from the collection
ansible-playbook kush_gupt.ed_25_03.collect_artifacts

# Or run using the full path
ansible-playbook ansible_collections/cisa/ed_25_03/playbooks/collect_artifacts.yml
```

Legal and Risk Notice
    - Use at your own risk. This may disrupt production traffic and trigger device reloads when core dump is executed.
    - Review and adapt to your environment. Consult CISA if compromise is suspected before deviating from guidance.
    - See repository `disclaimer.md`.


