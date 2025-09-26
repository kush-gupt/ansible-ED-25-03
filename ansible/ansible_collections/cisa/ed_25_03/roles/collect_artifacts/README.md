### Role: collect_artifacts

Implements ED 25-03 Part One (artifact collection) for Cisco ASA:
- show version (device details)
- show checkheaps twice with a wait and summary
- show tech-support detail
- implant grep: `more /binary system:/text | grep 55534154 41554156 41575756 488bb3a0`

Outputs are saved locally under `{{ collect_artifacts_ed_25_03_output_dir }}`.

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `collect_artifacts_ed_25_03_output_dir` | `{{ (playbook_dir \| default('.')) + '/artifacts/' + inventory_hostname }}` | Directory on control node where outputs will be stored per-host |
| `collect_artifacts_ed_25_03_checkheaps_wait_seconds` | `300` | Wait time between checkheaps runs (seconds). CISA recommends 5+ minutes |
| `collect_artifacts_ed_25_03_command_timeout` | `1800` | Command timeout for long-running commands like 'show tech-support detail' |
| `collect_artifacts_ed_25_03_filesystem` | `disk0:` | Filesystem for ASA core dump or directory listing in later steps |

## Usage Examples

### Basic Usage in Playbook

```yaml
---
- name: Collect ED 25-03 Artifacts
  hosts: asa
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.collect_artifacts
```

### With Custom Output Directory

```yaml
---
- name: Collect ED 25-03 Artifacts with Custom Output
  hosts: asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.collect_artifacts
      vars:
        collect_artifacts_ed_25_03_output_dir: "/tmp/ed25_artifacts/{{ inventory_hostname }}"
```

### With Custom Timing

```yaml
---
- name: Collect ED 25-03 Artifacts with Extended Wait
  hosts: asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.collect_artifacts
      vars:
        collect_artifacts_ed_25_03_checkheaps_wait_seconds: 600  # 10 minutes
        collect_artifacts_ed_25_03_command_timeout: 3600        # 1 hour
```

### Using the Collection Playbook

```bash
# Run the pre-built playbook
ansible-playbook kush_gupt.ed_25_03.collect_artifacts

# Or with inventory
ansible-playbook -i inventory.ini kush_gupt.ed_25_03.collect_artifacts
```

### Task-Level Usage

```yaml
---
- name: Include artifact collection tasks
  hosts: asa
  gather_facts: false
  tasks:
    - name: Collect artifacts
      include_role:
        name: kush_gupt.ed_25_03.collect_artifacts
      vars:
        collect_artifacts_ed_25_03_output_dir: "{{ ansible_date_time.date }}_artifacts/{{ inventory_hostname }}"
```

### With Group Variables

```yaml
# group_vars/asa.yml
collect_artifacts_ed_25_03_output_dir: "/shared/ed25_artifacts/{{ inventory_hostname }}"
collect_artifacts_ed_25_03_checkheaps_wait_seconds: 450
```

```yaml
# playbook.yml
---
- name: Collect ED 25-03 Artifacts
  hosts: asa
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.collect_artifacts
```

## Output Structure

After execution, the following files will be created in the output directory:

```
artifacts/
└── <hostname>/
    ├── show_version.txt
    ├── show_checkheaps_1.txt
    ├── show_checkheaps_2.txt
    ├── show_tech_support_detail.txt
    └── implant_grep.txt
```

## Prerequisites

- Ansible 2.15+
- Collections: `cisco.asa`, `ansible.netcommon`
- SSH connectivity to Cisco ASA devices
- Sufficient disk space on control node for output files

## Important Notes

- Follow CISA ED 25-03 guidance exactly and in order
- The `show tech-support detail` command may take significant time to complete
- Ensure adequate timeout values for your environment
- Review outputs for indicators before proceeding to next steps

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)