# Enrichment (Phase 5 of `init` pipeline)

After scanning files, enrich the extracted data with git history, PR metadata,
Jira ticket context, and cross-service dependency discovery. Each step is
independent but feeds into doc generation.

---

## 5.1 — Git History Scan

Run these git commands:

```bash
# Full commit log
git log --oneline --all

# Detailed log with bodies (for ticket IDs in commit bodies)
git log --format="%H %s %b" --all

# Detailed history for the top 20 files by importance score
git log --follow -p -- <file-path>   # repeat for each of the top 20

# Version tags for changelog grouping
git tag --sort=-version:refname | head -50
```

**Extract from git history:**
- Commit intent and messages
- Ticket IDs matching the `jira.ticket_pattern` regex from `repodoc.yaml`
  - Found in: commit subject (e.g. `[IE-4191] Fix Kaspi emit`), commit body,
    branch names
- Deprecation signals: commits mentioning "remove", "deprecate", "sunset",
  "migrate away from", "replaced by"
- Version/release tags for changelog grouping (see `specialized-docs.md` §Changelog)
- Files most frequently changed together (co-change coupling)

---

## 5.2 — PR Metadata Fetch

**Prerequisite:** `gh_available = true` from pre-flight check.

If available, run:
```bash
gh pr list --state merged --limit 50 --json number,title,body,headRefName
```

**Use PR data to:**
- Map PR titles to commits and ticket IDs (PR titles often contain ticket IDs
  not in commit messages)
- Extract PR body descriptions for feature context and changelog entries
- Identify PRs that describe intentional limitations or known issues

**If `gh` is not available:** skip silently, proceed without PR metadata. Do not
fail or warn — this is an optional enhancement.

Do **not** fetch individual review comments via `gh` — out of scope for this version.

---

## 5.3 — Jira Enrichment

**Prerequisites:** `jira.enabled: true` in config AND Atlassian MCP connected
(from pre-flight probe).

If both are true, for every unique ticket ID found across git commits and PR
titles/bodies:

1. **Fetch via Atlassian MCP:** title, description, acceptance criteria, labels,
   epic link, status, assignee
2. **Map to code:** associate each ticket with the commits and files it touches
3. **Use ticket content to populate:**
   - The *why* in feature docs (why was this built, what problem does it solve)
   - Architecture narrative (tickets often describe system-level decisions)
   - Changelog entries (ticket title is usually a better description than commit subject)
   - Deprecation context (tickets often specify removal timelines)
4. **Error handling:** if a ticket returns 404 or access denied, log a note and
   continue. Never fail the scan because a ticket is inaccessible.

**If Jira is disabled or not connected:** skip entirely. The documentation will
still be generated, just without the "why" context that tickets provide.

---

## 5.4 — Cross-Service Dependency Discovery

Scan for references to other services in:

| Signal | Where to look |
|---|---|
| HTTP base URLs | Hardcoded strings, config files, env var values |
| Service name env vars | Keys like `ORDER_SERVICE_HOST`, `PAYMENT_SERVICE_URL` |
| Shared internal libraries | Import statements (e.g. `com.deliveryhero.shared:keymaker-client`) |
| gRPC proto imports | Proto files importing external packages |
| Event topic names | Kafka, RabbitMQ, SQS topic/queue names suggesting cross-service flows |
| Docker Compose links | `depends_on`, `links`, network aliases |
| Kubernetes Service refs | `Service` resources, `ExternalName`, ingress rules |

**For each discovered dependency, record:**
- Service name (inferred)
- Call type: REST / gRPC / events / shared-lib / database
- Data direction: inbound / outbound / both
- Timeout and retry config if found
- The source file and line where the reference was found

**Ambiguous dependencies:** If you find an env var with no corresponding HTTP client
or SDK usage, add it to the questions queue (see `questions.md`) with the ambiguity
clearly described. Do not guess — ask.

**Org repo matching:** If `multi_repo.github_org` is set in config and `gh` CLI
is available:
```bash
gh repo list <org> --json name,url --limit 200
```
Use this to match discovered service names to actual repo names in the org, adding
repo URLs to integration docs where matched.
