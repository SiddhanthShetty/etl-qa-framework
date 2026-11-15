# Prompt 01: Recreate the CDP QA Testing Framework Specification

Use this prompt to rebuild the specification from scratch in a new AI session. It contains all the context and decisions from the original working session. Paste everything below the line.

If you have the whiteboard photos (`context/whiteboard/`), attach them too.

---

You are helping me, a QA Test Automation Lead, write the QA Testing Framework Specification for CDP.

## Context

- **CDP** is our data product. Databricks is the central platform: Unity Catalog, Databricks jobs (Workflows), Delta tables, Auto Loader ingestion and a bronze, silver and gold layout.
- **Goal:** make CDP a trustworthy, higher-quality product, with the testing framework in place first.
- **Reviewers:** Kedar, Gagan and Sukesh review the spec. The Product team then prioritises the backlog in its last section.
- **Length:** 15 to 20 pages for sections 1 to 19. Detailed code goes in an appendix.

## What already exists (reuse it; don't rebuild it)

- A **data quality framework built on Databricks DQX**. It writes results to its own Delta log table.
- A **CDP reconciliation framework** for source-to-target matching. It writes results to its own Delta log table.
- **Developer-owned unit tests** in pytest. QA does not write unit tests and only reads the developer build result.
- An existing **pytest framework with Allure HTML reporting**. The QA suite **extends it**; it is not replaced.
- **Jira with the Xray plugin**, which the team wants to use more. Assume the **latest Xray release**; the import script reads the Cloud or Server/Data Center endpoint from configuration.
- **Jenkins** for CI/CD. Not GitHub Actions.
- A **ReportPortal proof of concept** was done. The decision is not to adopt it for now: it overlaps with Allure, Xray and the Databricks dashboard, and it needs its own server.
- **CDP's Databricks Apps** use a FastAPI backend and a React frontend, hosted in Databricks. There are several, including a **lineage detection** app and a **dashboard** app.
- **Jobs can be parameterised.** Lakeflow Declarative Pipelines (DLT) are not used today but may be later.

## Requirements from the planning whiteboard

1. **What to test:** progression and regression, each black-box and white-box. Test types: functional, integration, data, boundary value, failure, transformation and NFR.
2. **How:** 80 to 85% automated, 15 to 20% process or manual. Integrated with spec-driven development.
3. **Content:** a generic CDP layer plus line-of-business (LoB) context.
4. **V-model:** requirements map to UAT, architecture and design to QA system and integration testing, and code to unit tests.
5. **Last section:** epics, stories and tasks for the Product team to prioritise.

## Decisions already made (keep them)

- **One tool per need:**
  - unit tests → developers;
  - data validity → DQX;
  - source-to-target totals → reconciliation;
  - functional, integration, transformation, failure and NFR → QA-owned pytest;
  - orchestration → Jenkins.
  
  Include a first-match decision table for which tool a new check belongs in.
- **Unified results view** `cdp_qa.results.test_results_v`. It is a view, not a copy, over the DQX log, the reconciliation log and a new QA results table. A run correlation table links each Databricks run to its `git_commit` and Jenkins `build_tag`.
- **Reporting is split three ways:**
  - Allure: why a test failed in a run.
  - Xray: traceability, manual tests, defects and release evidence.
  - Databricks dashboard: data health and trends across all frameworks.
  
  Each automated test carries its Xray key. A `conftest.py` hook writes it with `record_property("test_key", ...)` and `junit_family = xunit1`. DQX and reconciliation each count as one Xray gate test per pipeline.
- **Test isolation:** each build gets its own schemas (for example `cdp_test.qa_<id>_bronze`), landing and checkpoint paths, and a bundle `name_prefix`. All are passed as job parameters. Cleanup runs in `post { cleanup }`, and a nightly sweep catches leftovers.
- **Testability requirements for pipelines** are the exit criterion of the V-model design stage. Pipelines must take as job parameters:
  - catalog and schema per layer;
  - landing and checkpoint paths;
  - `as_of_date`;
  - `fail_before`, for deterministic failure injection;
  - `git_commit` and `build_tag`.
  
  They must also write audit logging and support `availableNow` triggers for streams.
- **Progression versus regression** is derived from Xray (story fix version), not from markers in code. There are three regression levels:
  - smoke and targeted regression on every merge;
  - full regression nightly and on release candidates.
  
  Targeted tests are chosen from the changed pipelines plus their downstream dependants, read from production `system.access.table_lineage`.
