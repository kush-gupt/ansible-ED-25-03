### Role: version_assessment

Parses ASA version and flags potential vulnerability to CVE-2025-20333 (RCE) and CVE-2025-20362 (Privilege Escalation) and whether running trains are in the post-handler-removal range (>= 9.20), based on CISA guidance context.

Auto-detects device type from show version output and outputs `version_assessment.json` under `artifacts/<host>/`.

## Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `version_assessment_ed25_device_type` | `asa` | Device classification: `asa` \| `asav` \| `ftd` \| `unknown` (auto-detected if not specified) |
| `version_assessment_ed25_is_public_facing` | `false` | Whether this device is public-facing (per ED 25-03 scope) |
| `version_assessment_ed25_support_status` | `unknown` | Support status: `supported` \| `eos_2025_09_30_or_earlier` \| `eos_2026_08_31` \| `unknown` |
| `version_assessment_ed25_support_end_date` | `""` | Specific end-of-support date if known |
| `version_assessment_ed_25_03_output_dir` | `{{ (playbook_dir \| default('.')) + '/artifacts/' + inventory_hostname }}` | Output directory on control node |

## Usage Examples

### Basic Usage in Playbook

```yaml
---
- name: ED 25-03 Version Assessment
  hosts: asa
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.version_assessment
```

### With Device Classification

```yaml
---
- name: ED 25-03 Version Assessment for ASAv
  hosts: asav
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.version_assessment
      vars:
        version_assessment_ed25_device_type: asav
        version_assessment_ed25_is_public_facing: true
```

### With Support Status Information

```yaml
---
- name: ED 25-03 Version Assessment with Support Info
  hosts: asa
  gather_facts: false
  roles:
    - role: kush_gupt.ed_25_03.version_assessment
      vars:
        version_assessment_ed25_support_status: eos_2025_09_30_or_earlier
        version_assessment_ed25_support_end_date: "2025-09-30"
        version_assessment_ed25_is_public_facing: true
```

### Using the Collection Playbook

```bash
# Basic assessment
ansible-playbook kush_gupt.ed_25_03.version_assessment

# With public-facing flag
ansible-playbook kush_gupt.ed_25_03.version_assessment \
  -e "version_assessment_ed25_is_public_facing=true"

# With support status
ansible-playbook kush_gupt.ed_25_03.version_assessment \
  -e "version_assessment_ed25_support_status=eos_2025_09_30_or_earlier"
```

### Multiple Device Types

```yaml
---
- name: ED 25-03 Assessment for Mixed Fleet
  hosts: all
  gather_facts: false
  tasks:
    - name: Assess physical ASA devices
      include_role:
        name: kush_gupt.ed_25_03.version_assessment
      vars:
        version_assessment_ed25_device_type: asa
        version_assessment_ed25_is_public_facing: "{{ 'dmz' in group_names }}"
      when: "'physical_asa' in group_names"
    
    - name: Assess virtual ASA devices
      include_role:
        name: kush_gupt.ed_25_03.version_assessment
      vars:
        version_assessment_ed25_device_type: asav
        version_assessment_ed25_is_public_facing: false
      when: "'virtual_asa' in group_names"
    
    - name: Assess FTD devices
      include_role:
        name: kush_gupt.ed_25_03.version_assessment
      vars:
        version_assessment_ed25_device_type: ftd
        version_assessment_ed25_is_public_facing: "{{ 'perimeter' in group_names }}"
      when: "'ftd' in group_names"
```

### Group Variables Configuration

```yaml
# group_vars/public_asa.yml
version_assessment_ed25_is_public_facing: true
version_assessment_ed25_device_type: asa
version_assessment_ed25_support_status: supported

# group_vars/internal_asav.yml
version_assessment_ed25_is_public_facing: false
version_assessment_ed25_device_type: asav
version_assessment_ed25_support_status: eos_2026_08_31

# group_vars/legacy_asa.yml
version_assessment_ed25_is_public_facing: false
version_assessment_ed25_device_type: asa
version_assessment_ed25_support_status: eos_2025_09_30_or_earlier
version_assessment_ed25_support_end_date: "2025-09-30"
```

