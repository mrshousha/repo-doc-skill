# Deep Scan (`/repo-doc deep-scan <target>` command)

Targeted scanning that reads files in full, regardless of context cost. Used to
resolve `[needs-deeper-scan]` markers and open questions.

`<target>` can be a section name, a file/directory path, or a question ID.

---

## Target: Section Name

Valid section names: `architecture`, `security`, `runbook`, `deprecations`,
`integrations`, `api`, `data`, `glossary`, `events`, `infrastructure`

1. Look up which source files feed that section via `generated_sections` in
   `.repodoc-state.json`
2. Check the section's doc file for `[needs-deeper-scan]` markers pointing to
   unread files
3. Read **all** unread files related to this section in full
4. Regenerate only the affected sentinel blocks in the relevant doc files
5. Update `.repodoc-state.json`:
   - Remove resolved files from `deep_scan_needed`
   - Update `file_hashes` for newly read files
   - Update `generated_sections` with new sentinel metadata
6. If questions were unblocked by this scan, resolve them or update their status
   in `.repodoc/unknowns.md`

---

## Target: File or Directory Path

1. Read the specified file(s) in full
2. Extract all signals (same extraction list as `scanner.md` §Tiered File Scan)
3. Determine which sentinel blocks reference this file via `source_files` in
   `.repodoc-state.json` `generated_sections`
4. Regenerate those sentinel blocks with the new data
5. Check if any open questions in `.repodoc/unknowns.md` reference this file:
   - If the scan answers them: propose the answer to the engineer and ask for
     confirmation before closing
   - If not answerable from code: report what was found, ask the engineer for
     the rest

---

## Target: Question ID

Format: `Q-001`, `Q-023`, etc.

1. Read the question from `.repodoc/unknowns.md`
2. Find the files listed in the `Reference:` field
3. Read those files in full
4. Attempt to answer the question from code analysis alone
5. **If answerable:** propose the answer and ask the engineer for confirmation
   before writing it to docs. Show:
   ```
   Based on reading <file>, I believe the answer to Q-001 is:
   <proposed answer>

   This would update:
     - docs/architecture/sequence-diagrams.md
     - docs/runbook/index.md

   Accept this answer? (y/n/edit)
   ```
6. **If not answerable:** report what was found and what remains unclear:
   ```
   Read <file> in full. Found:
     - <what was discovered>

   Still unclear:
     - <what remains unknown>

   Please provide the missing context in .repodoc/unknowns.md and re-run
   /repo-doc answers.
   ```

---

## After Any Deep Scan

Always perform these updates:

1. Update `docs/coverage.md` — recompute coverage metrics
2. Update `.repodoc-state.json`:
   - `deep_scan_needed`: remove resolved entries
   - `file_hashes`: add hashes for newly read files
   - `generated_sections`: update affected blocks
   - `coverage`: update metrics
   - `last_scan`: update timestamp
3. Report summary:
   ```
   Deep scan complete: <target>
   - Files read: N
   - Sentinel blocks updated: M
   - Questions resolved: K
   - Coverage: X% → Y%
   ```
