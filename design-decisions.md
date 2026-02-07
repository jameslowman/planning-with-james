# Design Decisions

This document captures the reasoning behind the design of `planning-with-james`, a Claude Code plugin for deep codebase knowledge indexing and feature planning.

---

## Core Philosophy

> "Great features aren't built with shallow knowledge. If I'm going to invest my money and time, it's going to be in the plan, and the plan requires the most detailed knowledge possible."

**Cost is not a concern. Knowledge depth is the goal.**

This plugin prioritizes thoroughness over efficiency. It uses Opus for all analysis tasks. A full indexing run may consume an entire Claude Code session - this is acceptable and expected.

---

## Problem Statement

Existing planning plugins had issues:
1. Never sure they were working as intended
2. Planning methodology didn't align with user's mental model
3. Shallow knowledge leading to shallow plans

The solution: build a comprehensive knowledge base *before* planning, so planning agents have deep context to work from.

---

## Architecture Overview

### Six Commands

| Command | Purpose |
|---------|---------|
| `/planning-with-james:create-knowledge` | Full codebase indexing from scratch |
| `/planning-with-james:update-knowledge [context\|verify\|repair]` | Incremental updates, integrity checks, repairs |
| `/planning-with-james:explore` | Query the knowledge graph for understanding, impact, flows |
| `/planning-with-james:plan` | Structured 7-phase planning with adaptive checkpoints |
| `/planning-with-james:go-time [plan-id\|pause\|status]` | Execute plan tasks with full continuity |
| `/planning-with-james:epic [review\|status\|epic-id]` | Create and manage multi-plan epics |

### Knowledge Location

```
.claude/planning-with-james/knowledge/
```

- **Project-specific**: Each repo has its own knowledge base
- **Git-ignored**: Knowledge is generated, not tracked
- **Plugin-namespaced**: Under `planning-with-james/` to avoid conflicts

---

## Key Design Decisions

### 1. File-Based Knowledge Graph

**Decision**: Store knowledge as markdown files with YAML frontmatter, plus a `_graph.json` for machine-readable relationships.

**Why**:
- Files survive context compaction (autocompact)
- Human-readable and debuggable
- Agents can navigate by reading files and following links
- Natural checkpointing - if interrupted, progress is preserved

**Structure**:
```
knowledge/
├── _overview.md          # Root index, last commit, stats
├── _graph.json           # Machine-readable relationship graph
├── _architecture.md      # System-wide connectivity narrative
├── _discovery.json       # Module inventory from Phase 1
└── modules/
    └── {module-id}/
        ├── _index.md     # Module overview + frontmatter
        ├── _refs.json    # Import/export relationships
        └── *.md          # Detail files as needed
```

### 2. Three-Phase Generation

**Decision**: Split indexing into distinct phases with different agent strategies.

**Phase 1: Structural Discovery** (single agent, broad)
- One agent analyzes the whole codebase structure
- Identifies module boundaries
- Outputs `_discovery.json` as foundation for Phase 2

**Phase 2: Deep Dives** (parallel agents, deep)
- One Opus subagent per module
- All launched in parallel for speed
- Each writes its own `_index.md` and `_refs.json`

**Phase 3: Connectivity Mapping** (single agent, cross-cutting)
- Reads all `_refs.json` files
- Builds `_graph.json` with nodes and edges
- Writes `_architecture.md` narrative
- Updates frontmatter with `external_refs`

**Why this split**:
- Phase 1 needs holistic view (one agent)
- Phase 2 benefits from parallelism (many agents)
- Phase 3 needs to see all outputs (one agent)

### 3. Cascading Graph Updates

**Decision**: When ANY module is updated, automatically update all connected modules.

**Why**:
- Knowledge graph must stay consistent
- A change to module A's interface affects modules B and C that depend on it
- Partial updates create stale/incorrect relationships
- Cost is acceptable for consistency

