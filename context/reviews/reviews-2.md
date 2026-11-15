# Independent Review 2 – CDP QA Testing Framework

**Review basis:**
- `CDP_QA_Testing_Framework_Specification.pdf`
- `CDP_QA_Backlog.pdf`
- `decisions-log.md`
- `02-independent-review.md` (review instructions)

**Review context:** This is the second independent review. Decisions recorded in the decisions log and the additional confirmations supplied for this review are treated as confirmed unless technically incorrect. The review does not propose reopening those decisions. No source documents have been modified.

## 1. Overall verdict

**Verdict: Minor revision recommended before reviewer circulation.**

The specification has retained its overall architecture and most of its important decisions despite the substantial reduction in length. The split between QA automation, DQX, reconciliation, developer-owned unit tests, Jenkins orchestration, Allure, Xray and the Databricks dashboard remains coherent.

However, a small number of corrections should be made before sending it out. These are not reasons to reverse the architectural decisions. They are mainly specification defects, implementation dependencies and a few technical inaccuracies.

| Area | Assessment |
|---|---|
| Overall architecture | Strong |
| Alignment with confirmed decisions | Strong |
| Sections 1–19 clarity | Good, with exceptions |
| Cross-reference accuracy | One definite incorrect reference; one misleading location reference |
| Backlog consistency | Several issues requiring correction |
| Databricks technical accuracy | Generally good; some important corrections |
| Jenkins execution design | Needs refinement |
| GDPR and data protection | Good foundation; erasure needs strengthening |
| Standalone reviewer readiness | Partial |
| Length and organisation | Good |

## 2. Findings – prioritised

### High severity

#### H1. Release gate cannot be fully derived from the specified unified results view

- **Location:** Sections 11.4, 15.2, 16.1; Appendix C.2.
- **Problem:** Section 15.2 says the release gate is one query against the unified results view. However, Appendix C.2 does not define enough information to evaluate all five gate conditions.
- **Why it matters:** The gate must determine whether defects are S1/S2, whether S3/S4 defects are linked, whether waivers are unexpired and approved, and whether UAT has signed off. The listed columns do not include defect status/severity, waiver expiry/approval or an explicit gate-stage identifier.
- **Fix:** Define the minimum gate contract. Either add the necessary defect, waiver, approval and stage information to the unified view, or explicitly allow the gate to join the relevant Jira/Xray evidence. Do not claim that the listed view alone supports the gate until this is proven.

#### H2. Pilot app testing depends on reporting functionality scheduled for a later epic

- **Location:** Section 6.5; Backlog Stories 3.6 and 4.1.
- **Problem:** Story 3.6 requires app tests to write to the unified results view. However, Epic 4 implements the unified results infrastructure, while Epic 3 is the pilot and does not depend on Epic 4.
- **Why it matters:** The pilot cannot meet its acceptance criteria using an infrastructure component that has not yet been delivered.
- **Fix:** Either make the initial pilot report to Allure/Xray and add unified-view publishing after Epic 4, or introduce an explicit dependency on the required Epic 4 deliverables.

#### H3. GDPR erasure testing is incomplete and the Delta purge statement is too broad

- **Location:** Section 6.2, Section 14.4, Story 5.5; Appendix C.5.
- **Problem:** The erasure scenario covers gold tables and views, then says the data is purged from Delta history after retention and VACUUM. This does not establish complete erasure across the CDP data lifecycle.
- **Why it matters:** Personal data can also exist in bronze, silver, source files, checkpoints, derived datasets and other retained copies. Furthermore, tables using deletion vectors may require `REORG TABLE ... APPLY (PURGE)` before VACUUM can physically remove deleted records.
- **Fix:** Define the erasure scope across all relevant storage and downstream copies. Clarify the retention and physical deletion procedure, including deletion-vector handling, and test that a subsequent Delta RESTORE cannot reintroduce erased records.

#### H4. Auto Loader schema-evolution behaviour is described inaccurately

- **Location:** Section 8.2; Appendix C.5.
- **Problem:** The specification states that default `addNewColumns` silently turns a renamed column into nulls.
- **Why it matters:** Current Auto Loader documentation describes a different sequence: with `addNewColumns`, detecting a new column causes an `UnknownFieldException`, updates the schema, and requires a restart. The old column can subsequently be null for new records. Also, the default depends on whether a schema is supplied.
- **Fix:** Describe the detection, failure, schema update and restart sequence. State the relevant schema-evolution mode explicitly for each tier rather than assuming one universal default.

