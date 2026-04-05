# Custom Skills for Hermes Agent

A collection of custom skills for [Hermes Agent](https://github.com/NousResearch/hermes-agent) -- a CLI-first AI agent built for autonomous terminal workflows.

These are not skills that ship with Hermes by default. They were designed, tested, and iterated on through real usage and represent opinionated approaches to specific problems.

## Installation

Skills live in `~/.hermes/skills/`. Symlink or copy from this repo:

```bash
# Symlink all skills at once
for skill in */; do
  ln -sf "$(pwd)/${skill}" ~/.hermes/skills/
done

# Or symlink a single skill
ln -sf "$(pwd)/tiered-memory" ~/.hermes/skills/tiered-memory
```

Then verify:

```
skills_list
skill_view tiered-memory
```

## Skills

<details>
<summary><strong>tiered-memory</strong> -- 3-layer file-based persistent memory</summary>

**Background:** Inspired by [Claude Code's memory architecture](https://x.com/himanshustwts/status/2038924027411222533), which was reverse-engineered from source code and went viral in March 2026. The post revealed that Claude Code uses index-based memory (MEMORY.md as pointers, not storage), a 3-layer design, background consolidation via forked subagents, and treats memory as hints not truth.

**Problem solved:** Hermes' default `memory` tool uses flat append-only entries injected into every session. This works for a few dozen entries but degrades with scale: context bloat, contradictory entries accumulate, and stale facts are treated as current truth.

**Design:**

- Layer 1: MEMORY.md (index, always loaded, ~150 chars/line)
- Layer 2: topic/\*.md (knowledge files, lazy-loaded on demand)
- Layer 3: transcripts/ (session archives, grep only)
- Write discipline: topic file first, then update index pointer
- Self-healing: merge, dedupe, prune during consolidation
- Staleness: entries older than 30 days are hints, not facts

**When to use:** Always active. Controls how persistent user knowledge is read, written, and maintained across sessions.

**See:** [references/architecture.md](tiered-memory/references/architecture.md) for the full design rationale and comparison to MemGPT, LangGraph, and vector+RAG approaches.

</details>

<details>
<summary><strong>skill-name</strong> -- one-line description</summary>

**Background:** TODO -- what inspired this skill, what problem it solves, what approach it takes.

**Problem solved:** TODO

**Design:** TODO

**When to use:** TODO

</details>

## Adding Your Own Skills

To create a new skill, follow this pattern:

```
skill-name/
  SKILL.md              # Required. YAML frontmatter + markdown instructions.
                        # This is what the agent reads when the skill is loaded.
  README.md             # Optional but recommended for the top-level repo docs.
                        # Can also live as references/README.md inside the skill dir.
  references/           # Optional. Supporting docs, architecture deep-dives, etc.
  templates/            # Optional. Starter templates the skill references.
  scripts/              # Optional. Automation scripts the skill uses.
  assets/               # Optional. Static files the skill needs.
```

The `SKILL.md` is the only required file. Everything else is supporting material. The `name` field in the YAML frontmatter must match the directory name exactly.

For detailed SKILL.md formatting, check out existing skills in `~/.hermes/skills/` or run `skill_view tiered-memory` in Hermes to see a real example.

## Philosophy

Good skills are:

- **Specific to a task type** -- not "everything about X" but "how to do X correctly"
- **Numbered and procedural** -- clear steps, not vibes
- **Honest about pitfalls** -- what breaks, what to watch out for, what the docs lie about
- **Self-contained** -- the agent should not need to search the web to use the skill
- **Maintained** -- stale skills are worse than no skills. Update them when they drift.

Bad skills are:

- Generic re-documentations of tool docs
- Unmaintained lists of outdated commands
- Opinionated workflows baked in without explanation
- Bloated with context that should be in a topic file or reference doc
