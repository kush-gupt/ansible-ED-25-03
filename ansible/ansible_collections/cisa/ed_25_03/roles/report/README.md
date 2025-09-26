### Role: report

Generates a consolidated ED 25-03 assessment report by combining outputs from other roles:
- Version assessment results
- Artifact collection findings  
- Syslog analysis counts
- Compromise detection status
- Recommended next actions per CISA guidance

The report includes automated recommendations based on:
- Compromise detection results
- Public-facing status
- Device support lifecycle status
- Vulnerability assessment outcomes

Output: `{{ ed_25_03_output_dir }}/ed25_report.json`

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ed_25_03_output_dir` | `{{ (playbook_dir \| default('.')) + '/artifacts/' + inventory_hostname }}` | Output directory for the consolidated report |
| `ed_25_03_report_format` | `json` | Report format: `json`, `yaml`, or `both` |
| `ed_25_03_include_raw_data` | `false` | Include raw command outputs in report |
| `ed_25_03_report_timestamp` | `{{ ansible_date_time.iso8601 }}` | Timestamp for report generation |

## Usage Examples

### Basic Usage - Final Report Generation

```yaml
---
- name: Generate ED 25-03 Final Report
  hosts: asa
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.version_assessment
    - kush_gupt.ed_25_03.collect_artifacts
    - kush_gupt.ed_25_03.patch_checks
    - kush_gupt.ed_25_03.hunt_syslog_checks
    - kush_gupt.ed_25_03.report  # Always run last
```

### Standalone Report Generation

```yaml
---
- name: Generate Report from Existing Data
  hosts: asa
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.report
```

### With Custom Output Format

```yaml
---
- name: Generate Multiple Format Reports
  hosts: asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.report
      vars:
        ed_25_03_report_format: both  # Generate both JSON and YAML
        ed_25_03_include_raw_data: true
```

### Using the Collection Playbook

```bash
# Generate report using the comprehensive assessment playbook
ansible-playbook kush_gupt.ed_25_03.assess_and_report

# Generate standalone report
ansible-playbook -c local -i "localhost," kush_gupt.ed_25_03.report
```

### Complete Assessment Workflow

```yaml
---
- name: Complete ED 25-03 Assessment and Reporting
  hosts: asa
  gather_facts: false
  serial: 1  # Process one device at a time
  tasks:
    # Phase 1: Initial Assessment
    - name: Version and vulnerability assessment
      include_role:
        name: kush_gupt.ed_25_03.version_assessment
    
    # Phase 2: Artifact Collection
    - name: Collect forensic artifacts
      include_role:
        name: kush_gupt.ed_25_03.collect_artifacts
    
    # Phase 3: Patch Status Check
    - name: Check for firmware update indicators
      include_role:
        name: kush_gupt.ed_25_03.patch_checks
    
    # Phase 4: Syslog Analysis (on control node)
    - name: Hunt syslog indicators
      include_role:
        name: kush_gupt.ed_25_03.hunt_syslog_checks
      vars:
        hunt_syslog_checks_asa_syslog_glob: "/var/log/siem/{{ inventory_hostname }}-*.log"
      delegate_to: localhost
    
    # Phase 5: Conditional Core Dump (if compromise detected)
    - name: Core dump if compromise detected
      include_role:
        name: kush_gupt.ed_25_03.core_dump
      vars:
        core_dump_ed_25_03_perform_core_dump: "{{ compromise_detected | default(false) }}"
        core_dump_ed_25_03_confirm_manual_power_cycle: "{{ compromise_detected | default(false) }}"
        core_dump_ed_25_03_core_dump_ack_text: "Automated core dump triggered by compromise detection"
      when: compromise_detected | default(false)
    
    # Phase 6: Final Report Generation
    - name: Generate comprehensive assessment report
      include_role:
        name: kush_gupt.ed_25_03.report
      vars:
        ed_25_03_report_format: both
        ed_25_03_include_raw_data: true
```

### Multi-Device Batch Reporting

```yaml
---
- name: Generate Reports for Multiple Devices
  hosts: asa_fleet
  gather_facts: false
  vars:
    report_base_dir: "/reports/ed25_assessment_{{ ansible_date_time.date }}"
  tasks:
    - name: Run assessment roles for each device
      include_role:
        name: "{{ item }}"
      loop:
        - kush_gupt.ed_25_03.version_assessment
        - kush_gupt.ed_25_03.collect_artifacts
        - kush_gupt.ed_25_03.patch_checks
    
    - name: Generate individual device reports
      include_role:
        name: kush_gupt.ed_25_03.report
      vars:
        ed_25_03_output_dir: "{{ report_base_dir }}/{{ inventory_hostname }}"
    
    # Generate fleet summary on control node
    - name: Generate fleet summary report
      template:
        src: fleet_summary.j2
        dest: "{{ report_base_dir }}/fleet_summary.json"
      delegate_to: localhost
      run_once: true
