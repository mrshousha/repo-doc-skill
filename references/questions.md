# Questions Queue (Phase 8 of `init` pipeline + `/repo-doc answers` command)

The questions system collects ambiguities during scanning and provides a structured
way for engineers to fill in gaps. Questions are never asked interactively during
a scan — they are always collected and written at the end.

---

## 8.1 — Writing the Questions Queue

### When to create questions

Throughout scanning and doc generation, whenever you encounter something that:
- Cannot be determined from code alone
- Would be wrong to guess about (especially security, auth, failure behavior)
- Blocks a diagram or runbook entry from being complete
- Produces a materially incomplete doc section

**Do not interrupt.** Add to an internal running list and write them all at the end.

### File location

`.repodoc/unknowns.md` — create the `.repodoc/` directory if it doesn't exist.
This file lives **outside** `docs/` so it never appears in the MkDocs site.

### Question format

```markdown
# Open Questions for the Engineer

**Instructions:**
1. Fill in the `Your answer:` field for each question below
2. Run `/repo-doc answers` to incorporate your answers into the documentation
3. Answered questions are moved to the `## Resolved` section automatically

Run `/repo-doc init` to discover new questions from changed files.
Run `/repo-doc deep-scan <target>` to answer questions by scanning deeper.

---

## Q-001

**Status:** open
**Priority:** high
**Affects:**
  - docs/architecture/sequence-diagrams.md (flow name — incomplete)
  - docs/runbook/index.md (entry name — incomplete)

**What I observed:**
<Describe exactly what you found in the code — file, line, pattern>

**Why it matters:**
<Explain which doc sections are blocked or incomplete because of this>

**Reference:** `<file-path>`, line <N>

**Your answer:** _[fill in here]_

---

## Resolved

<!-- repo-doc moves answered questions here automatically after /repo-doc answers -->
```

### Question ID convention

- Format: `Q-<zero-padded-3-digit>` starting from `Q-001`
- IDs are **never reused** — resolved questions keep their ID in the Resolved section
- Next question always gets the next available integer
- If `.repodoc/unknowns.md` already exists, read it to find the highest existing ID
  and continue from there

### Priority rules

| Priority | Criteria |
|---|---|
| `high` | Blocks a diagram or runbook entry from being written at all |
| `medium` | Produces a materially incomplete doc section |
| `low` | Minor clarification — section is otherwise complete |

### Cap enforcement

Cap at `questions.max_per_run` from `repodoc.yaml` (default: 20). If there are
more unknowns than the cap:
- Prioritize: high > medium > low
- Write the top N questions
- Note at the top: "X additional lower-priority questions were suppressed. Re-run
  after addressing these."

### State update

After writing questions, update `.repodoc-state.json`:
- `open_questions`: array of all open question IDs
- `resolved_questions`: array of all resolved question IDs (preserve existing)

---

## 8.2 — `/repo-doc answers`: Answer Re-ingestion

When the engineer runs `/repo-doc answers`:

### 8.2.1 — Read and parse

1. Read `.repodoc/unknowns.md`
2. Find all questions where `Your answer:` contains text other than `_[fill in here]_`
3. Process each answered question based on its answer type:

### 8.2.2 — Answer types

**a. Standard answer** — the answer explains how something works:
- Locate the sentinel block(s) listed in `Affects:`
- **Rewrite** the affected section(s) naturally incorporating the answer — not as
  an appended note, but as if the information was always known
- Remove any `[needs-deeper-scan]` markers that this answer resolves
- Remove any Mermaid `%% [needs-deeper-scan]` comments that this answer resolves
- Update the sentinel `hash` to reflect the new content

**b. Answer implies legacy/dead code** — e.g. "That env var is from an old
integration, we don't use it anymore":
- Do **not** remove the related doc section automatically
- Flag for engineer review:
  ```
  Q-002 answer suggests ORDER_SERVICE_HOST is legacy.

  Suggested actions (choose one):
    1. Remove from docs/integrations/overview.md and add to docs/deprecations.md
    2. Add a deprecation notice to the integration doc but keep it
    3. Do nothing — leave as-is

  What would you like to do? (1/2/3)
  ```
- Wait for the engineer's choice before making changes

**c. Answer is unclear** — e.g. "I'm not sure, ask Javier":
- Do not close the question
- Add a sub-note: `[Clarification needed: answer was unclear. Suggested: ask Javier]`
- Leave `Status: open`
- If the answer reveals a new unknown, generate a new question `Q-N`

### 8.2.3 — Move resolved questions

In `.repodoc/unknowns.md`, move processed questions to the `## Resolved` section:

```markdown
## Resolved

### Q-001 — resolved <date>

**Original question:** <question text>
**Answer:** <the engineer's answer>
**Updated docs:**
  - docs/architecture/sequence-diagrams.md (flow now documented)
  - docs/runbook/index.md (entry added)
```

### 8.2.4 — Update state

- Update `.repodoc-state.json` `open_questions` and `resolved_questions` arrays
- Update `coverage` if resolving questions increased coverage

### 8.2.5 — Report summary

```
Processed N answers:
   Q-001 → Updated sequence diagram and runbook entry
   Q-002 → Awaiting your decision (legacy/dead code — see prompt above)

M questions still open in .repodoc/unknowns.md
```
