### Role: hunt_syslog_checks

Local helper to analyze exported ASA syslogs for key message IDs mentioned in ED 25-03 (302013, 302014, 710005, 609002). Produces counts JSON to help detect tampering timelines.

Vars
```
asa_syslog_glob: "/var/log/asa/*.log"
```

Note: This role runs on the control node and expects logs already exported from ASA or your SIEM.

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)


