# Specialized Documentation Sections (Phase 7 of `init` pipeline, continued)

These sections are generated during the doc-writing phase of `/repo-doc init`.
Each section has specific scan targets and output formats.

**Inline coverage tracking:** As you generate each section, maintain a running
count of documented vs total items for the coverage report. For example, when
generating `api/endpoints.md`, count how many endpoints were documented vs how
many were found in the router scan. Pass these counts to the coverage report
(§7.6) at the end — do not re-scan to compute them separately.

---

## 7.1 — Security Documentation

### `docs/security/auth.md`

**Scan for:**
- Auth middleware: JWT validation, API key checks, OAuth flows, session handling
- Role and scope annotations: `@RequiresRole`, `@PreAuthorize`, custom decorators,
  route guards, `@RolesAllowed`, `@Secured`
- Token lifecycle: how obtained, validated, refreshed, revoked
- IDP integration: endpoints that call it, expected token format, validation logic

**Generate:**

1. **Auth flow sequence diagram** (`sequenceDiagram`) showing end-to-end token lifecycle
2. **Endpoint auth table** — every endpoint with its auth requirement and required scopes/roles:

   | Endpoint | Method | Auth Required | Scopes/Roles |
   |---|---|---|---|
   | `/v2/payment/initiate` | POST | JWT | `payment:write` |
   | `/health` | GET | None | — |

3. **IDP integration narrative** — prose description of identity provider integration

**Important:** If auth logic is complex or ambiguous, add targeted questions to the
queue rather than guessing. Security documentation that is confidently wrong is
worse than `[needs-deeper-scan]`.

### `docs/security/data-privacy.md`

**Scan for:**
- PII annotations: `@PII`, `@Sensitive`, `@Encrypted`, `@PersonalData`
- Field names suggesting PII: `email`, `phone`, `address`, `dob`, `ssn`,
  `national_id`, `ip_address`, `device_id`, `full_name`
- Log masking patterns: `maskField()`, `redact()`, custom log formatters
- Data retention or deletion logic (TTL, cleanup jobs, GDPR handlers)

**Generate:**

1. **PII fields table:**

   | Field | Model | Storage | Encrypted? | Masked in logs? |
   |---|---|---|---|---|
   | `email` | User | PostgreSQL | Yes | Yes |
   | `phone` | Customer | PostgreSQL | No | Yes |

2. **Retention/deletion logic** described in prose

---

## 7.2 — Runbook Documentation

### `docs/runbook/index.md`

**Scan for:**
- Error handler patterns: caught exceptions, propagation behavior
- Retry configs: max attempts, delay strategy (fixed/exponential/jitter)
- Circuit breaker configs: threshold, half-open probe interval, fallback
- Timeout values: per integration, per endpoint
- Fallback paths: feature flags, secondary service calls, graceful degradation
- DLQ consumers: topic name, feeder, reprocessing logic
- Health checks: endpoints, k8s liveness/readiness probes

**For each pattern found, generate a structured entry.** Even partial entries are
valuable — always create the entry with whatever can be inferred, mark gaps explicitly.

**Minimum viable runbook entry:**

```markdown
## Payment dispatch timeout

**Symptom:** HTTP 503 returned to caller
**Likely cause:** Payment service unreachable, timeout at 5000ms (config/integrations.yaml:42)
**How to confirm:** Look for ERROR PaymentClient in logs
**Resolution:** Check downstream health, verify network connectivity
**Escalation:** [needs-deeper-scan: add escalation owner here]
```

### `docs/runbook/alerts.md`

**Scan for:**
- Prometheus `*.rules.yaml` or `alerts.yaml`
- Datadog monitor configs
- PagerDuty policy definitions
- Metric names in code instrumentation (counters, histograms, gauges)

For each alert: name, condition, severity, suggested response.

### `docs/runbook/known-issues.md`

**Mine from:**
- `TODO`, `FIXME`, `HACK`, `WORKAROUND` comments (with 3 lines of surrounding context)
- PR body content describing intentional limitations
- Jira tickets labeled `bug` or `tech-debt` linked to commits but not closed

---

## 7.3 — Deprecation Tracking

**`docs/deprecations.md`**

Think carefully across **all** signal types — do not just grep for `@Deprecated`.

### Signal sources

| Signal | What to look for |
|---|---|
| Code annotations | `@Deprecated` (Java), `# deprecated` (Python), `/** @deprecated */` (JS/TS), `[Obsolete]` (.NET) |
| Comment patterns | "use X instead", "kept for backward compat", "legacy", "old implementation", "remove after", "will be removed in" |
| TODOs with removal intent | `// TODO: remove after [date/version/ticket]` |
| Feature flag signals | Flags permanently `true` in all envs, or kill switches permanently "off" |
| Version suffix signals | `/v1/` endpoint where `/v2/` also exists |
| Git history signals | Commits containing "removing", "sunsetting", "migrating away from", "replaced by" |
| PR body signals | Statements about upcoming removal or replacement |
| Jira signals | Tickets labeled `cleanup`, `sunset`, `migration`, `remove-legacy`, `tech-debt` |

### Output format per deprecation

