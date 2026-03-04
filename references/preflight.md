# Pre-flight Checks (Phase 1 of `init` pipeline / pre-step for all commands)

Run these checks **before every command**, in this exact order. They are fast and
cheap — no file reads, just existence checks and lightweight probes.

---

## 1.1 — Config File Check

Check whether `repodoc.yaml` exists in the repo root.

**If missing and the command is `config`:** proceed normally — config creates this file.

**If missing and the command is anything else:** stop immediately with:

```
⚠️  No repodoc.yaml found.

repo-doc needs to be configured before it can run.
Please run /repo-doc config first — it only takes a minute.
```

Do not attempt to infer config or run with defaults. Stop here.

**If present:** read it and validate:
- Required fields: `version`, `service.name`
- If any required field is missing or malformed, report the exact field and stop:
  ```
  ⚠️  repodoc.yaml is invalid: missing required field "service.name"
  Run /repo-doc config to fix.
  ```

---

## 1.2 — Atlassian MCP Connection Check

Attempt a lightweight probe to verify the Atlassian MCP is active in this session.

**How to probe:** Call `atlassianUserInfo` (or equivalent no-op Atlassian MCP call).
- If it succeeds → mark `jira_connected = true` in session memory
- If it throws an auth or connection error → mark `jira_connected = false`

**If not connected** and `jira.enabled` is not `false` in config, and `--skip-jira` was
not passed, display this and **block**:

```
⚠️  Atlassian MCP is not connected.

repo-doc uses Jira to enrich documentation with the "why" behind features —
ticket descriptions, acceptance criteria, and epic context.

To connect:
  1. Open your AI agent's settings → Integrations
  2. Connect your Atlassian workspace
  3. Re-run this command

To skip Jira for this session only: re-run with --skip-jira
To permanently disable Jira: set jira.enabled: false in repodoc.yaml
```

**If `--skip-jira` is passed** or `jira.enabled: false` in config: proceed without
Jira, log a note:
```
[repo-doc] Jira enrichment skipped for this run.
```

**Important:** Store probe results in session memory only — never in `.repodoc-state.json`.
MCP availability can change between sessions and the probe is cheap enough to run every time.

---

## 1.3 — gh CLI Check

Run:
```bash
gh --version
```

- If available: store `gh_available = true` in session memory. This enables PR
  metadata fetching and org repo listing as a fallback — not for GitHub MCP features
  (which are out of scope for this skill).
- If not available: proceed silently. PR metadata and org discovery will be skipped.

**GitHub MCP is out of scope.** Do not check for it, do not reference it, do not
plan any GitHub MCP usage anywhere.
