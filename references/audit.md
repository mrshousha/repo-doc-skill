# Audit — Documentation Drift Detection (`/repo-doc audit` command)

`/repo-doc audit` detects when code has changed but documentation hasn't kept up.
It produces a local report file and optionally adds staleness banners to docs.

---

## A.1 — Determine Changed Files

Detect the current branch context:

- **On a feature branch:** `git diff main...HEAD --name-only`
  (falls back to `master` if `main` doesn't exist)
- **On main/master:** `git diff HEAD~1 HEAD --name-only`

**Always exclude** from the changed files list:
- Everything under `docs_site.output_dir` from `repodoc.yaml` (defaults to `docs/`)
- Everything under `.repodoc/`
- `.repodoc-state.json` itself

---

## A.2 — Compare Against State

For each changed file:

1. Compute its current SHA-256 hash
2. Compare to the hash stored in `.repodoc-state.json` `file_hashes`
3. If changed: re-scan the file for the extraction signals listed in
   `scanner.md` §Tiered File Scan
4. Compare extracted signals against the sentinel blocks in related doc files
   (lookup via `generated_sections` in `.repodoc-state.json`)

---

## A.3 — Classify Drift

| Severity | Condition | Example |
|---|---|---|
| **Breaking** | Endpoint removed from code but still documented | Docs will mislead callers |
| **Breaking** | Auth requirement changed but not in security docs | Security gap |
| **Breaking** | Event schema changed but not in events doc | Consumers will break |
| **Notable** | New endpoint added, not in `docs/api/endpoints.md` | Gap in API docs |
| **Notable** | New env var added, not in `docs/infrastructure/configuration.md` | Missing config |
| **Notable** | New external service call, no integration doc | Undocumented dependency |
| **Notable** | Human-edited sentinel block where source also changed | Conflict |
| **Minor** | New commit with ticket ID not in changelog | Incomplete changelog |
| **Minor** | Data model field added, not in models doc | Minor gap |

---

## A.4 — Write Audit Report

Write `repodoc-audit-report.md` in the repo root (this file is gitignored —
set during `/repo-doc config`).

```markdown
# repo-doc Audit Report

Generated: <timestamp>
Branch: <current branch>
Files changed: <count>

## Summary
<breaking-count> Breaking | <notable-count> Notable | <minor-count> Minor

---

## Breaking

### <Title describing the drift>
- **Source:** `<file>` — <what changed>
- **Still documented in:** `<doc-file>` (<sentinel-id>)
- **Action needed:** <specific action>
- **Auto-fixable:** Yes/No — <how to fix if yes>

## Notable

### <Title>
- **Source:** `<file>:<line>`
- **Action needed:** <specific action>
- **Auto-fixable:** Yes/No

### Conflict: human edits AND source changed
- **Block:** `<sentinel-id>` in `<doc-file>`
- **Source changed:** `<file>` (modified <date>)
- **Human edit detected:** Content hash mismatch
- **Action needed:** Manual review before overwriting

## Minor

### <Title>
- **Action needed:** <specific action>
```

---

## A.5 — Staleness Banners

For doc sections where the source file changed more than 30 days ago without a
re-scan, add a staleness banner **inside** the sentinel block (before the content):

```markdown
> **Possibly outdated.** Source `src/handlers/PaymentHandler.java` last changed
> 45 days ago. Run `/repo-doc audit` to check for drift.
```

Only add banners inside sentinel blocks — never modify human-owned content.

---

## A.6 — State Update

After the audit:
- Update `file_hashes` in `.repodoc-state.json` for any files that were re-scanned
- Update `coverage` if the audit revealed new gaps
- Update `last_scan` timestamp
- If new questions emerged during re-scan, add them to `.repodoc/unknowns.md`
  following the format in `questions.md`

---

## A.7 — Summary Output

```
repo-doc Audit Report
━━━━━━━━━━━━━━━━━━━━━
Branch: feature/kaspi-v2
Files changed: 4

Breaking: 1 | Notable: 2 | Minor: 1

Full report: repodoc-audit-report.md
Run /repo-doc init to regenerate affected sections.
```
