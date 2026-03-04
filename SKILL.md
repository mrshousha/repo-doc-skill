---
name: repo-doc
description: >
  Stack-agnostic repository documentation engine. Scans code, git history, Jira
  tickets (via Atlassian MCP), and cross-service dependencies to produce living,
  structured, navigatable documentation with auto-generated Mermaid architecture
  diagrams. Audits documentation drift when code changes and maintains a questions
  loop to resolve ambiguities. Use this skill whenever someone wants to document a
  repository, generate architecture diagrams from code, audit docs for staleness,
  create onboarding guides, track deprecations, build a docs site from source code,
  or any variation of "document this repo", "generate docs", "what does this service do",
  "create architecture diagrams", "audit my docs", "set up a docs site", or
  "scan this codebase". Also triggers on /repo-doc commands.
tags:
  - backend
---

# repo-doc — Repository Documentation Engine

Generate, maintain, and audit living documentation for any codebase. Works with
any language, any framework, any repo shape (single service, monorepo, library).

## Commands

| Command | What it does |
|---|---|
| `/repo-doc config` | Interactive setup wizard — run first on any new repo |
| `/repo-doc init` | Full scan → generates all docs, diagrams, MkDocs site |
| `/repo-doc audit` | Diff-aware scan → detects documentation drift |
| `/repo-doc answers` | Re-ingests answered questions into docs |
| `/repo-doc serve` | Spins up local MkDocs Material site |
| `/repo-doc deep-scan <target>` | Zooms into a section, file, or question marked `[needs-deeper-scan]` |
| `/repo-doc coverage` | Prints documentation coverage metrics |

All commands also work as natural language (e.g. "run repo-doc audit on this repo").

---

## How to use this skill

Every command starts with **pre-flight checks** (read `references/preflight.md`).
Then route to the relevant subsystem. The reference files below contain the full
procedures — read only the ones you need for the current command.

### Command routing

| Command | Read these references (in order) |
|---|---|
| `config` | `preflight.md` → `config-wizard.md` |
| `init` | `preflight.md` → `serve-and-setup.md` §mkdocs.yml → `scanner.md` → `enrichment.md` → `diagrams.md` → `doc-structure.md` → `specialized-docs.md` → `questions.md` → `scanner.md` §State Persistence (final) |
| `audit` | `preflight.md` → `audit.md` |
| `answers` | `preflight.md` → `questions.md` §Answer Re-ingestion |
| `serve` | `preflight.md` → `serve-and-setup.md` |
| `deep-scan` | `preflight.md` → `deep-scan.md` |
| `coverage` | `preflight.md` → `specialized-docs.md` §Coverage Report |

### Reference file index

| File | Contents | Lines |
|---|---|---|
| `references/preflight.md` | Config check, MCP probe, gh CLI check (Phase 1) | ~85 |
| `references/config-wizard.md` | Full interactive setup wizard, repodoc.yaml schema | ~330 |
| `references/scanner.md` | Backup, skeleton scan, importance scoring, tiered read, state persistence (Phases 3–4, 9) | ~155 |
| `references/enrichment.md` | Git history, PR metadata, Jira enrichment, cross-service discovery (Phase 5) | ~120 |
| `references/diagrams.md` | Mermaid diagram generation (architecture, data flow, sequence, entity) (Phase 6) | ~220 |
| `references/doc-structure.md` | File tree layout, sentinel system, conflict resolution, writing rules (Phase 7) | ~145 |
| `references/specialized-docs.md` | Security, runbook, deprecation, glossary, onboarding, coverage, changelog (Phase 7 cont.) | ~295 |
| `references/questions.md` | Questions queue format, answer re-ingestion, question lifecycle (Phase 8) | ~170 |
| `references/audit.md` | Drift detection, severity classification, audit report format | ~135 |
| `references/deep-scan.md` | Targeted scanning by section, path, or question ID | ~100 |
| `references/serve-and-setup.md` | MkDocs serve, PR template, pre-commit hook, mkdocs.yml generation (Phase 2) | ~270 |

---

## `/repo-doc init` — Master Execution Pipeline

When `init` is called, execute these steps **in this exact order**. Each step
feeds the next. Tell the engineer at the start of each major phase what is happening.

