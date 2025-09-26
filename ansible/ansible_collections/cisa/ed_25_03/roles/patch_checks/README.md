### Role: patch_checks

Implements parts of ED 25-03 Part Three highlighting checks:
- Look for `firmwareupdate.log` on `disk0:`
- If present, save content locally to assist case creation with Cisco TAC

Outputs go to `artifacts/<host>/`.

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `patch_checks_ed_25_03_filesystem` | `disk0:` | Filesystem to check for firmware update logs |
| `patch_checks_ed_25_03_output_dir` | `{{ (playbook_dir \| default('.')) + '/artifacts/' + inventory_hostname }}` | Output directory on control node |
| `patch_checks_ed_25_03_command_timeout` | `300` | Timeout for filesystem operations |

## Usage Examples

### Basic Usage in Playbook

```yaml
---
- name: ED 25-03 Patch Checks
  hosts: asa
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.patch_checks
```

### With Custom Filesystem

```yaml
---
- name: ED 25-03 Patch Checks on Flash
  hosts: asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.patch_checks
      vars:
        patch_checks_ed_25_03_filesystem: "flash:"
```

### With Custom Output Directory

```yaml
---
- name: ED 25-03 Patch Checks with Custom Output
  hosts: asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.patch_checks
      vars:
        patch_checks_ed_25_03_output_dir: "/forensics/{{ inventory_hostname }}/patch_analysis"
```

### Using the Collection Playbook

```bash
# Basic usage
ansible-playbook kush_gupt.ed_25_03.patch_checks

# With custom filesystem
ansible-playbook kush_gupt.ed_25_03.patch_checks \
  -e "patch_checks_ed_25_03_filesystem=flash:"

# With inventory file
ansible-playbook -i inventory.ini kush_gupt.ed_25_03.patch_checks
```

### Combined with Other ED 25-03 Roles

```yaml
---
- name: Complete ED 25-03 Device Assessment
  hosts: asa
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.version_assessment
    - kush_gupt.ed_25_03.collect_artifacts
    - kush_gupt.ed_25_03.patch_checks
    - kush_gupt.ed_25_03.report
```

### Multiple Filesystem Check

```yaml
---
- name: Check Multiple Filesystems for Patch Logs
  hosts: asa
  gather_facts: false
  tasks:
    - name: Check disk0 for patch logs
      include_role:
        name: kush_gupt.ed_25_03.patch_checks
      vars:
        patch_checks_ed_25_03_filesystem: "disk0:"
        patch_checks_ed_25_03_output_dir: "{{ playbook_dir }}/artifacts/{{ inventory_hostname }}/disk0"
    
    - name: Check flash for patch logs
      include_role:
        name: kush_gupt.ed_25_03.patch_checks
      vars:
        patch_checks_ed_25_03_filesystem: "flash:"
        patch_checks_ed_25_03_output_dir: "{{ playbook_dir }}/artifacts/{{ inventory_hostname }}/flash"
```

### Conditional Execution Based on Version

```yaml
---
- name: Conditional Patch Checks
  hosts: asa
  gather_facts: false
  tasks:
    - name: Get ASA version
      cisco.asa.asa_command:
        commands:
          - show version
      register: version_output
    
    - name: Run patch checks only for vulnerable versions
      include_role:
        name: kush_gupt.ed_25_03.patch_checks
      when: "'9.18' in version_output.stdout[0] or '9.19' in version_output.stdout[0]"
```

### With Extended Timeout for Slow Devices

```yaml
---
- name: ED 25-03 Patch Checks with Extended Timeout
  hosts: slow_asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.patch_checks
      vars:
        patch_checks_ed_25_03_command_timeout: 600  # 10 minutes
```

### Group Variables Configuration

```yaml
# group_vars/asa.yml
patch_checks_ed_25_03_filesystem: "disk0:"
patch_checks_ed_25_03_output_dir: "/shared/ed25_forensics/{{ inventory_hostname }}"
patch_checks_ed_25_03_command_timeout: 450
```

```yaml
# playbook.yml
---
- name: ED 25-03 Patch Checks
  hosts: asa
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.patch_checks
```

### Automated Batch Processing

