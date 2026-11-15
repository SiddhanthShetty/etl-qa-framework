# Prompt 02: Independent review of the specification

Use this prompt in a fresh session, or with a separate reviewer agent that has not seen how the spec was written. Attach `spec/CDP_QA_Testing_Framework_Specification.md`, or the PDF.

---

You are a senior data QA and test architect who has run testing for lakehouse data products on Databricks. Review the attached CDP QA Testing Framework Specification cold.

Context the author worked to (treat these as requirements):

- CDP is a Databricks data product: Unity Catalog, jobs, Delta, Auto Loader, and a bronze, silver and gold layout. It also has Databricks Apps with a FastAPI backend and a React frontend.
- Existing tools:
  - a data quality framework built on DQX and a reconciliation framework, each with its own Delta log table;
  - Jenkins CI;
  - developer-owned unit tests;
  - pytest with Allure;
  - Jira with Xray.
  
  ReportPortal was trialled and not adopted.
- Whiteboard requirements:
  - progression and regression, each black-box and white-box;
  - functional, integration, data, boundary, failure, transformation and NFR tests;
  - 80 to 85% automation;
  - spec-driven development;
  - a generic CDP layer plus LoB context;
  - the V-model;
  - a 15 to 20 page main body ending in epics, stories and tasks.
- Table names, SLAs and targets are intentionally illustrative.

Deliver, in plain text and ranked most important first:

1. **Verdict:** two or three sentences on whether the spec is ready to send to reviewers, and what would improve it most.
2. **Findings:** for each one, give a severity (High, Medium or Low), the section, the problem stated concretely, why it matters, and the specific fix. Cover at least:
   - internal contradictions and stale cross-references;
   - technical accuracy of the Databricks, Delta, Auto Loader, Unity Catalog, DQX, Jenkins, pytest, Allure and Xray content, including the code;
   - gaps against the requirements;
   - whether the automation ratio is measurable;
   - whether the time budgets and test isolation are feasible;
   - whether the Databricks Apps approach (API, access, Playwright and load tests, and unattended sign-in) is realistic;
   - data privacy (GDPR);
   - anything a reviewer would push back on;
   - length and redundancy.
3. **Strengths:** what is genuinely strong and should not change.

Be specific and honest, and don't pad. Do not edit the document.