```
Phase 1/9 — Pre-flight checks                    → references/preflight.md
Phase 2/9 — Site scaffolding                      → generate mkdocs.yml + docs/ directory structure NOW
                                                    → references/serve-and-setup.md §mkdocs.yml
Phase 3/9 — Scanning repository structure         → references/scanner.md §Backup + §Skeleton Scan
Phase 4/9 — Scoring and reading files             → references/scanner.md §Importance Scoring + §Tiered File Scan
            State persistence (initial write)     → references/scanner.md §State Persistence
Phase 5/9 — Enrichment (git, PRs, Jira, deps)    → references/enrichment.md
Phase 6/9 — Generating diagrams                   → references/diagrams.md
Phase 7/9 — Writing documentation                 → references/doc-structure.md + references/specialized-docs.md
            (track coverage counts inline as you generate each section)
Phase 8/9 — Questions queue + coverage report     → references/questions.md + references/specialized-docs.md §Coverage
Phase 9/9 — Final state persistence               → references/scanner.md §State Persistence (final)
```

**Why this order matters:** `mkdocs.yml` and the docs/ directory structure are
generated early (Phase 2) because they don't depend on file scanning — they only
need `repodoc.yaml` config. This ensures the site scaffold exists even if later
phases are interrupted. Coverage counts are tracked inline during Phase 7 to
avoid a separate expensive pass.

Report progress to the engineer at each phase. When all phases are complete,
print the full completion banner with setup instructions:

```
[repo-doc] Phase 1/9: Running pre-flight checks...
[repo-doc] Phase 2/9: Setting up docs site scaffold...
[repo-doc] Phase 3/9: Scanning repository structure...
...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Documentation generated in docs/

  To view your docs locally:

    1. Install dependencies (one-time):
       pip install mkdocs mkdocs-material mkdocs-mermaid2-plugin mkdocs-awesome-pages-plugin mkdocs-panzoom-plugin

    2. Start the dev server:
       mkdocs serve

    3. Open http://localhost:8000

  Or run: /repo-doc serve  (does steps 1-3 for you)

  Next steps:
    • N questions in .repodoc/unknowns.md — fill them in and run /repo-doc answers
    • Run /repo-doc audit after code changes to detect documentation drift
    • Run /repo-doc deep-scan <target> to expand [needs-deeper-scan] sections

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

This banner is important — many engineers will see it as their first "what do I do
next" prompt. Always print it, even if some phases were partial.

---

## Shared Data Contracts

These three files are the shared state across all commands. Every reference file
that reads or writes them must use the exact schemas below.

### 1. `repodoc.yaml` — User configuration (repo root)

Defined and written by `config-wizard.md`. Read by every other subsystem.
Required fields: `version`, `service.name`. Full schema in `config-wizard.md`.

Key fields used across subsystems:
- `service.name` / `service.description` / `service.owner` — identity, used in onboarding + index
- `jira.enabled` / `jira.ticket_pattern` — controls enrichment.md behavior
- `multi_repo.github_org` — controls cross-service discovery in enrichment.md
- `scan.exclude_dirs` / `scan.exclude_patterns` — controls scanner.md file filtering
- `questions.max_per_run` — caps question count in questions.md
- `diagrams.*` — controls which diagrams to generate in diagrams.md
- `audit.on_commit` — controls pre-commit hook in serve-and-setup.md
- `pr_template.enabled` — controls PR template in serve-and-setup.md

### 2. `.repodoc-state.json` — Scan state (repo root, must be committed)

Written by `scanner.md`, updated by `audit.md`, `deep-scan.md`, `questions.md`.

```json
{
  "version": "1.0",
  "last_scan": "<ISO 8601 timestamp>",
  "scanner_version": "1.0",
  "file_hashes": { "<relative-path>": "sha256:<hash>" },
  "generated_sections": {
    "<sentinel-id>": {
      "doc_file": "<path-in-docs/>",
      "generated_at": "<ISO 8601>",
      "source_files": ["<path>"],
      "human_edited": false
    }
  },
  "deep_scan_needed": ["<path>"],
  "open_questions": ["Q-001"],
  "resolved_questions": ["Q-002"],
  "coverage": {
    "endpoints_documented": 0, "endpoints_total": 0,
    "events_published_documented": 0, "events_published_total": 0,
    "events_consumed_documented": 0, "events_consumed_total": 0,
    "integrations_documented": 0, "integrations_total": 0,
    "data_models_documented": 0, "data_models_total": 0,
    "env_vars_documented": 0, "env_vars_total": 0,
    "deprecations_documented": 0, "deprecations_total": 0
  }
}
```

**Merge conflict rule:** union `file_hashes` and `generated_sections`, merge
`open_questions` arrays (dedup by ID), take most recent `last_scan`.

### 3. `.repodoc/unknowns.md` — Questions queue (outside docs/)

Format defined in `questions.md`. Written during `init`, updated by `answers`
and `deep-scan`. Never rendered in the docs site.

**Question ID convention:** `Q-<zero-padded-3-digit>` starting from `Q-001`.
IDs are never reused — resolved questions keep their ID.

---

## Sentinel System (cross-cutting)

Every auto-generated content block is wrapped in sentinel HTML comments. This is
the **single source of truth** for whether a block has been human-edited.

```markdown
<!-- repo-doc:generated
     id="<sentinel-id>"
     source="<source-file-path>"
     scan="<ISO 8601 timestamp>"
     hash="<sha256 of content between tags>"
