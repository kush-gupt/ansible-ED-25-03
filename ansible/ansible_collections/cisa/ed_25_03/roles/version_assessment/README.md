### Role: version_assessment

Parses ASA version and flags potential vulnerability to CVE-2025-20333 (RCE) and CVE-2025-20362 (Privilege Escalation) and whether running trains are in the post-handler-removal range (>= 9.20), based on CISA guidance context.

Auto-detects device type from show version output and outputs `version_assessment.json` under `artifacts/<host>/`.

Vars
```
ed25_device_type: asa | asa-sm | asav | ftd | unknown (auto-detected if not specified)
ed25_is_public_facing: false
ed25_support_status: supported | eos_2025_09_30_or_earlier | eos_2026_08_31 | unknown
ed25_support_end_date: ""
```

References:
- ED 25-03 (Required Actions, timelines): https://www.cisa.gov/news-events/directives/ed-25-03-identify-and-mitigate-potential-compromise-cisco-devices
- CVE-2025-20333: https://www.cve.org/CVERecord?id=CVE-2025-20333
- CVE-2025-20362: https://www.cve.org/CVERecord?id=CVE-2025-20362


