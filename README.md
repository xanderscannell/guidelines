# Project Guidelines Plugin

An agent plugin that generates and enforces project coding guidelines. Write your
rules, review your code against them.

Compatible with Claude Code, GitHub Copilot in VS Code, and Copilot CLI.

## Quick Start

```
/guidelines-init
```

It asks one question: "What rules matter to you?" Write your rules, get a
`PROJECT_GUIDELINES.md`, and start reviewing:

```
/guidelines-review --pr
```

That's it. The rest of this README covers the details.

## Installation

**Marketplace (recommended)**
```bash
# Add marketplace
/plugin marketplace add xanderscannell/guidelines

# Install plugin
/plugin install project-guidelines
```

**Claude Code CLI:**
```bash
claude --plugin-dir /path/to/Guidelines
```

**VS Code:**

Open the Command Palette (Ctrl+Shift+P) and run:
```bash
Chat: Install Plugin From Source
```
then point to this directory or its Git repository URL.

## Two Modes

### Simple Mode (default)

A flat list of rules in plain language. No categories, no severity levels, no ceremony.
You write rules like:

```markdown
## Rules

- Make sure everything has test cases
- Don't add any more code to `src/legacy/handler.ts`, refactor elsewhere
- Architectural logic belongs in `src/core/`, not in route handlers
- All API responses must use the `ApiResponse` wrapper
```

The review checks your code against these rules and tells you what's broken. A rule is
a rule — if it's in the file, it matters.

### Full Mode

Categorized guidelines with severity levels (`error` / `warning` / `info`), structured
interviews, per-category review heuristics, and detailed reports with summary tables.
Use this when you want comprehensive, team-wide coding standards.

```
/guidelines-init --full
```

See [Full Mode Details](#full-mode-details) below.

### Upgrading

Start simple, go full when you need it:

```
/guidelines-init --upgrade
```

This reads your existing simple rules, maps them into categories, adds severity tags,
and runs the full auto-detection and interview flow. Your original rules are preserved.

---

## Skills

### `/guidelines-init`

Generates a `PROJECT_GUIDELINES.md` file.

| Argument | Mode |
|---|---|
| *(empty)* | Simple — quick scan, one question, flat rule list |
| `--simple` | Same as empty |
| `--full` | Full — comprehensive scan, 8-category interview, severity tags |
| `--upgrade` | Converts existing simple file to full mode |

**Simple mode workflow:**
1. Quick scan to identify tech stack
2. "What rules matter to you?"
3. Generates `PROJECT_GUIDELINES.md` with a flat `## Rules` list

**Full mode workflow:**
1. Auto-detect conventions from 13+ signal files
2. Structured interview across 8 categories (skips what's already detected)
3. Generate categorized guidelines with severity tags
4. Confirm — summary table, adjust severities, then commit

**Edge cases handled:**
- **Monorepo** — Detects workspaces and asks about global vs per-package rules
- **Existing guidelines** — Asks to merge or replace; warns about mode mismatches
- **Empty project** — Generates a starter template with TODO markers

### `/guidelines-review [scope]`

Reviews code against your guidelines. Auto-detects simple or full mode from the file.

| Argument | What gets reviewed |
|---|---|
| *(empty)* | All source files in the project |
| `src/api/` | All files in that directory (recursive) |
| `src/api/handler.ts` | Just that file |
| `--changed` | Files changed since the last commit |
| `--staged` | Git-staged files only |
| `--pr` | Files changed in the current branch vs main/master |

**Simple mode report:**
```markdown
# Guidelines Review
**Scope:** --pr | **Files scanned:** 8

## Issues Found

### Don't add any more code to `src/legacy/handler.ts`
- `src/legacy/handler.ts:142` — New function added — Move to `src/orders/`

### All API responses must use the `ApiResponse` wrapper
- `src/api/users.ts:38` — Returns raw object — Wrap with `ApiResponse.success(data)`

## Clean
All other rules passed.
```

**Full mode report** includes a summary table with pass/warn/fail counts per category,
findings grouped by severity (violations, warnings, suggestions), and recommendations
for new guidelines.

**After the report**, both modes offer:
- Fix issues automatically
- Add new rules / update guidelines
- Save report to `docs/reviews/`

---

## Subagent

### `reviewer`

Internal subagent used by `/guidelines-review` for parallel processing of large
codebases (51+ files). Not invoked directly by users.

- Receives file batches of ~20 files
- **Simple mode:** receives rules list, returns findings per rule
- **Full mode:** receives guidelines + severity map + category focus, returns
  categorized findings with severity levels
- Read-only — never modifies files
- Returns structured JSON for the parent skill to merge into the report

---

## Full Mode Details

### The Severity System

In full mode, every section of `PROJECT_GUIDELINES.md` has a severity tag:

```markdown
## Security
<!-- severity: error -->
- Never commit secrets or API keys...
```

| Severity | Meaning | Review behavior |
|---|---|---|
| `error` | Violation — must fix | Reported as a hard failure |
| `warning` | Should fix | Flagged but not critical |
| `info` | Nice to have | Suggestion only |

Default when no tag is present: `warning`.

**Default severity mapping set by init:**

| Category | Default |
|---|---|
| Security, Error Handling, Architecture | `error` |
| Code Style, Testing, Git Workflow, Documentation | `warning` |
| Performance | `info` |

Severities can be adjusted during init's confirm step or by editing the HTML comment
in the file at any time.

### Full Mode Review Heuristics

In full mode, the review applies targeted checks per category:

- **Code Style** — Naming patterns, import ordering, formatter detection
- **Architecture** — Directory structure, module boundary crossings, circular deps
- **Error Handling** — Bare catch/except, unhandled promises, missing logging
- **Testing** — Test file existence, naming conventions, anti-patterns
- **Git Workflow** — Commit message format, branch naming
- **Documentation** — Missing docstrings/JSDoc, README completeness
- **Security** — Hardcoded secrets, insecure patterns (`eval`, `innerHTML`, SQL concat)
- **Performance** — N+1 queries, missing pagination, sync I/O in async contexts

Rules already enforced by tooling (Prettier, ESLint, Black, etc.) running in CI are
marked as "Enforced by tooling" and skipped.

### Reference Material

`references/guideline-categories.md` ships with the plugin and contains per-stack
example rules (JavaScript/TypeScript, Python, Go, Rust) across all 8 categories.
Used by full-mode init as inspiration — adapted to the project, not copied verbatim.
Not used in simple mode.

## Project Structure

```
.claude-plugin/
  plugin.json                    # Plugin manifest
skills/
  guidelines-init/
    SKILL.md                     # /guidelines-init (simple + full + upgrade)
  guidelines-review/
    SKILL.md                     # /guidelines-review (auto-detects mode)
agents/
  reviewer.md                   # Parallel review subagent (both modes)
references/
  guideline-categories.md       # Per-stack example rules (full mode only)
```

## License

MIT
