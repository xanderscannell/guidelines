# Project Guidelines Plugin

An agent plugin that generates and enforces project coding guidelines. Creates a
`PROJECT_GUIDELINES.md` with severity-tagged conventions, then reviews your codebase
against it.

Compatible with Claude Code, GitHub Copilot in VS Code, and Copilot CLI.

## Installation

**Claude Code CLI:**
```bash
claude --plugin-dir /path/to/Guidelines
```

**VS Code:**
Open the Command Palette and run **Chat: Install Plugin From Source**, then point to
this directory or its Git repository URL.

## Skills

### `/guidelines-init`

Generates a `PROJECT_GUIDELINES.md` file — the single source of truth for your
project's coding standards.

**Workflow:**

1. **Auto-detect** — Scans your project for signal files (`package.json`, `tsconfig.json`,
   `pyproject.toml`, `go.mod`, `Cargo.toml`, CI configs, linter configs, etc.) and
   detects patterns like naming conventions, test file organization, import style, and
   directory structure using Glob and Grep.

2. **Interview** — Presents findings and asks targeted questions to fill gaps across
   8 categories: Code Style, Architecture, Error Handling, Testing, Git Workflow,
   Documentation, Security, and Performance. Skips categories already fully covered
   by auto-detection or irrelevant to the project.

3. **Generate** — Writes `PROJECT_GUIDELINES.md` with severity-tagged sections.
   Every section includes a `<!-- severity: error|warning|info -->` comment that
   controls how strictly `/guidelines-review` enforces it.

4. **Confirm** — Shows a summary table of categories, rule counts, and severity
   levels. Lets you adjust before committing.

**Default severity mapping:**

| Category | Severity |
|---|---|
| Security, Error Handling, Architecture | `error` (must fix) |
| Code Style, Testing, Git Workflow, Documentation | `warning` (should fix) |
| Performance | `info` (nice to have) |

**Edge cases handled:**
- **Monorepo** — Detects `packages/`, `apps/`, `lerna.json`, `pnpm-workspace.yaml`,
  or `turbo.json` and asks whether guidelines should be global or per-package.
- **Existing guidelines** — Asks whether to merge new detections or replace entirely.
- **Empty project** — Generates a starter template with TODO markers for customization.

### `/guidelines-review [path|--changed|--staged|--pr]`

Reviews your codebase against `PROJECT_GUIDELINES.md` and produces an actionable
compliance report.

**Scope options:**

| Argument | What gets reviewed |
|---|---|
| *(empty)* | All source files in the project |
| `src/api/` | All files in that directory (recursive) |
| `src/api/handler.ts` | Just that file |
| `--changed` | Files changed since the last commit |
| `--staged` | Git-staged files only |
| `--pr` | Files changed in the current branch vs main/master |

**Review process:**

1. **Load guidelines** — Reads `PROJECT_GUIDELINES.md` (checks root, `docs/`, and
   `.github/`). Parses severity comments to build a severity map. Missing severity
   comments default to `warning`.

2. **Determine scope** — Collects files based on the argument. Excludes `node_modules`,
   `.git`, `dist`, `build`, `vendor`, `__pycache__`, lockfiles, minified files, and
   source maps.

3. **Review** — For 50 files or fewer, reviews inline. For 51+, delegates to the
   `reviewer` subagent in parallel batches of ~20 files each.

4. **Report** — Produces a structured report with a summary table and findings grouped
   by severity (violations, warnings, suggestions), each citing the specific guideline,
   file, line number, and a concrete fix.

5. **Next steps** — Offers to fix violations automatically, update guidelines with
   newly discovered patterns, or save the report to `docs/reviews/`.

**Review heuristics include:**
- Naming and import pattern checks
- Module boundary and directory structure validation
- Bare `catch`/`except` detection, unhandled promise checks
- Test file existence and anti-pattern detection
- Hardcoded secret scanning (`sk-`, `ghp_`, API keys, etc.)
- N+1 query pattern detection, missing pagination checks
- Tooling-enforced rule detection (skips rules already covered by Prettier, ESLint,
  Black, gofmt, etc. running in CI)

## Subagent

### `reviewer`

Internal subagent used by `/guidelines-review` for parallel processing of large
codebases. Not invoked directly by users.

- Receives a batch of ~20 files, the guidelines text, a severity map, and which
  categories to focus on
- Returns structured JSON findings (file, line, severity, issue, suggestion,
  guideline text)
- Read-only — never modifies files
- Uses the severity from the guidelines, not its own judgment (ambiguous cases
  get `info` severity with a question)

## The Severity System

The severity HTML comment is the contract between init and review:

```markdown
## Security
<!-- severity: error -->
- Never commit secrets or API keys...
```

- `error` — **Violation.** Must fix. Directly contradicts a stated guideline.
- `warning` — **Warning.** Should fix. Inconsistent with convention but not explicitly
  forbidden. Also the default when no severity comment is present.
- `info` — **Suggestion.** Nice to have. Opportunity to better align with guidelines.

Severities are set by `/guidelines-init` based on category defaults, adjusted by the
user during the confirm step, and can be edited at any time by changing the HTML
comment in `PROJECT_GUIDELINES.md`.

## Reference Material

`references/guideline-categories.md` ships with the plugin and contains per-stack
example rules (JavaScript/TypeScript, Python, Go, Rust) across all 8 categories.
Used by `/guidelines-init` as inspiration during generation — examples are adapted
to the specific project, not copied verbatim.

## Project Structure

```
.claude-plugin/
  plugin.json                    # Plugin manifest
skills/
  guidelines-init/
    SKILL.md                     # /guidelines-init skill
  guidelines-review/
    SKILL.md                     # /guidelines-review skill
agents/
  reviewer.md                   # Parallel review subagent
references/
  guideline-categories.md       # Per-stack example rules
```

## License

MIT
