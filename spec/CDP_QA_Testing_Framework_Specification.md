# CDP QA Testing Framework Specification

Nov 12, 2026 · @Siddhanth

## 1. Summary

This spec defines one QA testing framework for CDP, our Databricks data product, so that every release can be shown to be trustworthy before it reaches consumers.

The framework does not replace what exists. It joins the DQX data quality framework, the CDP reconciliation framework and developer unit tests with QA tests built on the existing pytest framework, triggers and gates all of them from Jenkins, and reports them in one place.

- **Status:** draft for review by Kedar, Gagan and Sukesh.
- **Decision needed:** the Product team prioritises the backlog in section 20.
- **Automation split:** 80–85% automated, 15–20% manual or process.
- **New work for QA:** functional, transformation, failure, NFR and app tests; Jenkins orchestration; one unified results view and dashboard.
- **Reused as is:**
  - DQX for data quality;
  - the reconciliation framework;
  - developer unit tests;
  - the existing pytest framework with Allure.

Reviewers are asked to confirm four things:

1. The scope in section 2.
2. The ownership split in sections 5 and 17.
3. The targets marked "proposed" in sections 3 and 9. They are placeholders until the baseline is measured.
4. The order of the epics in section 20.

## 2. Purpose and scope

CDP needs one agreed way of testing, so that quality does not depend on who built a pipeline or which line of business it serves.

**In scope:** ingestion into bronze through Auto Loader; bronze to silver to gold transformations; consumer-facing gold tables and views; CDP's Databricks Apps (FastAPI and React), such as the lineage detection and dashboard apps (section 6.5); the jobs that run them, including schedules, dependencies, retries and alerts; Unity Catalog access control and masking; and deployment to test through Jenkins.

**Out of scope:** the correctness of source systems; downstream applications outside CDP; the internals of DQX and reconciliation (this spec covers how they are used and gated); unit tests, which developers own; and the Databricks platform itself.

**Principles**

1. **Reuse, don't rebuild.** QA never writes a second version of a check that DQX, reconciliation or an existing test already covers.
2. **Test from the spec.** Every test traces to an acceptance criterion or a mapping rule.
3. **One results store.** All results are read through one unified results view, giving one answer to "is this release good?".
4. **Fail safe.** A broken pipeline never leaves partial or misleading data in gold.
5. **Risk first.** The most critical tables get the deepest coverage first.

## 3. Goals and success measures

"Trustworthy" is defined as nine measures. They give the team a release gate that is the same every time, and give the Product team a trend they can read without knowing the tools. Targets are proposed until Epic 1 measures the baseline.

1. **Automation ratio:** 80–85%. Measured as automated Xray tests in scope (tier 1 and 2) ÷ all Xray tests in scope, per release.
2. **Automated coverage of tier 1 tables:** 80% within two quarters.
3. **Regression pass rate:** stable at or above 95%. This is a trend, not a gate (section 15).
4. **Defect leakage:** at most 2 production defects per release, and no S1.
5. **Tier 1 data incidents:** down 50% from the baseline.
6. **Critical DQX checks on gold at release:** 100% passing.
7. **Unexplained reconciliation breaks at release:** 0.
8. **Merge pipeline duration:** 30 minutes or less.
9. **Flaky tests:** under 2% of the suite.

## 4. Test strategy

CDP follows the V-model: each build stage has a matching test stage, designed when that stage is specified. A pipeline's design is complete only when it meets the testability requirements in section 4.9.

### 4.1 V-model levels

- **Requirements ↔ UAT:** the product does what the business asked. Owned by the line of business, supported by QA.
- **Architecture and design ↔ QA system and integration testing.** Owned by QA.
- **Code ↔ unit testing.** Owned by developers.

### 4.2 Progression and regression

- **Progression tests** prove that new or changed functionality works. They are written from the story's acceptance criteria.
- **Regression tests** prove that existing functionality still works.

A test's suite comes from Xray, not from code. A test linked to a story in the current release is progression; every other test is regression. Pytest markers cover only `smoke`, test type and line of business.

**Example:** a release adds a `loyalty_tier` column.

- **Progression, black-box:** load €1,200 of orders and check that gold shows Gold.
- **Progression, white-box:** check the silver value, the €999.99 and €1,000 boundaries, and that the `MERGE` updates rather than inserts.
- **Regression:** address history, consent filtering and rerun safety still work, and run time has not grown.

### 4.3 Black-box and white-box

Black-box tests use only inputs and outputs. White-box tests use knowledge of the code, the job structure or intermediate tables. Progression and regression tests use both.

### 4.4 Test types

QA tests are functional and integration (section 6), transformation and boundary value (section 7), failure (section 8) and NFR (section 9). Data validity belongs to DQX and reconciliation (section 10).

### 4.5 Risk-based tiers

Each table, pipeline and app is given a tier in Epic 1.

