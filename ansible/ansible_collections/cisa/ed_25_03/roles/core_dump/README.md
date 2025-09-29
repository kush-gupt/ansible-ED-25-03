### Role: core_dump

Implements ED 25-03 Part Two (Core Dump Collection) steps on Cisco ASA.

⚠️ **WARNING**: The action `crashinfo force page-fault` causes immediate reload and may lead to outages. Strong gating variables are required to run. This role does NOT perform the power cycle; ensure manual reboot as advised by CISA prior to capture.

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `core_dump_ed_25_03_perform_core_dump` | `false` | **REQUIRED**: Must be explicitly set to `true` to enable core dump |
| `core_dump_ed_25_03_confirm_manual_power_cycle` | `false` | **REQUIRED**: Must be explicitly set to `true` to confirm manual power cycle understanding |
| `core_dump_ed_25_03_core_dump_ack_text` | `""` | **REQUIRED**: Explicit acknowledgment message (must be non-empty) |
| `core_dump_ed_25_03_filesystem` | `disk0:` | Filesystem typically disk0: |
| `core_dump_ed_25_03_attempt_reconnect` | `false` | Optional attempt to reconnect after reload (best effort) |
| `core_dump_ed_25_03_expected_reload_seconds` | `180` | Expected time for device reload |
| `core_dump_ed_25_03_output_dir` | `{{ (playbook_dir \| default('.')) + '/artifacts/' + inventory_hostname }}` | Output directory on control node for notes |

## Usage Examples

### ⚠️ DANGEROUS - Production Core Dump

```yaml
---
- name: ED 25-03 Core Dump Collection (DANGEROUS)
  hosts: asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.core_dump
      vars:
        core_dump_ed_25_03_perform_core_dump: true
        core_dump_ed_25_03_confirm_manual_power_cycle: true
        core_dump_ed_25_03_core_dump_ack_text: "I acknowledge this will cause immediate device reload and potential service outage. I have coordinated with stakeholders and am prepared for manual power cycle as required by CISA ED 25-03."
```

### Safe Mode - Preparation Only

```yaml
---
- name: ED 25-03 Core Dump Preparation (Safe)
  hosts: asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.core_dump
      # All safety variables remain false - no actual core dump will occur
```

### With Custom Filesystem and Reconnect Attempt

```yaml
---
- name: ED 25-03 Core Dump with Custom Settings
  hosts: asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.core_dump
      vars:
        core_dump_ed_25_03_perform_core_dump: true
        core_dump_ed_25_03_confirm_manual_power_cycle: true
        core_dump_ed_25_03_core_dump_ack_text: "Explicit acknowledgment of service disruption and manual power cycle requirement"
        core_dump_ed_25_03_filesystem: "flash:"
        core_dump_ed_25_03_attempt_reconnect: true
        core_dump_ed_25_03_expected_reload_seconds: 300
```

### Using the Collection Playbook

```bash
# Safe mode (preparation only)
ansible-playbook kush_gupt.ed_25_03.core_dump

# DANGEROUS - With explicit variables (not recommended via CLI)
ansible-playbook kush_gupt.ed_25_03.core_dump \
  -e "core_dump_ed_25_03_perform_core_dump=true" \
  -e "core_dump_ed_25_03_confirm_manual_power_cycle=true" \
  -e "core_dump_ed_25_03_core_dump_ack_text='Explicit acknowledgment message'"
```

### With Group Variables for Multiple Devices

```yaml
# group_vars/critical_asa.yml
core_dump_ed_25_03_perform_core_dump: true
core_dump_ed_25_03_confirm_manual_power_cycle: true
core_dump_ed_25_03_core_dump_ack_text: "Coordinated maintenance window - core dump authorized for ED 25-03 compliance"
core_dump_ed_25_03_attempt_reconnect: true
core_dump_ed_25_03_expected_reload_seconds: 240
```

```yaml
# playbook.yml
---
- name: ED 25-03 Core Dump Collection - Maintenance Window
  hosts: critical_asa
  gather_facts: false
  serial: 1  # Process one device at a time
  roles:
    - kush_gupt.ed_25_03.core_dump
```

### Conditional Execution Based on Compromise Detection

```yaml
---
- name: Conditional Core Dump Based on Assessment
  hosts: asa
  gather_facts: false
  tasks:
    - name: Run version assessment first
      include_role:
        name: kush_gupt.ed_25_03.version_assessment
    
    - name: Core dump only if compromise detected
      include_role:
        name: kush_gupt.ed_25_03.core_dump
      vars:
        core_dump_ed_25_03_perform_core_dump: "{{ version_assessment_compromise_detected | default(false) }}"
        core_dump_ed_25_03_confirm_manual_power_cycle: "{{ version_assessment_compromise_detected | default(false) }}"
        core_dump_ed_25_03_core_dump_ack_text: "Automated core dump triggered by compromise detection in ED 25-03 assessment"
      when: version_assessment_compromise_detected | default(false)
```

## Safety Mechanisms

The role includes multiple safety mechanisms to prevent accidental execution:

1. **Triple Confirmation Required**: All three gating variables must be explicitly set to `true`
2. **Non-empty Acknowledgment**: The acknowledgment text cannot be empty
3. **Explicit Variable Names**: Variable names clearly indicate the dangerous nature
4. **Default Safe State**: All dangerous operations are disabled by default

## Output Files

After execution, the following files may be created:

```
artifacts/
└── <hostname>/
    ├── core_dump_preparation.txt    # Always created
    ├── core_dump_execution.log      # Created if core dump executed
    └── core_dump_notes.txt          # Manual notes and timestamps
```

## Prerequisites

- Ansible 2.17+
- Collections: `cisco.asa`, `ansible.netcommon`
- **Maintenance Window**: Coordinate with stakeholders
- **Manual Power Cycle Capability**: Physical access or remote power management
- **Backup Connectivity**: Alternative management access if primary fails

## Important Notes

- **CAUSES IMMEDIATE DEVICE RELOAD**: Service disruption is guaranteed
- **Manual Power Cycle Required**: Follow CISA guidance exactly
- **One Device at a Time**: Use `serial: 1` for multiple devices
- **Monitor Execution**: Watch for successful reconnection
- **Follow ED 25-03 Exactly**: Deviations may trigger anti-forensics

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)