**How it works** (linear, one at a time):
1. Module A is updated, agent returns brief summary (interface_changed, dependencies_changed)
2. Check `_graph.json` edges for modules connected to A
3. Spawn ONE agent per connected module to re-verify relationships
4. Update `_refs.json` and frontmatter for each
5. Rebuild `_graph.json` after all cascade updates complete
6. Progress file tracks every completed step for resume capability

### 4. Multi-Mode Update Command

**Decision**: Single `update-knowledge` command with four modes based on argument.

| Argument | Mode | Purpose |
|----------|------|---------|
| empty | Standard | Git-based incremental update |
| text | Guided | Interpret user intent, targeted update |
| `verify` | Verify | Check integrity, report problems, no changes |
| `repair` | Repair | Fix all problems found by verify |

**Standard mode**: Git-based, updates only affected modules, cascades through graph.
**Guided mode**: Interprets intent (deep dive, correction, business context), asks clarifying questions, executes targeted update, cascades.
**Verify mode**: Checks module completeness, staleness, codebase alignment, graph consistency, missing modules. Reports only.
**Repair mode**: Runs verify, then fixes each problem linearly with progress tracking.

**Why not separate commands**: Reduces cognitive overhead. All modes share cascade logic. Verify/repair are maintenance operations on the same knowledge base.

### 5. Clarification Before Execution

**Decision**: In guided mode, ask clarifying questions before doing work.

**Why**:
- User rants are unstructured
- Better to confirm understanding than guess wrong
- Prevents wasted work
- Creates dialogue that refines intent

**Intent categories**:
| Intent | Indicators | Action |
|--------|------------|--------|
| Deep dive | "go deeper", "more detail" | Full re-analysis |
| Correction | "wrong", "actually", "missed" | Fix documentation |
| Business context | "exists because", "historically" | Add human knowledge |
| Relationship fix | "connected", "calls", "depends" | Update graph |
| New area | "missed the X system" | Create new module |

### 6. Knowledge-First Principle

**Decision**: NEVER explore code without first consulting the knowledge graph.

**Pattern**:
```
1. READ _overview.md         → Understand the system landscape
2. READ _graph.json          → Find related modules and connections
3. READ relevant _index.md   → Load existing knowledge about target areas
4. THEN explore code         → Fill gaps, verify, go deeper
```

