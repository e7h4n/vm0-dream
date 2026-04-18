# vm0-dream

A portable, learnable prompt that teaches any agent to **dream** — consolidate the past 24 hours of working memory into long-term memory, the way REM sleep does for a human brain.

This repo is not a runtime. It is a **prompt you install** as a skill (or a system-prompt fragment) into an agent harness that already supports:

1. **A file-based long-term memory system** — a directory of Markdown memory files plus a `MEMORY.md` index.
2. **A log of past agent runs** — retrievable by run ID.
3. **A scheduler** — something that can invoke the agent on a cron / daily trigger.

If your harness has these three primitives, the prompt below will work as-is. The only things you should need to change are the command names (how you list runs, how you schedule the agent) and the memory directory path.

---

## What "dreaming" is, in one paragraph

An agent accumulates a lot of short-lived context during a day of work: failed tool calls, user corrections, newly-discovered APIs, team members mentioned in passing, issue numbers referenced, decisions made. Most of that is lost the moment the conversation ends. **Dreaming** is a nightly pass where the agent reads *every* run from the past 24 hours, extracts what's worth keeping, writes it into its persistent memory files, and prunes anything that has gone stale. It is a memory consolidation pass, not a summarization pass — the output is updates to the memory system, not a report for a human.

---

## Installation

### 1. Install as a skill