- **Tier 1:** feeds external consumers, regulatory reports or key decisions. Every test type is automated and in the release gate.
- **Tier 2:** feeds internal analytics. Functional, transformation, data quality and reconciliation tests.
- **Tier 3:** staging, reference or low-use data. Data quality and reconciliation only.

### 4.6 Regression suite levels

- **Smoke (every merge):** about five key tests per touched tier 1 pipeline or app.
- **Targeted (every merge):** regression tests for the touched pipelines and their downstream dependants. Smoke and targeted share the 18-minute merge test stage.
- **Full (nightly and release candidate):** every regression test, within 90 minutes.

Touched pipelines come from the changed files through the LoB configuration (section 18), and their dependants from production lineage in `system.access.table_lineage`. If the mapping is unclear, the merge runs smoke for all tier 1 pipelines, and the full suite runs that night. The budget is met by running each pipeline once with many assertions, running pipelines in parallel with `pytest-xdist`, and using serverless or pooled compute. All budgets are validated in the Epic 3 pilot.

### 4.7 Lifecycle of a test

A progression test is written from a story's acceptance criteria and linked to its Xray test. Once the story is released, the test counts as regression with no code change; core tier 1 checks are also marked `smoke`. A test changes only when its requirement changes, and is retired when its feature is removed.

### 4.8 Keeping the regression suite healthy

QA owns the suite, and developers fix the regressions their changes cause. A regression failure blocks the merge, and the test is never edited to make it pass unless the requirement changed. Each release, retired and duplicate tests are removed and slow tests leave smoke. Time budgets are limits: an over-budget test is sped up or moved to a slower level, not deleted.

### 4.9 Testability requirements

These requirements are the exit criterion of the design stage, agreed with developers in Epic 2. CDP jobs already take parameters, so they change how parameters are used, not the platform.

1. **Catalog and schema per layer are job parameters**, and every table reference resolves through them, so each build writes to its own schemas (section 11.2).
2. **Landing and Auto Loader checkpoint paths are job parameters**, so each build starts from a clean checkpoint.
3. **`as_of_date` is a job parameter**, defaulting to today, so date-window rules give the same result on every run.
4. **`fail_before` makes a no-op task fail before a named task**, so failure tests stop at an exact point (Appendix A.3).
5. **`git_commit` and `build_tag` reach every task**, including DQX and reconciliation, so every result traces to its build (section 11.4).
6. **Each run logs its run ID, rows in, rows out and status**, for auditability and diagnosis.
7. **Streams can run with `trigger(availableNow=True)`**, so tests can wait for them to finish.
8. **Apps read catalog, schema and warehouse from configuration, publish their OpenAPI schema, and give critical React elements `data-testid` attributes** (section 6.5).

## 5. Tooling and ownership

Each testing need has one tool and one owner, so no rule is written twice and no gap goes unowned.

- **Unit tests for pipelines and apps:** developers, using pytest, FastAPI `TestClient`, and Vitest or Jest. These exist; coverage is to be confirmed.
- **Data quality:** the CDP framework built on DQX, run by its owning team. Exists.
- **Source-to-target matching:** the CDP reconciliation framework, run by its owning team. Exists.
- **Functional, integration, transformation, failure and NFR tests:** QA, using the existing pytest framework, extended.
- **App API, access, UI and load tests:** QA, using pytest with `httpx`, Playwright and Locust. New.
- **Orchestration:** Jenkins, run by QA with the platform team. Exists; the test stages are new.
- **Reporting:** QA, using Allure (exists), Xray in Jira (exists) and a Databricks dashboard (new).

The QA suite **extends the existing pytest framework**: its fixtures, configuration and Allure set-up stay. This spec adds four things:

- build isolation fixtures;
- the Xray key hook;
- results publishing;
- the folders in Appendix A.4.

Story 2.8 lists the gaps.

### 5.1 Which tool does a new check belong in?

Use the first row that matches.

| If the check is about… | It belongs in… | Example |
| --- | --- | --- |
| One function, with no cluster or tables | Developer unit test | `clean_name("  John ")` returns `"John"` |
| Validity of data in a table | DQX | `customer_id` is unique in gold |
| Totals or counts matching between layers | Reconciliation | Bronze row count equals the source extract |
| A business rule's value for a given input | QA transformation test | Cancelled orders are excluded from `total_spend_12m` |
| Behaviour of a pipeline or app as a whole | QA functional test | An address change creates a history row |
| What happens when something breaks | QA failure test | A missing file leaves gold unchanged |
| Speed, freshness, access, recovery or cost | QA NFR test | The daily load finishes in its window |

### 5.2 Coverage matrix

The coverage matrix lists every tier 1 and tier 2 table and app down the side and every test type across the top. Each cell names the framework that covers it, or says "gap". Gaps become QA stories, and the matrix is reviewed at the start of every release.

## 6. Functional and integration testing

Functional tests prove that a pipeline behaves as its requirement says, from source input to gold output, without looking at the code.

### 6.1 Approach

1. The test places controlled input in the build's landing volume or bronze table.
2. It runs the real production job definition, deployed with the build's own parameters (section 4.9).
3. It checks the output.
4. The build drops its schemas.

