# tiered-memory

A 3-layer file-based memory system for Hermes agents that prevents context bloat, memory drift, and hallucinated history.

Inspired by [Claude Code's memory architecture](https://x.com/himanshustwts/status/2038924027411222533) -- index-based, bandwidth-aware, and self-healing.

## Quick Start

Drop this skill directory into your Hermes skills folder:

```
~/.hermes/skills/tiered-memory/
  SKILL.md        # Agent instructions for using the memory system
  references/
    templates/
      MEMORY.md.example
      topic-file.example
    architecture.md
    README.md
```

Or clone into a custom skills repo and symlink.

## What It Does

Most agent memory systems fail because they:
- Append everything, prune nothing (context explosion)
- Treat stale entries as truth (hallucinated history)
- Store derivable facts alongside real knowledge (bloat)

This system fixes all three by enforcing:
- **Separation of index and storage** -- MEMORY.md contains only pointers (~150 chars/line), actual knowledge lives in topic files
- **Lazy loading** -- topic files loaded only when relevant, never all at once
- **Write discipline** -- write to topic file first, then update the index pointer
- **Self-healing** -- merge, dedupe, prune contradictions during consolidation
- **Staleness awareness** -- entries older than 30 days are hints, not facts; must be re-verified
- **No derivable storage** -- if the agent can compute it, don't store it

## Architecture

```
MEMORY.md (always loaded, tiny index)
  |
  +-> identity-work.md (lazy, full details)
  +-> preferences.md (lazy, full details)
  +-> projects.md (lazy, full details)
  |
  transcripts/ (grep only, never bulk-loaded)
```

### Layer 1: MEMORY.md (Always Loaded)

A compact index injected into every session. Each line is a pointer, not raw knowledge.

```
student: Tejas Garg, CSE AI/ML, IIIT Nagpur '27, CGPA 8.32 -> topics/identity-work.md
routine: sleep 11pm-7:30am, 2-3hr afternoon nap, free after 2pm -> topics/preferences.md
```

### Layer 2: Topic Files (Lazy-Loaded)

Self-contained knowledge files loaded only when their theme is relevant.

```
~/.hermes/memories/topics/
  identity-work.md     # Name, institution, goals, resume strategy, LinkedIn
  preferences.md       # Output style, routine, accountability, discipline
  projects.md          # Project references with stacks, metrics, repos
```

### Layer 3: Transcripts (Grep Only)

Session summaries in `transcripts/YYYY-MM-DD-<slug>.md`. Never loaded in bulk. Searchable via grep or session_search.

## File Templates

See `references/templates/` for starter templates:
- `MEMORY.md.example` -- the index format with proper structure
- `topic-file.example` -- how topic files should be organized

## Architecture Deep Dive

See `references/architecture.md` for the full design rationale, comparisons to other systems, and why this approach works better than vector stores or append-only logs.

## Design Decisions

| Decision | Why |
|----------|-----|
| Plain Markdown files | No infra, no dependencies, human-readable, diffable |
| No vector DB | Not needed for structured knowledge; grep is fast and deterministic |
| 3 layers, not 1 | Different bandwidth costs require different load strategies |
| Write discipline enforced | Index-first writes cause silent drift when writes fail |
| Staleness is first-class | Old memory that contradicts reality is worse than no memory |
| Derivable facts banned | File structures, API results, and debug logs bloat context without adding value |

## License

MIT
