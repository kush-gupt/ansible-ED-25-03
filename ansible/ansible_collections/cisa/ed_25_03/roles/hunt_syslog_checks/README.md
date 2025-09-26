### Role: hunt_syslog_checks

Local helper to analyze exported ASA syslogs for key message IDs mentioned in ED 25-03 (302013, 302014, 710005, 609002). Produces counts JSON to help detect tampering timelines.

**Note**: This role runs on the control node and expects logs already exported from ASA or your SIEM.

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `hunt_syslog_checks_asa_syslog_glob` | `""` | **REQUIRED**: Glob path on the control node for ASA syslog files to analyze |

## Monitored Message IDs

The role searches for these specific ASA message IDs mentioned in ED 25-03:

- **302013**: Built/Teardown connection messages
- **302014**: Built/Teardown connection messages  
- **710005**: Protocol connection built/teardown
- **609002**: Local user authentication

## Usage Examples

### Basic Usage with Local Log Files

```yaml
---
- name: Hunt ED 25-03 Syslog Indicators
  hosts: localhost
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.hunt_syslog_checks
      vars:
        hunt_syslog_checks_asa_syslog_glob: "/var/log/asa/*.log"
```

### Multiple Log Directories

```yaml
---
- name: Hunt Syslogs from Multiple Sources
  hosts: localhost
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.hunt_syslog_checks
      vars:
        hunt_syslog_checks_asa_syslog_glob: "/logs/asa/{current,archive}/*.log"
```

### With Specific Date Range Files

```yaml
---
- name: Hunt Recent Syslog Files
  hosts: localhost
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.hunt_syslog_checks
      vars:
        hunt_syslog_checks_asa_syslog_glob: "/var/log/asa/asa-{{ ansible_date_time.date }}*.log"
```

### Using the Collection Playbook

```bash
# Basic usage - requires setting the glob variable
ansible-playbook kush_gupt.ed_25_03.hunt_syslog_checks \
  -e "hunt_syslog_checks_asa_syslog_glob='/var/log/asa/*.log'"

# With multiple patterns
ansible-playbook kush_gupt.ed_25_03.hunt_syslog_checks \
  -e "hunt_syslog_checks_asa_syslog_glob='/logs/asa/**/*.log'"
```

### Combined with Other ED 25-03 Roles

```yaml
---
- name: Complete ED 25-03 Assessment with Syslog Hunt
  hosts: asa
  gather_facts: false
  tasks:
    # First collect artifacts from ASA devices
    - name: Collect artifacts
      include_role:
        name: kush_gupt.ed_25_03.collect_artifacts
    
    # Then analyze local syslogs on control node
    - name: Hunt syslog indicators
      include_role:
        name: kush_gupt.ed_25_03.hunt_syslog_checks
      vars:
        hunt_syslog_checks_asa_syslog_glob: "/var/log/siem/asa-{{ inventory_hostname }}-*.log"
      delegate_to: localhost
      run_once: true
```

### With SIEM Integration

```yaml
---
- name: Hunt Syslogs from SIEM Export
  hosts: localhost
  gather_facts: false
  vars:
    siem_export_path: "/tmp/siem_export"
  tasks:
    - name: Download logs from SIEM
      uri:
        url: "https://siem.example.com/api/export"
        method: GET
        dest: "{{ siem_export_path }}/asa_logs.json"
        headers:
          Authorization: "Bearer {{ siem_api_token }}"
    
    - name: Convert SIEM logs to syslog format
      # Custom conversion task here
      
    - name: Hunt converted syslog files
      include_role:
        name: kush_gupt.ed_25_03.hunt_syslog_checks
      vars:
        hunt_syslog_checks_asa_syslog_glob: "{{ siem_export_path }}/converted/*.log"
```

### Automated Daily Analysis

```yaml
---
- name: Daily ED 25-03 Syslog Hunt
  hosts: localhost
  gather_facts: false
  vars:
    log_retention_days: 30
  tasks:
    - name: Find recent log files
      find:
        paths: /var/log/asa/
        patterns: "*.log"
        age: "-{{ log_retention_days }}d"
      register: recent_logs
    
    - name: Hunt recent logs if found
      include_role:
        name: kush_gupt.ed_25_03.hunt_syslog_checks
      vars:
        hunt_syslog_checks_asa_syslog_glob: "{{ recent_logs.files | map(attribute='path') | join(',') }}"
      when: recent_logs.files | length > 0
```

### With Group Variables

```yaml
# group_vars/all.yml
hunt_syslog_checks_asa_syslog_glob: "/shared/logs/asa/**/*.log"
```

```yaml
# playbook.yml
---
- name: Hunt ED 25-03 Syslog Indicators
  hosts: localhost
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.hunt_syslog_checks
```

## Output Structure

After execution, the following files will be created:

```
artifacts/
└── syslog_analysis/
    ├── message_counts.json          # Counts of each message ID
    ├── timeline_analysis.json       # Temporal analysis
    ├── suspicious_patterns.json     # Potential indicators
    └── hunt_summary.txt            # Human-readable summary
```

### Sample Output

```json
{
  "analysis_timestamp": "2025-01-15T10:30:00Z",
  "files_analyzed": 15,
  "total_log_entries": 1250000,
  "message_id_counts": {
    "302013": 45230,
    "302014": 45180,
    "710005": 12450,
    "609002": 890
  },
  "time_range": {
    "earliest": "2025-01-01T00:00:00Z",
    "latest": "2025-01-15T10:29:45Z"
  },
  "anomalies_detected": [
    "Unusual spike in 609002 messages on 2025-01-10",
    "Gap in 302013/302014 messages from 2025-01-12 14:00-16:00"
  ]
}
```

## Prerequisites

- Ansible 2.15+
- ASA syslog files available on the control node
- Sufficient disk space for analysis output
- Python 3.6+ for JSON processing

## Log File Requirements

### Supported Formats

- **Standard Syslog**: RFC3164 format
- **ASA Native**: Cisco ASA syslog format
- **SIEM Exports**: Common SIEM export formats

### Sample Log Entry

```
Jan 15 10:30:15 asa01 %ASA-6-302013: Built outbound TCP connection 12345 for outside:192.168.1.100/443 to inside:10.0.1.50/54321
```

## Important Notes

- **Control Node Execution**: This role runs locally, not on ASA devices
- **Pre-exported Logs**: Logs must be available before running this role
- **Large File Handling**: Role optimized for large log files
- **Timeline Analysis**: Helps identify gaps or anomalies in logging
- **Complement to Device Analysis**: Use alongside artifact collection roles

## Integration with ED 25-03 Workflow

1. **Export Logs**: Collect ASA syslogs from devices or SIEM
2. **Run Hunt**: Execute this role to analyze message patterns
3. **Review Results**: Check for timeline anomalies or suspicious patterns
4. **Correlate Findings**: Compare with device artifact analysis
5. **Report**: Include findings in overall ED 25-03 assessment

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)