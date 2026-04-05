---
name: tiered-memory
description: 3-layer file-based memory system for persistent user knowledge. Index (always loaded), topic files (lazy-loaded), transcripts (grep only). Prevents memory bloat, enforces write discipline, handles staleness.
version: 0.1.0
tags: [memory, persistence, user-context, long-term-memory]
---

# Tiered Memory System

A 3-layer file-based memory architecture for Hermes agents. Inspired by Claude Code's memory design:

- Memory is an index, not storage. Pointers, not content.
- Three layers with different bandwidth costs.
- Self-healing: merge, dedupe, prune stale entries.
- Skeptical retrieval: memory is a hint, never truth.

## Why This Design

Naive agent memory systems fail in two ways:

1. **Context explosion** -- everything gets appended, nothing is pruned
2. **Hallucinated history** -- stale entries contradict reality, and the model trusts them

This system prevents both by enforcing:

- Separation of index and storage
- Lazy loading of topic knowledge
- Write-then-index discipline (not index-and-forget)
- Active consolidation that removes contradictions
- Staleness as a first-class concept

## Architecture

```
~/.hermes/memories/
  MEMORY.md                 # Layer 1: Always-loaded index
  topics/
    identity-work.md        # Name, institution, goals, resume strategy
    preferences.md          # Communication style, routine, accountability
    projects.md             # Project references with stacks and metrics
  transcripts/              # Session archives (grep only)
```

### Layer 1: MEMORY.md (Always Loaded)

A compact index injected into every session. Each line is a pointer, NOT raw knowledge.

Format:

```
topic_key: brief_summary (~150 chars max) -> topics/filename.md
```

Rules:

- Never dump full content into MEMORY.md
- If a line exceeds ~150 chars, it belongs in a topic file
- Update the pointer after every topic file write
- Stale pointers are worse than no pointer (they mislead the model)

Example:

```
student: Tejas Garg, CSE AI/ML, IIIT Nagpur '27, CGPA 8.32 -> topics/identity-work.md
routine: sleep 11pm-7:30am, 2-3hr afternoon nap, free after 2pm -> topics/preferences.md
```

### Layer 2: Topic Files (Lazy-Loaded)

Self-contained knowledge files loaded only when their theme is relevant.

Structure:

```markdown
# Topic Name

## Subsection

- fact 1
- fact 2

---

last-verified: YYYY-MM-DD
```

Current topic files:

- `identity-work.md`: Name, institution, CGPA, internship status, resume variants, LinkedIn
- `preferences.md`: Output style rules, accountability structure, sleep and energy, discipline patterns
- `projects.md`: All project references with tech stacks, metrics, repos, and live links

### Layer 3: Transcripts (Grep Only)

Session summaries stored as `transcripts/YYYY-MM-DD-<slug>.md`.

Rules:

- NEVER load all transcripts into context
- Use grep or session_search for specific lookups
- Only write summaries for sessions with decisions worth remembering
- Skip routine sessions that add no new knowledge

## Write Discipline (CRITICAL)

Follow this exact order when writing new memory:

1. **Write to the correct topic file FIRST** -- update existing entries or add new ones
2. **Then update MEMORY.md** -- add or update the pointer line to match
3. **Never** dump raw content into MEMORY.md
4. **Delete** old entries when superseded (do NOT keep both old and new)
5. **Create** new topic files if the domain warrants it (keep the 150-char rule)

The write-then-index order is non-negotiable. Writing index first causes drift when the topic write fails or gets interrupted.

## Staleness Rules

- **Derivable facts are NOT stored.** If you can compute it from the current session or system state, don't persist it. This includes: file structures, debug logs, PR history, API results, installed packages.
- **Entries older than 30 days are hints,** not facts. Re-verify before using. If re-verification fails, delete the entry.
- **User corrections replace, never accumulate.** If the user says "actually my routine changed," delete the old entry and write the new one. Both cannot be true.
- **Temporary states get cleaned up.** Exam stress, current sprint status, "this week's focus" all become stale after the event passes. Auto-clean during consolidation.

## Consolidation

Merges duplicates, resolves contradictions, prunes stale entries, updates the index.

Triggers:

- **Session-end:** If any topic file was modified during the session, verify MEMORY.md pointers are current
- **On request:** Run `consolidate memory` at any time
- **Auto-clean:** During consolidation, remove entries about events that have already passed (e.g., "exams in May" after May is over)

Consolidation process:

1. Scan all topic files for contradictions or overlapping entries
2. Merge similar entries, keep the most specific version
3. Remove entries marked stale or contradicted by newer info
4. Verify each MEMORY.md pointer still points to valid content
5. Remove MEMORY.md pointers to deleted topics
6. Update `last-verified` timestamps on touched files

## Reading

Use `read_file(path)` to load MEMORY.md or specific topic files.
Use `search_files(target="content", pattern=..., path="~/.hermes/memories/topics/")` to search across all topic files.
Use `session_search(query=...)` for historical cross-session recall of transcripts.

## Pitfalls

- **Index-first writes:** Always write the topic file BEFORE updating MEMORY.md, not after
- **Keeping both old and new:** When a fact changes, DELETE the old entry. Two contradictory entries are worse than none
- **Storing derivable facts:** If the agent can compute it (list files, check API call results, read config), do NOT store it
- **Memory as truth:** Memory entries are hints. The model must verify against reality before acting on them
- **Over-indexing:** If MEMORY.md has more than 30-40 lines, it's doing too much. Push detail into topic files
