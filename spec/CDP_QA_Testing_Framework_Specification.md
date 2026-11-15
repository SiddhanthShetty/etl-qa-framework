# CDP QA Testing Framework Specification

Nov 12, 2026 · @Siddhanth

## 1. Summary

This spec defines one QA testing framework for CDP, our Databricks data product, so that every release can be shown to be trustworthy before it reaches consumers.

The framework does not replace what exists. It joins the DQX data quality framework, the CDP reconciliation framework and developer unit tests with a new QA-owned pytest suite, triggers and gates all of them from Jenkins, and reports them in one place.

| Item | Detail |
| --- | --- |
| Status | Draft for review |
| Reviewers | Kedar, Gagan, Sukesh |
| Decision needed | The Product team prioritises the backlog in section 20 |
| Automation split | 80–85% automated, 15–20% manual or process |
| New work for QA | Functional, transformation, failure and NFR tests; Jenkins orchestration; one unified results view and dashboard |
| Reused as is | DQX (data quality), reconciliation framework (source-to-target matching), developer unit tests |

Reviewers are asked to confirm four things:

1. The scope in section 2.
2. The ownership split in sections 5 and 17.
3. The targets marked "proposed" in sections 3 and 9. They are placeholders until the baseline is measured.
4. The order of the epics in section 20.

## 2. Purpose and scope

The purpose is to give CDP one agreed way of testing, so that quality does not depend on who built a pipeline or which line of business it serves.

### In scope

- Ingestion into bronze, including file and stream landing through Auto Loader or equivalent.
- Transformations from bronze to silver to gold.
- Consumer-facing gold tables and views.
- CDP's Databricks Apps (FastAPI backend, React frontend) and the data they serve (section 6.5).
- Databricks jobs and workflows that run these pipelines, including scheduling, dependencies, retries and alerts.
- Access control and masking on CDP data in Unity Catalog.
- Deployment of pipeline and app code to the test environment through Jenkins.

### Out of scope

- The internal correctness of source systems. CDP tests what it receives, not how the source produced it.
- Downstream applications outside CDP that consume its data. CDP's own Databricks Apps are in scope (section 6.5).
- The internals of the DQX and reconciliation frameworks. This spec covers how they are used and gated, not how they are built.
- Writing unit tests. Developers own them. This spec only sets the expectation that they exist and pass.
- The Databricks platform itself, such as cluster provisioning and workspace administration.

### Guiding principles

1. **Reuse, don't rebuild.** If DQX or reconciliation already covers a check, QA does not write a second version of it.
2. **Test from the spec.** Every test traces back to an acceptance criterion or a mapping rule.
3. **One results store.** Every framework's results are read through one unified results view, so there is one answer to "is this release good?".
4. **Fail safe.** A pipeline that breaks must never leave partial or misleading data in gold.
5. **Risk first.** The most critical tables get the deepest coverage first.

## 3. Goals and success measures

"Trustworthy" is defined here as nine measures, each with a proposed target that the team confirms once the baseline is known.

| Measure | How it is measured | Proposed target |
| --- | --- | --- |
| Automation ratio | Automated Xray tests in scope for tier 1 and 2 ÷ all Xray tests in scope, per release | 80–85% |
| Automated coverage of critical tables | Share of tier 1 tables with every applicable test type automated | 80% within two quarters |
| Regression pass rate (trend) | Passed ÷ executed regression tests per run, tracked over time. A trend measure, not a gate (section 15) | Stable at or above 95% |
| Defect leakage | Defects found in production per release | ≤ 2 per release, zero S1 |
| Data incidents | Incidents per month on tier 1 tables | Down 50% from baseline |
| Data quality on gold | DQX checks of critical severity passing at release | 100% |
| Reconciliation | Unexplained source-to-target breaks at release | 0 |
| Feedback time | Duration of the merge pipeline in Jenkins | ≤ 30 minutes |
| Flaky tests | Tests that change result on rerun with no code change | < 2% of the suite |

The baseline for each measure is captured in Epic 1 of the backlog. Until then the targets are a starting point for discussion, not commitments.

### What these measures are for

- **For the team:** a release gate that is the same every time (section 15).
- **For the Product team and stakeholders:** a trend they can read without knowing the tools.
- **For QA:** evidence of where coverage is thin, which feeds the next round of prioritisation.

## 4. Test strategy

CDP follows the V-model: every build stage has a matching test stage, and the test for a stage is designed when that stage is specified, not after the code is written. The architecture and design stage is complete only when the pipeline meets the testability requirements in section 4.9.

### 4.1 V-model levels

| Build stage | Matching test stage | What it proves | Owner |
| --- | --- | --- | --- |
| Requirements | User acceptance testing (UAT) | The product does what the business asked for | Line of business, supported by QA |
| Architecture and design | QA system and integration testing | Pipelines work end to end and with each other | QA |
| Code | Unit testing | Each function or transformation works in isolation | Developers |

### 4.2 Progression and regression

Progression and regression are the top-level split of what QA tests. Every functional, transformation, failure and NFR test is one or the other.

- **Progression testing** proves that new or changed functionality in the current release works. Tests are written from the story's acceptance criteria.
- **Regression testing** proves that existing functionality still works. Every progression test that passes joins the regression suite once its story is released.

|  | Progression: the new change works | Regression: existing behaviour still works |
| --- | --- | --- |
| Black-box | Load a customer with €1,200 of completed orders, run the job, check `loyalty_tier = Gold` in gold | An address change still creates a history row; customers without consent still stay out of the marketing view |
| White-box | Check the intermediate silver value, the €999.99 and €1,000 boundaries, and that the Delta `MERGE` updates the row instead of inserting a second one | Downstream views still resolve after the schema change; the job still reruns with no duplicates; run time has not grown |

A test's suite is not hard-coded. It is derived at run time from its Xray test key: a test linked to a story whose fix version is the current release is progression, and every other test is regression. Tests carry pytest markers only for `smoke`, test type and line of business, all registered in `pytest.ini`.

### 4.3 Black-box and white-box

Both progression and regression use both techniques.

- **Black-box (BB):** the tester only uses inputs and outputs. Example: load a customer in the source and check the gold profile.
- **White-box (WB):** the tester uses knowledge of the code, the job structure or the intermediate tables. Example: stop a job after the silver task to prove gold is not half-written.

### 4.4 Test types

| Test type | Question it answers | Technique | Section |
| --- | --- | --- | --- |
| Functional | Does the pipeline behave as the requirement says? | BB | 6 |
| Integration | Do pipelines, jobs and consumers work together? | BB | 6 |
| Transformation | Is each mapping rule applied correctly, row by row? | BB and WB | 7 |
| Boundary value | Do edge values behave correctly? | BB and WB | 7 |
| Failure | Does the pipeline fail safely and recover? | BB and WB | 8 |
| NFR | Is it fast, fresh, secure, recoverable and affordable? | BB and WB | 9 |
| Data | Is the data valid, complete and consistent? | BB | 10 (DQX and reconciliation) |

### 4.5 Risk-based tiers

Not every table needs the same depth. Each table or pipeline is placed in a tier during the inventory in Epic 1.

| Tier | Definition | Minimum coverage |
| --- | --- | --- |
| 1 | Feeds external consumers, regulatory reports or key business decisions | All test types, automated, in the release gate |
| 2 | Feeds internal analytics | Functional, transformation, data quality and reconciliation |
| 3 | Staging, reference or low-use data | Data quality and reconciliation only |

### 4.6 Regression suite levels

The regression suite runs at three levels, so a merge gets feedback inside the 30-minute target while full coverage still runs every night.

| Level | Contents | When it runs | Time budget |
| --- | --- | --- | --- |
| Smoke | About five key tests for each tier 1 pipeline the change touches: load, update, rerun, one key rule, consumer view | Every merge | Shares the 18-minute merge test stage (section 11.3) |
| Targeted regression | All regression tests for the touched pipelines and their downstream dependants | Every merge | Shares the 18-minute merge test stage |
| Full regression | Every regression test across all pipelines and lines of business | Nightly, and on every release candidate | 90 minutes |

**Choosing the targeted set.** A pytest option takes the list of touched pipelines from the changed files, using the line-of-business configuration in section 18. Downstream dependants are read from production lineage in `system.access.table_lineage`, not from test runs, which would pollute it. If the mapping is unclear, the merge runs smoke for all tier 1 pipelines and the full suite runs that night.

**How the budget is met.** Each pipeline runs once per merge with many assertions against its output, as transformation tests already do (section 7.1). Pipelines run in parallel with `pytest-xdist`, each worker in its own schemas. Jobs use serverless compute or a warm instance pool, so cluster start-up does not eat the budget. All budgets are validated in the Epic 3 pilot and adjusted before rollout.

### 4.7 Lifecycle of a test

1. A progression test is written from a story's acceptance criteria and linked to that story's Xray test.
2. It runs on every merge that touches its pipeline, as progression, while the story is in the current release.
3. Once the story is released, the same test counts as regression with no code change. If it protects a core behaviour of a tier 1 pipeline, it is also marked `smoke`.
4. It is changed only when the requirement changes, and retired only when the feature is removed.

### 4.8 Keeping the regression suite healthy

- **Owner:** QA owns the regression suite. Developers fix the regressions their changes cause.
- **A regression failure blocks the merge.** It is never worked around by editing the test, unless the requirement itself has changed.
- **Review each release:** remove tests for retired features, merge duplicates, and move slow tests out of smoke.
- **Time budgets are limits.** When a level exceeds its budget, tests are sped up or moved to a slower level, not deleted.
- **Progression is partly manual at first.** New features get exploratory testing alongside their automated progression tests. Regression is fully automated.

### 4.9 Testability requirements for CDP pipelines

QA can only test pipelines that are built to be tested. These requirements are the deliverable of the V-model's architecture and design stage: a pipeline's design is not complete until it meets them, and they are agreed with developers in Epic 2.

