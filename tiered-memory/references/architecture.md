# Architecture Deep Dive

## The Problem with Agent Memory

Most agent memory systems follow one of these patterns:

### Pattern 1: Append-Only Log
Every fact the user shares gets appended to a single file or vector store. Over time, this grows unboundedly. The agent must either truncate (losing context) or load everything (explosion of tokens).

### Pattern 2: Vector Database
Everything gets embedded and stored. Retrieval is semantic search over the vector space. This sounds good but has three problems:
- Embeddings are imprecise for structured knowledge (dates, version numbers, exact commands)
- No concept of staleness -- a year-old wrong answer has the same embedding as a fresh fact
- Infrastructure cost and complexity for what is fundamentally a lookup problem

### Pattern 3: Context Dump
The agent writes everything into a single MEMORY.md file. This works for the first few sessions, then the file exceeds the always-loaded token budget and quality degrades.

## The Claude Code Approach (and Why It Works)

Anthropic's Claude Code (reverse-engineered by the community) uses a fundamentally different model:

### Principle 1: Memory is an Index, Not Storage

MEMORY.md contains only pointers. Each line is ~150 characters pointing to a topic file. The actual knowledge lives elsewhere and is loaded lazily.

This means the always-loaded context stays tiny and deterministic. There is no semantic search, no embedding computation, no "maybe this is relevant." The index says "if you need identity context, read identity-work.md."

### Principle 2: Three Layers with Different Bandwidth Costs

| Layer | What | When Loaded | Cost |
|-------|------|-------------|------|
| 1 | Index (MEMORY.md) | Always | ~50 tokens per line |
| 2 | Topic files (*.md) | On-demand (when pointer says so) | ~300-800 tokens per file |
| 3 | Transcripts (.json) | Never bulk-loaded (grep only) | ~50 tokens per match |

The key insight: not all memory deserves the same loading treatment. Some information (who the user is) should always be available. Other information (detailed project specs) should only load when we are actually discussing projects. Archival information (old session transcripts) should only come up via targeted search.

### Principle 3: Write-Then-Index Discipline

The order matters. Write to the topic file first, then update the index pointer. If you do it in reverse:
1. Index points to content that does not exist yet
2. Topic write fails or gets interrupted
3. Agent trusts the index and finds nothing
4. Model hallucinates the missing content

This is a subtle failure mode that only appears at scale.

### Principle 4: Self-Healing (autoDream)

Background consolidation merges duplicates, removes contradictions, and prunes stale entries. Memory is continuously edited, not just appended.

Without this, memory drift is inevitable. Every user correction, every change in circumstances, every new project adds content. Eventually the file contains contradictory information about who the user is and what they want.

### Principle 5: Skeptical Retrieval

Memory entries are hints, not truth. The model is instructed to verify before acting. This is critical because:
- Users change their minds
- Circumstances evolve (new internships, different routines, updated goals)
- The agent may have misinterpreted or miscategorized something

If memory says "user prefers X" but the latest conversation says "I actually prefer Y now," the memory must yield. The most recent signal always wins.

### Principle 6: What NOT to Store

This is the hardest engineering decision. The rule is: if it is derivable from the current system state, do not store it.

Examples of things NOT to store:
- File system structure (can be ls'd)
- Installed packages (can be queried)
- API results (can be re-fetched)
- Debug logs (ephemeral by nature)
- PR history (queryable from GitHub)
- Current working directory (trivial)
- Session state (the agent knows what it is doing right now)

Examples of things TO store:
- User's name, preferences, and goals
- Historical decisions and their rationale
- Long-term project information
- Communication style preferences
- Patterns and tendencies (discipline, routine, energy levels)

## Comparison to Other Systems

| System | Approach | Pros | Cons |
|--------|----------|------|------|
| MemGPT/Letta | Core + peripheral memory with external compute | Good concept | Peripheral memory grows unboundedly |
| LangGraph checkpointers | Full conversation serialization | Complete history | No summarization or pruning |
| Vector+RAG | Embed all facts, search on demand | Semantic matching | Imprecise for structured data, no staleness handling |
| **Tiered Memory** | Index + topic files + archival grep | Predictable bandwidth, self-healing, simple | Requires write discipline |

## Implementation Notes for Hermes Agents

1. MEMORY.md lives at `~/.hermes/memories/MEMORY.md` (or path of your choice)
2. Topic files live at `~/.hermes/memories/topics/`
3. Transcripts live at `~/.hermes/memories/transcripts/`
4. The memory tool injects a compact pointer into every session that tells the agent "use the tiered-memory skill for memory operations"
5. All actual memory I/O is done via read_file, write_file, patch, and search_files

## When to Add a New Topic File

Add a new topic file when:
- You have 5+ facts about a topic that are not covered by existing files
- The topic is accessed frequently enough to deserve its own file
- The combined pointer in MEMORY.md would exceed ~150 characters

Do NOT add a new topic file for:
- 1-2 facts that fit in an existing file
- Temporary information (use transcripts instead)
- System state that can be queried
