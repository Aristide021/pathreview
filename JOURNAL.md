# JOURNAL

## Week 7 - Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/148

**Issue title:** Skill extractor fails to detect JavaScript and TypeScript

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Selection reasoning:**
This is my first time contributing to a codebase this size, so I deliberately started at
Tier 1 rather than reaching for something bigger. Against the "is this right for me?"
checklist: the issue is reproducible in two lines (the report gives exact input/output
pairs), the blast radius is contained to a single function (`_detect_languages` in
`ingestion/parsers/skill_extractor.py`) with no cross-module or DB/API side effects, and the
four failing unit tests named in the issue already define what "done" looks like, so I don't
have to guess at acceptance criteria. It's also labeled "good first issue," which lines up
with wanting a bounded, well-specified first PR rather than something open-ended.

**Problem summary:**
`SkillExtractor.extract_skills()` in `ingestion/parsers/skill_extractor.py` is supposed to
detect programming languages from resume/repo text, but its JavaScript/TypeScript detection
only fires when a filename with a `.js`/`.ts` extension is explicitly passed in, or when the
text literally contains the word "import" or "require". It ignores the `JS_TS_KEYWORDS` set
already defined on the class (`const`, `let`, `function`, `async`, `await`, etc.) and never
scans the text body for TypeScript signals such as `.ts`/`.tsx` mentions or the word
"TypeScript" the way Python detection checks multiple signals (imports, `def`, type
annotations). As a result, text describing JS/TS work with arrow functions, async/await, or
in-body `.tsx`/`.ts` file mentions returns no language detection at all, or in the TypeScript
case gets misclassified as only "React" (via the unrelated `REACT_INDICATORS` substring
match). A successful fix broadens `_detect_languages` to use the existing keyword set and
add TypeScript-specific signals, making the four currently-failing tests in
`tests/unit/test_skill_extractor.py` (`test_javascript_detection`,
`test_text_with_typescript_files`, `test_devops_tool_detection`,
`test_docker_compose_detection`) pass.

**Branch name:** fix/148-detect-javascript-typescript

**Setup confirmation:** [x] App runs locally at localhost:5173

**Setup note:** On a fresh clone, `make setup` failed at `alembic upgrade head` because it
assumed `.env` already existed and that the Postgres/Redis Docker containers were already
running. Fixed by having the `setup` target copy `.env.example` to `.env` if missing and run
`docker compose up -d --wait db redis` before the migration step (see the first commit on
this branch).

**Cohort ledger:** [x] Issue added to cohort ledger