| Requirement | Why the tests need it |
| --- | --- |
| Catalog and schema for every layer are job parameters, and every table reference resolves through them | Each build writes to its own schemas instead of shared tables (section 11.2) |
| Landing paths and Auto Loader checkpoint paths are job parameters | Rerun, duplicate and out-of-order tests start from a clean checkpoint every build |
| A reference date (`as_of_date`) is a job parameter, defaulting to today | Date-window rules such as `total_spend_12m` give the same result every run |
| A `fail_before` parameter makes a no-op task fail on purpose before a named task | Failure tests stop a run at an exact point instead of racing a cancel (Appendix A.3) |
| `git_commit` and `build_tag` are job parameters passed to every task, including DQX and reconciliation | Every result can be traced to the commit and Jenkins build under test (section 11.4) |
| Each run logs its run ID, rows in, rows out and status to the audit table | Auditability checks and failure diagnosis |
| Streaming sources can be run with `trigger(availableNow=True)` | A test can wait for a stream to finish processing what was landed |
| Databricks Apps read their catalog, schema and SQL warehouse from configuration, and the FastAPI backend publishes its OpenAPI schema | Each build's app copy serves the build's test data, and API contract tests have a schema to check against (section 6.5) |
| React elements used in critical journeys carry stable data-testid attributes | UI tests do not break when layout or styling changes |

CDP jobs can already take parameters, so these are changes to how parameters are used, not to the platform.

## 5. Tooling and ownership

Each testing need has exactly one tool and one owner, so no rule is written twice and no gap is left unowned.

| Need | Tool | Owner | Status |
| --- | --- | --- | --- |
| Unit tests on transformation code | pytest | Developers | Exists, coverage to be confirmed |
| Data quality checks | CDP data quality framework built on Databricks DQX | Its current owning team | Exists |
| Source-to-target matching | CDP reconciliation framework | Its current owning team | Exists |
| Functional, integration, transformation, failure and NFR tests | pytest suite | QA | New |
| Orchestration of all test runs | Jenkins | QA with the platform team | Exists, test stages are new |
| Results store and dashboard | Unified view over Delta result tables, and a Databricks dashboard | QA | New |
| Detailed report of each test run | Allure, published by Jenkins | QA | Exists |
| Traceability, manual tests, defects and release evidence | Xray in Jira | QA with the Product team | Exists, use to be expanded |
| Databricks Apps unit tests (FastAPI and React) | pytest with FastAPI TestClient; Vitest or Jest | Developers | To be confirmed |
| Databricks Apps API, access, UI and performance tests | pytest with httpx, Playwright, Locust | QA | New |

### 5.1 Which tool does a new check belong in?

Use the first row that matches.

| If the check is about… | It belongs in… | Example |
| --- | --- | --- |
| A single function, with no cluster or tables needed | Developer unit test | `clean_name("  John ")` returns `"John"` |
| The validity of data in a table, whatever produced it | DQX | `customer_id` is unique and not null in gold |
| Totals or counts matching between two layers | Reconciliation | Row count in bronze equals source extract |
| A business rule producing the right value for a given input | QA transformation test | Cancelled orders are excluded from `total_spend_12m` |
| Behaviour of the pipeline as a whole | QA functional test | An address change creates a new history row |
| What happens when something breaks | QA failure test | A missing file leaves gold unchanged |
| Speed, freshness, access, recovery or cost | QA NFR test | The daily load finishes inside its window |

### 5.2 Coverage matrix

The coverage matrix is the working tool that keeps this split honest. It lists tier 1 and tier 2 tables down the side and test types across the top. Each cell names the framework that covers it, or is marked as a gap.

| Table (illustrative) | Unit | Data quality | Reconciliation | Functional | Transformation | Failure | NFR |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `gold.customer_profile` | Dev | DQX | Recon | Gap | Gap | Gap | Gap |
| `silver.orders` | Dev | DQX | Recon | Gap | Gap | Gap | Not needed |

The gaps in the real matrix become QA stories. The matrix is reviewed at the start of every release.

## 6. Functional and integration testing

Functional tests prove that a pipeline behaves as its requirement says, from source input to gold output, without looking at the code.

### 6.1 Approach

Every functional test follows the same four steps:

1. **Arrange:** a pytest fixture places controlled input in the test landing volume or bronze table.
2. **Act:** the test starts the real Databricks job and waits for it to finish.
3. **Assert:** the test queries the output table and compares it with the expected result.
4. **Clean up:** the test schema is dropped at the end of the run.

The job under test is the production job definition, deployed to the test target. The build passes its own catalog, schemas, landing and checkpoint paths as job parameters (section 4.9), so the test never touches shared tables. Tests never call transformation functions directly. That is the unit-test level. An example test is in Appendix A.1.

### 6.2 Standard scenarios

Every tier 1 pipeline is tested against these scenarios where they apply.

| Scenario | Input | Expected result |
| --- | --- | --- |
| New record | Customer C100 arrives in the source | C100 is in `gold.customer_profile` with matching name, email and country |
| Update with history (SCD Type 2) | C100's address changes from Dublin to Cork | Dublin row has `is_current = false` and an end date; Cork row has `is_current = true` |
| Delete | Source flags C100 as deleted | C100 is removed or flagged as the requirement states, never left active |
| Rerun with no new data | Job runs twice on the same input | Row count unchanged, no duplicates |
| Late-arriving data | An order dated last week arrives today | The order lands in the correct period and aggregates are updated |
| Consumer view filter | Customer with `consent = N` | Customer does not appear in the marketing view |
| Empty input | A valid file with zero rows | Job succeeds, gold unchanged, run is logged |
| Erasure request | A right-to-erasure request arrives for C100 | C100 is removed from every gold table and view; older Delta versions holding C100 are purged once the agreed retention and VACUUM have run |

### 6.3 Integration scenarios

Integration tests cover the joins between pipelines and jobs.

- **Job dependencies:** the gold job starts only after the silver job succeeds.
- **Cross-pipeline joins:** a customer loaded by one pipeline joins correctly to orders loaded by another.
- **Consumer contract:** the columns, types and names of a gold table match the published contract. A removed or renamed column fails the test.
- **Hand-off to DQX and reconciliation:** both run as tasks in the job and write their results (section 10).

### 6.4 Pass criteria

A functional test passes only when the job succeeds and every assertion holds. A job that succeeds with wrong data is a failure, and so is a job that produces right data but ends in a failed state.

### 6.5 Databricks Apps (FastAPI and React)

CDP's Databricks Apps are tested in layers, with most checks at the API level and only a few critical journeys in a browser. API tests are fast and stable; browser tests are slow and break when the layout changes.

| Layer | What it proves | Tool | Owner | When |
| --- | --- | --- | --- | --- |
| Backend unit | FastAPI route and service logic, with the Databricks SQL calls mocked | pytest with FastAPI `TestClient` | Developers | Developer build |
| Frontend unit | React components render and behave correctly | Vitest or Jest with React Testing Library | Developers | Developer build |
| API functional and contract | The deployed app's endpoints return the right status codes, match the published OpenAPI schema and handle bad input | pytest with `httpx` | QA | Every merge that touches the app |
| Data correctness | Numbers returned by the API equal a direct SQL query on the build's gold tables for the same filters | pytest | QA | Every merge that touches the app or its tables |
| Access | Each role sees only its permitted data; an unauthenticated call is refused | pytest as the role-based test principals (section 9.2) | QA | Every merge; priority 1 |
| UI end to end | Three to five critical user journeys work in a real browser | Playwright through `pytest-playwright` | QA | Nightly and release candidate |
| Failure | When the SQL warehouse is stopped or slow, a table is missing or the backend errors, the UI shows a clear message and never shows stale or partial data as current | pytest and Playwright | QA | Nightly |
| Performance | API and page response times under expected load (section 9) | Locust | QA | Release candidate |

**Isolation.** Each build deploys its own copy of the app with the bundle prefix (section 11.2). The app reads its catalog and schema from configuration, so the build's copy serves the build's test data. If the workspace's app limits or start-up time make one app per build impractical, a single shared test app is used and app tests run one build at a time.

