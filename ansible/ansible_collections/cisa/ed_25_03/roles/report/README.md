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

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)
