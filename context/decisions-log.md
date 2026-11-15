# Decisions log

The decisions taken while drafting the spec, and why. Update this file whenever a decision changes, so that the prompts in `prompts/` stay accurate.

| Date | Decision | Reason |
| --- | --- | --- |
| 2026-11-09 | CDP runs on Databricks; the spec is written for Databricks jobs, Delta, Unity Catalog and Auto Loader | CDP's platform |
| 2026-11-09 | Reuse the existing DQX data quality framework and the CDP reconciliation framework; QA does not rebuild them | They already exist and have owners |
| 2026-11-09 | Jenkins is the CI/CD tool | The team's CI/CD |
| 2026-11-09 | Unit tests are owned by developers; QA reads their build result | Ownership split |
| 2026-11-12 | Reporting: Allure for run detail, Xray for traceability and release evidence, Databricks dashboard for trends | Each answers a different question |
| 2026-11-12 | ReportPortal not adopted for now | Overlaps the other three tools and needs its own server |
| 2026-11-12 | Reviewers are Kedar, Gagan and Sukesh; the Product team prioritises | Confirmed by the QA lead |
| 2026-11-12 | Progression and regression covered in depth, with smoke, targeted and full regression levels | Whiteboard requirement |
| 2026-11-13 | Jobs can be parameterised; per-build isolation through job parameters | Confirmed by the QA lead |
| 2026-11-13 | Lakeflow Declarative Pipelines are not used today; sections to revisit if they are adopted are listed | Confirmed by the QA lead |
| 2026-11-13 | Unified results view over the existing log tables plus a QA results table and a run correlation table | Single release gate without changing the other frameworks |
| 2026-11-13 | Automation ratio measured as automated Xray tests ÷ all in-scope Xray tests | Makes 80 to 85% measurable |
| 2026-11-13 | Test priority 1 to 3 and defect severity S1 to S4; pass rate is a trend, not a gate | Removes the P1 ambiguity |
| 2026-11-13 | DPO approval before using masked production data; no real row values in Jira, Allure or results | GDPR |
| 2026-11-15 | Code samples moved to Appendix A; ReportPortal rationale to Appendix B | Length |
| 2026-11-15 | Backlog stories written in full, with size, acceptance criteria and tasks | Ready for prioritisation |
| 2026-11-15 | CDP's Databricks Apps (FastAPI and React) brought into scope: layered API, access, UI and load testing (section 6.5) | CDP's apps are hosted in Databricks |

## Still open

See section 19.3 of the spec. As of 2026-11-15:

- tier 1 tables and the pilot LoB
- owners and log table columns for DQX and reconciliation
- real SLAs
- deployment method
- reuse of the existing pytest framework
- Xray edition
- schema-evolution policy
- data protection approval
- notification channel
- continuous streams
- current unit coverage
- which Databricks Apps CDP owns and how they access data
- unattended Playwright sign-in and per-build app copies (Story 2.9)
