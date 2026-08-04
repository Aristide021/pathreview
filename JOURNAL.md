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

## Week 8 - Reproduction & solution planning

**Reproduction commit link:** https://github.com/Aristide021/pathreview/commit/dd1b0ba565f41ba882d47f8baddd19f0391f3b8e

**Reproduction summary:**
I reproduced the bug two ways in my local venv. Running the snippet from the issue against
`SkillExtractor.extract_skills()` returned `[]` for JavaScript text and `['React']` for
TypeScript text, matching the reported behavior exactly. Running
`pytest tests/unit/test_skill_extractor.py` gave 5 failures out of 18, including all four
tests named in the issue.

### How to reproduce

```bash
.venv/bin/python -c "
from ingestion.parsers.skill_extractor import SkillExtractor
e = SkillExtractor()
print('JS  :', [d.name for d in e.extract_skills('Wrote index.js using const arrow functions and async/await callbacks')])
print('TS  :', [d.name for d in e.extract_skills('Built app.tsx and types.ts with strict TypeScript interfaces')])
"
```

Observed output:

```
JS  : []
TS  : ['React']
```

Expected: `['JavaScript']` and a list containing `TypeScript`.

```bash
.venv/bin/python -m pytest tests/unit/test_skill_extractor.py -v
```

Observed: `5 failed, 13 passed`.

```
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_text_with_typescript_files
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_database_technology_detection
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_devops_tool_detection
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_javascript_detection
FAILED tests/unit/test_skill_extractor.py::TestSkillExtractor::test_docker_compose_detection
```

### What I found while tracing it

All four in-scope failures come from `_detect_languages` and `_detect_tools` in
`ingestion/parsers/skill_extractor.py`:

- The `JS_TS_KEYWORDS` set on line 30 is defined but never referenced anywhere in the repo.
  JS detection never looks at `const`, `let`, `function`, or `=>`.
- `re.search(r"\b(import|require)\s+", text)` on line 179 requires whitespace after the
  keyword, so `require('fs')` does not match.
- TypeScript is only ever selected by filename (line 186). Text-only TS input can never
  produce a TypeScript detection, and line 186 makes JS and TS mutually exclusive.
- The TS test text gets labeled Python, because the annotation regex on line 160,
  `:\s*(int|str|float|bool|list|dict)`, matches `: string` (`str` is a prefix of `string`).
- `_detect_tools` substring-matches the literal word "docker". Dockerfile content
  (`FROM`, `RUN`, `EXPOSE`) and compose YAML (`services:`, `ports:`) never contain it.

The fifth failure, `test_database_technology_detection`, is a separate pre-existing bug in
the test itself: line 138 reads `skill_names = [s.name for s in skill_names]`, which raises
`UnboundLocalError`. It is unrelated to #148 and I plan to report it as its own issue rather
than widen this PR.

**PLAN.md link:** https://github.com/Aristide021/pathreview/blob/fix/148-detect-javascript-typescript/PLAN.md

**Walkthrough video (recommended):** not recorded

**Blockers or open questions:**
Two judgment calls I want feedback on. First, whether TypeScript input should also report
JavaScript, since TS is a superset. The test only asserts TypeScript is present, so either
reading passes. Second, whether to fix the unrelated broken test at line 138 in this PR or
file it separately. I lean toward filing it separately to keep the diff scoped to #148.

## Week 9 - Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
All six sub-tasks from `PLAN.md` are implemented and committed in
`fix(ingestion): detect JavaScript, TypeScript, and Dockerfiles in skill extractor`.
JavaScript now detects from strong signals (`require()`, `module.exports`, arrow functions,
ES6 import-from, `.js` references) or any two weak ones (`const`, `let`, `var`, `function`,
`export`). TypeScript detects from body text with no filename. The Python annotation regex
gained a `\b` so `: string` no longer registers as Python. Docker detects Dockerfile
directives and compose keys structurally. All four tests named in the issue pass, and I added
7 regression tests.

**Next steps:**
Request peer review on a draft PR, then open the PR against upstream with the pre-existing
failure counts documented.

**Blockers:**
None blocking. One thing to flag in review: the pre-commit `mypy` hook cannot pass on
`tests/unit/test_skill_extractor.py` because of 21 pre-existing errors, one of which comes
from the broken `test_database_technology_detection`. I committed with `--no-verify` and said
so in the commit message rather than widening scope to fix unrelated tests.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/799

**Branch:** `fix/148-detect-javascript-typescript`

**What you built:**
A rewrite of JavaScript and TypeScript detection in
`ingestion/parsers/skill_extractor.py`, replacing a filename-only check and a regex that
required trailing whitespace with pattern sets matched against the text body. TypeScript is
now detectable without a filename and no longer misreported as Python. Docker detection
recognizes Dockerfile directives and compose file structure, neither of which contains the
literal word "docker".

**Tests added or updated:**
`tests/unit/test_skill_extractor.py`. Added 7 regression tests covering the false positives
the existing tests do not exercise: TypeScript detected without a filename, TypeScript not
reported as Python, `require('fs')` with no trailing space, English prose containing
"constant"/"classic" not registering as JavaScript, Python imports not registering as
JavaScript, Dockerfile directives detected without the word "docker", and Docker not
duplicated when both named and structural evidence appear.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes

Both commands fail on this repo before any of my changes, so "passes" means no new failures,
per the Week 9 guidance on pre-existing failures:

| Command | Before | After |
|---|---|---|
| `make check` (ruff) | 182 errors | 179 errors |
| `make test-unit` | 53 failed, 375 passed | 49 failed, 386 passed |

Newly failing tests: none. Newly passing: the four named in the issue. The ruff count drops
because the repo's own pre-commit `ruff --fix` hook reformatted the files I touched.

**Draft PR feedback received from:** none yet
