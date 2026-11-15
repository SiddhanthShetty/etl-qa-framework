# CDP QA Testing Framework

The QA testing framework specification for CDP, our Databricks data product, together with the prompts and context needed to recreate and refine it with an AI assistant.

**Status:** draft for review by Kedar, Gagan and Sukesh, followed by Product team prioritisation of the backlog (section 20).

## Contents

```text
cdp-qa-framework/
├── spec/
│   ├── CDP_QA_Testing_Framework_Specification.pdf    # read-only copy for reviewers
│   ├── CDP_QA_Testing_Framework_Specification.docx   # editable Word copy
│   ├── CDP_QA_Testing_Framework_Specification.md     # plain-text source, best for diffs and AI prompts
│   └── CDP_QA_Backlog.pdf                            # full user stories (the doc's Backlog tab)
├── prompts/
│   ├── 01-recreate-spec.md        # rebuild the whole spec from scratch
│   ├── 02-independent-review.md   # a cold review by a separate reviewer
│   └── 03-refine-and-continue.md  # next steps: apply reviews, real values, Jira, test repo, SQL
└── context/
    ├── whiteboard-notes.md        # transcription of the planning whiteboard
    ├── decisions-log.md           # what was decided, when and why
    └── whiteboard/                # original whiteboard photos
```

## How to use the prompts

1. **To recreate the spec**, open a new AI session and paste `prompts/01-recreate-spec.md`. Attach the whiteboard photos if you have them.
2. **To review it**, use `prompts/02-independent-review.md` in a session that did not write the spec, so the review is independent.
3. **To move the work forward**, pick the matching prompt from `prompts/03-refine-and-continue.md` and attach the current Markdown spec. Attach `context/decisions-log.md` too if the session is new.

After any change, update the Markdown spec first, regenerate the PDF and DOCX from it, and add a row to `context/decisions-log.md`.

## Notes

- Table names, SLAs and targets in the spec are illustrative until Epic 1 measures the real baseline.
- Sections 1 to 19 fill exactly 20 A4 pages (down from about 39). The spec PDF is 30 pages including the section 20 epic summary and Appendices A to C. The full user stories are in `spec/CDP_QA_Backlog.pdf` (8 pages).
- This repository contains internal project content, including reviewer names and whiteboard photos. Keep it in a private repository.

## Push to GitHub

```bash
cd cdp-qa-framework
git init -b main
git add .
git commit -m "Add CDP QA testing framework spec, prompts and context"
git remote add origin git@github.com:<your-org>/cdp-qa-framework.git
git push -u origin main
```