```yaml
# playbook.yml
---
- name: ED 25-03 Version Assessment by Group
  hosts: all_asa
  gather_facts: false
  roles:
    - kush_gupt.ed_25_03.version_assessment
```

### Conditional Assessment Based on Inventory

```yaml
---
- name: Conditional Version Assessment
  hosts: cisco_devices
  gather_facts: false
  tasks:
    - name: Get device type from show version
      cisco.asa.asa_command:
        commands:
          - show version
      register: version_check
    
    - name: Run assessment for ASA devices only
      include_role:
        name: kush_gupt.ed_25_03.version_assessment
      vars:
        version_assessment_ed25_device_type: "{{ 'ASAv' in version_check.stdout[0] | ternary('asav', 'asa') }}"
        version_assessment_ed25_is_public_facing: "{{ ansible_host | ipaddr('public') | bool }}"
      when: "'Cisco Adaptive Security Appliance' in version_check.stdout[0]"
```

### Batch Assessment with Custom Output

```yaml
---
- name: Batch Version Assessment with Timestamped Output
  hosts: asa_fleet
  gather_facts: false
  vars:
    assessment_timestamp: "{{ ansible_date_time.date }}_{{ ansible_date_time.time }}"
  roles:
    - role: kush_gupt.ed_25_03.version_assessment
      vars:
        version_assessment_ed_25_03_output_dir: "/assessments/{{ assessment_timestamp }}/{{ inventory_hostname }}"
```

### Integration with CMDB

```yaml
---
- name: ED 25-03 Assessment with CMDB Integration
  hosts: asa
  gather_facts: false
  vars:
    cmdb_api_url: "https://cmdb.example.com/api"
  tasks:
    - name: Get device info from CMDB
      uri:
        url: "{{ cmdb_api_url }}/devices/{{ inventory_hostname }}"
        method: GET
        headers:
          Authorization: "Bearer {{ cmdb_token }}"
      register: cmdb_info
      delegate_to: localhost
    
    - name: Run assessment with CMDB data
      include_role:
        name: kush_gupt.ed_25_03.version_assessment
      vars:
        version_assessment_ed25_device_type: "{{ cmdb_info.json.device_type | lower }}"
        version_assessment_ed25_is_public_facing: "{{ cmdb_info.json.network_zone == 'dmz' }}"
        version_assessment_ed25_support_status: "{{ cmdb_info.json.support_status }}"
        version_assessment_ed25_support_end_date: "{{ cmdb_info.json.support_end_date }}"
```

### Vulnerability-Focused Assessment

```yaml
---
- name: CVE-Focused Version Assessment
  hosts: asa
  gather_facts: false
  tasks:
    - name: Run version assessment
      include_role:
        name: kush_gupt.ed_25_03.version_assessment
    
    - name: Read assessment results
      slurp:
        src: "{{ version_assessment_ed_25_03_output_dir | default(playbook_dir + '/artifacts/' + inventory_hostname) }}/version_assessment.json"
      register: assessment_results
      delegate_to: localhost
    
    - name: Parse assessment data
      set_fact:
        assessment_data: "{{ assessment_results.content | b64decode | from_json }}"
    
    - name: Flag high-risk devices
      set_fact:
        high_risk_device: true
      when: 
        - assessment_data.cve_2025_20333_vulnerable | default(false)
        - assessment_data.public_facing | default(false)
        - assessment_data.support_status in ['eos_2025_09_30_or_earlier', 'unknown']
    
    - name: Create high-risk device list
      lineinfile:
        path: "/tmp/high_risk_devices.txt"
        line: "{{ inventory_hostname }}: {{ assessment_data.software_version }} (Public: {{ assessment_data.public_facing }}, Support: {{ assessment_data.support_status }})"
        create: yes
      delegate_to: localhost
      when: high_risk_device | default(false)
```