#### H5. NFR execution in the Jenkins example does not guarantee execution for release candidates

- **Location:** Section 11.3; Appendix A.5.
- **Problem:** Section 11.3 schedules NFR testing for release candidates. Appendix A.5 runs NFR only when `RUN_NFR` is true. `RELEASE_CANDIDATE` does not automatically enable it.
- **Why it matters:** A release candidate can complete without the NFR stage, despite the stated release-gate requirements.
- **Fix:** Make the NFR stage execute whenever `RELEASE_CANDIDATE` is true, or explicitly enforce and validate the required parameter combination.

#### H6. Cost measurement timing conflicts with release-candidate NFR execution

- **Location:** Sections 9.1, 11.3; Backlog Story 5.3.
- **Problem:** Cost is checked from `system.billing.usage` the next day, but NFR execution has a two-hour release-candidate budget.
- **Why it matters:** The cost result cannot necessarily be available when the release gate evaluates the candidate.
- **Fix:** Distinguish synchronous NFR checks from asynchronous billing verification. Define whether cost is a blocking release criterion or a subsequent monitoring result, including how the gate handles pending billing data.

### Medium severity

#### M1. Incorrect backlog cross-reference for app coverage

- **Location:** Backlog Story 6.5.
- **Problem:** Its acceptance criteria refer to the app rows in section 4.9. Section 4.9 contains testability requirements, not app coverage rows.
- **Why it matters:** The implementation team is directed to the wrong section.
- **Fix:** Reference section 6.5 for app test requirements and section 5.2 for the coverage matrix, as appropriate.

#### M2. Coverage matrix scope differs between the specification and backlog

- **Location:** Section 5.2; Stories 1.2 and 6.5.
- **Problem:** Section 5.2 says the matrix covers tier 1 and tier 2 tables and apps. Story 1.2 initially defines its matrix only for tables, while Story 6.5 adds app rows during rollout.
- **Why it matters:** It is unclear whether the initial inventory and baseline deliverable includes app coverage or whether apps are intentionally deferred.
- **Fix:** Explicitly distinguish the initial table matrix from later app coverage, or include app rows in Story 1.2. Add app-specific dimensions: API, data correctness, access, UI and load.

#### M3. Xray gate-test counting is ambiguous

- **Location:** Section 11.1; Section 16.4; Backlog Story 4.8.
- **Problem:** Section 11.1 says DQX and reconciliation each count as one automated gate test per pipeline. Story 4.8 describes one Xray gate test per pipeline covering both frameworks.
- **Why it matters:** The automation ratio could count one combined test or two separate tests.
- **Fix:** Explicitly choose one combined gate test or two independent gate tests, then use the same definition everywhere.

#### M4. App authorisation model is unresolved despite security requirements being specified

- **Location:** Section 6.5; Section 9.2; Section 19.3; Story 2.9.
- **Problem:** The specification requires role-based app access tests but leaves open whether apps access data using their own service principal or the user's permissions.
- **Why it matters:** These are materially different security models. An app's own service principal does not automatically enforce the individual user's Unity Catalog permissions.
- **Fix:** Define the intended authorisation model for each app and design the tests accordingly. For API calls, explicitly include the app's `CAN USE` permission and the required app-scoped OAuth token. The Playwright spike should test the actual user-authorisation journey where required.

#### M5. The automation ratio needs an operational counting rule

- **Location:** Sections 3 and 11.1; Story 1.3.
- **Problem:** The ratio is measurable, as confirmed in the decisions log, but the treatment of quarantined, skipped, newly created and combined gate tests is not fully defined.
- **Why it matters:** Different teams could calculate different percentages for the same release.
- **Fix:** Keep the confirmed formula, but define the release snapshot, test identity, duplicate handling and inclusion/exclusion rules.

#### M6. Transformation-test execution conflicts with parallel pytest workers

- **Location:** Sections 4.6 and 11.2; Appendix A.2.
- **Problem:** The transformation example uses a session-scoped fixture to prepare one pipeline output for parameterised assertions. Jenkins runs pytest with `-n auto`.
- **Why it matters:** With pytest-xdist, session-scoped fixtures are instantiated separately in each worker process. The pipeline may therefore run multiple times rather than once for all cases.
- **Fix:** Separate data preparation and pipeline execution from parallel assertions, or explicitly control worker grouping and shared fixture behaviour. Ensure parallel workers cannot overwrite one another's test data.

#### M7. Changed-file-to-pipeline detection is under-specified