**Why**:
- Prevents redundant exploration (rediscovering what we already know)
- Ensures continuity (builds on existing knowledge, not parallel to it)
- Guides exploration (graph shows what's connected, what to check)
- Cost efficiency (don't spend tokens re-learning documented information)

This is non-negotiable for all operations: updates, deep dives, corrections, new areas.

### 7. Choose-Your-Own-Adventure Navigation

**Decision**: Design documents so agents can traverse them like a graph.

**How**:
- `_overview.md` links to `_architecture.md` and module indexes
- Each `_index.md` has `relates_to` and `depends_on` in frontmatter
- `_graph.json` provides programmatic lookup
- Agents read overview, decide what's relevant, follow links

**Why**:
- Not all knowledge is needed for every task
- Agents can find relevant context efficiently
- Mimics how humans navigate documentation

### 7. Frontmatter for Machine + Human Reading

**Decision**: Every markdown file has YAML frontmatter with structured metadata.

```yaml
---
module_id: auth
last_updated: 2024-01-15T10:30:00Z
last_commit: a3f2b1c
relates_to:
  - ../payments/_index.md
depends_on:
  - ../database/users.md
keywords: [oauth, jwt, session]
---
```

**Why**:
- Agents can parse frontmatter programmatically
- Humans can read the prose below
- Timestamps enable staleness detection
- Commit hashes enable git-based updates

### 8. Session Start Hook for Auto-Detection

**Decision**: Hook on `SessionStart` to check knowledge base status.

**Checks**:
1. Does knowledge folder exist? If not, prompt to create.
2. Is knowledge behind current commit? If so, prompt to update.

**Why**:
- Users shouldn't have to remember to update
- Proactive prompts prevent working with stale knowledge

---

## Planning System

### `/planning-with-james:plan`

The planning skill guides users through a structured 7-phase process:

```
Phase 0: Setup           → Create folder, copy templates, register plan
Phase 1: Context         → Gather problem description from user
Phase 2: Scoping         → Identify modules, set boundaries (knowledge-first)
Phase 3: Discovery       → Parallel deep exploration (Opus subagents)
Phase 4: Approach        → Decide on technical direction
Phase 5: Detailed Plan   → Full implementation specification
Phase 6: Task Breakdown  → Executable checklist with dependencies
Phase 7: Finalize        → Cold start doc ready for implementation
```

### Key Design Decisions

**Conversational Checkpoints**
Every phase has a checkpoint, and they all present findings and invite discussion. The point of planning is the conversation -- drawing context out of the user that isn't visible in code.

The system always stops at:
- Phase 1 (problem description) - Verify understanding
- Phase 2 (scoping) - "What modules should I consider? What constraints?"
- Phase 3 (discovery) - "Does this match your understanding? Is this data correct?"
- Phase 4 (approach) - "Do you see a better way? Any constraints I should know?"
- Phase 5 (detailed plan) - "Any concerns before I break this into tasks?"
- Phase 6 (task breakdown) - Last chance before implementation

**Why not auto-advance?** Initial designs allowed auto-advancing through phases when "scope was obvious" or "only one approach existed." In practice, this led to:
1. Discovery agents reporting wrong data (stale timing numbers) that went uncorrected
2. Approaches chosen without discussion when the user had better ideas
3. Constraints missed because the user was never asked

The user is the best source of truth for: real performance numbers, business constraints, historical context, edge cases from production, and their own mental model. The knowledge graph is a starting point, not ground truth.

**Speed comes from clarity, not from skipping steps.** A well-understood problem executes fast. A misunderstood problem wastes days.

**Problem Type Adaptation**
The system adapts its weights based on problem type:

| Problem Type | Discovery | Approach | Detailed Plan | Tasks |
|--------------|-----------|----------|---------------|-------|
| Bug | Heavy | Light | Light | Light |
| Feature | Medium | Heavy | Heavy | Medium |
| Refactor | Light | Medium | Heavy | Heavy |
| New System | Heavy | Heavy | Heavy | Medium |

**Scoping Before Discovery**
Phase 2 (Scoping) uses the knowledge graph to identify which modules are relevant BEFORE spawning discovery agents. This prevents unfocused exploration.

**Parallel Discovery with Synthesis**
Phase 3 spawns parallel Opus agents for different areas, then a synthesis agent combines findings. This is faster than sequential exploration and surfaces conflicts between areas.

**Cold Start Document**
`context_scratch_pad.md` is maintained throughout planning (not just at the end). If autocompact hits mid-planning, the document provides recovery context.

### Template Files

| Template | Purpose |
|----------|---------|
| `_plan_state.json` | Plugin state tracking |
| `context_scratch_pad.md` | Cold restart document |
| `problem_description.md` | User's problem/goal |
| `scope.md` | In/out scope, modules |
| `findings_area.md` | Per-area discovery |
| `findings_summary.md` | Synthesized discovery |
| `approach.md` | Options and decision |
| `detailed_plan.md` | Implementation spec |
| `tasks.md` | Executable checklist |

## Implementation System

### `/planning-with-james:go-time`

The go-time skill executes a plan's tasks one at a time with rigorous state tracking through an Orient-Execute-Record-Evaluate loop.

### Key Design Decisions

**Plan Lifecycle States**

Plans have explicit states that control both behavior and hook safety:

```
planning → planned → active ⇄ paused → completed
```

| State | Meaning | Who sets it |
|-------|---------|-------------|
| `planning` | Still in planning phases | `/plan` skill |
| `planned` | Planning complete, ready for implementation | `/plan` skill (Phase 7) |
| `active` | Currently being implemented | `/go-time` on activation |
| `paused` | Implementation paused, safe to do other work | `/go-time pause` or user request |
| `completed` | All tasks done | `/go-time` when final checklist passes |

Only ONE plan can be `active` at a time. Multiple plans can exist in various states simultaneously.

**Why explicit states matter**: The user raised a critical concern - what if you have an active plan but need to do a deployment? You don't want hooks reminding Claude about the plan during unrelated work. The active/paused distinction solves this: pausing sets `active_plan` to null, making the hook completely silent. The user can have 10 plans in various stages and freely switch between them.

**Plan Registry**

All plans tracked in `.claude/planning-with-james/plans/_registry.json`:

```json
{
  "active_plan": "SB-1112",
  "plans": {
    "SB-1112": {
      "name": "SB-1112 Return UNLOCODEs in RFQ Rate Search",
      "path": "/full/path/to/plan/folder",
      "created_at": "2026-02-04T12:00:00Z",
      "status": "active",
      "current_task": "2.1"
    }
  }
}
```

**Conditional PreToolUse Hook**

A `PreToolUse` prompt hook fires before `Edit` and `Write` operations. It checks the registry for an active plan:

- If no active plan: outputs nothing. Zero interference.
- If active plan exists: outputs one line like `[ACTIVE PLAN: SB-1112 | Task 2.1: Modify _transform_rate()]`

This is the re-orientation signal that prevents drift after autocompact. It's safe because:
1. Only fires when a plan is explicitly active (not paused)
2. Outputs nothing when paused or no plan exists
3. The user controls activation/pausing through the go-time skill
4. A 5-second timeout prevents any blocking

**Orient-Execute-Record-Evaluate Loop**

Every task follows this cycle:

1. **Orient**: Read `_plan_state.json`, `context_scratch_pad.md`, `tasks.md`, relevant section of `detailed_plan.md`. This happens before EVERY task, not just at session start.
2. **Execute**: Implement the task following the detailed plan. Lint check after changes.
3. **Record**: Check off task in `tasks.md`, update scratch pad, update plan state. This happens IMMEDIATELY - never deferred.
4. **Evaluate**: Did anything unexpected happen? Does the plan need adjustment? Is this a checkpoint?

**Why Orient happens every time**: After autocompact, Claude's memory is foggy but the files are pristine. By re-reading state files before every task (not just at session start), the system self-corrects continuously. The hook provides a lightweight nudge; the Orient phase provides the full re-orientation.

**Plan Deviations**

Three levels of deviation handling:
- **Additive** (add/skip tasks): Just do it, note in scratch pad, continue.
- **Tactical** (minor approach tweaks): Note in scratch pad, continue.
- **Strategic** (approach is wrong): STOP. Tell user. Do not continue until resolved.

The key principle: if a change affects the plan's *direction*, stop and ask. If it's *tactical*, document and continue.

**Checkpoints as Gates**

Checkpoints in `tasks.md` are hard gates. All verification checks must pass before proceeding. Failed checks get fixed in place, not deferred. This prevents small errors from compounding across phases.

---

## Epic System

### `/planning-with-james:epic`

An epic is a large initiative that breaks down into multiple sequential plans. Each sub-plan goes through the full plan/go-time lifecycle independently, but they share context through a coordination layer.

### Why Epics Exist

Some initiatives span weeks or months across dozens or hundreds of Claude Code sessions. A single plan can't handle this because:
- Context loss across sessions compounds (no single scratch pad survives months)
- Later phases depend heavily on what earlier phases discovered and decided
- Live validation (deploy, test with real data, measure costs) must happen between phases
- The plan itself evolves as earlier phases reveal reality

### Key Design Decisions

**The `learnings.md` File**

This is the single most important file in the epic system. It's a curated, growing document that accumulates architectural decisions, discoveries, and insights from every completed sub-plan. Any fresh Claude session reads this file and immediately has months of context.

Unlike a log or dump, `learnings.md` is selective. Each entry must be specific, actionable, and concise. It has sections for: architecture decisions (locked in), discoveries that changed the plan, assumptions proved wrong, technical debt introduced deliberately, patterns established, and risks materialized vs avoided.

**Epic Review Process**

After each sub-plan completes, an epic review runs before the next sub-plan starts. This is the context bridge. An Opus agent reads everything from the completed sub-plan and produces:
1. A `completion_review.md` - detailed record of what happened
2. New entries for `learnings.md` - distilled insights for future use
3. Revised outlines for remaining sub-plans based on what was learned

This ensures context carries forward, plans adapt to reality, and no session starts from scratch.

**Detail One, Outline the Rest**

When creating an epic, the first sub-plan gets a detailed outline (enough for `/plan` to use as a starting point). Later sub-plans get rough outlines only. This is deliberate - later plans depend so heavily on earlier results that detailed planning before those results exist is wasted work. Each epic review revises the outlines based on reality.

**Validation Gates Are Human-Driven**

Between sub-plans, the system defines validation gates (deploy to staging, measure costs, test with real data). These are not automated checks - they require human judgment and real-world observation. The system surfaces them and tracks their completion; the human executes them.

**Integration with Existing Skills**

The epic skill doesn't replace `/plan` or `/go-time` - it coordinates them:
- `/epic` creates the coordination layer and manages reviews
- `/plan` creates detailed plans for sub-plans, automatically reading epic context
- `/go-time` executes plans, reading `learnings.md` during ORIENT for epic sub-plans
- The registry links plans to their parent epic

### Epic Template Files

| Template | Purpose |
|----------|---------|
| `_epic_state.json` | Epic metadata, sub-plan tracking |
| `epic_context.md` | Business story, goals, constraints, history |
| `epic_plan.md` | Overall shape with sub-plan outlines |
| `learnings.md` | Accumulated wisdom across sub-plans |
| `validation_gates.md` | Human-driven verification between sub-plans |
| `completion_review.md` | Per-sub-plan review (written during epic review) |

---

## Knowledge Resilience System

### The Context Exhaustion Problem

The original update-knowledge skill launched ALL module update agents in parallel, then received all their outputs simultaneously. For large codebases, this filled the main context to capacity. When autocompact hit mid-update, Claude lost track of what was done and what wasn't, leaving the knowledge graph in an inconsistent state.

Three contributing factors:
1. **Parallel agent outputs**: N agents returning results at once = N times the context cost
2. **Preamble bloat**: Reading _overview.md, _graph.json, _architecture.md, and all module _index.md files into the main context before any work started consumed 30-40% of context before a single agent launched
3. **No resume capability**: When context was lost, there was no way to know what had completed

### Design Principles

**Lean Main Context**: The orchestrator reads only what it needs to plan (overview frontmatter, discovery.json, git diff). Subagents read their own module knowledge. The main context never holds the full knowledge graph.

**Linear Execution for Updates**: Process modules ONE AT A TIME. Spawn agent, receive brief JSON summary, update progress file, move to next. Context grows linearly by ~100 tokens per module (just the summary), not exponentially.

**Progress File as Lifeline**: `_update_progress.json` and `_create_progress.json` track every completed step. If autocompact hits, re-invoking the skill detects the in-progress file and resumes from exactly where it stopped. No work is lost.

**Brief Return Format**: Subagents write detailed work to disk files. They return ONLY a brief JSON summary to the main context. This is the key constraint that prevents context exhaustion.

### Why Creation Stays Parallel

`create-knowledge` keeps parallel execution because:
- There's no existing state to read (no preamble bloat)
- Subagents just explore code and write files (pure creation, no comparison)
- It worked successfully on real codebases

But it now has progress tracking for resilience. If interrupted, the resume logic checks which modules already have `_index.md` files and only processes the missing ones.

### Verify and Repair

The verify/repair modes provide ongoing knowledge maintenance:

**Verify** checks (no changes, no subagents):
- Module folder completeness (_index.md + _refs.json exist)
- Staleness (module last_commit vs overview last_commit)
- Codebase alignment (module paths still exist, no orphaned knowledge)
- Graph consistency (all edges reference valid nodes)
- Missing modules (codebase directories that should be modules)
- Interrupted updates (in-progress progress files)

**Repair** fixes everything verify finds, linearly with progress tracking:
- Orphaned modules: deleted
- Incomplete modules: re-indexed
- Stale modules: refreshed (or just commit hash updated if no code changed)
- Missing modules: created
- Broken graph: rebuilt from _refs.json files

This is designed to be run periodically as maintenance, or after a known failed update.

---

## Explore System

### `/planning-with-james:explore`

A lightweight, stateless skill for querying the knowledge graph during everyday work -- understanding how things work, impact analysis, PR review context.

### Why This Exists

The knowledge graph was originally only consumed during `/plan`. But the graph contains deep analysis of every module: boundaries, interfaces, patterns, dependencies, gotchas. That information is valuable for everyday work -- understanding unfamiliar code, assessing the blast radius of a change, reviewing PRs, tracing request flows. Without a dedicated skill, users had to either read the knowledge files manually or let Claude rediscover everything from source code.

### Stateless Design

Unlike `/plan` and `/go-time`, the explore skill has:
- No phases or phase progression
- No state files (`_plan_state.json`, etc.)
- No registry entries
- No hooks

It reads the knowledge graph, answers the question, and optionally saves findings. This keeps it fast and low-friction -- something you invoke mid-conversation without ceremony.

### Query Type Classification

The skill classifies queries into four types (Understanding, Impact, Flow, Diff Impact) to determine what knowledge to load and how to structure the answer. This prevents loading the entire graph for a simple "how does X work" question, and ensures impact/flow queries traverse the graph edges properly.

Default is Understanding, which covers the most common case: "tell me about this module."

### Saved Explorations

Findings can optionally be saved to `.claude/planning-with-james/explorations/`. These are timestamped markdown files with frontmatter (query, type, modules involved). They serve as a lightweight record of what was explored and when, useful for:
- Reviewing what you looked at before starting a plan
- Sharing exploration context with teammates
- Picking up where you left off in a new session

The directory is created lazily on first save.

### Future Consideration: Knowledge Discovery Hook

A natural extension is a `PreToolUse` hook on `Glob` and `Grep` that passively nudges Claude toward the knowledge graph whenever it's about to search source code. The hook would check if a knowledge graph exists and output a brief reminder like: "A knowledge graph exists for this repo. Consider checking `.claude/planning-with-james/knowledge/` first."

This was deferred from the initial implementation to keep scope focused. The hook would need careful gating (only fire when knowledge exists, only fire once per session or per-query, don't fire during knowledge creation/updates) to avoid being annoying.

---

## Future Considerations

### Depth Levels

Consider allowing configurable depth:
- Quick index (shallow, fast)
- Standard index (current behavior)
- Deep index (exhaustive, expensive)

For now, one depth level (thorough) aligns with "cost is not a concern" philosophy.

---

## Lessons Learned

### Autocompact Survival

The file-based approach provides autocompact resilience. When context is compacted:
1. The skill instructions are summarized
2. But written files persist on disk
3. Progress files enable exact resume
4. Agent can read state to understand what's done and what's next

This is a strong argument for file-based workflows over pure conversation state. Progress tracking makes this explicit rather than relying on implicit file existence checks.

### Parallel vs Linear Subagents

Launching multiple Task agents in parallel is critical for initial indexing (Phase 2 of create-knowledge), where there's no existing state to load and context cost is just the agent outputs.

But for updates, parallel execution is dangerous. Each update agent needs to read existing knowledge, compare it to source code, and return results -- all of which fills context. Linear execution with brief return summaries keeps context growth predictable and bounded.

---

## User's Original Vision

From initial discussion:

> "I would want for my plugin to have full knowledge of the codebase it works in. A 'pre' tool that collects data comprehensively. You would run this the first time you use the plugin, or update it whenever new code was implemented. This would be the '10,000 foot view' but could have more detailed branch documents to explain individual components. It would act as a knowledge repository, a starting point when beginning a plan."

> "The system should be able to traverse a graph of documents. An overview and architecture as the index for all modules, each of which could have its own folder and indexes - so an agent could traverse the knowledge graph as a 'choose your own adventure' as necessary."

> "When an update is called, we should have memory about when we updated last. Store it in the overview - read when we updated last and what commit that was. Check github for all commits since, and use that information to update all documents."

This plugin is the implementation of that vision.