Tests never call transformation functions directly; that is the unit-test level. There is an example in Appendix A.1.

### 6.2 Standard scenarios

Every tier 1 pipeline is tested against these scenarios where they apply.

- **New record:** C100 appears in `gold.customer_profile` with matching attributes.
- **Update with history:** after a move from Dublin to Cork, the old row is closed and the new row is current.
- **Delete flag:** the record is removed or flagged as the requirement states, never left active.
- **Rerun on the same input:** no new rows and no duplicates.
- **Late-arriving data:** lands in the correct period, and aggregates update.
- **Consent filter:** customers with `consent = N` are absent from the marketing view.
- **Empty file:** the job succeeds, gold is unchanged, and the run is logged.
- **Erasure request:** removed from every gold table and view, and purged from Delta history after retention and VACUUM.

### 6.3 Integration scenarios

Integration tests check that:

- gold jobs start only after silver succeeds;
- joins across pipelines are correct;
- gold tables match their published contract, so a removed or renamed column fails;
- DQX and reconciliation run as tasks and write their results (section 10).

### 6.4 Pass criteria

A functional test passes only when the job succeeds and every assertion holds. Right data with a failed job, or wrong data with a successful job, are both failures.

### 6.5 Databricks Apps (FastAPI and React)

CDP has several UI apps, tested mostly at API level, with only a few critical journeys in a browser. The two known apps set the pattern; Story 1.1 lists the rest.

- **Lineage detection app:**
  - lineage for a known table matches its source of truth (for example `system.access.table_lineage`);
  - users see lineage only for tables they can access;
  - new and dropped tables appear or disappear after refresh.
- **Dashboard app:**
  - each KPI equals the matching SQL on gold for the same filters and dates;
  - filters and empty states work;
  - stale data is never shown as current.

* **Unit (developers):** FastAPI `TestClient`, and Vitest or Jest, in the developer build.
* **API, contract and data correctness (merge):** pytest with `httpx` checks status codes, the OpenAPI schema and bad input, and that API numbers equal a direct query on the build's gold.
* **Access (merge, priority 1):** each role principal sees only its data, and unauthenticated calls are refused.
* **UI journeys (nightly):** three to five critical journeys in Playwright.
* **Failure (nightly):** a stopped warehouse, missing table or backend error shows a clear message, never stale data.
* **Load (release candidate):** Locust against the targets in section 9.

- **Isolation:** each build deploys its own app copy, pointed at its own test data. If workspace app limits make that impractical, one shared test app is used and builds run one at a time.
- **Sign-in:** API tests send a test service principal's OAuth token in the `Authorization` header ([Databricks docs](https://docs.databricks.com/gcp/dev-tools/databricks-apps/connect-local)). Unattended Playwright sign-in is unproven; Story 2.9 is a spike to settle it.
- **Reporting:** results carry Xray keys and appear in Allure and the unified view as test type `app`. Failed UI tests attach a Playwright trace. There are examples in Appendix A.7.

## 7. Transformation and boundary value testing

Transformation tests prove that every rule in the source-to-target mapping document gives the right value for each row, inside the real pipeline. They sit between developer unit tests, which check a function on its own, and reconciliation, which checks that totals match.

### 7.1 Approach

Each mapping rule is written as cases: a rule ID, input values and the expected output. Cases are kept in version-controlled files, one per target table. The test seeds all inputs for a table in one batch and runs the pipeline once. A parametrised test then checks each case, so every rule shows its own pass or fail. There is an example in Appendix A.2.

### 7.2 Rule categories

Cases cover derived fields, lookups (including no match), aggregations, deduplication, date formats, type casts and conditional rules. Worked examples are in Appendix C.1.

### 7.3 Boundary values

Every rule with a threshold, range or length limit gets cases on, just below and just above the boundary, plus null and empty values. For example, spend of 999.99, 1,000.00 and 1,000.01 for a €1,000 rule, or orders exactly 12 months old for a 12-month window. Timestamps also cover midnight, daylight-saving changes, month end and 29 February.

### 7.4 Traceability

Every case carries its mapping rule ID. A rule with no case is a coverage gap and is shown on the dashboard.

## 8. Failure testing

Failure tests prove that when something goes wrong, the pipeline stops safely, alerts someone, recovers cleanly and never leaves partial or misleading data in gold.

### 8.1 What every failure test asserts

1. **Safe state:** gold's latest version in `DESCRIBE HISTORY` is unchanged, and related gold tables are consistent with each other. Delta commits are atomic per table, so the real risk is a run that commits to some tables and not others.
2. **Detection:** the run ends in the expected state, with a clear error.
3. **Alerting:** the agreed alert fires.
4. **Recovery:** a rerun after the fault is removed gives correct data and no duplicates.

### 8.2 Scenario catalogue

The catalogue in Appendix C.5 covers 15 scenarios:

