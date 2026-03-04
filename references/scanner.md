# Scanner (Phases 3–4 of `init` pipeline + Phase 9 state persistence)

Covers: backup check, skeleton scan, importance scoring, tiered file reading,
and state persistence. This is the foundation — everything else builds on its output.

---

## 3.1 — Backup Check

Check if the docs output directory (`docs_site.output_dir` from `repodoc.yaml`) already exists and contains files.

**If it does:**
1. Copy it to `.repodoc-backup/docs-<timestamp>/` (e.g. `.repodoc-backup/docs-20250226-1400/`)
2. Inform the engineer:
   ```
   [repo-doc] Existing docs/ found. Backed up to .repodoc-backup/docs-20250226-1400/
   Proceeding with generation...
   ```
3. Continue — backup is automatic, no confirmation needed.

**If it does not exist:** proceed directly.

---

## 3.1.1 — Seed From Existing Docs (optional)

Check `docs_site.seed_from_existing` in `repodoc.yaml`.

**If `true`:** Before scanning source code, read all Markdown files in the
**original** `docs/` directory (not the output directory — this is the existing
human-written docs the user opted to reference). For each file:

- Extract human-written context: architecture notes, rationale, onboarding
  instructions, known gotchas, glossary terms
- Store this as a **context layer** in session memory, keyed by topic/section
- During doc generation (Phases 6–7), consult this context layer to:
  - Fill gaps where code doesn't reveal intent
  - Improve "why" sections in architecture and feature docs
  - Pre-populate glossary terms found in existing docs
  - Validate your understanding of integration behavior

**Do not** copy content verbatim from existing docs into generated output.
**Do not** treat existing docs as ground truth — code always takes precedence
for factual claims (endpoints, models, config). Use existing docs only for
narrative context, rationale, and intent.

**If `false` or unset:** skip this step entirely. Do not read the existing docs directory.

---

## 3.2 — Skeleton Scan

Build a structural map **without reading any file content** (besides the first 50
lines for import extraction):

1. List all files, filtered by `scan.exclude_dirs` and `scan.exclude_patterns`
   from `repodoc.yaml`
2. Deduplicate by resolved path to handle symlinks — never visit the same inode twice
3. Skip binary files: detect via file extension lists (images, archives, compiled
   binaries) or `file --mime-type` if available
4. Record:
   - Total file count
   - Language distribution by extension
   - Directory structure depth
5. Build a lightweight **import graph**: read only the first 50 lines of each file
   to extract `import` / `require` / `#include` / `use` / `from X import` statements.
   This reveals file centrality without full reads.

---

## 4.1 — Importance Scoring

Score every file to determine read order. Higher score = read first.

| Signal | Score |
|---|---|
| Filename contains: `main`, `app`, `server`, `bootstrap`, `handler`, `controller`, `router`, `service`, `gateway`, `consumer`, `producer`, `dispatcher` | +40 |
| Imported/required by 5–9 other files | +30 |
| Imported/required by 10+ other files | +50 |
| Modified in the last 30 commits | +20 |
| Contains HTTP route definitions (`@Get`, `@Post`, `router.get`, `app.route`, `@RequestMapping`, etc.) | +30 |
| Contains database schema or migration | +25 |
| Contains event topic definitions or Kafka producer/consumer setup | +25 |
| Contains proto / thrift / avro schema | +30 |
| Is a Dockerfile, docker-compose, or k8s manifest | +20 |
| Is a CI/CD config | +15 |
| File size > 500KB | −50 (likely generated or binary) |
| Matches generated patterns (`*_generated.*`, `*.pb.go`, `*.min.js`, `*_pb2.py`) | −100 (skip entirely) |
| In a test directory (`__tests__/`, `spec/`, `test/`, `tests/`) | −30 |
| In `docs_site.output_dir` (e.g. `docs/`, `docs-gen/`) | −200 (always excluded from scanning) |

**Repetitive directories:** If a directory has 10+ files of the same type, read at
most 3 representative samples and note:
```
[X more files of this pattern — sampled, not exhaustively read]
```

**Context heuristic:** After reading ~150 files in full, switch to
**header-and-signature-only** mode for remaining files (first 30 lines + exported
symbols). This preserves signal without burning context on implementation details.

---

## 4.2 — Tiered File Scan

Read files in descending importance score order. For each file, extract:

- **Entry points**: main files, server bootstraps, CLI entrypoints, Lambda handlers,
  Dockerfile `CMD`/`ENTRYPOINT`
- **Modules/packages**: directory structure, package boundaries, module exports
- **Data models**: classes, structs, schemas, DTOs, ORM models, proto files,
  OpenAPI/Swagger specs
- **External dependencies**: package manifests — group by category (HTTP clients,
  databases, messaging, auth, observability, utilities)
- **Configuration**: env files, config YAMLs, feature flags, secrets references
  (names only — **never log values**)
- **Infrastructure**: Dockerfiles, docker-compose, k8s manifests, Terraform, Helm charts
- **APIs exposed**: REST endpoints, GraphQL schemas, gRPC protos, event topics published
- **APIs consumed**: HTTP clients, SDK calls, queue consumers, event subscriptions
- **Auth & security**: middleware, token validation, role/scope annotations,
  PII-annotated fields
- **Data stores**: database connections, cache clients, object storage clients
- **Cross-service signals**: hostnames, service URLs, internal DNS, shared library imports
- **Deprecations**: anything suggesting removal or replacement (see `specialized-docs.md`
  §Deprecation Tracking for the full signal list)

For each unread high-priority file (score >= 60), mark inline in the relevant
generated doc section using the `[needs-deeper-scan]` format from SKILL.md.

---

## 4.3 / 9.1 — State Persistence

Write `.repodoc-state.json` to the repo root after the initial scan **and** again
after all generation is complete. This file must be committed — it is shared team state.

Use the schema defined in SKILL.md §Shared Data Contracts.

**When to write:**
- **Initial write** (after tiered scan, Phase 4 of the pipeline): populate
  `file_hashes`, `generated_sections` (empty initially), `deep_scan_needed`.
  Write this file **immediately** — it serves as a checkpoint. If later phases
  are interrupted, this file still records what was scanned.
- **Final write** (after all doc generation + questions, Phase 9): update
  `generated_sections` with all sentinel block metadata, `open_questions`,
  `resolved_questions`, `coverage`

**Computing file hashes:** Use SHA-256 of the file content. Store as `sha256:<hex>`.
Only hash files that were actually read (not skipped or sampled).

**Merge conflict rule** (documented here for reference — same as SKILL.md):
Union `file_hashes` and `generated_sections`, merge `open_questions` arrays (dedup
by ID), take the most recent `last_scan` timestamp.
