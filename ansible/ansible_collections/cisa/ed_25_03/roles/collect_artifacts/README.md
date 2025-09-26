### Role: collect_artifacts

Implements ED 25-03 Part One (artifact collection) for Cisco ASA:
- show version (device details)
- show checkheaps twice with a wait and summary
- show tech-support detail
- implant grep: `more /binary system:/text | grep 55534154 41554156 41575756 488bb3a0`

Outputs are saved locally under `{{ collect_artifacts_ed_25_03_output_dir }}`.

Defaults
```
collect_artifacts_ed_25_03_output_dir: "{{ (playbook_dir | default('.')) + '/artifacts/' + inventory_hostname }}"
collect_artifacts_ed_25_03_checkheaps_wait_seconds: 300
collect_artifacts_ed_25_03_command_timeout: 1800
```

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)