```yaml
---
- name: Batch Patch Checks for Multiple ASA Groups
  hosts: all_asa
  gather_facts: false
  serial: 5  # Process 5 devices at a time
  roles:
    - role: kush_gupt.ed_25_03.patch_checks
      vars:
        patch_checks_ed_25_03_output_dir: "{{ ansible_date_time.date }}_patch_checks/{{ inventory_hostname }}"
```

### Integration with TAC Case Creation

```yaml
---
- name: ED 25-03 Patch Checks with TAC Integration
  hosts: asa
  gather_facts: false
  tasks:
    - name: Run patch checks
      include_role:
        name: kush_gupt.ed_25_03.patch_checks
      
    - name: Check if firmware update log found
      stat:
        path: "{{ patch_checks_ed_25_03_output_dir | default(playbook_dir + '/artifacts/' + inventory_hostname) }}/firmwareupdate.log"
      register: firmware_log
      delegate_to: localhost
    
    - name: Create TAC case preparation note
      copy:
        content: |
          TAC Case Preparation for {{ inventory_hostname }}
          Date: {{ ansible_date_time.iso8601 }}
          Firmware Update Log Found: {{ firmware_log.stat.exists }}
          
          {% if firmware_log.stat.exists %}
          URGENT: firmwareupdate.log detected on {{ inventory_hostname }}
          Location: {{ patch_checks_ed_25_03_filesystem }}
          Size: {{ firmware_log.stat.size | default('unknown') }} bytes
          
          Next Steps:
          1. Open TAC case immediately
          2. Provide firmwareupdate.log content
          3. Follow CISA ED 25-03 guidance exactly
          {% else %}
          No firmwareupdate.log found on {{ inventory_hostname }}
          Filesystem checked: {{ patch_checks_ed_25_03_filesystem }}
          {% endif %}
        dest: "{{ patch_checks_ed_25_03_output_dir | default(playbook_dir + '/artifacts/' + inventory_hostname) }}/tac_case_notes.txt"
      delegate_to: localhost
```

## Output Structure

After execution, the following files may be created in the output directory:

```
artifacts/
└── <hostname>/
    ├── patch_check_results.json     # Always created - summary
    ├── filesystem_listing.txt       # Directory listing of filesystem
    ├── firmwareupdate.log           # Only if found on device
    └── patch_check_notes.txt        # Analysis notes and timestamps
```

### Sample Output Files

**patch_check_results.json**:
```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "hostname": "asa01",
  "filesystem_checked": "disk0:",
  "firmwareupdate_log_found": true,
  "firmwareupdate_log_size": 15234,
  "additional_files_found": [
    "firmware_backup_20250110.bin",
    "update_status.txt"
  ],
  "analysis_status": "URGENT - TAC case required",
  "next_actions": [
    "Open Cisco TAC case immediately",
    "Provide firmwareupdate.log content to TAC",
    "Follow CISA ED 25-03 Part Three guidance"
  ]
}
```

## Prerequisites

- Ansible 2.15+
- Collections: `cisco.asa`, `ansible.netcommon`
- SSH connectivity to Cisco ASA devices
- Sufficient privileges to read filesystem contents
- Disk space on control node for log files

## Important Notes

- **Critical Indicator**: Finding `firmwareupdate.log` indicates potential compromise
- **Immediate Action Required**: Open TAC case if firmware update log is found
- **Preserve Evidence**: Do not modify or delete found log files
- **Multiple Filesystems**: Consider checking both `disk0:` and `flash:`
- **Large Files**: Firmware logs can be substantial in size

## What the Role Checks

1. **Directory Listing**: Lists contents of specified filesystem
2. **Firmware Update Log**: Specifically looks for `firmwareupdate.log`
3. **Related Files**: Identifies other potentially relevant files
4. **File Retrieval**: Downloads found logs to control node
5. **Analysis Notes**: Creates summary and next-action recommendations

## Integration with ED 25-03 Workflow

1. **After Artifact Collection**: Run after basic artifact gathering
2. **Before Core Dump**: Complete patch checks before proceeding to Part Two
3. **TAC Case Preparation**: Use outputs to support TAC case creation
4. **Evidence Chain**: Maintain proper forensic handling of downloaded files

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)