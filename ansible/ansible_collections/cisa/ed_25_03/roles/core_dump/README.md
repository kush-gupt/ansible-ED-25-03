### Role: core_dump

Implements ED 25-03 Part Two (Core Dump Collection) steps on Cisco ASA.

WARNING: The action `crashinfo force page-fault` causes immediate reload and may lead to outages. Strong gating variables are required to run. This role does NOT perform the power cycle; ensure manual reboot as advised by CISA prior to capture.

Vars
```
core_dump_ed_25_03_perform_core_dump: false
core_dump_ed_25_03_confirm_manual_power_cycle: false
core_dump_ed_25_03_core_dump_ack_text: "Type a long explicit acknowledgment message here"
core_dump_ed_25_03_filesystem: "disk0:"
core_dump_ed_25_03_attempt_reconnect: false
core_dump_ed_25_03_expected_reload_seconds: 180
```

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)