```markdown
## `POST /v1/payment/initiate` — Endpoint

**Status:** Deprecated
**Type:** REST endpoint
**Deprecated since:** 2024-11-03 (commit a3f9c12, ticket IE-3891)
**Still functional?** Yes — returns valid responses but logs a deprecation warning
**Replacement:** `POST /v2/payment/initiate`
**Migration guide:** [extracted from commit/PR/Jira — or [needs-deeper-scan]]
**Removal target:** Q2 2025 (from Jira IE-3891) — or "Unknown [needs-confirmation]"
```

---

## 7.4 — Glossary

**`docs/glossary.md`**

Extract high-frequency domain nouns from: class names, variable names, function
names, Jira ticket titles, PR titles, commit messages, inline comments.

**Filter out generic technical terms:** `request`, `response`, `handler`, `config`,
`util`, `helper`, `manager`, `factory`, `controller` (generic), `service` (generic).

**Keep domain-specific terms:** anything a new engineer wouldn't understand from
context alone — payment provider names, internal platform names, business domain
concepts, market-specific terms.

**For each term:**
1. Search Jira descriptions and PR bodies for a human-written definition
2. If found: use verbatim with attribution
3. If not found: infer from usage context, flag with `[needs-confirmation]`

```markdown
## Kaspi

A payment provider integration for Kazakhstan markets.
Source: `src/integrations/kaspi/`
Events: publishes to `kaspi.payment.events` Kafka topic
See: [Kaspi payment flow](architecture/sequence-diagrams.md#kaspi-payment-flow)
```

---

## 7.5 — Onboarding Guide

**`docs/onboarding.md`**

Synthesize from all available data:

1. **What this service does** — one paragraph (from Jira epics, `repodoc.yaml`
   description, or README)
2. **Local setup** — exact commands extracted from `Makefile`, `package.json` scripts,
   `README`, or docker-compose. Maximum 8 steps, exact commands not prose.
3. **First 3 things to read** — the 3 highest-importance-scored files from scanning,
   with one sentence on why each matters
4. **Key concepts** — top 10 glossary terms, linked to `glossary.md`
5. **Who to ask** — team owner from `repodoc.yaml`, plus CODEOWNERS owners of
   critical paths if available
6. **Where things live** — quick directory map: "API handlers in X, models in Y,
   event consumers in Z"

If any section cannot be inferred, add a question rather than leaving it blank.

---

## 7.6 — Coverage Report

**`docs/coverage.md`** — auto-generated after every `init`, `audit`, or `coverage` command.

### Standalone `/repo-doc coverage` command

When run as a standalone command (not as part of `init`):

1. Read `.repodoc-state.json` to get existing `coverage` metrics and `generated_sections`
2. Read the current doc files and count documented items per category
3. Regenerate `docs/coverage.md` with updated metrics
4. Update `.repodoc-state.json` `coverage` field
5. Print the coverage table to the console

This is a lightweight operation — it does **not** re-scan source files. It only
recomputes coverage from the existing docs and state. For a full re-scan, use
`/repo-doc init` or `/repo-doc audit`.

### Coverage report format

Add a prominent notice at the top. Update `.repodoc-state.json` `coverage` field.

```markdown
# Documentation Coverage

> Auto-generated by repo-doc. Do not edit manually.
> Run `/repo-doc coverage` to refresh without a full scan.
> Last updated: <timestamp> | Scanner v1.0

| Category | Documented | Total | Coverage |
|---|---|---|---|
| API Endpoints | 12 | 14 | 85% |
| Events Published | 3 | 3 | 100% |
| Events Consumed | 2 | 3 | 67% |
| External Integrations | 3 | 4 | 75% |
| Data Models | 8 | 10 | 80% |
| Env Vars | 24 | 27 | 89% |
| Deprecations Tracked | 4 | ~6* | ~67% |

*Estimated — run `/repo-doc deep-scan deprecations` to surface more.

## Gaps
- `POST /v2/orders/cancel` — found in router, missing from docs/api/endpoints.md
- `notifications` Kafka consumer — found, no integration doc
- `UserPreference` model — exists, not in docs/data/models.md

## Open Questions
N unanswered questions in .repodoc/unknowns.md — run `/repo-doc answers`

## Deep Scans Pending
- src/legacy/order_processor.java (importance: 85)
Run `/repo-doc deep-scan <path>` to process these.
```

---

## 7.7 — Changelog

**`docs/changelog/index.md`**

Use [Keep a Changelog](https://keepachangelog.com) conventions. Grouped by semantic
version tag, then by date for untagged work. Reverse chronological order.

```markdown
# Changelog

All notable changes to this service are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com).

---

## [2.4.1] — 2025-02-20

### Fixed
- IE-4201: Payment dispatch retry storm on downstream timeout

## [2.4.0] — 2025-02-10

### Added
- IE-4191: Kaspi payment provider integration
- IE-4150: Settlement batch export endpoint

### Deprecated
- `POST /v1/payment/initiate` — use `/v2/payment/initiate` instead (IE-3891)

---

## [Unreleased]

Changes not yet associated with a version tag:
- Refactor Kaspi retry logic
```

**Generation rules:**
- Group commits by nearest preceding git version tag
- Within a group, categorize: Added / Changed / Fixed / Deprecated / Removed / Security
- If a commit has a Jira ticket: use ticket title from MCP if available, else commit subject
- If no version tags exist: group by calendar month
- Limit to last 100 commits for initial generation — note truncation at bottom