**Authentication.** API tests call the app with an OAuth token for a test service principal in the `Authorization` header, as Databricks documents for calling apps from outside the browser ([Databricks docs](https://docs.databricks.com/gcp/dev-tools/databricks-apps/connect-local)). How Playwright signs in unattended for UI tests is not yet proven and is the subject of a spike (Story 2.9). If the app uses on-behalf-of-user access, access tests run once per role principal.

**Results.** App tests carry Xray keys like every other QA test, appear in Allure, and write to the unified results view with `test_type` set to app. Playwright saves a trace and screenshot for every failed UI test and attaches them to the Allure report.

An example API test and UI test are in Appendix A.7.

## 7. Transformation and boundary value testing

Transformation tests prove that every rule in the source-to-target mapping document produces the right value for each individual row.

This is different from the two neighbouring levels. Developer unit tests check a function on its own. Reconciliation checks that totals match. Transformation tests check each rule inside the real pipeline, against the mapping document.

### 7.1 Approach

1. Each mapping rule is written as one or more cases: rule ID, input values, expected output.
2. Cases are kept in a version-controlled file next to the tests, one file per target table.
3. A fixture seeds the bronze table with all inputs for a table in one batch.
4. The pipeline runs once.
5. A parametrised pytest test checks each case, so every rule shows as its own pass or fail.

Running the job once for many rules keeps the Jenkins stage fast. An example is in Appendix A.2.

### 7.2 Rule categories and examples

| Category | Rule (illustrative) | Input | Expected output |
| --- | --- | --- | --- |
| Derived field | `full_name` = trimmed first name + last name | `"  John "`, `"Smith"` | `"John Smith"` |
| Derived field, null handling | Same rule | `"John"`, null | `"John"`, not `"John null"` |
| Lookup | Country code to country name | `IE` | `Ireland` |
| Lookup, no match | Same rule | `XX` | `Unknown`, not null |
| Aggregation | `total_spend_12m` sums completed orders in the last 12 months | €100 completed, €50 cancelled, €30 from 14 months ago | `100` |
| Deduplication | Latest record wins per email | Two records, updated 1 Nov and 5 Nov | One profile with the 5 Nov values |
| Date format | Source dates are dd/MM/yyyy | `09/11/2026` | `2026-11-09` |
| Type cast | Amount string to decimal | `"1,250.50"` | `1250.50` |
| Conditional | `segment` = Gold when spend ≥ 1,000 | Spend 1,000 | `Gold` |

### 7.3 Boundary values

Each rule with a threshold, range or length limit gets cases on, just below and just above the boundary, plus null and empty.

| Rule | Boundary cases |
| --- | --- |
| `segment` = Gold when spend ≥ 1,000 | 999.99, 1,000.00, 1,000.01, 0, negative, null |
| Age must be 0–120 | -1, 0, 120, 121, null |
| 12-month window | Order dated exactly 12 months ago, one day older, one day newer |
| Name length limit of 100 | 99, 100 and 101 characters, empty string |
| Timestamps | Midnight, daylight-saving change, end of month, 29 February |

### 7.4 Traceability

Every case carries the rule ID from the mapping document. A mapping rule with no case is a coverage gap and is reported on the dashboard.

## 8. Failure testing

Failure tests prove that when something goes wrong, the pipeline stops safely, tells someone, recovers cleanly and never leaves partial or misleading data in gold.

### 8.1 What every failure test asserts

Each test checks up to four things after the fault is injected:

1. **Safe state:** the gold table's latest version in `DESCRIBE HISTORY` is unchanged, and every related gold table is consistent with the others. Delta commits are atomic per table, so the real risk is a run that commits to one table and not another, or makes several commits to one table.
2. **Detection:** the job run ends in the expected state, with a clear error message.
3. **Alerting:** the agreed alert fires to the agreed channel.
4. **Recovery:** a rerun after the fault is removed completes with correct data and no duplicates.

### 8.2 Scenario catalogue

| Scenario | How the fault is injected | Expected behaviour |
| --- | --- | --- |
| Missing source file | No file is placed for the day | Job skips or fails with an alert; gold is unchanged, not emptied |
| Schema drift, renamed column | File arrives with `email` renamed to `email_address` | Behaves as the schema policy states. Under Auto Loader's default `addNewColumns` mode the stream stops, then on restart adds the new column and leaves `email` null, so the policy must block or alert on that. Never loaded as silent nulls |
| Schema drift, new column | File arrives with an extra column | Handled as the schema policy states; existing columns unaffected |
| Unparseable rows | 5 rows that cannot be parsed in a file of 1,000 | 995 rows load. The 5 land in `_rescued_data` or the bad-records path, as the policy states, and a DQX rule on `_rescued_data` raises them |
| Rule-breaking rows | 5 parseable rows that break a critical DQX rule | 995 rows reach gold, 5 go to the DQX quarantine table, alert fires above the agreed threshold |
| Corrupt or truncated file | File cut off mid-record | As the policy states: in FAILFAST mode the batch fails and nothing loads; in PERMISSIVE mode the readable rows load and the rest are rescued |
| Crash before gold | `fail_before = gold_customers` makes the run fail just before the gold task | Gold is unchanged; a rerun completes with no duplicates and no partial gold data |
| Task retry | A task fails once with a transient error and is retried | The retried task gives the same result as a clean run; no duplicates |
| Duplicate delivery, same file | The same file is landed twice | Second delivery is ignored by the Auto Loader checkpoint; row counts do not double |
| Duplicate delivery, new file name | The same content lands under a new file name | The pipeline removes duplicates by business key. Auto Loader alone would load it twice |
| Upstream job failure | The silver job is forced to fail | Gold job does not run on stale data |
| Out-of-order delivery | Tuesday's file arrives before Monday's | Final state is the same as in-order delivery |
| Volume spike or drop | File is 10 times, or one tenth of, the normal size | Job completes or fails safely; a volume alert fires |
| Permission revoked | The job's service principal loses access to a table | Job fails with a clear permission error; no partial write |
| Rollback | A bad load is allowed into gold in the test environment | Delta `RESTORE` to the previous version works and the steps are documented. Restore is limited by the VACUUM retention period, and downstream streams reading the table may need restarting |

The schema-evolution and bad-record policy for each tier is agreed in Epic 2. Until then, the expected behaviour for the two schema-drift rows, the unparseable rows and the truncated file is open. An example failure test is in Appendix A.3.

### 8.3 Rules for running failure tests

- They run only in the build's own isolated test schemas, never against shared, pre-production or production data.
- They run nightly and before each release, not on every merge, because they are slow and deliberately disruptive.
- Each scenario that is not yet handled by the pipeline is raised as a defect against the pipeline, not skipped in the test suite.

## 9. Non-functional (NFR) testing

NFR tests prove how well CDP works, not what it does: speed, scale, freshness, security, recovery, auditability and cost. Each has a measurable target agreed with the Product team.

NFR checks come in two kinds: pre-release tests that Jenkins runs before a release can go out, and production monitoring that runs continuously after it does. All targets are proposed until agreed in Epic 5.

### Pre-release NFR tests

| Category | Check | How it is measured | Proposed target |
| --- | --- | --- | --- |
| Performance | Daily load of normal volume completes in its window | Run duration from the Jobs API, read during the Jenkins run | Under 30 minutes for about 10M rows |
| Performance | Consumer query on gold responds quickly | Query history duration on the SQL warehouse | Under 5 seconds for the standard queries |
| Scalability | Load at three times normal volume completes | Same job on a 3x synthetic data set | Inside the SLA, no out-of-memory failure |
| Security, access | Roles see only what they should | Queries run as test principals in each group | Analyst cannot read raw PII; ungranted principal gets `PERMISSION_DENIED` |
| Security, masking | PII is masked for non-privileged roles | Select the masked column as an analyst principal | Email shown as `j***@***.com` |
| Recoverability | A table can be restored to a previous version | Timed Delta `RESTORE` in the performance schema | Inside 15 minutes |
| Auditability | Every run is traceable | Audit table has run ID, rows in, rows out, status | 100% of runs logged |
| Cost | A standard run stays within budget | DBUs for the run from `system.billing.usage`, checked the next day because billing data arrives hours late | Within the agreed DBU budget per run |
| App performance | The Databricks App stays responsive under expected load | Locust load test against the build's app copy, with response times from Locust | 95% of API calls under 1 second and pages usable within 3 seconds, with 20 concurrent users |

### 9.1 How NFR tests are run

- **Data:** production-sized volumes in a separate performance schema, built from masked copies or synthetic data (section 14).
- **Compute:** the same cluster or warehouse size as production. A result on a smaller cluster is not valid evidence.
- **Timing:** before each release, and after any major change to a tier 1 pipeline, using a Jenkins parameter.
- **Trend, not just pass or fail:** each run's duration and DBU cost is stored. A run that passes but is 20% slower than the last three is flagged for review.

### 9.2 Security tests need test principals

Security tests connect as dedicated service principals, one for each consumer role. These principals are members of the same Unity Catalog groups as real users in that role. They are requested in Epic 2.

### 9.3 Production monitoring

These checks can only be measured in production, so they are not part of the release gate. The platform team owns them; QA owns the check definitions and reviews their trend each release.

| Check | How it is measured | Proposed target |
| --- | --- | --- |
| Freshness | `max(load_timestamp)` on each tier 1 gold table, checked by a scheduled job | Updated by 06:00, data no older than 24 hours |
| Reliability | Success rate of scheduled runs over 30 days, from `system.lakeflow.job_run_timeline` | ≥ 99% |
| Cost trend | DBUs per pipeline per day from `system.billing.usage` | No unexplained rise above 20% week on week |
| Data quality and reconciliation | The production runs of DQX and reconciliation | No critical failures unresolved after one working day |

## 10. Data quality and reconciliation integration

Data quality and reconciliation stay with their existing frameworks and owners. QA's job is to make sure they are switched on for every tier 1 and tier 2 table, wired into the release gate, and reported with everything else.

### 10.1 What each framework covers

| Framework | Covers | Typical checks |
| --- | --- | --- |
| Data quality (DQX-based) | Validity of data inside a table | Not null, unique, allowed values, ranges, formats, referential integrity, schema |
| Reconciliation | Agreement between two layers or systems | Row counts, sums of key measures, key-level matching, missing and extra records |

### 10.2 What QA does and does not do

| QA does | QA does not |
| --- | --- |
| Checks that every tier 1 and tier 2 table has rules in both frameworks, using the coverage matrix | Write or maintain the rule engines |
| Raises requests for missing rules from acceptance criteria | Duplicate a DQX or reconciliation rule in pytest |
| Tests that the wiring works: bad rows are quarantined, thresholds trigger alerts, a failed critical check stops the job | Decide rule thresholds alone. These are agreed with data owners |
| Makes both frameworks' results visible in the unified results view | Change how either framework stores its own detail |
| Includes both in the release gate |  |

### 10.3 Wiring tests

These are QA-owned pytest tests that prove the frameworks are connected correctly. They do not test the rules themselves.

- A row that breaks a critical DQX rule lands in the quarantine table and not in gold.
- A batch above the agreed error threshold fails the job and raises an alert.
- A deliberately mismatched load produces a reconciliation break with the correct difference reported.
- Both frameworks write a result row for every run, including runs with zero failures.

### 10.4 Requesting a new rule

1. A story's acceptance criterion describes a data rule. Example: "`loyalty_points` is never negative."
2. QA classifies it using the guide in section 5.1.
3. If it is a DQX or reconciliation rule, QA raises a request to the owning team with the table, column, rule and severity.
4. The story is not done until the rule is live in the test environment and appears in the unified results view. If the owning team cannot add it within the sprint, the story closes with the request tracked as an open dependency.

## 11. Automation and Jenkins execution

Jenkins deploys the code, starts the tests and collects the results. Databricks does all the data work. The Jenkins agent runs pytest, and pytest drives Databricks: it loads test data, starts jobs and queries the output.

### 11.1 What is automated

The 80–85% target is one ratio, measured per release: automated Xray tests in scope for tier 1 and 2 pipelines, divided by all Xray tests in scope. A test is in scope when it is linked to a requirement or mapping rule of a tier 1 or 2 pipeline. DQX and reconciliation each count as one automated gate test per pipeline (section 16.4). What stays manual is listed in section 12.

### 11.2 One-time setup

- **Identity:** a Databricks service principal for Jenkins, authenticated with OAuth. Its client ID and secret are held in Jenkins credentials.
- **Agent:** a labelled Jenkins agent, or a pinned container image, with Python, the Databricks CLI, the Databricks SDK for Python, and Databricks Connect or the SQL connector.
- **Deployment:** each build deploys the jobs with Databricks Asset Bundles to the test target with its own `name_prefix`, so concurrent builds never overwrite each other's job definitions.
- **Isolation:** each build gets a unique ID from Jenkins `BUILD_TAG`, which includes the Jenkins job name, so merge and nightly builds cannot collide. The build creates its own schemas, such as `cdp_test.qa_<id>_bronze`, `_silver` and `_gold`, plus its own landing and checkpoint paths. All of these are passed to the jobs as parameters (section 4.9).
- **Cleanup:** at the end of every build, pass or fail, the build runs `bundle destroy` for its jobs, drops its schemas with `DROP SCHEMA ... CASCADE` and removes its paths. A nightly job removes anything older than two days that a crashed build left behind.

### 11.3 Pipeline stages and triggers

The merge pipeline is budgeted at 30 minutes in total, the feedback-time target in section 3. All budgets are validated in the Epic 3 pilot. The repository layout is in Appendix A.4 and the Jenkinsfile outline in Appendix A.5.

| Stage | What runs | Trigger | Budget |
| --- | --- | --- | --- |
| Deploy | Create this build's schemas and paths; bundle deploy with this build's prefix | Every build | 5 minutes |
| Unit gate | Read the developer build result for this commit. Unit tests are not rerun | Every merge | 2 minutes |
| Merge tests | Smoke and targeted regression for the touched pipelines and apps, their progression tests, wiring tests, and app API and access tests, in parallel | Every merge | 18 minutes |
| Data quality and reconciliation gate | Read DQX and reconciliation results for this build's runs from the unified view | Every build | 2 minutes |
| Full regression | Every regression test, including app UI journeys in Playwright | Nightly, and release candidate | 90 minutes |
| Failure | `tests/failure` | Nightly, and release candidate | 60 minutes |
| NFR | `tests/nfr` on production-sized data | Release candidate, by parameter | 2 hours |
| Publish and cleanup | JUnit and Allure reports, QA results and run correlation to Delta, import to Xray, then remove this build's jobs, schemas and paths | Always, even on failure | 3 minutes |

Every stage runs in the test environment against the release candidate's commit, including release-candidate runs. Pre-production is used only for a deployment smoke check and UAT, because its masked copy of production cannot give the exact results these tests assert. The NFR stage runs in the performance schema.

Tests are selected through pytest options, not hand-written marker expressions. For example, a merge that touches the customer pipeline runs:

```bash
pytest --pipelines=$(tools/changed_pipelines.py) --levels=smoke,targeted --with-progression
```

### 11.4 Unified results view

The unified results view, `cdp_qa.results.test_results_v`, is the single place every result is read from. It is a view, not a copy, over three sources:

1. The existing DQX log table, unchanged.
2. The existing reconciliation log table, unchanged.
3. A new QA results table, written by `publish_results.py`. It holds pytest results, the developer unit test result for the commit, and release-gating manual results exported from Xray.

DQX and reconciliation log their results by Databricks job run. To link them to the commit and Jenkins build under test, `publish_results.py` also writes a run correlation table, `cdp_qa.results.run_correlation`, with one row per job run that Jenkins started: `build_tag`, `git_commit`, `environment` and `databricks_run_id`. Because every job also receives `git_commit` and `build_tag` as parameters (section 4.9), the view can join each framework's rows to the build.

| Column | Meaning |
| --- | --- |
| `build_tag` and `git_commit` | The Jenkins build and commit under test, through the correlation table |
| `databricks_run_id` | The job run that produced the result |
| `executed_at` and `environment` | When and where the check ran |
| `source_framework` | `pytest`, `dqx`, `reconciliation`, `unit` or `manual` |
| `test_type` | Functional, transformation, failure, NFR, data quality, reconciliation, unit, manual, app |
| `suite` | Progression or regression, derived from Xray (section 4.2) |
| `priority` | Test priority 1 to 3 (section 16.1) |
| `tier` and `line_of_business` | Tier of the table under test, and its LoB or generic |
| `pipeline` and `table_name` | What was tested |
| `test_name`, `xray_key` and `rule_id` | Which check, its Xray test, and the mapping rule it traces to |
| `status` | Passed, failed, error, skipped, quarantined or waived |
| `duration_seconds` and `message` | Run time, and failure detail with no row values from non-synthetic data (section 14.4) |

## 12. Manual and process testing

About 15–20% of in-scope tests stay manual on purpose (the ratio in section 11.1), because they need human judgement or run too rarely to be worth automating. Alongside them, QA does process work that is not testing but keeps the framework honest. The table covers both.

| Activity | Why it stays manual | When it happens |
| --- | --- | --- |
| Exploratory testing of new features | Finds problems nobody thought to write a test for | Each release, on new tier 1 functionality |
| Review of requirements and mapping documents | Catches ambiguity before code is written | Story refinement |
| Test design | Deciding what to test is a judgement call; running it is not | When a story is refined |
| UAT support | Business users need help with data set-up and with reading results | UAT phase |
| Dashboard and report checks | Layout, labels and readability are judged by eye | When a consumer-facing view changes |
| One-off data migrations and backfills | Run once, so automation does not pay back | As they occur |
| Failure investigation | Deciding whether a failure is a product defect, a test defect or an environment issue | After every failed run |
| Release sign-off | A named person accepts the evidence | Each release |

### 12.1 Rules for manual work

- Any manual check that is repeated in three releases is reviewed for automation.
- Exploratory sessions are time-boxed and their findings are logged, so a useful finding becomes an automated test.
- Manual tests are planned and executed in Xray. Release-gating manual results are also added to the unified results view with `source_framework = manual`, so the dashboard shows the whole picture.

## 13. Integration with spec-driven development

Tests are derived from the spec before the code is written, so a story cannot be called done without the checks that prove it.

### 13.1 Flow for one story

1. **Spec:** the story is written with acceptance criteria in Given / When / Then form, plus any mapping rules.
2. **Classify:** QA maps each criterion to a test type and tool using section 5.1.
3. **Design:** QA writes the test cases and expected results while development starts. Developers write unit tests alongside the code.
4. **Build:** QA automates the pytest tests. DQX and reconciliation rule requests go to their owners.
5. **Run:** Jenkins runs everything on merge.
6. **Trace:** each automated test carries its Xray test key and rule ID, so Jira shows which stories are proven and the dashboard shows which mapping rules are.

### 13.2 Example

Story: merge duplicate customer profiles that share an email address.

| Acceptance criterion | Test type | Tool |
| --- | --- | --- |
| Given two profiles with the same email, when the pipeline runs, then one profile remains | Functional | pytest |
| The surviving profile keeps the most recently updated address | Transformation | pytest |
| `email` is unique in `gold.customer_profile` | Data quality | DQX rule request |
| The count of distinct emails in silver equals the row count in gold | Reconciliation | Reconciliation rule request |
| A crash during the merge leaves no half-merged profiles | Failure | pytest |
| The merge step adds no more than 5 minutes to the daily run | NFR | pytest |

### 13.3 Definition of ready

A story can enter a sprint only when:

- Acceptance criteria are written and testable.
- Mapping rules are documented for any new or changed column.
- The table's tier is known.
- Test data needs are identified.

### 13.4 Definition of done

A story is done only when:

- Unit tests exist and pass.
- Every acceptance criterion has at least one automated check, or a recorded reason for a manual one.
- Requested DQX and reconciliation rules are live in the test environment.
- All checks pass in Jenkins and appear in the unified results view.
- New tests are tagged to join the regression suite.

### 13.5 AI assistance

AI tools may draft test cases from acceptance criteria and mapping rules. A QA engineer reviews every generated case before it is merged, and the expected results always come from the spec, never from the current behaviour of the code.

## 14. Test data and environments

Tests run in dedicated, isolated schemas with data that is either hand-made, synthetic or masked. No test reads or writes unmasked production PII.

### 14.1 Environments

| Environment | Used for | Data | Who runs tests |
| --- | --- | --- | --- |
| Development | Unit tests, developer checks | Small hand-made samples | Developers |
| Test | Functional, transformation, failure and wiring tests | Hand-made and synthetic, in a per-build schema | Jenkins |
| Performance schema in test or pre-production | NFR tests | Production-sized, masked or synthetic | Jenkins, by parameter |
| Pre-production | UAT, release rehearsal, full regression | Masked copy of production, only once approved under section 14.4 | QA and business users |
| Production | Monitoring only: DQX, reconciliation, freshness checks | Live data | Scheduled jobs |

The exact environment names and catalogs are confirmed in Epic 1.

### 14.2 Types of test data

| Type | Used for | How it is made |
| --- | --- | --- |
| Hand-made cases | Functional and transformation tests | Written in the test or the mapping case file; a handful of rows with known expected results |
| Synthetic volume data | NFR and scalability tests | Generated by script to match production row counts, key distribution and skew |
| Masked production samples | UAT and exploratory testing | Copied with PII columns masked or tokenised before leaving production |
| Broken data | Failure tests | Hand-made files with deliberate faults: wrong schema, bad rows, truncation |

### 14.3 Rules

- **Isolation:** every Jenkins build uses its own schemas and paths and removes them at the end, even when it fails (section 11.2).
- **Determinism:** a test creates the data it needs. It never depends on data left by another test or another run.
- **PII:** test data contains no real personal data. Masked copies are approved by the data owner before use.
- **Version control:** hand-made data and mapping cases live in the test repository and are reviewed like code.
- **Reference data:** lookups such as country codes are loaded from a fixed snapshot, so a production change does not break tests unexpectedly.
- **Time:** tests that depend on today's date, such as a 12-month window, set the reference date explicitly.

### 14.4 Privacy

Masked or tokenised production data is still personal data under GDPR, so its use in testing needs the same care as production.

- **Approval:** using any copy of production data outside production needs sign-off from the data protection officer as well as the data owner. Until then, only synthetic data is used, including for NFR volume tests.
- **Masking standard:** join keys are tokenised deterministically so joins still work. Names, emails, phone numbers and addresses are replaced. Free-text fields are dropped. Combinations of fields that could identify a person, such as birth date with postcode, are generalised.
- **Access:** masked copies sit in a catalog that only QA, the test service principals and named UAT users can read.
- **Retention:** per-build schemas live only for the build. Performance and pre-production copies are refreshed or deleted at least every 90 days.
- **No data in reports:** Jira defects, Allure attachments, Xray executions and the `message` column of the unified results view never contain row values from non-synthetic data. They reference the test case and the run ID instead.
- **Erasure:** the erasure scenario in section 6.2 proves that a deletion request reaches every gold table and drops out of Delta history once retention has passed.

## 15. Entry and exit criteria

A release goes to production only when every exit criterion below is met, or a named approver has signed a waiver for the one that is not.

### 15.1 Criteria by stage

| Stage | Entry: testing can start when… | Exit: the stage is complete when… |
| --- | --- | --- |
| Unit | Code is committed | The developer build for this commit passed. Any coverage threshold is set by developers |
| Functional and transformation | Code is deployed to test; unit gate is green; test data is ready | All priority 1 tests pass; every other non-passed test has a linked S3 or S4 defect or an unexpired waiver; no open S1 or S2 defects |
| Data quality and reconciliation | Pipeline has run in test | All critical DQX checks pass; zero unexplained reconciliation breaks |
| Failure | Functional stage is green | All tier 1 failure scenarios pass or have an accepted defect with a workaround |
| NFR | Release candidate is frozen; performance data is loaded | Every NFR target is met, or the miss is accepted in writing |
| UAT | All stages above are green in pre-production | Business sign-off is recorded |

### 15.2 Release gate

The release gate is one query on the unified results view for the release candidate's commit, joined through the run correlation table (section 11.4). The release is go only when all of these hold:

1. Every stage in 15.1 has results for this commit.
2. Every priority 1 test passed. Quarantined tests do not count as passed.
3. Every other result that did not pass has a linked S3 or S4 defect, or an unexpired waiver.
4. No open S1 or S2 defect is linked to the release.
5. UAT sign-off is recorded in Xray.

The overall pass rate is reported as a trend (section 3), not used as the gate, so no failure is accepted just because it falls inside a percentage.

The gate result is shown on the dashboard as go or no-go, with the failing items listed.

### 15.3 Waivers

- A waiver names the failing check, the risk, the workaround and an expiry date.
- Waivers for priority 1 tests or S1 defects are approved by the product owner and the QA lead together.
- An expired waiver fails the gate again.

### 15.4 Suspension and resumption

Testing is suspended when the environment is unavailable, the build cannot be deployed, or more than 30% of tests fail for the same cause. It resumes when the cause is fixed and the smoke suite passes.

## 16. Defect management and reporting

Every failed check ends in one of three outcomes within one working day: a product defect, a test fix or an environment ticket. Nothing stays red without an owner.

### 16.1 Test priority and defect severity

Two separate scales are used, so a test's importance is never confused with a defect's impact.

**Test priority** is set when a test is written, from the tier of the table it protects:

| Priority | Applies to |
| --- | --- |
| 1 | Smoke tests, and tests of security, privacy or regulatory rules, on any tier |
| 2 | Other tests on tier 1 tables |
| 3 | Tests on tier 2 and 3 tables |

**Defect severity** is set at triage:

| Severity | Definition | Example | Target to fix |
| --- | --- | --- | --- |
| S1 | Wrong or missing data in a tier 1 table, a security or privacy breach, or a pipeline that cannot run | Duplicated customers in gold; analyst can read raw PII | Before release; blocks the gate |
| S2 | Wrong data in a tier 2 table, or a tier 1 issue with a workaround | Aggregation wrong for one edge case | Before release unless waived |
| S3 | Minor issue with no effect on consumers | Misleading log message | Within two releases |
| S4 | Cosmetic or improvement | Naming inconsistency | Backlog |

### 16.2 Triage flow

1. Jenkins marks the build failed and posts to the team channel.
2. The QA engineer on rotation classifies each failure: product, test or environment.
3. A product defect is logged in Jira, linked to the failing Xray test execution, with the test name, Xray key, rule ID, expected and actual result, run ID and commit. Row values from non-synthetic data are never copied in (section 14.4).
4. The defect is assigned in the daily stand-up.
5. The fix is verified by rerunning the failing test. That test stays in the regression suite.

### 16.3 Flaky tests

A test that passes on rerun with no code change is flaky. It is quarantined within one day, fixed within one sprint, and counted in the flaky-test measure. Quarantined tests do not count as passed in the release gate.

### 16.4 Reporting tools

Three tools carry reporting, and each answers a different question. ReportPortal, which was trialled in a proof of concept, is not adopted for now (Appendix B).

| Tool | Question it answers | Fed by | Audience | Status |
| --- | --- | --- | --- | --- |
| Allure | Why did this test fail in this run? | pytest, published by Jenkins on every build | QA, developers | Exists |
| Xray in Jira | Is this story proven? Is the release signed off? | pytest results imported by Jenkins after each run; manual executions entered directly | Product team, QA lead | Exists, use to be expanded |
| Databricks dashboard | Is the data healthy? What are the trends across all frameworks? | Unified results view over the DQX and reconciliation log tables and the QA results table | Everyone, especially data owners | New |

**How they connect**

- The Jenkins publish stage does four things on every run: JUnit results to Jenkins, the Allure report, pytest results to Delta, and a test execution in Xray against the release.
- Each automated test carries its Xray test key as a pytest marker. A conftest.py hook writes the key into the JUnit report with record\_property("test\_key", ...), which needs junit\_family = xunit1 in pytest.ini. A test with no key is not imported, so Xray never creates stray Test issues.
- DQX and reconciliation results are not pushed to Xray rule by rule. Each pipeline has one gate test in Xray that passes or fails on the run's results and links to the dashboard for detail.
- Defects are raised in Jira and linked to the failing Xray execution, with a link to the Allure report for that run.
- Manual tests, exploratory sessions and UAT are planned and executed in Xray.

### 16.5 Dashboard

One Databricks dashboard reads the unified results view. It is the only place where pytest, DQX and reconciliation results are seen together.

| View | Shows | Audience |
| --- | --- | --- |
| Release readiness | Go or no-go for the current candidate, with failing items | Product team |
| Latest run | Pass, fail and skipped by test type and framework | QA, developers |
| Trend | Pass rate, duration and flaky rate over the last 30 runs | QA lead |
| Coverage | Coverage matrix by table and test type, with gaps | QA lead, reviewers |
| Traceability | Mapping rules with and without passing checks. Story-level traceability is in Xray | Product owner |
| NFR trend | Run duration, freshness and DBU cost by pipeline | Platform team |

### 16.6 Reporting rhythm

- **Every build:** Jenkins notification with links to the Allure report and the latest-run view.
- **Weekly:** a short quality summary to the team: pass rate, new defects, coverage change.
- **Each release:** the Xray test execution report for the release, with the dashboard's release readiness view, attached to the sign-off.

## 17. Roles and responsibilities

Quality is shared, but each activity has one accountable role. R = does the work, A = accountable, C = consulted, I = informed.

| Activity | Developers | QA | DQX and reconciliation owners | Platform team | Product owner / LoB |
| --- | --- | --- | --- | --- | --- |
| Acceptance criteria and mapping rules | C | C | I | I | A, R |
| Unit tests | A, R | I | I | I | I |
| Data quality rules | C | C | A, R | I | C |
| Reconciliation rules | C | C | A, R | I | C |
| Functional, transformation, failure and NFR tests | C | A, R | I | C | I |
| Coverage matrix | C | A, R | C | I | I |
| Jenkins pipeline and test stages | C | A, R | I | R | I |
| Service principals, catalogs, environments | I | C | I | A, R | I |
| Unified results view and dashboard | I | A, R | C | C | I |
| Defect triage | R | A, R | R | C | I |
| NFR targets and SLAs | C | R | I | C | A |
| UAT | I | R (support) | I | I | A, R |
| Release sign-off | C | R | I | I | A |
| Regression suite upkeep | C | A, R | I | I | I |

### 17.1 Role summaries

- **QA Test Automation Lead:** owns this framework, the coverage matrix, the QA test suite, the unified results view and the release evidence.
- **QA engineers:** design and automate tests, triage failures, support UAT.
- **Developers:** write unit tests with their code, fix defects, review QA tests that touch their pipelines.
- **DQX and reconciliation owners:** maintain their frameworks, add rules on request, keep their result log tables stable, warn QA before changing their columns, and pass the build parameters through to their tasks.
- **Platform team:** provide Jenkins agents, service principals, test catalogs and compute.
- **Product owner and line-of-business representatives:** write and accept requirements, agree NFR targets, sign off UAT and releases.

## 18. Generic CDP layer and line-of-business context

The framework has two layers: a generic layer that applies to every CDP pipeline, and a line-of-business (LoB) layer that adds rules specific to one business area. A new LoB reuses the generic layer and only writes what is different.

### 18.1 What sits in each layer

|  | Generic CDP layer | LoB context layer |
| --- | --- | --- |
| Purpose | The same baseline of trust everywhere | Business rules that only make sense for one LoB |
| Functional | New, update, delete, rerun, empty input | LoB-specific processes, such as a loyalty tier upgrade |
| Transformation | Standard cleansing, dates, deduplication, lookups | LoB mapping rules and calculations |
| Failure | The full scenario catalogue in section 8 | Faults specific to an LoB's sources, such as a partner feed arriving late |
| NFR | Platform-wide targets | LoB-specific SLAs, such as a tighter freshness window |
| Data quality | Schema, nulls on keys, uniqueness, freshness | Value rules, such as loyalty points never being negative |
| Owner of the rules | QA lead and platform owners | LoB representative with QA |

### 18.2 How it works in practice

- **Generic tests are parametrised.** They take a pipeline name, table name, key columns and history type from configuration, so the same test runs against any pipeline.
- **Each LoB has a configuration file** in `config/` listing its pipelines, tables, tiers, keys and SLAs.
- **LoB tests live in their own folder** and carry an LoB marker, so Jenkins can run one LoB or all of them.
- **Results carry the LoB**, so the dashboard can be filtered by line of business.

An example configuration file is in Appendix A.6.

### 18.3 Onboarding a new LoB

1. List the LoB's pipelines and tables and assign tiers.
2. Add the configuration file. Generic tests now run against the new pipelines with no new code.
3. Add the LoB's rows to the coverage matrix.
4. Write mapping cases and LoB-specific tests from its requirements.
5. Request LoB-specific DQX and reconciliation rules.
6. Agree LoB NFR targets and add them to the configuration.

## 19. Risks, assumptions and open questions

The biggest risk to this plan is dependency on teams outside QA: access from the platform team and results from the DQX and reconciliation owners.

### 19.1 Risks

| Risk | Effect | Mitigation |
| --- | --- | --- |
| Service principal or test catalog is delayed | Jenkins cannot run any test | Raise the access request in week one; pilot from a personal workspace schema meanwhile |
| DQX or reconciliation results are hard to join into the unified results view | No single view; gate stays partly manual | Agree the log table columns with the owners early; link runs to builds through the run correlation table. A nightly copy is a temporary fallback only |
| Mapping documents are incomplete or out of date | Transformation tests have no reliable expected results | Start with tier 1 tables; treat missing rules as requirement defects |
| Test runs are slow or expensive | Feedback time target is missed; compute cost rises | One job run per table for many cases; small data on merge; NFR only before release |
| Flaky tests erode trust in the suite | Failures are ignored | Quarantine policy in section 16.3; per-build schema isolation |
| Rules duplicated across pytest, DQX and reconciliation | Conflicting results, double maintenance | Decision guide in section 5.1; coverage matrix review each release |
| Unit test coverage is lower than assumed | Defects reach QA stages late | Baseline it in Epic 1; add a unit gate in Jenkins |
| Team capacity | Automation target slips | Prioritised backlog; pilot proves effort per pipeline before committing dates |
| Pipelines do not meet the testability requirements in section 4.9 | Builds share tables; tests become flaky and cannot run in parallel | Agree the requirements in Epic 2; make them a design exit criterion; until a pipeline meets them, its tests run one build at a time in a shared test schema |
| Approval to use masked production data is slow or refused | UAT and NFR tests run on less realistic data | Start the approval in Epic 1; build synthetic data generators that match production distributions |

### 19.2 Assumptions

- CDP uses a bronze, silver and gold layout in Unity Catalog. Table and schema names in this document are illustrative.
- Pipelines run as Databricks jobs that Jenkins can deploy and trigger. Jobs can take parameters, confirmed by the team.
- CDP does not use Lakeflow Declarative Pipelines today. If it adopts them, sections 4.9, 8 and 11.2 are revisited: isolation moves to the pipeline's target schema, and Lakeflow expectations and change-data-capture handling overlap with DQX and failure testing.
- The DQX and reconciliation frameworks run as tasks in those jobs and write results to their own Delta log tables.
- Developers write unit tests with pytest, and their build reports a pass or fail per commit.
- A test workspace or catalog separate from production is available.

### 19.3 Open questions

- [ ] Which tables and pipelines are tier 1, and which line of business does the pilot pipeline belong to?
- [ ] Who owns the DQX framework and the reconciliation framework, and what columns do their log tables have?
- [ ] What are the real SLAs for run time, freshness and cost?
- [ ] How are pipelines deployed today: Asset Bundles or another method?
- [ ] Does the new QA suite extend the existing pytest framework (fixtures, Allure set-up) or replace it?
- [ ] Which Xray edition is used, Cloud or Server/Data Center? Its import endpoint and authentication differ.
- [ ] What schema-evolution and bad-record policy should each tier follow?
- [ ] Who in data protection approves the use of masked production data, and on what basis?
- [ ] Which notification channel does the release gate post to?
- [ ] Do any CDP sources run as continuous streams rather than triggered runs?
- [ ] What is the current unit test coverage?
- [ ] Which Databricks Apps does CDP own, who uses them, and which tables does each read?
- [ ] Do the apps use the app's own service principal or on-behalf-of-user access to data?
- [ ] How can Playwright sign in to a Databricks App unattended, and can each build have its own app copy within the workspace's limits? (Story 2.9)

## 20. Backlog: epics, stories and tasks

The work is split into eight epics, listed in the proposed order. The Product team is asked to confirm or change this order. Epics 1 to 3 are the foundation and the pilot; the rest can be reordered once the pilot has shown what each kind of test costs.

| # | Epic | Outcome | Depends on | Size | Proposed priority |
| --- | --- | --- | --- | --- | --- |
| 1 | Inventory, coverage matrix and baseline | We know what exists, what is covered and where we start from | None | M | Must, first |
| 2 | Test foundation | Any CDP pipeline can be tested in isolation from Jenkins | Platform team, developers | L | Must, first |
| 3 | Pilot on one tier 1 pipeline | Every QA test type proven on one pipeline, with its real cost known | 1, 2 | L | Must |
| 4 | Unified results view and reporting | One view, one dashboard and Xray evidence across every framework | 2, DQX and reconciliation owners | M | Must |
| 5 | NFR targets and test suite | Agreed SLAs, tested before every release and monitored after | 2, product owner | M | Should |
| 6 | Rollout | Coverage target met for tier 1, then tier 2 | 3 | L | Should |
| 7 | Release gate and spec-driven process | Automated go or no-go; definition of ready and done in use | 4 | M | Should |
| 8 | Further LoB onboarding | Every LoB in scope runs on the framework through configuration | 6 | M | Could |

**Sizes:** S is up to 3 days, M is up to one sprint, and L is more than one sprint and must be split before it starts. Each story lists its acceptance criteria (AC) and main tasks. Stories are written for the illustrative pilot; names and numbers change once Epic 1 is done.

### Epic 1: Inventory, coverage matrix and baseline

- **Story 1.1 (S).** As QA lead, I want every CDP pipeline and table listed with a tier, so that testing effort goes to the highest risk first.
  - AC: every job and table is listed with its layer, owner, consumers and tier.
  - AC: the product owner has agreed the tier 1 list, and the pilot pipeline is chosen from it.
  - Tasks: export jobs from the Jobs API and tables from Unity Catalog; workshop tiers with the product owner.
- **Story 1.2 (M).** As QA lead, I want a coverage matrix for tier 1 and 2 tables, so that gaps and overlaps between frameworks are visible.
  - AC: each cell names unit tests, DQX, reconciliation, QA or "gap" (section 5.2).
  - AC: duplicated checks between frameworks are listed with a proposed owner.
  - Tasks: collect DQX rules, reconciliation rules and unit tests per table; fill the matrix; review it with the framework owners.
- **Story 1.3 (S).** As QA lead, I want a baseline for each measure in section 3, so that progress can be shown from the first release.
  - AC: each of the nine measures has a current value, or a note on why it cannot be measured yet.
  - AC: the Product team has agreed the proposed targets, or changed them.
  - Tasks: count incidents and production defects for the last six months; record current pass rates, unit coverage and run times.
- **Story 1.4 (S).** As QA lead, I want the DQX and reconciliation log tables documented, so that the unified results view can be designed.
  - AC: the columns, keys and retention of both log tables are written down and confirmed by their owners.
  - AC: the fields needed in section 11.4 are mapped, or listed as missing.
- **Story 1.5 (S).** As QA lead, I want data protection approval, or a decision to use synthetic data only, so that UAT and NFR test data is lawful (section 14.4).
  - AC: the data protection officer has approved or refused the masking standard, in writing.
  - AC: section 14 is updated to match the decision.

### Epic 2: Test foundation

- **Story 2.1 (S).** As a QA engineer, I want a Jenkins service principal, a test catalog and role-based test principals, so that tests can run unattended and security tests can act as real roles.
  - AC: Jenkins authenticates to Databricks with OAuth and can create and drop schemas in the test catalog only.
  - AC: one test principal exists for each consumer role, in the same Unity Catalog groups as real users (section 9.2).
  - Tasks: raise the access request; store credentials in Jenkins; prove access with a one-line connection test.
- **Story 2.2 (M).** As a QA engineer, I want a test repository with shared fixtures, so that every test is written the same way.
  - AC: the layout in Appendix A.4 exists, with `tables`, `landing` and `jobs` fixtures that resolve to the build's own schemas and paths.
  - AC: markers are registered in `pytest.ini`, and an unregistered marker fails the run.
  - Tasks: create the repository; build the fixtures; add the Xray key hook (section 16.4).
- **Story 2.3 (S).** As a QA engineer, I want a Jenkins pipeline that deploys to test and runs one smoke test, so that the end-to-end path is proven before real tests are written.
  - AC: a merge triggers deploy, one smoke test, publish and cleanup, and the build shows the JUnit result.
  - AC: a deliberately failing test marks the build unstable and cleanup still runs.
- **Story 2.4 (M).** As QA lead, I want the testability requirements in section 4.9 agreed with developers and added to the design exit criteria, so that every pipeline can be tested in isolation.
  - AC: developers have signed off the requirements, and the design template includes them.
  - AC: the pilot pipeline accepts every parameter in section 4.9, and a run with test parameters writes only to the test schemas.
  - Tasks: walk developers through section 4.9; add the parameters to the pilot pipeline; review one other pipeline for effort.
- **Story 2.5 (M).** As a QA engineer, I want build set-up and cleanup tooling, so that every build has its own schemas, paths and jobs and leaves nothing behind (section 11.2).
  - AC: two builds started at the same time both pass without touching each other's data or jobs.
  - AC: after a build is aborted halfway, the nightly sweep removes everything it left within 48 hours.
  - Tasks: write `create_build_env.py` and `cleanup_build_env.py`; set the bundle prefix; schedule the sweep job.
- **Story 2.6 (S).** As QA lead, I want the line-of-business configuration format from section 18 in place, so that the pilot is built the way every LoB will be.
  - AC: the pilot pipeline is described only in its LoB configuration file, with no pipeline names hard-coded in tests.
  - AC: `--pipelines` and LoB filters select tests from the configuration.
- **Story 2.7 (S).** As QA lead, I want a schema-evolution and bad-record policy for each tier, so that failure tests have agreed expected results (section 8.2).
  - AC: for each tier, the policy states the Auto Loader schema-evolution mode, the parse mode and where rejected rows go.
  - AC: section 8.2's open rows are updated to match.
- **Story 2.8 (S).** As QA lead, I want a decision on reusing the existing pytest and Allure framework, so that the new suite builds on it instead of duplicating it.
  - AC: a short comparison records what is reused, what is migrated and what is retired.
  - AC: section 5 and Story 4.6 are updated to match.

* **Story 2.9 (S).** As a QA engineer, I want a spike on testing a Databricks App unattended, so that the app test approach in section 6.5 is proven before we build on it.
  - AC: from Jenkins, an API test calls a deployed app with a test service principal's OAuth token and gets a 200 response.
  - AC: a Playwright test signs in and completes one journey without a human, or the spike records why it cannot and proposes an alternative.
  - AC: the spike records whether one app copy per build fits the workspace's app limits and start-up time.
  - Tasks: deploy the app with a bundle prefix; try service principal and user sign-in for Playwright; write up the result in section 6.5.

### Epic 3: Pilot on one tier 1 pipeline

- **Story 3.1 (M).** As a QA engineer, I want functional tests for the pilot pipeline, so that its end-to-end behaviour is proven against its requirements.
  - AC: each scenario in section 6.2 that applies to the pilot has an automated test linked to an Xray key.
  - AC: the smoke tests for the pilot are marked, and the merge stage runs them within its budget.
  - Tasks: write the scenarios as Xray tests; automate them; record each test's run time.
- **Story 3.2 (M).** As a QA engineer, I want transformation cases for every mapping rule of the pilot table, so that each rule is proven row by row.
  - AC: every rule in the pilot's mapping document has at least one case, and every threshold rule has boundary cases (section 7.3).
  - AC: the pipeline runs once for all cases, and a rule with no case shows as a gap.
- **Story 3.3 (M).** As a QA engineer, I want five failure scenarios on the pilot, so that safe failure and recovery are proven.
  - AC: missing file, renamed column, unparseable rows, crash before gold and duplicate delivery each have a test asserting safe state, detection, alerting and recovery (section 8.1).
  - AC: any scenario the pipeline does not yet handle is raised as a defect, not skipped.
- **Story 3.4 (S).** As a QA engineer, I want wiring tests for DQX and reconciliation on the pilot, so that both frameworks are proven to be connected correctly.
  - AC: the four checks in section 10.3 pass on the pilot.
  - AC: both frameworks' results for a test run carry the build's `git_commit` and `build_tag`.
- **Story 3.5 (S).** As QA lead, I want to demo the pilot to the Product team with its real costs, so that Epics 4 to 8 can be prioritised and sized.
  - AC: the demo shows a merge build, a nightly build and a failed build with its reports.
  - AC: effort per test type and actual stage durations are recorded, and the time budgets in section 11.3 are confirmed or changed.

* **Story 3.6 (M).** As a QA engineer, I want the CDP app that reads the pilot tables covered by API, access and UI tests, so that the pilot proves app testing end to end.
  - AC: the app's main endpoints have API and data correctness tests, and each role principal has an access test.
  - AC: one critical journey runs in Playwright nightly, with traces attached to Allure on failure.
  - AC: app tests write to the unified results view with `test_type` set to app.

### Epic 4: Unified results view and reporting

- **Story 4.1 (M).** As QA lead, I want the QA results table, the run correlation table and the Jenkins publisher, so that every QA result is stored with its commit and build (section 11.4).
  - AC: every Jenkins build writes one row per test and one correlation row per Databricks run it started.
  - AC: a build that fails at Deploy still writes its build row.
- **Story 4.2 (S).** As QA lead, I want the DQX log table mapped into the unified results view, so that data quality results sit beside every other result.
  - AC: DQX results for a build appear in the view with its commit, tier and priority, without changing the DQX log table.
- **Story 4.3 (S).** As QA lead, I want the reconciliation log table mapped into the unified results view, so that reconciliation breaks are reported in the same place.
  - AC: reconciliation results for a build appear in the view with the difference reported, without changing the reconciliation log table.
- **Story 4.4 (S).** As QA lead, I want the developer build result recorded for each commit, so that the unit gate can be read rather than rerun.
  - AC: the QA results table holds the developer build status for every commit Jenkins tests.
- **Story 4.5 (M).** As QA lead, I want the dashboard views in section 16.5, so that each audience sees the answer it needs.
  - AC: the latest run and release readiness views are live first; the others follow in priority order.
  - AC: every view reads only from the unified results view.
- **Story 4.6 (S).** As a developer, I want the Allure report published from Jenkins on every build, so that I can see why a test failed.
  - AC: every build links to its Allure report, including failed builds. Scope follows the decision in Story 2.8.
- **Story 4.7 (M).** As QA lead, I want pytest results imported into Xray from Jenkins, so that requirement coverage and release evidence live in Jira.
  - AC: each build creates one Xray test execution against the release, and every result lands on its existing Xray test.
  - AC: tests without an Xray key are reported and not imported.
- **Story 4.8 (S).** As QA lead, I want one Xray gate test per pipeline for DQX and reconciliation, so that their outcome counts in Xray without importing every rule.
  - AC: the gate test passes only when the pipeline has no failing critical DQX check and no unexplained reconciliation break, and it links to the dashboard.
- **Story 4.9 (S).** As QA lead, I want release-gating manual results exported from Xray, so that the release gate sees manual and automated evidence together.
  - AC: a manual execution marked as release-gating appears in the unified view within one hour.

### Epic 5: NFR targets and test suite

- **Story 5.1 (S).** As QA lead, I want agreed NFR targets for each tier 1 pipeline, so that "fast enough" and "fresh enough" are testable.
  - AC: the product owner has agreed run time, freshness, recovery and cost targets, replacing the proposed values in section 9.
- **Story 5.2 (M).** As a QA engineer, I want a production-sized data set in a performance schema, so that NFR results reflect real volumes.
  - AC: the data matches production row counts, key distribution and skew within 10%.
  - AC: it is synthetic, or masked under the approval from Story 1.5.
- **Story 5.3 (M).** As a QA engineer, I want performance, scalability and cost checks automated, so that every release candidate is measured the same way.
  - AC: run duration is read from the Jobs API during the build; cost is checked from `system.billing.usage` the next day.
  - AC: a run more than 20% slower than the last three is flagged, even when it passes.
- **Story 5.4 (S).** As a QA engineer, I want access and masking checks automated with the role-based test principals, so that security rules are proven before every release.
  - AC: for each role, a test proves what it can read, what it cannot, and how masked columns appear.
- **Story 5.5 (S).** As QA lead, I want a timed Delta restore rehearsal, so that recovery is proven, not assumed.
  - AC: a gold table is restored to an earlier version inside the target time, and the steps are documented.
- **Story 5.6 (S).** As QA lead, I want the production monitoring checks in section 9.3 handed to the platform team, so that freshness, reliability and cost are watched after release.
  - AC: each check runs on a schedule in production, with an owner and an alert route.

* **Story 5.7 (S).** As QA lead, I want a load test for each CDP app, so that apps stay responsive before release.
  - AC: a Locust test runs against the release candidate's app copy and checks the app performance target in section 9.

### Epic 6: Rollout

- **Story 6.1 (M).** As a QA engineer, I want generic functional and failure tests driven by configuration, so that a new pipeline gets baseline coverage without new code.
  - AC: adding a pipeline to its LoB configuration file runs the generic scenarios against it.
- **Story 6.2 (M, one per pipeline).** As QA lead, I want each remaining tier 1 pipeline brought to the pilot's standard, so that the coverage target in section 3 is met.
  - AC: the pipeline meets section 4.9, has mapping cases and specific tests, and its coverage matrix row has no gaps.
- **Story 6.3 (L, split by pipeline).** As QA lead, I want tier 2 pipelines covered for functional and transformation tests, so that internal analytics is protected too.
  - AC: every tier 2 pipeline meets the minimum coverage in section 4.5.
- **Story 6.4 (S).** As QA lead, I want the nightly full regression scheduled and stable, so that the whole product is checked every day.
  - AC: the nightly run finishes inside 90 minutes, and its flaky-test rate is under 2% for two weeks running.

* **Story 6.5 (M, one per app).** As QA lead, I want every CDP Databricks App covered to the pilot's standard, so that app regressions are caught before release.
  - AC: each app meets the app rows in section 4.9 and has API, data correctness, access and critical-journey tests.
  - AC: the coverage matrix has a row per app with no gaps.

### Epic 7: Release gate and spec-driven process

- **Story 7.1 (M).** As the Product team, we want an automated go or no-go for each release candidate, so that release decisions rest on the same evidence every time.
  - AC: the gate query in section 15.2 runs on the release candidate's commit and shows go or no-go with the failing items.
  - AC: a deliberately failing priority 1 test turns the gate to no-go.
- **Story 7.2 (S).** As QA lead, I want the definition of ready and done in section 13 adopted by the team, so that tests are planned with stories, not after them.
  - AC: both definitions are in the team's Jira workflow, and the next sprint's stories meet the definition of ready.
- **Story 7.3 (S).** As the Product team, we want requirement coverage reported in Xray, so that we can see which acceptance criteria are proven.
  - AC: every story in the current release shows its linked tests and their latest result in Xray.
- **Story 7.4 (S).** As QA lead, I want the waiver process and flaky-test quarantine in place, so that exceptions are visible and expire.
  - AC: a waiver needs an owner, risk, workaround and expiry date, and an expired waiver fails the gate.
  - AC: a quarantined test shows as quarantined, not passed, in the view and the gate.
- **Story 7.5 (S).** As QA lead, I want to trial AI-assisted test case drafting from acceptance criteria, so that test design is faster without lowering quality.
  - AC: for one sprint, drafted cases are compared with hand-written ones for time taken and defects found, and the result decides whether to continue.

### Epic 8: Further LoB onboarding

- **Story 8.1 (S).** As QA lead, I want the configuration from Story 2.6 extended to every LoB in scope, so that each one runs on the framework.
  - AC: each LoB has a configuration file, and the dashboard can filter by it.
- **Story 8.2 (M).** As QA lead, I want the second LoB onboarded using the checklist in section 18.3, so that onboarding by configuration is proven.
  - AC: the generic tests run against the new LoB's pipelines with no new code, and its specific rules are tested.
- **Story 8.3 (S).** As the Product team, we want release readiness by LoB, so that one LoB's issue does not hide or block another's status.
  - AC: the release readiness view shows go or no-go for each LoB.

### Suggested first increment

Epics 1 and 2 run in parallel and need no prioritisation decision. Stories 2.4 and 2.7 depend on developers, so they should start in the first week. Epic 3 follows as soon as Stories 2.1 to 2.6 are done, and its demo is the best input for prioritising Epics 4 to 8.

## Appendix A. Code samples

These samples are illustrative. Names such as `tables`, `landing` and `jobs` stand for the shared fixtures in the test repository.

### A.1 Functional test (section 6)

```python
def test_address_change_keeps_history(landing, run_job, tables):
    landing.put("customers", [{"customer_id": "C100", "city": "Dublin"}])
    run_job("cdp_customer_pipeline")
    landing.put("customers", [{"customer_id": "C100", "city": "Cork"}])
    run_job("cdp_customer_pipeline")

    # tables.gold() resolves to this build's own gold schema
    rows = tables.gold("customer_profile").filter("customer_id = 'C100'")
    assert rows.count() == 2
    assert rows.filter("is_current").first().city == "Cork"
    assert rows.filter("NOT is_current").first().city == "Dublin"
```

### A.2 Transformation test (section 7)

```python
CASES = load_cases("mappings/customer_profile.yaml")  # one unique key per case

@pytest.fixture(scope="session")
def output_by_key(tables, run_pipeline_once):
    # one job run and one query for every case of this table
    rows = tables.gold("customer_profile").collect()
    return {r.customer_id: r for r in rows}

@pytest.mark.parametrize("case", CASES, ids=lambda c: c.rule_id)
def test_mapping_rule(case, output_by_key):
    row = output_by_key.get(case.key)
    assert row is not None, f"{case.key} missing from output"
    expected = coerce(case.expected, to=case.target_type)  # e.g. Decimal, date
    assert row[case.target_column] == expected
```

Cases may hold several input rows for one key, which aggregation and deduplication rules need.

### A.3 Failure test (section 8)

```python
def test_crash_before_gold_is_recoverable(landing, jobs, tables):
    landing.put("customers", sample_customers(1000))
    version_before = tables.latest_version("gold", "customer_profile")

    # deterministic fault: a no-op task fails just before the gold task
    run = jobs.run_and_wait("cdp_customer_pipeline", fail_before="gold_customers")
    assert run.state == "FAILED"
    assert tables.latest_version("gold", "customer_profile") == version_before

    jobs.run_and_wait("cdp_customer_pipeline")
    gold = tables.gold("customer_profile")
    assert gold.count() == 1000
    assert gold.select("customer_id").distinct().count() == 1000
```

### A.4 Test repository layout (section 11)

```text
cdp-qa-tests/
  conftest.py            # shared fixtures: tables, landing, jobs, build schemas, Xray key hook
  pytest.ini             # registered markers, junit_family
  config/                # one file per environment and per line of business
  mappings/              # transformation cases, one file per target table
  tests/
    functional/
    transformation/
    failure/
    nfr/
    wiring/              # DQX and reconciliation wiring tests
  tools/
    create_build_env.py  # creates schemas and paths for this build
    publish_results.py   # writes QA results and the run correlation rows to Delta
    cleanup_build_env.py # destroys jobs, drops schemas, removes paths
  Jenkinsfile
```

### A.5 Jenkinsfile outline (section 11)

```groovy
pipeline {
  agent { label 'databricks-qa' }
  triggers { cron('H 1 * * *') }
  parameters {
    booleanParam(name: 'RELEASE_CANDIDATE', defaultValue: false)
    booleanParam(name: 'RUN_NFR', defaultValue: false)
  }
  environment {
    DATABRICKS_HOST          = credentials('dbx-host')
    DATABRICKS_CLIENT_ID     = credentials('dbx-sp-id')
    DATABRICKS_CLIENT_SECRET = credentials('dbx-sp-secret')
    BUILD_ID_SAFE            = "${env.BUILD_TAG}".replaceAll('[^A-Za-z0-9_]', '_')
  }
  stages {
    stage('Deploy') {
      steps {
        sh 'python tools/create_build_env.py --build $BUILD_ID_SAFE'
        sh 'databricks bundle deploy -t test --var="prefix=$BUILD_ID_SAFE"'
      }
    }
    stage('Unit gate') {
      when { not { triggeredBy 'TimerTrigger' } }
      steps { sh 'python tools/check_dev_build.py --commit $GIT_COMMIT' }
    }
    stage('Merge tests') {
      when { not { anyOf { triggeredBy 'TimerTrigger'; expression { params.RELEASE_CANDIDATE } } } }
      steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
          sh 'pytest --pipelines=$(tools/changed_pipelines.py) --levels=smoke,targeted --with-progression -n auto --junitxml=reports/merge.xml'
        }
      }
    }
    stage('Full regression and failure') {
      when { anyOf { triggeredBy 'TimerTrigger'; expression { params.RELEASE_CANDIDATE } } }
      steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
          sh 'pytest --levels=full --with-progression -n auto --junitxml=reports/full.xml'
        }
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
          sh 'pytest tests/failure --junitxml=reports/failure.xml'
        }
      }
    }
    stage('NFR') {
      when { expression { params.RUN_NFR } }
      steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
          sh 'pytest tests/nfr --junitxml=reports/nfr.xml'
        }
      }
    }
    stage('Data quality and reconciliation gate') {
      steps { sh 'python tools/check_dq_gate.py --build $BUILD_ID_SAFE' }
    }
  }
  post {
    always {
      junit testResults: 'reports/*.xml', allowEmptyResults: true
      allure results: [[path: 'allure-results']]
      sh 'python tools/publish_results.py --build $BUILD_ID_SAFE --commit $GIT_COMMIT || true'
      sh 'python tools/import_xray.py reports/*.xml || true'
    }
    cleanup {
      sh 'python tools/cleanup_build_env.py --build $BUILD_ID_SAFE || true'
    }
  }
}
```

Three details matter. `catchError` lets later stages run after a test failure, so one failing functional test does not hide the failure tests. Reporting steps tolerate a missing report, because a build that fails at Deploy produces none. Cleanup sits in `post { cleanup }`, which runs last whatever happened before it.

### A.6 Line-of-business configuration (section 18)

```yaml
# config/lob_retail.yaml (illustrative)
line_of_business: retail
pipelines:
  - name: cdp_customer_pipeline
    target: gold.customer_profile
    tier: 1
    keys: [customer_id]
    history: scd2
    freshness_hours: 24
    max_runtime_minutes: 30
```

### A.7 Databricks App tests (section 6.5)

```python
# API test: numbers from the app match gold, called as a test principal
def test_customer_summary_matches_gold(app_client, tables):
    resp = app_client.get("/api/customers/summary", params={"country": "IE"})
    assert resp.status_code == 200
    body = resp.json()
    expected = (tables.gold("customer_profile")
                .filter("country = 'IE' AND is_current").count())
    assert body["customer_count"] == expected

def test_summary_refuses_unauthenticated_call(app_url, httpx_client):
    resp = httpx_client.get(f"{app_url}/api/customers/summary")
    assert resp.status_code in (401, 302)

# UI test: one critical journey in a real browser
def test_search_customer_journey(app_page):  # app_page: signed-in Playwright page
    app_page.get_by_test_id("customer-search").fill("C100")
    app_page.get_by_test_id("search-submit").click()
    expect(app_page.get_by_test_id("customer-city")).to_have_text("Cork")
```

## Appendix B. Why ReportPortal is not adopted

- Its run reports overlap with Allure, its execution history with Xray, and its trends with the Databricks dashboard.
- It needs its own server to host and maintain, and a fourth place to send results.
- It is revisited only if failure triage across many runs becomes more than Allure and Xray can handle.