-->
...generated content...
<!-- repo-doc:end id="<sentinel-id>" -->
```

**Sentinel ID naming:** `rdoc:<category>-<slug>` where slug is lowercase,
hyphens for spaces/special chars, stable across runs (no timestamps or counters).

| Category | Example |
|---|---|
| API endpoint group | `rdoc:api-endpoints-payment` |
| Event definition | `rdoc:events-kaspi-payment` |
| Integration | `rdoc:integration-order-service` |
| Data model | `rdoc:model-order` |
| Runbook entry | `rdoc:runbook-payment-timeout` |
| Feature | `rdoc:feature-kaspi-integration` |
| Security auth | `rdoc:security-auth-flow` |
| Deprecation | `rdoc:deprecation-v1-payment-initiate` |
| Glossary entry | `rdoc:glossary-kaspi` |
| Changelog version | `rdoc:changelog-v2-4-0` |

**Conflict resolution on re-scan:**

| Hash matches content? | Source changed? | Action |
|---|---|---|
| Yes | No | Skip — nothing to do |
| Yes | Yes | Source evolved, human didn't edit → update silently |
| No | No | Human edited, source unchanged → preserve, note in coverage |
| No | Yes | Both changed → flag in audit report, never auto-overwrite |

Content **outside** sentinel blocks is fully human-owned. Never touch it.

---

## Writing Principles (apply to all generated docs)

- Open every section with a **one-paragraph human-readable summary** before diagrams or tables
- Every API endpoint: method, path, auth requirement, request body, response body, example curl
- Every integration file: failure mode, timeout config, what happens if it goes down
- Never write "TODO" or "TBD" — use `[needs-deeper-scan: reason — run /repo-doc deep-scan <target>]`
- New features → new file in `features/`, new integrations → new file in `integrations/`
- Secrets: document env var **names** only, never values
- For unread high-priority files (score >= 60), mark inline:
  ```
  [needs-deeper-scan: src/legacy/processor.java — importance 85, not read in this pass.
   Run /repo-doc deep-scan src/legacy/processor.java]
  ```

---

## Important Constraints

- **GitHub MCP is out of scope.** Do not check for it, reference it, or plan any GitHub MCP usage.
- **MCP connection status is session-only.** Never cache it in `.repodoc-state.json`.
- **`docs_site.output_dir` is always excluded from scanning.** Defaults to `docs/` — prevents circular scanning. If the user chose a custom output dir (e.g. `docs-gen/`), that directory is excluded instead.
- **`.repodoc-state.json` must be committed** (shared team state). Never in `.gitignore`.
- **`repodoc-audit-report.md` is gitignored.** It's a local output file.
- **`.repodoc-backup/` is gitignored.** Auto-backups are not committed.
- **Never interrupt the engineer with questions during scanning.** Collect all unknowns
  and write them to `.repodoc/unknowns.md` at the end.