- **Location:** Section 4.6; Section 18.1; Appendix A.5/A.6; Story 2.6.
- **Problem:** Section 4.6 says changed files map to pipelines through LoB configuration, but the example configuration does not contain source-code path mappings.
- **Why it matters:** The `changed_pipelines.py` example has no defined mapping contract to work from.
- **Fix:** Add a code-path-to-pipeline mapping or define another explicit mechanism for detecting affected pipelines.

#### M8. App performance targets mix API and browser performance

- **Location:** Section 9.1; Story 5.7.
- **Problem:** Locust is specified for both API response times and page response times.
- **Why it matters:** HTTP load testing does not measure React rendering, browser interaction or client-side performance in the same way as a real browser.
- **Fix:** Keep Locust for backend/API load and define a separate browser measurement for critical page performance.

#### M9. Xray JUnit mapping needs a precise contract

- **Location:** Section 16.4; Story 2.2 and Story 4.7.
- **Problem:** The specification says the pytest hook writes the Xray key into JUnit XML using `record_property`, but does not specify the property name or demonstrate that the importer maps it to an existing Test issue.
- **Why it matters:** A syntactically valid JUnit report does not automatically guarantee that Xray updates the intended existing tests.
- **Fix:** Specify the exact property/category mapping and prove it with a test execution against the configured Cloud or Server/Data Center release.

#### M10. Reporting failures are swallowed in the Jenkins example

- **Location:** Appendix A.5.
- **Problem:** Both `publish_results.py` and `import_xray.py` end with `|| true`.
- **Why it matters:** A build can appear successful even when reporting or Xray import has failed.
- **Fix:** Permit cleanup and later reporting attempts to continue, but record publisher/import failures as explicit build failures or missing-evidence conditions that block the release gate.

#### M11. Epic sizes do not clearly reconcile with story estimates

- **Location:** Section 20; Backlog, Epics 1, 4, 5 and 7.
- **Problem:** Several epics are sized M despite containing multiple M stories and several S stories.
- **Why it matters:** The epic size can give the Product team an unrealistic impression of delivery effort.
- **Fix:** Either recalculate epic estimates from the story estimates or state that epic sizes are independent high-level estimates and are not additive roll-ups.

#### M12. Release-gate stage completion needs a clearer definition

- **Location:** Sections 11.3 and 15.2; Appendix C.2.
- **Problem:** The gate requires results for every stage, but the unified view does not explicitly identify stage/run type, and the relationship between merge, nightly, release-candidate and UAT evidence is not fully described.
- **Why it matters:** It is unclear whether a missing result means failure, a stage not being applicable, or an earlier result being reused.
- **Fix:** Define required stages for each trigger type and how results are selected for a candidate commit.

#### M13. Data leakage controls do not cover every reporting channel explicitly

- **Location:** Section 14.4; Section 16.2; Section 6.5.
- **Problem:** The no-real-row-values rule mentions Jira, Allure, Xray and the results message column, but does not explicitly cover Jenkins console logs, Playwright traces, screenshots and API error payloads.
- **Why it matters:** Those artefacts can contain personal data even when the result table is clean.
- **Fix:** Extend the redaction and synthetic-data requirements to all logs, attachments, traces and diagnostic outputs.

#### M14. Story 4.4 acceptance criterion is ambiguous

- **Location:** Backlog Story 4.4.
- **Problem:** The phrase "for every commit Jenkins tests" is grammatically unclear.
- **Why it matters:** It does not define whether every commit, every Jenkins-tested commit or every developer build must be recorded.
- **Fix:** Rewrite the acceptance criterion to identify the exact commit population and expected status.

#### M15. Terms are used before being defined

- **Location:** Sections 4, 15, 16 and 18.
- **Problem:** `UAT` is used in section 4.1 without being expanded; `NFR` appears before its expansion in section 9; `LoB` is used in section 4.6 before being expanded in section 18. Priority 1 and S1/S2 are used in section 15.1 before their scales are defined in section 16.1.
- **Why it matters:** This is precisely the kind of ambiguity introduced by shortening a document.
- **Fix:** Define UAT, NFR and LoB at first use. Move a compact priority/severity definition to section 15 or introduce it before the entry/exit criteria.

#### M16. The backlog document is referenced as a tab

- **Location:** Section 20, page 21.
- **Problem:** The specification says the full user stories are in the "Backlog tab". The actual deliverable is a separate PDF.
- **Why it matters:** A reviewer following the reference may look for a tab that does not exist.
- **Fix:** Reference the companion document by its actual title, *CDP QA Backlog*, and make clear that section 20 contains only the epic summary.

