### Role: patch_checks

Implements parts of ED 25-03 Part Three highlighting checks:
- Look for `firmwareupdate.log` on `disk0:`
- If present, save content locally to assist case creation with Cisco TAC

Outputs go to `artifacts/<host>/`.

See: [CISA ED 25-03 guidance](https://www.cisa.gov/news-events/directives/supplemental-direction-ed-25-03-core-dump-and-hunt-instructions)