- **Missing or bad input:** missing files, schema drift, unparseable rows, rule-breaking rows, truncated files.
- **Interrupted runs:** crashes before gold, task retries, upstream failures.
- **Repeated or disordered input:** duplicate deliveries, out-of-order files, volume spikes.
- **Access and recovery:** revoked permissions, rollback with Delta `RESTORE`.

Auto Loader behaviour matters in several of these. Its default `addNewColumns` mode turns a renamed column into silent nulls, and its checkpoint de-duplicates by file path only. The schema-evolution and bad-record policy for each tier is still open (Story 2.7). There is an example test in Appendix A.3.

### 8.3 Rules

Failure tests run only in the build's own schemas, nightly and before release, never on every merge. A scenario the pipeline does not yet handle is raised as a defect, not skipped.

## 9. Non-functional (NFR) testing

NFR tests prove how well CDP works: speed, scale, security, recovery, auditability and cost. Pre-release tests gate each release; production monitoring watches it afterwards. All targets are proposed until the Product team agrees them in Epic 5.

### 9.1 Pre-release tests

- **Daily load:** under 30 minutes for about 10M rows, measured by the Jobs API.
- **Gold queries:** standard queries under 5 seconds, from SQL warehouse history.
- **Scalability:** at 3x synthetic volume, the load stays within SLA with no out-of-memory failure.
- **Access:** analysts cannot read raw PII, and ungranted principals are denied (queried as each role principal).
- **Masking:** analysts see email as `j***@***.com`.
- **Recoverability:** a Delta `RESTORE` completes inside 15 minutes.
- **Auditability:** 100% of runs are logged.
- **Cost:** within the agreed DBU budget, checked the next day from `system.billing.usage`.
- **App load:** 95% of API calls under 1 second and pages within 3 seconds at 20 users, measured with Locust.

These tests run on production-sized masked or synthetic data in a performance schema, on production-sized compute, before each release or major change. A run that passes but is 20% slower than the last three is flagged.

### 9.2 Test principals

Security tests connect as dedicated service principals, one per consumer role, in the same Unity Catalog groups as real users. They are requested in Epic 2.

### 9.3 Production monitoring

The platform team runs these checks; QA defines them and reviews the trend each release. They are not part of the release gate.

- **Freshness:** tier 1 gold is updated by 06:00 and is under 24 hours old.
- **Reliability:** at least 99% of scheduled runs succeed over 30 days.
- **Cost:** no unexplained rise above 20% week on week.
- **Production DQX and reconciliation:** no critical failure stays open beyond one working day.

## 10. Data quality and reconciliation integration

DQX checks the validity of data inside a table: nulls, uniqueness, allowed values, ranges, formats, referential integrity and schema. Reconciliation checks agreement between layers: counts, sums and key-level matches. Both stay with their owners. QA makes sure they cover every tier 1 and 2 table, are wired into the release gate, and are reported with everything else.

### 10.1 What QA does

QA uses the coverage matrix to check that every tier 1 and 2 table has rules in both frameworks, raises requests for missing rules from acceptance criteria, proves the wiring works (section 10.3), and brings both frameworks' results into the unified view and the release gate.

### 10.2 What QA does not do

QA does not maintain the rule engines or duplicate their rules in pytest. It doesn't set thresholds alone (those are agreed with data owners), and doesn't change how either framework stores its results.

### 10.3 Wiring tests

- A row breaking a critical DQX rule lands in quarantine, not gold.
- A batch above the error threshold fails the job and alerts.
- A deliberately mismatched load produces a reconciliation break with the right difference.
- Both frameworks write a result for every run, including runs with zero failures.

### 10.4 Requesting a new rule

When an acceptance criterion describes a data rule (for example, `loyalty_points` is never negative), QA classifies it with section 5.1. If it belongs in DQX or reconciliation, QA requests it from the owning team with the table, column, rule and severity. The story is done when the rule is live in test. If the owning team cannot add it within the sprint, the story closes with the request tracked as a dependency.

## 11. Automation and Jenkins execution

Jenkins deploys the code, starts the tests and collects the results. Databricks does the data work: pytest on the Jenkins agent loads test data, starts jobs and queries the output.

### 11.1 What is automated

The 80–85% target is one ratio per release: automated Xray tests in scope for tier 1 and 2, divided by all Xray tests in scope. DQX and reconciliation each count as one automated gate test per pipeline. Section 12 lists what stays manual.

### 11.2 Setup and isolation

- **Identity:** Jenkins uses a Databricks service principal with OAuth. Its credentials are held in Jenkins.
- **Agent:** a labelled agent or pinned image, with Python, the Databricks CLI and SDK, and Databricks Connect.
- **Isolation:** each build takes a unique ID from `BUILD_TAG`. It deploys its jobs with Asset Bundles under its own `name_prefix`, creates its own schemas (such as `cdp_test.qa_<id>_gold`) and its own landing and checkpoint paths, and passes them all to the jobs as parameters (section 4.9).
- **Cleanup:** every build, pass or fail, destroys its jobs, drops its schemas and removes its paths. A nightly sweep removes anything left by crashed builds.

