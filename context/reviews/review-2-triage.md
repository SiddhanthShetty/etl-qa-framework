# Review 2 triage

Triage of `reviews-2.md` (second independent review, 2026-11-15) against the spec as exported on 2026-11-15.

Technical claims were checked against current documentation before accepting them:

- **Auto Loader (H4):** with `addNewColumns`, the stream fails with `UnknownFieldException` after adding the new column to the schema, and resumes on restart. `addNewColumns` is the default only when no schema is provided; with a provided schema the default is `none`. ([Databricks: Auto Loader schema](https://docs.databricks.com/ingestion/auto-loader/schema.html))
- **Deletion vectors (H3):** deleted rows stay in the Parquet files until `REORG TABLE ... APPLY (PURGE)` (or a rewrite), then VACUUM removes the old files. This only applies to tables with deletion vectors enabled. ([Databricks: deletion vectors](https://docs.databricks.com/delta/deletion-vectors.html))
- **pytest-xdist (M6):** session-scoped fixtures run once per worker, not once per run. ([pytest-xdist how-to](https://github.com/pytest-dev/pytest-xdist))
- **Xray JUnit (M9):** Xray maps a JUnit test case to an existing Test through a `test_key` (or `test_id`) property on the `testcase` element. ([Xray: JUnit reports](https://confluence.getxray.app/display/XRAYCLOUD/Taking+advantage+of+JUnit+XML+reports))

**Outcome (2026-11-15):** applied with the recommended defaults: cost is not a gate condition (H6), and there are two gate tests per pipeline (M3). Sections 1–19 still end on page 20.

**Key:** Accept = fix it. Partly = the point is valid but the reviewer's reading or fix is off; a smaller change is proposed. Reject = no change. Defer = valid, settled by an open question or a backlog story. **Decision** = needs the QA lead's call before editing.

## Summary

| Verdict | Findings |
|---|---|
| Accept | H1, H2, H3, H4, H5, M2, M6, M7, M8, M10, M11, M12, M13, M14, M15, M17 |
| Accept, decision needed | H6 (is cost blocking?), M3 (one or two gate tests?) |
| Partly | M1, M4, M5, M9, M16 |
| Reject | Moving five failure scenarios back into 8.2 (section 5 of the review) |

## Findings

| # | Verdict | Checked against the spec | Proposed change |
|---|---|---|---|
| H1 | Accept | 15.2 says "one query on the unified results view", but C.2 has no defect severity, waiver expiry, UAT sign-off or stage column. | 15.2: the gate joins the unified view with Xray/Jira defects, waivers and UAT status for the candidate commit. Add `stage` to the view. Put a five-line gate contract in 11.4 (merged with M12). |
| H2 | Accept | Epic 3 depends on 1 and 2 only; Story 3.6 AC requires the unified view from Epic 4. | Story 3.6 AC: results in Allure and Xray. Unified-view publishing for app tests moves to Story 4.1's AC. Check Stories 3.1–3.5 for the same dependency. |
| H3 | Accept | 6.2 erasure covers gold tables and views only. | Erasure scope: bronze, silver, gold, landing files, checkpoints and downstream copies. Procedure: delete, `REORG ... APPLY (PURGE)` where deletion vectors are on, then VACUUM. Test that `RESTORE` to a pre-erasure version fails or returns no erased rows. |
| H4 | Accept (verified) | 8.2 and C.5 say `addNewColumns` "turns a renamed column into silent nulls". | Rewrite: the stream fails with `UnknownFieldException`, adds the new column, and on restart the old column is null for new rows. The default is `none` when a schema is supplied. The per-tier policy (Story 2.7) states the mode explicitly. |
| H5 | Accept | A.5 runs NFR only `when { params.RUN_NFR }`. | `when { expression { params.RUN_NFR \|\| params.RELEASE_CANDIDATE } }`. |
| H6 | Accept, **decision** | 9.1 checks cost "the next day from `system.billing.usage`", inside a 2-hour release-candidate budget. | Recommended: cost is not a blocking gate condition. The gate uses run duration (known at once); cost is checked the next day and an overrun raises an S3 defect. Alternative: the release waits a day for cost. |
| M1 | Partly | 4.9 does contain app testability rows (app configuration, OpenAPI, `data-testid`), so the reference is correct but reads ambiguously. | Story 6.5: "meets the app testability requirements in section 4.9 and the app tests in section 6.5". |
| M2 | Accept | 5.2 includes apps; Story 1.2 says tables only, though Story 1.1 already lists apps. | Story 1.2 covers tier 1 and 2 tables and apps, with app columns API, data correctness, access, UI and load. |
| M3 | Accept, **decision** | 11.1 says DQX and reconciliation "each count as one gate test per pipeline"; 16.4 and Story 4.8 say "one gate test per pipeline". | Recommended: two gate tests per pipeline (one DQX, one reconciliation), so a failure shows which framework failed. Align 11.1, 16.4 and Story 4.8. |
| M4 | Partly / Defer | Already an open question in 19.3, and Story 2.9 is the spike. | Add one line to 6.5: access tests are designed after the authorisation model is settled; API tests need the caller to have `CAN USE` on the app. No other change until Story 2.9 reports. |
| M5 | Partly | Formula confirmed in the decisions log. | Add the counting rules (snapshot at release-candidate build; quarantined and skipped count as not automated; one Xray key = one test) to Story 1.3's AC, not the body, to save pages. |
| M6 | Accept (verified) | A.2 uses a session fixture to run the pipeline once; Jenkins runs `-n auto`. | Group each table's cases on one worker with `@pytest.mark.xdist_group` and `--dist loadgroup`, and give each worker its own data prefix. Note it in 7.2; update A.2 and A.5. |
| M7 | Accept | 4.6 maps changed files through the LoB configuration, but A.6 has no code paths. | Add a `paths:` list per pipeline in A.6; `changed_pipelines.py` matches changed files against it, and falls back to smoke for all tier 1 (as 4.6 already says). |
| M8 | Accept | 9.1 measures API and page times with Locust. | Locust for API load; page times from Playwright on the release candidate. |
| M9 | Partly | The mechanism is already specified (`record_property`, `junit_family = xunit1`), but the property name isn't. Story 4.7 already proves the import. | 16.4: name the property `test_key`. |
| M10 | Accept | A.5 ends both publish steps with `\|\| true`. | Wrap publishing in `catchError(buildResult: 'UNSTABLE')`; keep `\|\| true` only on cleanup. Gate rule 1 already treats missing results as no-go; say so. |
| M11 | Accept | Epics 4 and 5 are sized M but contain three M stories each. | Resize Epics 4 and 5 to L. Add to the Sizes line: "Epic sizes are estimates, not sums of story sizes." |
| M12 | Accept | Merged with H1. | Required stages per trigger in the 11.4 contract: merge (smoke, targeted), nightly (full), release candidate (full, NFR, app load, UAT). The gate selects results by candidate commit. |
| M13 | Accept | 14.4 covers Jira, Allure, Xray and the message column only. | Extend the rule to Jenkins logs, Playwright traces and screenshots, and API error payloads. |
| M14 | Accept | Story 4.4: "for every commit Jenkins tests". | "for every commit that a QA Jenkins build tests, the result of the developer build for that commit". |
| M15 | Accept | UAT first used in 4.1, NFR in section 1, LoB in 4.6, priority and severity in 15.1 before 16.1. | Expand each at first use; in 15.1 add "(priority and severity scales in section 16.1)". |
| M16 | Partly | The doc does have a Backlog tab; the reviewer saw only the PDF. | Section 20: "in the companion *CDP QA Backlog* (the Backlog tab of the doc; `CDP_QA_Backlog.pdf` in the repo)". |
| M17 | Accept | Section 3 says 80% of tier 1 tables; 4.5 says every test type for tier 1 is automated. | Section 3: "share of tier 1 tables whose applicable QA test types are all automated and gated", distinct from the 80–85% Xray ratio. |

## Page budget

Sections 1–19 are at exactly 20 pages. The additions (gate contract, erasure, H4, M13, M15) add roughly half a page. Recover it with the reviewer's own suggestions:

- remove the automation-ratio repeat between sections 3 and 11.1;
- remove ownership repeats between sections 5 and 17;
- merge the classification overlap between 4.4 and 5.1.

Do not move the failure catalogue, RACI or code back into the body.