Save the prompt in the **"The dream prompt"** section below as `SKILL.md` (or your harness's equivalent) under a skill named `dream`. The trigger should be the user saying `dream`, `/dream`, or any equivalent phrase.

The skill should also expose two management operations:

- `dream auto` / `dream on` → creates the nightly schedule (see below)
- `dream off` → removes it

### 2. Set up the nightly schedule

Configure your scheduler to invoke the agent with the prompt `/dream` (or whatever triggers the skill) once per day. A reasonable default is **03:00 local time**, when no one is around to send new requests.

Concretely, for a harness that exposes a `schedule setup` command:

```bash
<your-cli> schedule setup <agent-id> \
  --name dream \
  --frequency daily \
  --time 03:00 \
  --timezone <your-tz> \
  --prompt "/dream" \
  --enable
```

### 3. Memory-system assumptions

The prompt assumes your agent stores long-term memory as:

- A root directory, e.g. `~/.agent/memory/`
- An index file `MEMORY.md` at the root, containing one line per memory file: `- [Title](file.md) — one-line hook`
- One Markdown file per memory topic, each with YAML frontmatter:

  ```markdown
  ---
  name: <memory name>
  description: <one-line description>
  type: <feedback|user|project|reference>
  ---

  <memory content>
  ```

If your harness uses a different structure, adapt the paths and file format in Phase 5 of the prompt.

---

## The dream prompt

> Everything below this line is the prompt itself. Copy it verbatim into your skill file. Replace bracketed placeholders (`<...>`) with values appropriate to your harness.

---

# Skill: Dream

Consolidate working memory from the past 24 hours into long-term memory — like REM sleep for the agent brain.

## Trigger

User says "dream", "做梦", "sleep consolidate", or invokes `/dream` → run the default dream workflow.
User says "dream auto", "dream on", or invokes `/dream auto` / `/dream on` → run the `auto` operation.
User says "dream off" or invokes `/dream off` → run the `off` operation.

## Operations

### Default — Run dream consolidation

Execute Phases 1–6 below in order, then emit the Phase 7 report.

### Operation: auto

Enable a daily schedule that runs dream automatically at 03:00 local time every night.

```bash
<scheduler-cli> schedule setup <agent-id> \
  --name dream \
  --frequency daily \
  --time 03:00 \
  --timezone <local-tz> \
  --prompt "/dream" \
  --enable
```

Confirm to the user: schedule name `dream`, daily at 03:00, enabled, `/dream off` to cancel.

### Operation: off

Check `<scheduler-cli> schedule list` for a schedule named `dream`. If present, delete it; otherwise tell the user no `dream` schedule was found.

---

## Goal

Simulate human sleep memory consolidation: scan all runs from the past 24 hours, extract insights worth retaining, enrich entity knowledge, write them into persistent memory, and audit existing memory for staleness and broken citations.

## Workflow

### Phase 1: Orient — Read existing memory state

Before reading any run logs, read what is already known.

1. Read `<memory-root>/MEMORY.md`.
2. Read every topic file referenced from the index.

Build a mental map of:

- What facts, patterns, and preferences are already recorded.
- Which entries reference specific tools, file paths, run IDs, or decisions that might have changed.
- Which entries *look* potentially stale and should be re-validated in the Prune phase.

This phase ensures the Extract and Prune phases are informed — it prevents duplicate writes and catches contradictions.

---

### Phase 2: Gather — Collect all runs from past 24h

List every agent run from the past 24 hours:

```bash
<runs-cli> logs list --since 24h --limit 100
```

Collect every run ID. If the listing is capped (e.g. "Showing 100 of more results"), note the cap for the Phase 7 report but process all returned runs.

For **every** run ID collected, read its full content:

```bash
<runs-cli> logs <run_id> --all
```

Do NOT skip runs to save time. Do NOT sample a subset. Read them all.

- If a run returns a 500/fetch error: skip it and count it.
- If a run is very short (e.g. a one-turn greeting): still read it, but don't belabor it.

For each run, mentally extract:

- What was the user's request?
- What happened? (tool calls, errors, corrections, retries)
- How did it end?

---

### Phase 3: Extract — Identify memory-worthy patterns

After reading ALL runs, analyze everything across six categories.

**Category A — Mistakes & Corrections**

- Tool calls that failed and required retry with a different approach.
- User explicitly corrected output ("no not that", "wrong", "不对", etc.).
- Self-corrections where the agent realized an approach was wrong mid-execution.
- API/tool errors that revealed gaps in the agent's knowledge.

**Category B — Repeated Patterns**

- Same operation performed multiple times across runs — what's the canonical approach?
- Recurring question types that warrant a standing answer.
- Workarounds discovered for known tool limitations.
- CLI/bash patterns that reliably worked vs. alternatives that failed.

**Category C — New Knowledge**

- New tools, APIs, repos, or projects researched and understood.
- Architectural decisions or technical conclusions reached.
- User preferences revealed through conversation flow.
- Team / domain / product context learned.

**Category D — Process Improvements**

- Faster or simpler approaches that replaced initial attempts.
- Cases where reading docs or code first would have saved time.
- Cases where asking a clarifying question first would have been better.

**Category E — User Intent Gaps**

- Cases where what the user *asked for* differed from what they *actually wanted*.
- Implicit expectations that weren't stated but became clear mid-conversation.
- Patterns where the stated request was a proxy for a deeper goal.
- Recurring themes in what the user cares about beyond the literal task.

**Category F — Friction Signals**

- Points where the agent caused rework, confusion, or required multiple corrections.
- Responses that missed the mark on tone, depth, or format.
- Cases where the user had to re-explain or re-phrase their request.
- Tool usage patterns that slowed down rather than helped the interaction.

---

### Phase 4: Entity Sweep — Enrich world knowledge

Scan every run for mentions of real-world entities: people, projects, repos, integrations, external tools, companies. For each entity, ask: *do I have adequate memory coverage?*

**Entity types to track**

| Entity type          | Examples                          | Memory file pattern          |
| -------------------- | --------------------------------- | ---------------------------- |
| Team member          | names, handles                    | `contacts-*.md` or inline    |
| External person      | a customer, a vendor contact      | `contacts-external.md`       |
| Issue / PR           | `#8501`, `org/repo#1234`          | `project-issues-tracked.md`  |
| Connector / integration | integration IDs, 3rd-party APIs | `project-*-connector.md`     |
| Project / feature    | a feature name, an internal tool  | `project-*.md`               |
| External tool / API  | an npm package, a SaaS vendor     | KB or `project-*.md`         |

**Sweep procedure**

For each distinct entity found across all runs:

1. **Check coverage** — Does a memory entry exist for this entity? Is it current?
2. **Score thinness** — If an entity appeared 2+ times today but has little or no coverage, it is a gap.
3. **Fill gaps** — For thin or missing entries, write or update the relevant memory file with what was learned today. Record direct assessments from the runs verbatim where useful — the exact language used, not a sanitized paraphrase.
4. **Skip transients** — Single-mention, low-signal entities (e.g. a random npm package name) don't need entries unless they are likely to recur.

**What "good coverage" looks like**

- **Person:** role, contact info, how they relate to current projects, last known interaction context.
- **Project / feature:** current status, owner, key decisions made, known blockers.
- **Connector / integration:** authentication method, known issues, integration ID if applicable.
- **Issue / PR:** current state (open/closed/merged), what it's for, whether it's resolved.

Do not create coverage for entities that are already well-documented. Only fill genuine gaps.

---

### Phase 5: Consolidate — Write new and updated memories

For each significant insight from Phase 3 and each entity gap from Phase 4, write or update a memory file at `<memory-root>/`.

Use the appropriate memory type:

- `feedback` — behavioral corrections, approach preferences.
- `user` — information about who the user is.
- `project` — team / product / domain context.
- `reference` — where to find information (external systems, dashboards, docs).

**Memory file format**

```markdown
---
name: <memory name>
description: <one-line description>
type: <feedback|user|project|reference>
---

<memory content>

**Why:** <reason this matters>
**How to apply:** <when to use this>
```

**Compiled-truth + timeline structure.** For project and reference memories that track evolving state (a connector's status, an ongoing feature), use a "current state" section at the top (rewrite when things change) and a "history" section at the bottom (append-only). This preserves how understanding evolved:

```markdown
## Current State
<best current understanding — rewrite this section when facts change>

## History
- <YYYY-MM-DD>: <what was learned or changed>
- <YYYY-MM-DD>: <what was learned or changed>
```

Before writing a new file, check the `MEMORY.md` index. If an existing file covers the same topic, update it instead of creating a new one.

---

### Phase 6: Prune — Audit existing memories for staleness and broken citations

Using the mental map from Phase 1 and the knowledge gained in Phases 2–4, review each existing memory entry across two dimensions.

**6A — Staleness audit**

- **Contradicted?** Does anything learned today directly contradict this memory? → Update or delete.
- **Superseded?** Has a newer, better-understood pattern replaced what this memory describes? → Replace content.
- **Duplicate?** Does this memory overlap significantly with another entry? → Merge into the more complete one and delete the other.

**6B — Citation hygiene**

Scan every memory file for references to external artifacts that can go stale.

| Reference pattern       | How to verify                                             |
| ----------------------- | --------------------------------------------------------- |
| Issue / PR `#NNNN`      | Check if open/closed/merged; update if status changed     |
| Integration / connector ID | Check if it still exists; remove if deleted            |
| File path `/path/...`   | Check if the file exists; correct or remove if not        |
| Scheduled job name      | Check your scheduler listing; update if renamed / removed |
| Agent ID                | Check your agent listing; update if stale                 |

For each stale reference:

- Artifact still exists but status is wrong → update the memory content.
- Artifact no longer exists → remove the reference or note it as resolved.
- Do NOT leave dangling references in memory files.

This phase is a stateless audit — no persistent state file is needed. Use only what was observed in this run plus quick verification checks.

After pruning, update the `MEMORY.md` index:

- Add one line per new file: `- [Title](file.md) — one-line hook`
- Remove lines pointing to deleted files.
- Keep the index concise (≤ ~200 lines) so it fits in context.

---

### Phase 7: Report — Output consolidation summary

Output a summary inline in the current conversation. Be specific and honest. Don't pad.

Include:

- Memory consolidation: total runs listed / successfully read / skipped (errors).
- New memories written / existing memories updated / memories pruned.
- Entity sweep results: entities found, gaps filled, entities skipped as transient.
- Citation hygiene results: stale references found and resolved.
- Key insights from Phase 3, grouped by category (A–F) — topic-level, not raw content.
- If the run-list hit its cap, note how many runs existed in total.

---

## Notes for adapters

- **No knowledge-base sync.** This prompt deliberately omits any "ingest external sources" phase. It only consolidates the agent's own runs. If you want KB ingestion, compose it as a separate skill and chain them.
- **Idempotency.** Dreaming the same 24-hour window twice should produce roughly the same memory state. Phase 1 (read existing memory first) and the "update, don't duplicate" rule in Phase 5 are what make this true.
- **Honesty over completeness.** If a phase was skipped (e.g. run logs unavailable), the Phase 7 report must say so. A dream that silently drops a phase is worse than a dream that reports the gap.
- **Language.** Default to the user's configured language. The report is for a human; the memory files are for the agent.