### 11.3 Pipeline stages and triggers

The merge pipeline is budgeted at 30 minutes. All budgets are validated in the Epic 3 pilot.

| Stage | Trigger | Budget |
| --- | --- | --- |
| Deploy: create schemas and paths; bundle deploy | Every build | 5 min |
| Unit gate: read the developer build result for this commit | Every merge | 2 min |
| Merge tests: smoke, targeted regression and progression for touched pipelines and apps; wiring; app API and access | Every merge | 18 min |
| DQX and reconciliation gate, read from the unified view | Every build | 2 min |
| Full regression, including app UI journeys | Nightly and release candidate | 90 min |
| Failure tests | Nightly and release candidate | 60 min |
| NFR tests in the performance schema | Release candidate | 2 h |
| Publish and cleanup: JUnit, Allure, results to Delta, Xray import | Always | 3 min |

Release-candidate runs also execute in the test environment. Pre-production is only for a deployment smoke check and UAT, because its masked data cannot give the exact results these tests assert. The repository layout is in Appendix A.4 and the Jenkinsfile in Appendix A.5.

### 11.4 Unified results view

`cdp_qa.results.test_results_v` is the single place every result is read from. It is a view over three sources:

- the DQX log table, unchanged;
- the reconciliation log table, unchanged;
- a new QA results table, holding pytest results, the developer build result and release-gating manual results from Xray.

DQX and reconciliation log by Databricks run. A run correlation table, `cdp_qa.results.run_correlation`, links each run to its `build_tag` and `git_commit`, and every job receives both as parameters. The view's columns are listed in Appendix C.2.

## 12. Manual and process testing

About 15–20% of in-scope tests stay manual on purpose, because they need judgement or run too rarely to automate. These are exploratory testing of new tier 1 features, UAT support, visual checks of dashboards and apps, one-off migrations and backfills, and release sign-off. QA also does process work: reviewing requirements and mapping documents, designing tests and investigating failures.

### 12.1 Rules for manual work

Manual tests are planned and executed in Xray, and release-gating results also go into the unified view. A manual check repeated in three releases is reviewed for automation. Exploratory sessions are time-boxed, and their findings logged so useful ones become automated tests.

## 13. Integration with spec-driven development

Tests come from the spec before the code is written, so a story cannot be done without the checks that prove it.

### 13.1 Flow for one story

1. The story has acceptance criteria in Given / When / Then form, plus mapping rules.
2. QA maps each criterion to a test type and tool (section 5.1).
3. QA designs tests while development starts; developers write unit tests with the code.
4. QA automates the pytest tests, and DQX and reconciliation rules are requested.
5. Jenkins runs everything on merge.
6. Xray keys and rule IDs show which stories and rules are proven.

Appendix C.3 walks through an example story.

### 13.2 Definition of ready

Acceptance criteria are testable, mapping rules are documented, the table's tier is known, and test data needs are identified.

### 13.3 Definition of done

A story is done when its unit tests pass, and every criterion has an automated check or a recorded reason for a manual one. Requested DQX and reconciliation rules must be live in test, all checks must pass in Jenkins and appear in the unified view, and new tests must join the regression suite.

### 13.4 AI assistance

AI may draft test cases from acceptance criteria and mapping rules. A QA engineer reviews every case, and expected results always come from the spec, never from the code's current behaviour.

## 14. Test data and environments

Tests run in isolated schemas on hand-made, synthetic or masked data. No test reads or writes unmasked production personal data.

### 14.1 Environments

- **Development:** unit tests, on small hand-made samples.
- **Test:** functional, transformation, failure, wiring and app tests, on hand-made and synthetic data per build.
- **Performance schema:** NFR tests, on production-sized synthetic data, or masked data if approved.
- **Pre-production:** UAT and release rehearsal, on a masked production copy only once approved (section 14.4).
- **Production:** monitoring only.

The exact environment and catalog names are confirmed in Epic 1.

### 14.2 Types of test data

Hand-made rows serve functional and transformation tests. Generated data matching production counts and skew serves NFR tests. Masked samples serve UAT and exploratory testing, and deliberately broken files serve failure tests.

### 14.3 Rules

Each build uses its own schemas and paths and removes them at the end (section 11.2). Every test creates its own data and never depends on another run. Test data and mapping cases are versioned and reviewed like code. Reference data comes from a fixed snapshot, and date-based tests set `as_of_date`.

### 14.4 Privacy

Masked or tokenised production data is still personal data under GDPR.

- **Approval:** any copy of production data outside production needs the data protection officer's sign-off as well as the data owner's. Until then, all test data is synthetic.
- **Masking:** join keys are tokenised deterministically. Names, emails, phone numbers and addresses are replaced. Free text is dropped. Identifying combinations, such as birth date with postcode, are generalised.
- **Access:** masked copies can be read only by QA, the test principals and named UAT users.
- **Retention:** they are refreshed or deleted at least every 90 days.
- **No real data in reports:** Jira, Allure, Xray and the results `message` column never contain row values from non-synthetic data.
- **Erasure:** the erasure scenario in section 6.2 proves that deletions reach every gold table and leave Delta history.

