# Ansible Collection for CISA ED 25-03 (Cisco ASA)

This repository contains Ansible content that automates key steps from CISA's Supplemental Direction ED 25-03 for Cisco ASA devices, helping organizations respond to CVE-2025-20333 and CVE-2025-20362 and potential compromise indicators.

## CISA ED 25-03 Critical Deadlines

- **September 26, 2025 11:59PM EDT**: 
  - Submit core dumps for all public-facing ASA hardware (Parts 1-3)
  - Apply latest Cisco updates for devices remaining in service
- **September 30, 2025**: 
  - Permanently disconnect ASA devices with end-of-support dates on/before this date
- **October 2, 2025 11:59PM EDT**: 
  - Report complete inventory and actions taken to CISA
- **Ongoing**: Apply updates within 48 hours of Cisco release

**References**:
- [CISA ED 25-03 Directive](https://www.cisa.gov/news-events/directives/ed-25-03-identify-and-mitigate-potential-compromise-cisco-devices)
- [CISA ED 25-03: Core Dump & Hunt Instructions](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)
- [CVE-2025-20333](https://www.cve.org/CVERecord?id=CVE-2025-20333)

## Quick Start

### 1. Prerequisites
```bash
# Install required collections
ansible-galaxy collection install -r ansible/requirements.yml

# Verify Ansible version (requires 2.15+)
ansible --version
```

### 2. Configuration
```bash
# Configure your ASA devices
cp ansible/inventory.ini.example ansible/inventory.ini
# Edit ansible/inventory.ini with your ASA hosts and credentials

# Configure device-specific settings
cp ansible/group_vars/asa.yml.example ansible/group_vars/asa.yml  
# Edit ansible/group_vars/asa.yml for your environment
```

### 3. Execution Workflow

#### Step 1: Assessment and Artifact Collection
```bash
# Safe initial assessment - no device impact
ansible-playbook -i ansible/inventory.ini \
  ansible/playbooks/collect_artifacts.yml
```

#### Step 2: Patch Verification
```bash
# Check for firmware update logs
ansible-playbook -i ansible/inventory.ini \
  ansible/playbooks/patch_checks.yml
```

#### Step 3: Syslog Analysis (Local)
```bash
# Analyze exported syslog files on control node
ansible-playbook -i ansible/inventory.ini \
  ansible/playbooks/hunt_syslog_checks.yml \
  -e hunt_syslog_checks_asa_syslog_glob="/path/to/your/asa/logs/*.log"
```

#### Step 4: Core Dump (⚠️ DANGEROUS - CAUSES OUTAGE)
```bash
# Only execute if compromise is suspected and you understand the risks
ansible-playbook -i ansible/inventory.ini \
  ansible/playbooks/core_dump.yml \
  -e core_dump_ed_25_03_perform_core_dump=true \
  -e core_dump_ed_25_03_confirm_manual_power_cycle=true \
  -e core_dump_ed_25_03_core_dump_ack_text="I acknowledge this triggers immediate reload and potential outage per CISA ED 25-03 guidance."
```

#### Step 5: Comprehensive Report
```bash
# Generate consolidated assessment report
ansible-playbook -i ansible/inventory.ini \
  ansible/playbooks/assess_and_report.yml
```

### Step 6. Review Results
All outputs are saved to: `artifacts/<inventory_hostname>/`
- Review `ed25_report.json` for consolidated findings and recommendations
- **If compromise detected**: Immediately disconnect device (do not power off), report to CISA at contact@cisa.dhs.gov
- **Submit core dumps**: Upload to CISA Malware Next Gen portal if public-facing ASA hardware
- **Complete inventory reporting**: Submit to CISA by October 2, 2025 using provided template

# Disclaimer

## Disclaimer of Warranty and Liability

This software is provided **“AS IS”** and **“AS AVAILABLE,”** without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, non-infringement, title, or quiet enjoyment.  

No oral or written information or advice given by the Author, the owner of the GitHub account where this software is hosted, GitHub, Inc., or Red Hat, Inc., or their respective employees, officers, directors, or affiliates (collectively, the **“Disclaimed Parties”**) shall create any warranty.  

The entire risk arising out of the use or performance of this software remains with you. To the maximum extent permitted by law, in no event shall any of the Disclaimed Parties be liable for any claim, damages, or other liability, whether in an action of contract, tort, or otherwise, arising from, out of, or in connection with the software or the use or other dealings in the software.  

This includes, without limitation, direct, indirect, incidental, special, exemplary, consequential, or punitive damages, or damages for loss of profits, loss of data, business interruption, or personal injury, even if advised of the possibility of such damages.  

---

## No Support or Obligation

The use of this software does not create any obligation on the part of the Author, GitHub, the GitHub Account holder, or Red Hat to provide maintenance, support, updates, enhancements, or modifications. Any support provided shall be at the sole discretion of such party and may be discontinued at any time without notice.  

---

## No Endorsement or Agency

The hosting of this software on GitHub does not imply any endorsement by GitHub, nor does it create any partnership, agency, employment, fiduciary, or joint venture relationship between GitHub and the Author or between GitHub and Red Hat.  

Likewise, any mention of Red Hat does not imply endorsement, sponsorship, or official involvement by Red Hat, unless explicitly stated in a separate written agreement.  

---

## User Responsibility

By using this software, you acknowledge and agree that:  

1. You are solely responsible for determining whether the software is suitable for your purposes.  
2. You are solely responsible for ensuring that your use of the software complies with all applicable laws, regulations, and third-party rights.  
3. You assume all risks associated with use of the software, including but not limited to the risk of data loss, security vulnerabilities, or operational failures.  

---

## Jurisdiction and Severability

This disclaimer shall be governed by the laws of the jurisdiction in which you use the software, without regard to its conflict of law principles.  

If any provision of this disclaimer is found to be unenforceable, the remaining provisions shall remain in full force and effect.  
