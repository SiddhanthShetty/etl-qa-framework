# Prompt 03: Refine the spec and carry the work forward

Each block below is a separate prompt. Attach the current `spec/CDP_QA_Testing_Framework_Specification.md`, and `context/decisions-log.md` if the session is new. Run them in roughly this order.

---

## 3.1 Apply review findings

> Here is the current spec and a review of it. Apply every finding that doesn't need information from me. For each finding that does, add it to the open questions in section 19.3 instead. Keep section numbering and cross-references consistent, and use one name for each concept throughout (for example "unified results view"). When you're done, list what you changed, by section, and what you left open.

## 3.2 Replace illustrative values with real ones (after Epic 1)

> Epic 1 is done. Here are the real values:
> - tier 1 and tier 2 tables and pipelines: [list]
> - the pilot pipeline and its line of business: [name]
> - agreed SLAs (run time, freshness, recovery, cost): [values]
> - baselines for the nine measures in section 3: [values]
> - Xray edition (Cloud or Server/DC): [answer]
> - DQX and reconciliation log table columns: [columns]
> - the schema-evolution and bad-record policy per tier: [policy]
> - the data protection decision on masked data: [decision]
>
> Update the spec everywhere these appear. Remove "illustrative" and "proposed" labels only where a real value now stands. Close the matching open questions.

## 3.3 Incorporate reviewer comments

> Here are comments from Kedar, Gagan and Sukesh: [paste]. For each comment, either change the spec or explain in one line why not. Show the changes as a table of section, change and reviewer. Don't change anything that no comment asked for.

## 3.4 Bring the main body to 15 to 20 pages

> The main body (sections 1 to 19) must fit in 15 to 20 A4 pages. Cut duplication first, then detail that belongs in the appendix or the backlog. Do not remove any decision, requirement or open question. Report the word count before and after.

## 3.5 Turn the backlog into Jira and Xray items

> Convert the epics and stories in section 20 into a CSV for Jira import, with these columns: Issue Type, Summary, Description (user story plus acceptance criteria as a bullet list), Epic Link, Story Points (S=3, M=5, L=8), Labels (cdp-qa, epic-N), Priority. Then list the Xray Test issues to create for the Epic 3 pilot, one per scenario in sections 6.2, 7.2, 8.2 and 10.3, each linked to its story.

## 3.6 Build the pilot test repository

> Using Appendix A and sections 4.9, 11 and 18, scaffold the `cdp-qa-tests` repository:
> - `conftest.py` fixtures (`tables`, `landing`, `jobs`, build schemas, the Xray key hook);
> - `pytest.ini` with registered markers and `junit_family = xunit1`;
> - the build environment and cleanup tools;
> - `publish_results.py`, which writes the QA results and run correlation tables;
> - the Jenkinsfile;
> - one working example per test type for the pilot pipeline;
> - API, access and Playwright examples for the pilot Databricks App (FastAPI and React), as in section 6.5 and Appendix A.7.
>
> Use the Databricks SDK for Python and Databricks Connect. Mark every place that needs a real workspace value with `TODO(cdp):`.

## 3.7 Write the unified results view

> Write the Databricks SQL for:
> - `cdp_qa.results.qa_results` and `cdp_qa.results.run_correlation` as Delta tables;
> - the view `cdp_qa.results.test_results_v`, which unions the DQX log, the reconciliation log and the QA results table into the schema in section 11.4 and joins through `run_correlation`.
>
> Here are the DQX and reconciliation log schemas: [paste]. Then write the release gate query from section 15.2.

## 3.8 Prepare the Product team prioritisation session

> From section 20, prepare a one-page brief for the Product team. Cover: what each epic delivers, its size, what it depends on, the risk of deferring it, and the questions they need to decide. Recommend an order, and show where Epics 1 to 3 are fixed and Epics 4 to 8 are open.