## 15. Entry and exit criteria

A release goes to production only when every exit criterion is met, or a named approver has signed a waiver for the one that is not.

### 15.1 Criteria by stage

- **Unit:** complete when the developer build for this commit has passed.
- **Functional and transformation:** start once deployed to test with the unit gate green. Complete when all priority 1 tests pass, every other failure has an S3 or S4 defect or a waiver, and no S1 or S2 defect is open.
- **DQX and reconciliation:** start once the pipeline has run in test. Complete when there is no critical DQX failure and no unexplained break.
- **Failure:** start once the functional stage is green. Complete when the tier 1 scenarios pass, or have an accepted defect with a workaround.
- **NFR:** start once the candidate is frozen and performance data is loaded. Complete when the targets are met, or a miss is accepted in writing.
- **UAT:** start once all the stages above are green. Complete when business sign-off is recorded in Xray.

### 15.2 Release gate

The gate is one query on the unified results view for the candidate's commit. The release is go only when:

1. Every stage has results for this commit.
2. Every priority 1 test passed. Quarantined tests don't count as passed.
3. Every other failure has a linked S3 or S4 defect, or an unexpired waiver.
4. No S1 or S2 defect is open.
5. UAT is signed off in Xray.

The pass rate is a trend, not the gate, so no failure is accepted just because it falls inside a percentage. The dashboard shows go or no-go, with the failing items.

### 15.3 Waivers and suspension

A waiver names the check, the risk, the workaround and an expiry date. Waivers for priority 1 tests or S1 defects need both the product owner and the QA lead, and an expired waiver fails the gate again.

Testing is suspended if the environment is down, the build cannot deploy, or more than 30% of tests fail for one cause. It resumes when the smoke suite passes.

## 16. Defect management and reporting

Within one working day, every failed check becomes a product defect, a test fix or an environment ticket. Nothing stays red without an owner.

### 16.1 Test priority and defect severity

Two separate scales keep a test's importance apart from a defect's impact.

**Test priority:**

1. Smoke tests, and security, privacy or regulatory tests on any tier.
2. Other tier 1 tests.
3. Tier 2 and 3 tests.

**Defect severity:**

| Severity | Definition | Fix by |
| --- | --- | --- |
| S1 | Wrong or missing tier 1 data, a security or privacy breach, or a pipeline that cannot run | Before release; blocks the gate |
| S2 | Wrong tier 2 data, or a tier 1 issue with a workaround | Before release unless waived |
| S3 | Minor, no effect on consumers | Within two releases |
| S4 | Cosmetic or improvement | Backlog |

### 16.2 Triage

Jenkins posts each failed build to the team channel, and the QA engineer on rotation classifies each failure as product, test or environment. Product defects go to Jira, linked to the failing Xray execution and its Allure report, with no real row values. A fix is verified by rerunning the test, which stays in the regression suite.

### 16.3 Flaky tests

A test that passes on rerun with no code change is flaky. It is quarantined within a day and fixed within a sprint, and it counts against the flaky-test measure.

### 16.4 Reporting tools

Each tool answers a different question. ReportPortal was trialled and is not adopted for now (Appendix B).

- **Allure:** why a test failed in a given run. For QA and developers.
- **Xray in Jira:** whether a story is proven and the release is signed off. For the Product team and QA lead.
- **Databricks dashboard:** whether the data is healthy, and trends across frameworks. For everyone.

**Xray integration:**

- Each test carries its Xray key as a pytest marker.
- A `conftest.py` hook writes the key to the JUnit report with `record_property`; this needs `junit_family = xunit1`.
- Jenkins imports the report through Xray's REST API. The latest Xray release is assumed, and the endpoint for Cloud or Server/Data Center is set in configuration.
- Tests without a key are not imported.
- DQX and reconciliation appear in Xray as one gate test per pipeline.

### 16.5 Dashboard

One Databricks dashboard reads the unified view. Its views cover release readiness for the Product team, the latest run and 30-run trend, the coverage matrix, mapping-rule traceability, and the NFR trend. Story-level traceability stays in Xray.

### 16.6 Reporting rhythm

Every build sends a Jenkins notification with links. A short quality summary goes out weekly. Each release attaches the Xray execution report and the release readiness view to its sign-off.

## 17. Roles and responsibilities

Quality is shared, but each activity has one accountable role. The full RACI is in Appendix C.4.

- **The QA lead** owns this framework, the coverage matrix, the QA suite, the unified view and the release evidence.
- **QA engineers** design and automate tests, triage failures and support UAT.
- **Developers** write unit tests, meet the testability requirements and fix defects.
- **The DQX and reconciliation owners** maintain their rules and log tables, add rules on request, and pass build parameters to their tasks.
- **The platform team** provides access, catalogs and compute, and runs production monitoring.
- **The product owner and LoB representatives** own requirements and NFR targets, and sign off UAT and releases.

