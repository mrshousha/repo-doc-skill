# Documentation Structure (Phase 7 of `init` pipeline)

Defines the file tree layout for generated documentation, the sentinel system
for tracking auto-generated vs human-edited content, and writing rules.

---

## File Tree

All documentation lives in `docs/`. For monorepos, each service gets its own
subdirectory (`docs/payment-service/`, `docs/notification-service/`), each with
the full structure below. MkDocs nav uses tabs per service.

For a single service, use `docs/` directly (no subdirectory).

```
docs/
├── index.md                        # Service one-liner, purpose, quick nav
├── onboarding.md                   # New engineer guide
├── architecture/
│   ├── overview.md                 # Architecture diagram + narrative
│   ├── data-flow.md                # Data flow diagrams per domain
│   ├── sequence-diagrams.md        # Sequence diagrams per flow
│   └── entity-model.md             # Domain/entity diagrams
├── api/
│   ├── endpoints.md                # All REST/GraphQL/gRPC APIs
│   └── events.md                   # Events published and consumed
├── security/
│   ├── auth.md                     # Auth flows, scopes/roles, token lifecycle
│   └── data-privacy.md             # PII fields, storage, log masking, retention
├── integrations/
│   ├── overview.md                 # Summary table of all external dependencies
│   └── <service-name>.md           # One file per external service
├── data/
│   ├── models.md                   # Data models, schemas, DTOs
│   └── storage.md                  # Databases, caches, object stores
├── infrastructure/
│   ├── deployment.md               # How deployed: environment, replicas, resources
│   ├── configuration.md            # Every env var: name, type, required, default
│   └── local-setup.md              # Exact commands to run locally — steps not prose
├── runbook/
│   ├── index.md                    # Common failure modes and resolution
│   ├── alerts.md                   # Alert definitions and response playbooks
│   └── known-issues.md             # Known bugs, workarounds, timelines
├── features/
│   └── <feature-name>.md           # One file per major feature
├── deprecations.md                 # Everything deprecated
├── glossary.md                     # Domain terms with definitions
├── coverage.md                     # Auto-generated coverage report
└── changelog/
    └── index.md                    # Grouped by version tag or date
```

**Outside docs/ (not rendered in docs site):**
```
.repodoc/
└── unknowns.md                     # Open questions — internal working file
```

---

## Sentinel System

The sentinel comment is the **single source of truth** for whether a content block
has been human-edited. There is no duplicate hash storage — the sentinel itself
carries all metadata.

### Format

```markdown
<!-- repo-doc:generated
     id="<sentinel-id>"
     source="<source-file-path>"
     scan="<ISO 8601 timestamp>"
     hash="<sha256 of content between tags>"
-->

...generated content here...

<!-- repo-doc:end id="<sentinel-id>" -->
```

### Sentinel ID Naming

Use `rdoc:<category>-<slug>` format. See SKILL.md §Sentinel System for the full
category table. Slugs are:
- Lowercase
- Hyphens for spaces and special characters
- Stable across runs (no timestamps, no counters)
- Derived from the subject name (e.g. "Payment Handler" → `payment-handler`)

### Conflict Resolution on Re-scan

Compute SHA-256 of the current content between sentinel tags and compare to the
`hash` attribute in the opening sentinel comment.

| Hash matches? | Source changed? | Action |
|---|---|---|
| Yes | No | Skip — nothing to do |
| Yes | Yes | Source evolved, human didn't edit → **update silently** |
| No | No | Human edited, source unchanged → **preserve**, note in coverage report |
| No | Yes | Both changed → **flag in audit report**: "Conflict: human edits AND source changed. Review required." **Never auto-overwrite.** |

### Human-Owned Content

Content **outside** sentinel blocks is fully human-owned. Never modify, move, or
delete it under any circumstances. Engineers can add their own sections, notes,
and context freely — repo-doc only manages content within its own sentinel blocks.

---

## File Creation Rules

- **New features** → new file in `features/<feature-name>.md` — never append to
  existing feature files
- **New integrations** → new file in `integrations/<service-name>.md` — never
  append to existing integration files
- **`index.md`** — always regenerated with current service summary and nav links
- **`coverage.md`** — always regenerated (it's a report, not documentation)
- **`changelog/index.md`** — append new entries, preserve existing ones

---

## `docs/index.md` Template

```markdown
# <service.name>

<service.description from repodoc.yaml>

## Quick Navigation

| Section | Description |
|---|---|
| [Onboarding](onboarding.md) | New to this service? Start here |
| [Architecture](architecture/overview.md) | System overview and diagrams |
| [API](api/endpoints.md) | REST/gRPC/GraphQL endpoints |
| [Integrations](integrations/overview.md) | External service dependencies |
| [Runbook](runbook/index.md) | Failure modes and resolution |
| [Coverage](coverage.md) | Documentation completeness |

**Owner:** <service.owner>
**Last documented:** <timestamp>
```