```

### Custom Report Configuration

```yaml
---
- name: Custom ED 25-03 Report Configuration
  hosts: asa
  gather_facts: false
  vars:
    custom_report_config:
      include_sections:
        - version_assessment
        - vulnerability_status
        - compromise_indicators
        - recommended_actions
      exclude_raw_data: false
      priority_level: "critical"
  roles:
    - role: kush_gupt.ed_25_03.report
      vars:
        ed_25_03_output_dir: "/priority_reports/{{ inventory_hostname }}"
        ed_25_03_report_format: json
        ed_25_03_custom_config: "{{ custom_report_config }}"
```

### Integration with Ticketing Systems

```yaml
---
- name: ED 25-03 Assessment with Ticket Creation
  hosts: asa
  gather_facts: false
  tasks:
    # Run assessment roles
    - include_role:
        name: "{{ item }}"
      loop:
        - kush_gupt.ed_25_03.version_assessment
        - kush_gupt.ed_25_03.collect_artifacts
        - kush_gupt.ed_25_03.patch_checks
        - kush_gupt.ed_25_03.report
    
    # Read generated report
    - name: Read assessment report
      slurp:
        src: "{{ ed_25_03_output_dir | default(playbook_dir + '/artifacts/' + inventory_hostname) }}/ed25_report.json"
      register: assessment_report
      delegate_to: localhost
    
    - name: Parse report data
      set_fact:
        report_data: "{{ assessment_report.content | b64decode | from_json }}"
    
    # Create tickets for high-priority findings
    - name: Create security ticket if compromise detected
      uri:
        url: "{{ ticketing_system_url }}/api/tickets"
        method: POST
        body_format: json
        body:
          title: "URGENT: ED 25-03 Compromise Detected - {{ inventory_hostname }}"
          description: "{{ report_data.summary }}"
          priority: "critical"
          category: "security_incident"
          affected_system: "{{ inventory_hostname }}"
          findings: "{{ report_data.compromise_indicators }}"
      when: report_data.compromise_detected | default(false)
```

### Scheduled Reporting

```yaml
---
- name: Scheduled ED 25-03 Compliance Check
  hosts: asa
  gather_facts: false
  vars:
    compliance_check_date: "{{ ansible_date_time.date }}"
  tasks:
    - name: Run lightweight assessment
      include_role:
        name: "{{ item }}"
      loop:
        - kush_gupt.ed_25_03.version_assessment
        - kush_gupt.ed_25_03.patch_checks
    
    - name: Generate compliance report
      include_role:
        name: kush_gupt.ed_25_03.report
      vars:
        ed_25_03_output_dir: "/compliance/{{ compliance_check_date }}/{{ inventory_hostname }}"
        ed_25_03_report_format: json
    
    - name: Archive previous reports
      archive:
        path: "/compliance/{{ (ansible_date_time.date | to_datetime('%Y-%m-%d') - timedelta(days=30)).strftime('%Y-%m-%d') }}/"
        dest: "/compliance/archives/ed25_{{ (ansible_date_time.date | to_datetime('%Y-%m-%d') - timedelta(days=30)).strftime('%Y-%m-%d') }}.tar.gz"
        remove: true
      delegate_to: localhost
      run_once: true
```

## Output Structure

The report role generates comprehensive output files:

```
artifacts/
└── <hostname>/
    ├── ed25_report.json             # Main assessment report
    ├── ed25_report.yaml             # YAML format (if requested)
    ├── executive_summary.txt        # Human-readable summary
    ├── technical_findings.json      # Detailed technical data
    ├── recommended_actions.json     # Prioritized action items
    └── compliance_status.json       # ED 25-03 compliance tracking
```

### Sample Report Structure

```json
{
  "assessment_metadata": {
    "timestamp": "2025-01-15T10:30:00Z",
    "hostname": "asa01",
    "ansible_collection": "kush_gupt.ed_25_03",
    "ed_25_03_version": "1.0.0",
    "assessment_duration": "00:15:23"
  },
  "device_information": {
    "model": "ASA5516-X",
    "software_version": "9.18(4)30",
    "serial_number": "ABC123DEF456",
    "uptime": "42 days, 3 hours",
    "public_facing": false
  },
  "vulnerability_assessment": {
    "cve_2025_20333_vulnerable": true,
    "cve_2025_20362_vulnerable": true,
    "train_safety_status": "unsafe",
    "support_status": "eos_2025_09_30_or_earlier"
  },
  "compromise_indicators": {
    "compromise_detected": false,
    "suspicious_patterns": [],
    "firmware_update_log_found": false,
    "syslog_anomalies": []
  },
  "compliance_status": {
    "ed_25_03_phase_1_complete": true,
    "ed_25_03_phase_2_required": false,
    "ed_25_03_phase_3_complete": true,
    "overall_compliance": "compliant"
  },
  "recommended_actions": [
    {
      "priority": "high",
      "action": "Schedule firmware upgrade",
      "timeline": "Before 2025-09-30",
      "reason": "Device approaching end of support"
    },
    {
      "priority": "medium",
      "action": "Continue monitoring",
      "timeline": "Ongoing",
      "reason": "No compromise indicators detected"
    }
  ],
  "next_steps": [
    "Review technical findings",
    "Schedule maintenance window for upgrades",
    "Continue ED 25-03 compliance monitoring"
  ]
}
```

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)