#### M17. The 80% tier 1 coverage target needs a clearer interpretation

- **Location:** Sections 3 and 4.5; Story 6.2.
- **Problem:** Section 3 targets 80% automated coverage of tier 1 tables, while section 4.5 says every test type for tier 1 is automated and gated.
- **Why it matters:** A reviewer cannot tell whether the 80% refers to the proportion of tables onboarded or the proportion of applicable checks covered.
- **Fix:** State the numerator and denominator for tier 1 table coverage and distinguish it from the 80–85% Xray automation ratio.

## 3. Cross-reference audit

**Assessment: References resolve, with qualifications.**

The specification and backlog references were checked against the document structure.

| Reference group | Result |
|---|---|
| Section 3 references | Correct |
| Section 4.5 and 4.9 | Correct sections; see Story 6.5 issue |
| Sections 5.1 and 5.2 | Correct |
| Sections 6.2 and 6.5 | Correct |
| Sections 7.3 and 8.1–8.2 | Correct |
| Sections 9, 9.2 and 9.3 | Correct |
| Sections 10.3 and 11.2–11.4 | Correct |
| Sections 13, 14.4 and 15.2 | Correct |
| Sections 16.1 and 16.5 | Correct |
| Sections 18 and 18.2 | Correct |
| Appendix C.1–C.5 | All exist and cover the referenced subjects |
| Story references | All identifiable story numbers resolve |

The following are the exceptions or qualifications:

1. **Story 6.5 → section 4.9:** Incorrect subject reference, as described in M1.
2. **Section 20 → Backlog tab:** Incorrect document-location reference, as described in M16.
3. **Story 3.6 → unified results view:** The referenced functionality exists conceptually, but its delivery dependency is missing.
4. **Story 4.8 → gate-test counting:** The reference is valid, but the acceptance criterion does not clearly match the automation-ratio counting rule.

No missing Appendix C subsection or nonexistent numbered story was identified.

## 4. Consistency between the backlog and specification

The overall epic structure is consistent.

**Consistent elements:**
- Eight epics in both documents.
- Story numbering follows the corresponding epic.
- The pilot, test foundation, reporting, NFR, rollout and onboarding work are represented.
- The first increment in section 20 aligns with the proposed foundation-first approach.
- The confirmed decisions around pytest, Allure, Xray and Databricks Apps are reflected in the backlog.

**Main consistency problems:**

| Item | Problem |
|---|---|
| Epic 3 / Story 3.6 | Requires unified results before Epic 4 delivers them |
| Epic 1 / Story 1.2 | Matrix initially covers tables, while section 5.2 also includes apps |
| Epic 4 / Story 4.8 | One combined gate versus two independently counted gates |
| Epic 6 / Story 6.5 | Incorrect section 4.9 reference |
| Epic 4 / Story 4.4 | Ambiguous acceptance criterion |
| Epic estimates | Several M-sized epics contain multiple M-sized stories |

The epic estimates should be reconciled with story estimates before Product prioritisation. This does not mean the estimates are necessarily wrong, but the relationship between the two levels should be explicit.

## 5. Would sections 1–19 stand alone?

**Assessment: Yes for strategic review; not completely for detailed technical sign-off.**

The main body successfully communicates:
- Why the framework is needed.
- What is in and out of scope.
- How testing is organised.
- Which existing frameworks are reused.
- How tests progress through CI/CD.
- Who owns the different activities.
- How quality is measured.
- How release decisions are made.

A reviewer can understand the proposed architecture without reading the appendices.

However, four areas depend too heavily on Appendix C.

| Body section | Appendix dependency | Assessment |
|---|---|---|
| 7.2 | C.1 – transformation examples | Acceptable |
| 8.2 | C.5 – 15 failure scenarios | Somewhat excessive |
| 11.4 | C.2 – results-view columns | Too much dependency |
| 13.1 | C.3 – worked example | Acceptable |
| 17 | C.4 – complete RACI | Acceptable |

The most important issue is section 11.4. The unified results view is central to the architecture, release gate and reporting strategy. A reviewer should not have to open an appendix to understand its minimum logical contract.

### What should move back into the main body