### Compliance Reporting Integration

```yaml
---
- name: ED 25-03 Compliance Assessment
  hosts: asa
  gather_facts: false
  vars:
    compliance_report_date: "{{ ansible_date_time.date }}"
  tasks:
    - name: Run version assessment
      include_role:
        name: kush_gupt.ed_25_03.version_assessment
    
    - name: Generate compliance summary
      template:
        src: compliance_summary.j2
        dest: "/compliance/{{ compliance_report_date }}/{{ inventory_hostname }}_compliance.json"
      delegate_to: localhost
      vars:
        assessment_file: "{{ version_assessment_ed_25_03_output_dir | default(playbook_dir + '/artifacts/' + inventory_hostname) }}/version_assessment.json"
```

## Output Structure

After execution, the following files will be created in the output directory:

```
artifacts/
└── <hostname>/
    ├── version_assessment.json      # Main assessment results
    ├── show_version_raw.txt         # Raw show version output
    ├── vulnerability_matrix.json    # CVE vulnerability mapping
    └── assessment_notes.txt         # Analysis notes and recommendations
```

### Sample Assessment Output

```json
{
  "assessment_metadata": {
    "timestamp": "2025-01-15T10:30:00Z",
    "hostname": "asa01",
    "assessment_version": "1.0.0"
  },
  "device_information": {
    "detected_type": "asa",
    "model": "ASA5516-X",
    "software_version": "9.18(4)30",
    "build_info": "9.18(4)30 built on Dec 15 2023",
    "serial_number": "ABC123DEF456",
    "hardware_info": "ASA5516 chassis, Hw Serial#: ABC123DEF456",
    "uptime": "42 days, 3 hours, 15 minutes"
  },
  "vulnerability_assessment": {
    "cve_2025_20333_vulnerable": true,
    "cve_2025_20333_details": {
      "severity": "critical",
      "cvss_score": 9.8,
      "description": "Remote Code Execution vulnerability"
    },
    "cve_2025_20362_vulnerable": true,
    "cve_2025_20362_details": {
      "severity": "high", 
      "cvss_score": 7.8,
      "description": "Privilege Escalation vulnerability"
    },
    "train_safety_status": "unsafe",
    "train_version": "9.18"
  },
  "configuration_assessment": {
    "public_facing": false,
    "support_status": "eos_2025_09_30_or_earlier",
    "support_end_date": "2025-09-30",
    "days_until_eos": 258
  },
  "risk_assessment": {
    "overall_risk": "high",
    "risk_factors": [
      "Multiple critical vulnerabilities",
      "Approaching end of support",
      "Unsafe software train"
    ],
    "mitigations_required": [
      "Immediate patching recommended",
      "Plan migration before EOS date",
      "Enhanced monitoring"
    ]
  },
  "ed_25_03_compliance": {
    "requires_assessment": true,
    "phase_1_applicable": true,
    "phase_2_required": false,
    "phase_3_applicable": true,
    "priority_level": "high"
  },
  "recommended_actions": [
    {
      "action": "Schedule emergency patching",
      "priority": "critical",
      "timeline": "immediate",
      "reason": "Critical vulnerabilities present"
    },
    {
      "action": "Plan hardware replacement",
      "priority": "high", 
      "timeline": "before 2025-09-30",
      "reason": "Device approaching end of support"
    }
  ]
}
```

References:
- ED 25-03 (Required Actions, timelines): https://www.cisa.gov/news-events/directives/ed-25-03-identify-and-mitigate-potential-compromise-cisco-devices
- CVE-2025-20333: https://www.cve.org/CVERecord?id=CVE-2025-20333
- CVE-2025-20362: https://www.cve.org/CVERecord?id=CVE-2025-20362