## 18. Generic CDP layer and line-of-business context

The framework has two layers.

- **Generic CDP layer:** applies to every pipeline. It covers standard scenarios, cleansing and lookups, the failure catalogue, platform NFR targets, and baseline data checks (schema, keys, uniqueness, freshness). QA and the platform owners own it.
- **Line-of-business (LoB) layer:** adds only what is specific to one business area, such as LoB processes, mapping rules, source faults, tighter SLAs and value rules (for example, loyalty points are never negative). The LoB representative owns it, with QA.

### 18.1 How it works

Generic tests are parametrised from a configuration file per LoB, which lists its pipelines, tables, tiers, keys and SLAs (example in Appendix A.6). LoB-specific tests carry an LoB marker, and results carry the LoB, so Jenkins and the dashboard can filter by line of business.

### 18.2 Onboarding a new LoB

1. List its pipelines, tables and apps, and assign tiers.
2. Add its configuration file. Generic tests then run with no new code.
3. Add its rows to the coverage matrix.
4. Write its mapping cases and specific tests.
5. Request its DQX and reconciliation rules.
6. Agree its NFR targets.

## 19. Risks, assumptions and open questions

### 19.1 Risks

- **Access or testability is late:** the platform team or developers deliver late. Request access in week one, make testability a design exit criterion, and run non-compliant pipelines one build at a time.
- **DQX or reconciliation results are hard to join:** agree the log columns early and use the run correlation table. A nightly copy is a temporary fallback only.
- **Mapping documents are incomplete:** start with tier 1, and treat missing rules as requirement defects.
- **Tests are slow, costly or flaky:** run each pipeline once, use small data on merge, isolate builds, and quarantine flaky tests.
- **Masked-data approval or team capacity falls short:** use realistic synthetic data and the prioritised backlog. The pilot measures real effort.

### 19.2 Assumptions

CDP uses bronze, silver and gold in Unity Catalog, and all names here are illustrative. Pipelines run as parameterised Databricks jobs that Jenkins can deploy, with DQX and reconciliation as job tasks writing to their own Delta log tables. Developer builds report a pass or fail per commit, and a separate test catalog is available. Lakeflow Declarative Pipelines are not used today; if they are adopted, sections 4.9, 8 and 11.2 are revisited.

### 19.3 Open questions

- [ ] Tier 1 tables and the pilot pipeline and LoB (Story 1.1).
- [ ] Owners and columns of the DQX and reconciliation log tables (Story 1.4).
- [ ] Real SLAs for run time, freshness and cost.
- [ ] Current deployment method: Asset Bundles or other.
- [ ] Xray Cloud or Server/Data Center (latest release assumed).
- [ ] Schema-evolution and bad-record policy per tier (Story 2.7).
- [ ] Who in data protection approves masked data, and on what basis.
- [ ] The release gate's notification channel.
- [ ] Whether any sources run as continuous streams.
- [ ] Current unit test coverage.
- [ ] Other CDP UI apps and the tables they read.
- [ ] App data access: the app's own service principal or the user's own permissions.
- [ ] Unattended Playwright sign-in and per-build app copies (Story 2.9).

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

**Sizes:** S is up to 3 days, M is up to one sprint, and L is more than one sprint and must be split before it starts. The full user stories, with acceptance criteria and tasks, are in the Backlog tab (exported here as `CDP_QA_Backlog.pdf`).

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

### A.4 Target layout of the extended pytest framework (section 11)

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
# Dashboard app: a KPI from the API equals the same query on gold
def test_active_customers_kpi_matches_gold(dashboard_client, tables):
    resp = dashboard_client.get("/api/kpis/active-customers", params={"country": "IE"})
    assert resp.status_code == 200
    expected = (tables.gold("customer_profile")
                .filter("country = 'IE' AND is_current").count())
    assert resp.json()["value"] == expected

# Lineage app: upstream tables for a known gold table match the lineage source
def test_lineage_for_customer_profile(lineage_client, expected_lineage):
    resp = lineage_client.get("/api/lineage", params={"table": "gold.customer_profile"})
    assert resp.status_code == 200
    upstream = {edge["source"] for edge in resp.json()["edges"]}
    assert upstream == expected_lineage("gold.customer_profile")

# Access: a call without sign-in is refused
def test_dashboard_refuses_unauthenticated_call(dashboard_url, httpx_client):
    resp = httpx_client.get(f"{dashboard_url}/api/kpis/active-customers")
    assert resp.status_code in (401, 302)

# UI: one critical journey in a real browser
def test_dashboard_filter_journey(dashboard_page):  # signed-in Playwright page
    dashboard_page.get_by_test_id("country-filter").select_option("IE")
    expect(dashboard_page.get_by_test_id("kpi-active-customers")).not_to_have_text("")
