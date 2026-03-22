---
name: guidelines-review
description: >
  Reviews code against PROJECT_GUIDELINES.md and produces an actionable compliance
  report. Use this skill when the user asks to review code against guidelines, check
  code standards compliance, audit the project, run a style check, or says things like
  "review against guidelines", "check my code", "are we following our standards",
  "audit this file", or "guidelines check". Also triggers when the user asks for a
  code review in a project that has a PROJECT_GUIDELINES.md file.
---

# Guidelines Review

Review code against the project's `PROJECT_GUIDELINES.md` and produce a compliance
report. Supports two modes (simple and full), auto-detected from the guidelines file.

## Arguments

`$ARGUMENTS` can be:

| Argument | Files to review |
|---|---|
| *(empty)* | All source files in the project |
| File path | Just that file |
| Directory path | All source files in that directory (recursive) |
| `--changed` | Files changed since the last commit (`git diff --name-only HEAD`) |
| `--staged` | Git-staged files only (`git diff --cached --name-only`) |
| `--pr` | Files changed in current branch vs main (`git diff --name-only main...HEAD`; falls back to `master` if `main` doesn't exist) |

---

## Step 1 — Load guidelines and detect mode

1. Look for `PROJECT_GUIDELINES.md` in the repo root.
2. If not found, check `docs/PROJECT_GUIDELINES.md` and `.github/PROJECT_GUIDELINES.md`.
3. If still not found, tell the user:
   > No `PROJECT_GUIDELINES.md` found. Run `/guidelines-init` first to set up your
   > project guidelines, or point me to your guidelines file.
   Then stop.
4. **Detect mode** by reading the first 10 lines of the file:
   - If `<!-- mode: simple -->` is found → **simple mode**.
   - If `<!-- mode: full -->` is found → **full mode**.
   - If no mode marker is found: check the entire file for any `<!-- severity: ... -->`
     tags. If found → full mode. If not found → simple mode.
5. Branch to the appropriate review workflow below.

---

## Step 2 — Determine scope (shared across both modes)

Based on `$ARGUMENTS`, collect the files to review.

For `--changed`, `--staged`, and `--pr`, use Bash to run the appropriate git command.
Normalize all file paths to forward slashes.

For full-project or directory scopes, use Glob to collect source files.

**Always exclude:** `node_modules`, `.git`, `dist`, `build`, `vendor`, `__pycache__`,
`.next`, `coverage`, `*.min.*`, `*.lock`, `*.map`, generated code, binary files,
lockfiles, and images.

---

## Simple Mode Review

For guidelines files with `<!-- mode: simple -->` or no severity tags.

### Parse rules

Read the `## Rules` section. Each bullet is a rule. No severity parsing needed.
Every rule has equal weight — a rule is a rule.

### Review

Read each file in scope and check it against the flat rule list.

- Do NOT apply per-category heuristics (no grep patterns for security, N+1, etc.).
- Do NOT assign severity levels. Every finding is simply a rule that was broken.
- Just read the code and the rules, and use judgment.
- For 50 files or fewer, review inline.
- For 51+ files, delegate to the `reviewer` subagent with simple-mode input
  (rules list + file batch). Split into batches of ~20 files.

### Report

Produce a clean, direct report:

```markdown
# Guidelines Review

**Scope:** [what was reviewed]
**Files scanned:** [count]

## Issues Found

### [Rule text]
- `src/api/handler.ts:42` — [what is wrong] — [how to fix]
- `src/api/router.ts:18` — [what is wrong] — [how to fix]

### [Rule text]
- `src/utils/db.ts:7` — [what is wrong] — [how to fix]

## Clean
[List of rules with no violations, or "All other rules passed."]
```

No summary table. No severity grouping. No pass/warn/fail columns. Just issues
grouped by rule, with file:line and a fix for each.

If no issues are found, say so clearly:

```markdown
# Guidelines Review

**Scope:** [what was reviewed]
**Files scanned:** [count]

All files pass all rules.
```

### Next steps

After presenting the report, offer:

1. **Fix issues:** "Want me to fix these automatically?"
2. **Add rules:** "Want to add any new rules based on what I found?"
3. **Save report:** "Want me to save this report to `docs/reviews/YYYY-MM-DD.md`?"

---

## Full Mode Review

For guidelines files with `<!-- mode: full -->` or `<!-- severity: ... -->` tags.

### Parse guidelines

Parse each section heading and look for a severity comment on the following line.
The format is `<!-- severity: (error|warning|info) -->`.

- Normalize whitespace when parsing — accept both `<!-- severity:error -->`
  and `<!-- severity: error -->`.
- If a section has no severity comment, default to `warning`.
- Build a severity map: `{ "Code Style > Naming": "warning", "Security": "error", ... }`

### Review

**For scopes of 50 files or fewer:** Review files directly (inline).

**For scopes of 51+ files:** Delegate to the `reviewer` subagent for parallel processing:
- Split files into batches of ~20 files each.
- Pass each batch: the full guidelines text, the parsed severity map, the list of
  file paths, and which guideline categories to focus on.
- Collect JSON results from all subagent instances.
- Merge findings into a single list.

**Per-category review heuristics:**

When reviewing (inline or via subagent), check these patterns per category:

**Code Style (Naming, Formatting, Imports)**
- Grep for naming pattern violations against the documented convention
- Check a sample of files for import ordering consistency
- If a formatter (Prettier, Black, gofmt) is configured AND running in CI,
  mark formatting as "Enforced by tooling" and skip detailed checks

**Architecture**
- Check directory structure against documented patterns
- Look for imports that cross documented module boundaries
- Check for circular dependencies if guidelines mention them

**Error Handling**
- Grep for bare `catch {}`, `catch (e) {}` without handling, `except: pass`
- Look for `console.log` / `print` used instead of proper logging framework
- Check for unhandled promises (missing `await` or `.catch()`)

**Testing**
- Check test file existence for source files that guidelines say must be tested
- Verify test naming conventions match guidelines
- Look for test anti-patterns: no assertions, commented-out tests, empty test bodies

**Git Workflow**
- Check recent commit messages against documented format
- Check current branch name against naming convention

**Documentation**
- Check for missing docstrings/JSDoc on public functions/classes if guidelines require them
- Verify README exists and has required sections

**Security**
- Grep for hardcoded secret patterns (API keys, passwords, tokens, `sk-`, `ghp_`, etc.)
- Check for `.env` files that should be gitignored
- Look for known insecure patterns (`eval()`, `innerHTML`, SQL string concatenation,
  `Function()`, `child_process.exec` with string interpolation)

**Performance**
- Look for N+1 query patterns (queries inside loops)
- Check for missing pagination in list endpoints
- Look for synchronous I/O in async contexts

### Report

Produce a structured report:

```markdown
# Guidelines Review Report

**Scope:** [what was reviewed]
**Date:** [today]
**Guidelines version:** [date from PROJECT_GUIDELINES.md header]
**Files scanned:** [count]

## Summary

| Category | Pass | Warn | Fail |
|---|---|---|---|
| Code Style | 12 | 3 | 1 |
| Architecture | 5 | 0 | 0 |
| ... | ... | ... | ... |

## Violations (must fix)

### [Category]: [Specific guideline]
**Severity:** error
**Files:** `src/api/handler.ts:42`, `src/api/router.ts:18`
**Guideline:** [quote the specific rule from PROJECT_GUIDELINES.md]
**Issue:** [what's wrong]
**Fix:** [concrete suggestion with corrected code]

---

## Warnings (should fix)

[same format as violations]

---

## Suggestions (nice to have)

[same format, for severity: info findings]

---

## Compliant

[brief list of categories and rules that passed cleanly]

## Recommendations

[patterns noticed that aren't covered by current guidelines — suggest additions
to PROJECT_GUIDELINES.md]
```

### Next steps

After presenting the report, offer:

1. **Fix violations:** "Want me to fix the violations automatically?"
2. **Update guidelines:** "I noticed some patterns not covered by your guidelines.
   Want me to add them to PROJECT_GUIDELINES.md?"
3. **Save report:** "Want me to save this report to `docs/reviews/YYYY-MM-DD.md`?"

---

## Review Principles (both modes)

- **Be specific:** Always cite `file:line`, quote the rule, and suggest a fix.
  Never say "some files have issues" without naming them.
- **No invented rules:** Every finding must reference a specific rule or section of
  PROJECT_GUIDELINES.md. Do not apply personal preferences or external standards
  that aren't in the guidelines.
- **Be proportional:** For a single-file review, be thorough. For a full-project
  review, focus on patterns and representative examples rather than listing every
  instance of a repeated violation. Group repeated violations with a count.
- **Acknowledge compliance:** Don't only flag problems. Call out areas where the code
  follows guidelines cleanly.
- **Respect severity (full mode only):** Don't escalate `info` items to sound like
  errors. Don't downplay `error` items. The guideline author chose the severity
  deliberately.
- **Skip tooling-enforced rules (full mode only):** If a linter or formatter already
  enforces a rule and runs in CI, mark it as "Enforced by tooling" and move on.
  Only mark as enforced if the tool clearly covers the entire category.