- **Merge pipeline budget is 30 minutes:** deploy 5, unit gate 2, merge tests 18, data quality gate 2, publish and cleanup 3. These are validated in the pilot.
- **Release-candidate testing** runs in the test environment. Pre-production is for UAT and a deployment smoke check only.
- **The automation ratio** is measured as automated Xray tests in scope ÷ all Xray tests in scope, per release.
- **Two separate scales:** test priority 1 to 3, and defect severity S1 to S4.
  - The release gate requires every priority 1 test to pass, every other failure to have a linked S3 or S4 defect or an unexpired waiver, no open S1 or S2 defects, and UAT signed off in Xray.
  - Pass rate is a trend measure, not a gate.
- **NFR is split in two:**
  - pre-release tests: performance, scalability, access, masking, recoverability, auditability, and cost checked the next day;
  - production monitoring: freshness, reliability, cost trend, and production DQX and reconciliation runs, owned by the platform team.
- **Privacy (GDPR):**
  - DPO approval is needed before any masked production copy is used;
  - a masking standard, access limits and retention rules;
  - no real row values in Jira, Allure or the results `message` column;
  - an erasure (right-to-be-forgotten) test.
- **Databricks Apps are in scope** (section 6.5), tested in layers:
  - developers own FastAPI `TestClient` and React unit tests;
  - QA owns API functional and contract tests (`httpx`, checked against the OpenAPI schema), data correctness against gold, access tests per role principal (priority 1), three to five Playwright UI journeys nightly, failure behaviour, and Locust load tests before release.
  
  Each build deploys its own app copy reading its own test data. API tests use a test service principal's OAuth token. Unattended Playwright sign-in is unproven and is a spike (Story 2.9). Apps must read catalog, schema and warehouse from configuration, publish OpenAPI, and use `data-testid` attributes.
- **Auto Loader failure scenarios** must reflect real behaviour:
  - `_rescued_data` and the bad-records path;
  - FAILFAST versus PERMISSIVE parse modes;
  - checkpoint de-duplication by file path only;
  - the default `addNewColumns` evolution mode.
  
  A schema-evolution and bad-record policy per tier is an open item.

## Document structure

1. Summary: status, reviewers, the decision needed, and what reviewers must confirm.
2. Purpose and scope: in scope, out of scope, guiding principles.
3. Goals and success measures: nine measures with proposed targets.
4. Test strategy:
   1. V-model levels
   2. Progression and regression (BB/WB grid with CDP examples)
   3. Black-box and white-box
   4. Test types
   5. Risk-based tiers
   6. Regression suite levels
   7. Lifecycle of a test
   8. Keeping the regression suite healthy
   9. Testability requirements
5. Tooling and ownership: tool table, "which tool" table, coverage matrix.
6. Functional and integration testing: approach, standard scenarios including erasure, integration scenarios, pass criteria, Databricks Apps (FastAPI and React).
7. Transformation and boundary value testing.
8. Failure testing: four assertions (safe state, detection, alerting, recovery), scenario catalogue, rules.
9. NFR: pre-release tests, how they run, test principals, production monitoring.
10. Data quality and reconciliation integration: what QA does and does not do, wiring tests, requesting a new rule.
11. Automation and Jenkins: automation ratio, one-time setup, pipeline stages and triggers, unified results view and run correlation.
12. Manual and process testing.
13. Spec-driven development: flow, example, definition of ready, definition of done, AI assistance (expected results always come from the spec).
14. Test data and environments: environments, data types, rules, privacy.
15. Entry and exit criteria: release gate, waivers, suspension.
16. Defects and reporting: priority and severity, triage, flaky tests, reporting tools, dashboard, rhythm.
17. Roles and responsibilities (RACI).
18. Generic CDP layer and LoB context: layers, configuration file per LoB, onboarding.
19. Risks, assumptions and open questions.
20. Backlog. Eight epics:
    1. inventory
    2. foundation
    3. pilot
    4. results and reporting
    5. NFR
    6. rollout
    7. release gate
    8. further LoB onboarding
    
    Each story is written as "As a…, I want…, so that…", with a size (S, M or L), acceptance criteria and tasks.

**Appendix A:** code samples (functional test, transformation test, failure test, repository layout, Jenkinsfile, LoB configuration YAML, Databricks App API and UI tests).
**Appendix B:** why ReportPortal is not adopted.

## Writing rules

- Plain English and short sentences. Lead each section with its point.
- Use tables for comparisons and catalogues, and numbered lists for steps.
- Mark all table names, SLAs and targets as illustrative or proposed until Epic 1 measures the baseline.
- Use realistic Databricks examples: `gold.customer_profile`, SCD Type 2 history, `total_spend_12m`, consent filtering, Auto Loader.

Write the specification section by section. Before you start, ask me for any real values I can give you now: tier 1 tables, SLAs, Xray edition, the LoB for the pilot.