```

## Appendix B. Why ReportPortal is not adopted

- Its run reports overlap with Allure, its execution history with Xray, and its trends with the Databricks dashboard.
- It needs its own server to host and maintain, and a fourth place to send results.
- It is revisited only if failure triage across many runs becomes more than Allure and Xray can handle.

## Appendix C. Reference tables

### C.1 Transformation rule examples (section 7)

| Category | Rule (illustrative) | Input | Expected output |
| --- | --- | --- | --- |
| Derived field | `full_name` = trimmed first + last name | `"  John "`, `"Smith"` | `"John Smith"` |
| Null handling | Same rule | `"John"`, null | `"John"`, not `"John null"` |
| Lookup | Country code to name | `IE` | `Ireland` |
| Lookup, no match | Same rule | `XX` | `Unknown`, not null |
| Aggregation | `total_spend_12m` sums completed orders in 12 months | €100 completed, €50 cancelled, €30 from 14 months ago | `100` |
| Deduplication | Latest record per email wins | Updated 1 Nov and 5 Nov | The 5 Nov values |
| Date format | Source is dd/MM/yyyy | `09/11/2026` | `2026-11-09` |
| Type cast | Amount string to decimal | `"1,250.50"` | `1250.50` |
| Conditional | `segment` = Gold when spend ≥ 1,000 | 1,000 | `Gold` |

### C.2 Unified results view columns (section 11.4)

| Column | Meaning |
| --- | --- |
| `build_tag`, `git_commit` | Jenkins build and commit, through the correlation table |
| `databricks_run_id` | The job run that produced the result |
| `executed_at`, `environment` | When and where the check ran |
| `source_framework` | `pytest`, `dqx`, `reconciliation`, `unit` or `manual` |
| `test_type` | Functional, transformation, failure, NFR, data quality, reconciliation, unit, manual, app |
| `suite` | Progression or regression, from Xray |
| `priority` | Test priority 1 to 3 (section 16.1) |
| `tier`, `line_of_business` | Tier and LoB of what was tested |
| `pipeline`, `table_name` | What was tested |
| `test_name`, `xray_key`, `rule_id` | The check, its Xray test and its mapping rule |
| `status` | Passed, failed, error, skipped, quarantined or waived |
| `duration_seconds`, `message` | Run time and failure detail, with no real row values |

### C.3 Spec-driven example (section 13)

Story: merge duplicate customer profiles that share an email address.

| Acceptance criterion | Test type | Tool |
| --- | --- | --- |
| Two profiles with the same email become one | Functional | pytest |
| The survivor keeps the latest address | Transformation | pytest |
| `email` is unique in gold | Data quality | DQX rule request |
| Distinct emails in silver equal rows in gold | Reconciliation | Reconciliation rule request |
| A crash during the merge leaves no half-merged profiles | Failure | pytest |
| The merge adds no more than 5 minutes to the run | NFR | pytest |

### C.4 RACI (section 17)

R = does the work, A = accountable, C = consulted, I = informed.

| Activity | Developers | QA | DQX and recon owners | Platform | Product owner / LoB |
| --- | --- | --- | --- | --- | --- |
| Acceptance criteria and mapping rules | C | C | I | I | A, R |
| Unit tests | A, R | I | I | I | I |
| Data quality and reconciliation rules | C | C | A, R | I | C |
| QA tests and regression upkeep | C | A, R | I | C | I |
| Coverage matrix | C | A, R | C | I | I |
| Jenkins test stages | C | A, R | I | R | I |
| Access, catalogs, environments | I | C | I | A, R | I |
| Unified view and dashboard | I | A, R | C | C | I |
| Defect triage | R | A, R | R | C | I |
| NFR targets | C | R | I | C | A |
| UAT and release sign-off | C | R | I | I | A, R |

### C.5 Failure scenario catalogue (section 8)

| Scenario | Expected behaviour |
| --- | --- |
| Missing source file | Skip or fail with an alert; gold unchanged, not emptied |
| Renamed column | As the schema policy states. Auto Loader's default `addNewColumns` adds the new column and leaves the old one null, so the policy must block or alert; never silent nulls |
| New column | As the schema policy states; existing columns unaffected |
| Unparseable rows | Good rows load; bad rows go to `_rescued_data` or the bad-records path, and a DQX rule raises them |
| Rule-breaking rows | Good rows reach gold; bad rows go to the DQX quarantine; alert above threshold |
| Truncated file | FAILFAST loads nothing; PERMISSIVE loads readable rows and rescues the rest, as the policy states |
| Crash before gold (`fail_before`) | Gold unchanged; rerun completes with no duplicates |
| Task retry | Same result as a clean run |
| Same file landed twice | Ignored by the Auto Loader checkpoint |
| Same content, new file name | Removed by business-key deduplication. Auto Loader alone would load it twice |
| Upstream job fails | Gold does not run on stale data |
| Files out of order | Same final state as in order |
| Volume spike or drop (10x) | Completes or fails safely; volume alert fires |
| Permission revoked | Clear permission error; no partial write |
| Rollback | Delta `RESTORE` works within VACUUM retention; downstream streams may need a restart |