| Content | Recommendation |
|---|---|
| Unified results view | Bring a compact logical schema and gate requirements into section 11.4 |
| Failure catalogue | Bring five representative failure scenarios and expected outcomes into section 8.2 |
| Priority and severity | Define before section 15 |
| App authorisation | State that the security model must be resolved before access-test implementation |
| Full RACI | Keep in C.4 |
| Detailed transformation examples | Keep in C.1 |
| Full 15-scenario catalogue | Keep in C.5 |
| Worked spec-driven example | Keep in C.3 |
| Code samples | Keep in Appendix A |
| ReportPortal rationale | Keep in Appendix B |
| Full user stories | Keep in the separate backlog PDF |

This would improve standalone readability without moving the entire appendices back into the specification.

## 6. Length and trimming assessment

**Assessment: 20-page target met.**

The current document structure is:

| Component | Pages |
|---|---:|
| Sections 1–19 | 20 |
| Section 20 – epic table | 1 |
| Appendices A, B and C | 9 |
| **Specification PDF total** | **30** |
| Separate backlog PDF | 8 |

The trimming has been successful.

The full failure catalogue, RACI, code examples and transformation tables should not be moved back into the main body.

Instead, recover space by tightening some existing sections:
- Sections 5 and 17 have some ownership repetition.
- Sections 3 and 11.1 repeat the automation-ratio discussion.
- Sections 4.4 and 5.1 overlap in explaining test classification.
- Sections 15 and 16 could be more clearly separated between gate conditions and defect management.

Use that space for the unified results contract and the minimum failure-scenario summary.

**Recommendation:** Retain the current 20-page limit; do not increase it.

## 7. What is genuinely strong and should not change

The following parts of the specification should be protected from unnecessary changes:

1. **Reuse rather than replacement:** The separation between DQX, reconciliation, developer unit testing and QA testing is well thought out.
2. **Existing pytest and Allure framework:** Extending the existing infrastructure rather than creating another framework is the right architectural direction.
3. **Progression and regression model:** The distinction between progression, regression, smoke, targeted and full regression is clear and useful.
4. **Test isolation:** Parameterised jobs, independent schemas, landing paths and checkpoints are a strong foundation for reliable parallel testing.
5. **Risk-based coverage:** Prioritising tier 1 pipelines and tables before expanding to tier 2 is sensible.
6. **Transformation testing:** Testing mapping rules inside real pipeline execution, rather than directly calling transformation functions, is an appropriate separation from developer unit testing.
7. **Failure testing:** Testing safe failure, detection, alerting and recovery is one of the strongest parts of the design.
8. **Unified reporting architecture:** Keeping the existing DQX and reconciliation log tables unchanged while building a unified results view is a good integration decision.
9. **App testing strategy:** API, access, UI and load testing form a sensible layered approach. The unattended sign-in spike is particularly appropriate.
10. **Spec-driven development:** The definition of ready, definition of done and traceability through Xray provide a useful development lifecycle.
11. **Data protection:** The DPO approval requirement, synthetic-data fallback and restriction on real row values in reporting are important safeguards.
12. **Separate backlog:** Keeping full stories, acceptance criteria and tasks outside the main specification is the right decision for maintaining readability.

These decisions should be preserved.

## 8. Top five fixes

In priority order, these are the five things to address before circulation:

1. **Fix the unified results/release-gate contract:** Include defect, waiver, approval and stage information needed to calculate a genuine go/no-go.
2. **Resolve Epic 3 versus Epic 4 sequencing:** The pilot must not depend on a unified-results implementation that has not yet been delivered.
3. **Correct GDPR erasure and Delta purge requirements:** Cover all relevant data copies, deletion vectors, VACUUM and restore behaviour.
4. **Correct Auto Loader failure and schema-evolution descriptions:** Explicitly document the actual detection, restart and bad-record behaviour.
5. **Make release-candidate execution deterministic:** Guarantee NFR execution and resolve the conflict between the two-hour NFR budget and next-day billing data.

## 9. Final recommendation

**Minor revision recommended before circulation.**

A major restructuring or another reduction in page count is not recommended.

The specification is substantially improved. The architectural decisions are sound, the backlog is much more actionable, and the 20-page main-body target has been achieved.

The remaining work is primarily about making the specification implementable without interpretation.

Resolve the first five fixes, correct the cross-reference and terminology issues, and make the minimal additions to sections 8, 11 and 15.

After those corrections, the specification should be ready for independent review by Kedar, Gagan and Sukesh.

---

**Review scope note:** This file captures the review findings and recommendations. Technical documentation citations shown in the conversational review are not embedded as live links here; the relevant product behaviours are identified in the findings so they can be verified against the current Databricks, pytest-xdist and Xray documentation during remediation.
