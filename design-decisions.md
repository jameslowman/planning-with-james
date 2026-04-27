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
| `/planning-with-james:dig` | Dig into the knowledge graph for understanding, impact, flows |
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

### 3. Recursive Module Discovery

**Decision**: Phase 2 agents classify each module as LEAF or CONTAINER after reading the code. Containers produce children that get classified in the next wave. Repeat until everything is a leaf.

**The Problem**: Flat module discovery is too shallow for complex codebases. In a monorepo where `parsers/` contains `ocean_fcl/` which contains `email/`, `attachment/excel/`, `attachment/pdf/` -- you either treat "parsers" as one module (too shallow, mixing unrelated concerns) or you require Phase 1 to enumerate every nested directory without reading code (impractical, since you can't tell depth boundaries from directory structure alone).

**Why Phase 2 Decides**: The agent reading the actual code is the one with context to judge. Phase 1 sees directories; Phase 2 sees imports, interfaces, and domain boundaries. A directory that *looks* like a single module from the outside might reveal three distinct sub-domains once you read the code. Only the deep-dive agent has that information.

**"When in doubt, lean LEAF"**: Over-splitting creates busywork. Under-splitting loses some granularity but the knowledge is still there -- a leaf module can document sub-areas in its prose and detail files. The cost of a false container (spawning extra agents, managing extra hierarchy) is higher than the cost of a false leaf (slightly less granular module boundaries).

**Layer-Organized vs Domain-Organized Codebases**: The original LEAF/CONTAINER heuristic worked well for domain-organized code (Python backends with `auth/`, `parsers/`, `rfq/`) but produced false LEAFs for large frontend applications. Frontend codebases typically organize by technical layer first (`components/`, `pages/`, `hooks/`, `utils/`) with domain boundaries one level deeper (`pages/quotes/`, `pages/crm/`, `components/shipments/`). The Phase 2 agent would see layer directories, classify them as "implementation details, not separate domains," and mark the entire app as a LEAF -- even when it contained hundreds of files spanning many distinct business domains. The fix: explicit guidance in the subagent prompt to look *inside* layer-based directories for domain boundaries, and a size signal (50+ files) that triggers more careful scrutiny before classifying as LEAF. The split should be by domain (quotes, crm, shipments), not by layer (pages, components, hooks).

**How Waves Work**:
- **Wave 0**: Phase 1 identifies top-level modules, all marked `module_type: "pending"`
- **Wave 1**: Parallel agents classify each pending module. Leaves write full docs. Containers write lightweight docs and return children.
- **Wave 2+**: New children (also `pending`) get classified in the next wave. Repeat until no pending modules remain.
- Within each wave, agents run in **parallel** (independent modules). Across waves, execution is **sequential** (children depend on parent classification).

**Graph Representation**:
- `_graph.json` nodes include `module_type` ("leaf" or "container") and `parent` (null for top-level)
- A `"contains"` edge type links containers to their children -- these are derived from `_discovery.json` hierarchy, not from import analysis
- Cross-cutting edges (`imports`, `calls`, `shares_types`, `depends_on`) connect any modules regardless of hierarchy level. A leaf can import from a sibling, a cousin, or a top-level module.

**Flat Folder Structure**: All modules live in `modules/{module-id}/` regardless of hierarchy depth. The hierarchy exists in `_discovery.json` (via `parent` and `children` fields) and `_graph.json` (via `contains` edges), not in the filesystem. This keeps paths simple and avoids deeply nested folder structures.

**Worked Example**:

```
Codebase:
  parsers/
    ocean_fcl/
      email/
      attachment/
        excel/
        pdf/
    air/

Wave 0 (Phase 1 output):
  _discovery.json modules:
    - {id: "parsers", path: "parsers/", module_type: "pending", parent: null, children: []}
    - {id: "air-parser", path: "parsers/air/", module_type: "pending", parent: null, children: []}

Wave 1 (Phase 2, first pass):
  Agent for "parsers" reads code → too many distinct domains → CONTAINER
    Returns: {type: "container", children: [{id: "ocean-fcl", path: "parsers/ocean_fcl/"}]}
    Writes lightweight _index.md (overview, children list, shared patterns)
  Agent for "air-parser" reads code → cohesive unit → LEAF
    Returns: {type: "leaf"}
    Writes full _index.md, _refs.json, detail files

  _discovery.json updated:
    - parsers: module_type: "container", children: ["ocean-fcl"]
    - air-parser: module_type: "leaf"
    - ocean-fcl: module_type: "pending", parent: "parsers" ← NEW

Wave 2:
  Agent for "ocean-fcl" reads code → still distinct sub-domains → CONTAINER
    Returns: {type: "container", children: [{id: "ocean-fcl-email", ...}, {id: "ocean-fcl-attachment", ...}]}

Wave 3:
  Agent for "ocean-fcl-email" → LEAF
  Agent for "ocean-fcl-attachment" → CONTAINER (excel and pdf are different enough)
    Returns children: [{id: "ocean-fcl-attachment-excel", ...}, {id: "ocean-fcl-attachment-pdf", ...}]

Wave 4:
  Agent for "ocean-fcl-attachment-excel" → LEAF
  Agent for "ocean-fcl-attachment-pdf" → LEAF

Done. All modules classified. Max depth: 4.
```

### 4. Cascading Graph Updates

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

### 5. Multi-Mode Update Command

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

### 6. Clarification Before Execution

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

### 7. Knowledge-First Principle

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

### 8. Choose-Your-Own-Adventure Navigation

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

### 9. Frontmatter for Machine + Human Reading

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

### 10. Session Start Hook for Auto-Detection

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

The planning skill guides users through a structured 10-phase process:

```
Phase 0: Setup              → Create folder, copy templates, register plan
Phase 1: Context            → Gather problem description and user flows
Phase 2: Scoping            → Identify modules, set boundaries (knowledge-first)
Phase 3: Discovery          → Parallel deep exploration (Opus subagents)
Phase 4: Test Architecture  → Map user flows to test scenarios
Phase 5: Approach           → Decide on technical direction
Phase 6: Approach Stress Test → Critique the approach + find what already exists
Phase 7: Detailed Plan      → Full implementation specification
Phase 8: Task Breakdown     → Executable checklist with dependencies
Phase 9: Finalize           → Cold start doc ready for implementation
```

### Key Design Decisions

**Conversational Checkpoints**
Every phase has a checkpoint, and they all present findings and invite discussion. The point of planning is the conversation -- drawing context out of the user that isn't visible in code.

The system always stops at:
- Phase 1 (problem description + user flows) - Verify understanding
- Phase 2 (scoping) - "What modules should I consider? What constraints?"
- Phase 3 (discovery) - "Does this match your understanding? Is this data correct?"
- Phase 4 (test architecture) - "Do these test scenarios capture the behavior you described?"
- Phase 5 (approach) - "Do you see a better way? Any constraints I should know?"
- Phase 6 (stress test) - "Here's what a fresh pair of eyes found — flaws in the approach and existing code we should leverage."
- Phase 7 (detailed plan) - "Any concerns before I break this into tasks?"
- Phase 8 (task breakdown) - Last chance before implementation

**Why not auto-advance?** Initial designs allowed auto-advancing through phases when "scope was obvious" or "only one approach existed." In practice, this led to:
1. Discovery agents reporting wrong data (stale timing numbers) that went uncorrected
2. Approaches chosen without discussion when the user had better ideas
3. Constraints missed because the user was never asked

The user is the best source of truth for: real performance numbers, business constraints, historical context, edge cases from production, and their own mental model. The knowledge graph is a starting point, not ground truth.

**Speed comes from clarity, not from skipping steps.** A well-understood problem executes fast. A misunderstood problem wastes days.

**Problem Type Adaptation**
The system adapts its weights based on problem type:

| Problem Type | Discovery | Approach | Stress Test | Detailed Plan | Tasks |
|--------------|-----------|----------|-------------|---------------|-------|
| Bug | Heavy | Light | **Heavy** | Light | Light |
| Feature | Medium | Heavy | **Heavy** | Heavy | Medium |
| Refactor | Light | Medium | **Heavy** | Heavy | Heavy |
| New System | Heavy | Heavy | Medium | Heavy | Medium |

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
| `planned` | Planning complete, ready for implementation | `/plan` skill (Phase 9) |
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

### Background Agents Need Explicit Turn Budgets

When v2.1.2 switched subagents to `run_in_background: true`, no `max_turns` was specified. Background agents default to a turn budget that's insufficient for deep research tasks. An Opus agent reading multiple knowledge files, then source files, then analyzing patterns can easily consume 15-20 turns before reaching the Write tool calls for findings and result files. The agent "completes" (hits its turn limit) with research done but no files written -- the orchestrator's polling loop times out finding 0 result files.

The fix: explicitly set `max_turns: 30` on all background Task agent launches. This gives agents enough headroom to read extensively AND write their output files. The affected launches are `create-knowledge` Phase 2 (deep dive agents) and `plan` Phase 3 (discovery agents).

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

## Dig System (formerly Query, originally Explore)

### `/planning-with-james:dig`

A lightweight, stateless skill for digging into the knowledge graph during everyday work -- understanding how things work, impact analysis, PR review context.

Originally named `/explore`, renamed to `/query` to avoid collision with Claude Code's built-in Explore agent type, then renamed again to `/dig` for clarity and brevity.

### Why This Exists

The knowledge graph was originally only consumed during `/plan`. But the graph contains deep analysis of every module: boundaries, interfaces, patterns, dependencies, gotchas. That information is valuable for everyday work -- understanding unfamiliar code, assessing the blast radius of a change, reviewing PRs, tracing request flows. Without a dedicated skill, users had to either read the knowledge files manually or let Claude rediscover everything from source code.

### Stateless Design

Unlike `/plan` and `/go-time`, the dig skill has:
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

## Concurrent Sessions

### The Problem

The original registry had a single `active_plan` pointer at the root level. Two terminals running `/go-time` simultaneously would fight over this pointer -- one would activate its plan, the other would overwrite it, and the PreToolUse hook would surface the wrong plan's context.

### Solution: Per-Session Plan Binding

Replace the singleton `active_plan` with a `sessions` map keyed by `$CLAUDE_SESSION_ID` (available in SKILL.md via substitution, and in hooks via stdin JSON's `session_id` field).

**Registry format change:**
```json
{
  "sessions": {
    "session-abc": { "active_plan": "SB-1133", "bound_at": "..." },
    "session-def": { "active_plan": "SB-1288", "bound_at": "..." }
  },
  "plans": {
    "SB-1133": { "locked_by_session": "session-abc", ... },
    "SB-1288": { "locked_by_session": "session-def", ... }
  }
}
```

Each terminal gets its own slot. No contention.

### Plan-Level Locking

The `locked_by_session` field on each plan prevents two terminals from working on the same plan simultaneously, which would cause file conflicts in `_plan_state.json`, `tasks.md`, and `context_scratch_pad.md`. If a plan is locked by another session, go-time refuses to activate it.

### Session Cleanup

Sessions are ephemeral. Stale entries accumulate when terminals close without pausing. The go-time skill prunes entries with `bound_at` older than 24 hours on every invocation, clearing their locks and setting affected plans to `"paused"`.

### Hook Session Awareness

Hooks receive `session_id` in their stdin JSON (not as an environment variable). The compact recovery hook and PreToolUse reminder hook both parse this from stdin with `jq` and look up the session-specific plan binding. If `session_id` is unavailable (older Claude Code versions), hooks exit silently -- safe degradation.

### What Changed

| Component | Before | After |
|-----------|--------|-------|
| Registry root | `active_plan: "id"` | `sessions: { sid: { active_plan: "id" } }` |
| Plan entry | (no lock field) | `locked_by_session`, `locked_at` |
| Hooks | Read `active_plan` directly | Parse `session_id` from stdin, look up `sessions[sid]` |
| Constraint | One active plan globally | One active plan per session, one session per plan |
| Pause/Complete | Set `active_plan` to null | Remove session entry, clear plan lock |

### Seamless Switching

Running `/go-time SB-1288` while SB-1133 is active in the same session automatically pauses SB-1133 (unbinds, unlocks) before activating SB-1288. No need to explicitly pause first.

---

## Lessons System

### Why Lessons Exist

The epic system has `learnings.md` to carry context across sub-plans. But there was no equivalent for standalone plans or for project-wide patterns. A user correction in one plan ("that API is rate-limited, don't call it in a loop") would be lost by the next plan. Fresh sessions kept repeating the same mistakes.

The lessons system fills this gap with a two-tier design: per-plan capture and project-level accumulation.

### Two-Tier Design

**Per-plan `lessons.md`**: Created alongside other plan files in Phase 0. Captures corrections, discoveries, effective patterns, and mistakes during planning and implementation. Sections: User Corrections, Discoveries, What Worked, Mistakes to Avoid. These are specific to the plan's problem and context.

**Project-level `.claude/planning-with-james/lessons.md`**: Accumulated from all plans. Created lazily on first promotion (not pre-created). Sections: Patterns & Conventions, Mistakes to Avoid, What Works Well. These are generalized -- plan-specific references are rewritten for broader applicability.

The two tiers serve different purposes. Per-plan lessons are raw and immediate -- useful during that plan's implementation. Project-level lessons are curated and general -- useful for any future plan.

### Capture Moments

Lessons are captured at natural pause points where the user provides feedback:

- **Plan checkpoints (Phases 2-6)**: After incorporating user feedback, if the feedback contained a correction, a reusable codebase insight, or context about what failed before, an entry is added to the per-plan `lessons.md`. Not every correction is a lesson -- only insights that would help a fresh agent on a different plan.
- **Go-time EVALUATE**: If a task reveals something unexpected about the codebase, it gets noted in both the scratch pad and `lessons.md` (Discoveries section).
- **Go-time CHECKPOINTS**: The checkpoint report includes a count of lessons captured during that phase.

### Consumption Points

Lessons are read at the start of work to inform decisions:

- **Plan Phase 0 (Step 9)**: Project-level lessons are read before planning begins. They inform scoping, discovery, and approach phases.
- **Plan Phase 3 (Discovery subagents)**: If project-level lessons mention any of the modules being investigated, those entries are included in the subagent prompt.
- **Go-time ORIENT (item 5)**: Per-plan lessons are read before every task. Project-level lessons are read on first orient of the session.
- **Epic review**: Per-plan lessons from the completed sub-plan are included in the synthesis agent's file list.

### Promotion Mechanics

Per-plan lessons are promoted to project-level at two points:

1. **Go-time COMPLETE**: After all tasks pass, before updating scratch pad. Each per-plan lesson is evaluated: promote if it's about the codebase itself, would help a fresh agent, or corrects a common assumption. Skip if it's specific to this plan's problem, already in the knowledge graph, or too narrow. Promoted entries are rewritten for generality.

2. **Epic review (Step 5)**: After updating the epic's `learnings.md`, universally-applicable lessons from the sub-plan's `lessons.md` are also promoted to project-level.

### Curation Strategy

The project-level file must stay scannable. When it exceeds ~200 lines of content: merge redundant entries, remove entries about deleted code, keep it useful. The file is curated on every promotion, not in a separate maintenance step.

### Elegance Check

Added to go-time CHECKPOINTS as step e2, between diff review and read-back verification. A brief self-check: is the solution the simplest it could be, or did complexity creep in? Catches "just in case" additions, premature abstractions, and over-engineering before they stick.

If the code could be simpler without losing correctness or clarity, simplify it now. Re-run lint and tests after any simplification. The checkpoint report includes the elegance check result.

### Relationship to Epic Learnings

The lessons system and epic `learnings.md` are complementary, not overlapping:

- **Epic `learnings.md`**: Carries context within an epic -- architecture decisions locked in by earlier sub-plans, assumptions proved wrong, patterns established for consistency across sub-plans. Focused on the epic's initiative.
- **Per-plan `lessons.md`**: Captures corrections and discoveries during a single plan's lifecycle. Raw material that feeds both epic learnings and project-level lessons.
- **Project-level `lessons.md`**: Generalized insights applicable across all plans regardless of epic membership. The broadest scope.

Epic reviews consider both: sub-plan lessons that are epic-specific go into `learnings.md`, while universally-applicable ones go to project-level `lessons.md`.

---

## Test-First Architecture

### Why Test-First

The planning system produced great plans, but testing was an afterthought -- a small table in `detailed_plan.md` and some tasks tacked onto the end of `tasks.md`. This led to:

1. Tests written after implementation confirmed the code worked as written, not that it solved the original problem
2. For bugs, no verification that the fix actually addressed the user's reported behavior
3. Mock boundaries decided ad-hoc during implementation instead of deliberately during planning
4. Existing test gaps went unanalyzed -- nobody asked "why don't the current tests catch this?"

The fix: make testing a first-class planning phase that produces a user-reviewable document before implementation begins.

### Two-Part Design

**Part 1: User Flow Extraction (Phase 1 Enhancement)**

Phase 1 (Context Gathering) now includes structured questioning to extract step-by-step user flows. These are plain English sequences like:

```
1. User opens email in RFQ
2. User sees no un-validated addresses
3. User clicks "Source Rates" button
4. Expected: button is active, rate sourcing begins
5. Actual: button is greyed out, no action possible
```

For bugs, at least one flow is required before proceeding. For features, flows describe the intended user journey. These flows are stored in a new **User Flows** section of `problem_description.md` and become the source material for test scenarios.

**Part 2: Phase 4 - Test Architecture (New Phase)**

A new phase between Discovery (Phase 3) and Approach (Phase 5, formerly Phase 4). This placement is deliberate:

- **After Discovery**: Agents need to know the code to map user flows to specific functions, identify mock boundaries, and analyze existing test coverage.
- **Before Approach**: The test strategy should inform the approach, not the other way around. An approach that's hard to test is a bad approach. The approach decision now reads `test_plan.md` as an input.

### Phase 4 Agents

Three specialized background agents investigate the test landscape in parallel:

1. **Test Infrastructure Agent**: Framework, fixtures, helpers, mocking patterns, CI integration. Answers: "how does this codebase test?"

2. **Existing Coverage Agent**: Maps existing tests to user flows, identifies gaps, explains why current tests don't catch the bug. Answers: "why don't existing tests catch this?"

3. **Mock Boundary Agent**: Traces each user flow through the code path, identifies where mocks should be placed based on the user's test preference. Answers: "what gets mocked and what gets tested directly?"

For hard-to-track bugs (user can't provide clean reproduction steps), additional investigation agents are launched to trace suspected code paths and identify fault locations. The difficulty escalates effort, not reduces it.

### The `test_plan.md` Document

The synthesis agent combines all findings into `test_plan.md` -- the central artifact of this phase. Key design decisions:

**Plain English test descriptions**: Every test is described in human-readable language so the user can review it. Example:

```
1. **Rate search returns UNLOCODEs for valid origin** —
   Verify that when searching rates with a valid origin port,
   the response includes the origin UNLOCODE in each rate result.
   - Setup: Create mock rate data with known UNLOCODE values
   - Action: Call rate_search(origin="USNYC", ...)
   - Assert: Each result in response has origin_unlocode == "USNYC"
   - Mock boundary: Mock the database layer, test the service logic
```

This is the test plan's most important property. Code-level test specifications can be wrong in ways non-engineers can't spot. Plain English can be validated by anyone who understands the problem.

**User co-authorship**: The Phase 4 checkpoint is a hard stop where the user reviews every test scenario. They can add scenarios ("also test what happens when the UNLOCODE is null"), correct mock boundaries ("don't mock the cache layer, that's where the bug lives"), and remove unnecessary tests. This conversation produces better tests than any automated analysis.

**Test preference as user config**: Stored in `.claude/planning-with-james/config.json`, the `test_preference` setting (`"mock"`, `"integration"`, or `"mixed"`) controls how mock boundaries are determined. Default is `"mock"` -- fully mocked tests that run fast and work reliably in CI/CD. Users who prefer integration tests change this once and all future plans adapt.

### Task Ordering: Tests Before Implementation

Phase 8 (Task Breakdown, formerly Phase 6) now mandates that Phase 1 of every task list is "Test Foundation" -- writing all tests from `test_plan.md` before any implementation begins:

```
Phase 1: Test Foundation
  - Task 1.1: Set up test scaffolding and fixtures
  - Task 1.2: Write tests for Scenario 1
  - Task 1.3: Write tests for Scenario 2
  ...
  - CHECKPOINT: All tests written. For bugs: tests FAIL (proving the bug exists)

Phase 2: Implementation
  ...

Phase N: Verification
  - Task N.1: Verify all tests from Phase 1 now PASS
```

For bugs, the Phase 1 checkpoint verifies that tests **fail** -- this proves the tests actually capture the bug. If the tests pass before implementation, they're testing the wrong thing.

### Phase Renumbering

The new phase shifts everything after Discovery:

| Before (v2) | After (v3) | After (v3.1) |
|-------------|------------|--------------|
| Phase 4: Approach | Phase 5: Approach | Phase 5: Approach |
| — | — | Phase 6: Approach Stress Test (new) |
| Phase 5: Detailed Plan | Phase 6: Detailed Plan | Phase 7: Detailed Plan |
| Phase 6: Task Breakdown | Phase 7: Task Breakdown | Phase 8: Task Breakdown |
| Phase 7: Finalize | Phase 8: Finalize | Phase 9: Finalize |

### Problem Type Adaptation

Test Architecture weight varies by problem type:

| Type | Weight | Why |
|------|--------|-----|
| Bug | **Heavy** | Extra agents for hard-to-track bugs. Deep gap analysis. Every flow must have a failing test. |
| Feature | Medium | Map user journey flows to test scenarios. Establish the test foundation. |
| Refactor | Medium | Regression guards are critical -- prove existing behavior doesn't change. |
| New System | Medium | Less existing test infrastructure to analyze, but flows guide the test foundation. |

---

## Approach Stress Test (v3.1.0 — New Phase)

### The Problem: Two Compounding Biases

Two biases compound in agentic planning and neither is addressed by the existing phases:

1. **Confirmation bias**: The agent that chose the approach in Phase 5 has investment in it. It won't naturally look for reasons it's wrong. The user checkpoint helps, but the user knows the business — they may not catch a technical flaw hiding three call stacks deep.

2. **Build-new bias**: Agentic AI prefers creating new code over leveraging what exists. Discovery (Phase 3) maps the problem area, but runs *before* an approach is decided. Once the approach is locked, nobody asks "given this specific approach, what already exists that serves it?"

Real example: In one plan, the approach was "fix the wrong URL string." A critique agent with fresh eyes found that the codebase already had account-type detection in the E2E tests — the real problem wasn't "wrong string" but "not detecting which string to use." The approach was solving a symptom, not the root cause.

### Phase 6: Approach Stress Test (Two Agents, One Phase)

A new phase between Approach (Phase 5) and Detailed Plan (Phase 7). Two agents run in parallel:

**Agent A (Critique)** — a skeptical reviewer with no stake in the approach:
- Verifies assumptions against actual code (does it really work that way?)
- Walks through edge cases (different user types, environments, concurrent execution)
- Looks for simpler alternatives (is the approach over-engineering?)
- Assesses risk (blast radius, rollback options, breaking changes)
- Classifies findings as BLOCKING, CONCERN, or NOTE

**Agent B (Reuse)** — searches the codebase through the lens of the approach:
- Finds functions/utilities that already do what the approach needs (or 80%+ of it)
- Identifies patterns the approach should follow rather than reinvent
- Spots code that's almost right (fix > rebuild)
- Flags reuse risks (looks reusable but isn't)
- Classifies assets as REUSE, EXTEND, ADAPT, or AVOID

This placement is deliberate:
- **After Approach**: Both agents need to know *what* we're building before they can critique it or search for existing assets. Running before the approach is decided would be unfocused.
- **Before Detailed Plan**: Both agents' findings directly shape the implementation spec. Blocking issues get resolved before we invest in detailed planning. Reuse findings turn "build new" into "extend existing." Concerns get addressed in the plan rather than discovered during implementation.

### The Output Documents

**`critique_report.md`**: Verdict, blocking issues (with evidence and suggestions), concerns, notes, simpler alternatives, and a risk summary with blast radius and rollback options.

**`reuse_analysis.md`**: Reuse score, asset inventory (REUSE/EXTEND/ADAPT/AVOID with file paths), patterns to follow, a Build-New List (the *only* things that should appear as new code), and impact on the approach.

### How It Feeds Into Later Phases

Phase 7 (Detailed Plan) must read both documents and:
- Address all CONCERN-level findings from the critique
- Incorporate reuse recommendations — preferring REUSE/EXTEND over building new
- Note any BLOCKING findings that were resolved and how

Phase 8 (Task Breakdown) also reads both — critique concerns may need dedicated tasks, and reuse decisions affect task size and dependencies.

### Blocking Issue Resolution

If the critique agent finds BLOCKING issues, they're presented to the user *before* the full checkpoint. The response can be:
- Amend the approach (update approach.md, note what prompted it)
- Return to Phase 5 if the approach is fundamentally flawed
- Re-run only the affected agent if the amendment is substantial

### Problem Type Adaptation

| Type | Critique Weight | Reuse Weight | Why |
|------|----------------|--------------|-----|
| Bug | **Heavy** | Medium | Bugs often stem from wrong assumptions. Critique verifies the approach targets root cause, not symptom. Existing code is often "almost right." |
| Feature | **Heavy** | **Heavy** | Highest risk of both over-engineering and unnecessary new code. Full treatment on both fronts. |
| Refactor | Medium | **Heavy** | Approach is usually sound for refactors, but reuse analysis is critical — the whole point is restructuring what exists. |
| New System | Medium | Light | Less existing code to reuse, but critique catches architectural assumptions early when stakes are highest. |

### Phase Renumbering

This phase shifts everything after Approach:

| Before (v3.0) | After (v3.1) |
|----------------|--------------|
| Phase 6: Detailed Plan | Phase 7: Detailed Plan |
| Phase 7: Task Breakdown | Phase 8: Task Breakdown |
| Phase 8: Finalize | Phase 9: Finalize |

---

## Pre/Post User Story Narrative (v3.5.0)

### The Problem: Bullet-Point User Flows Flatten the Mental Model

Phase 1's user-flow extraction was a Q&A — "walk me through the steps you take to trigger this." The output was a numbered list with `Expected:` and `Actual:` at the failure point. Mechanically correct, structurally clean, easy for Phase 4 test agents to consume.

The list also missed the things that actually break alignment between user and planner:

- **Side effects within a step** — "she glances back at the addin and notices the spinner is gone, even though the PDF is still being prepared, but figures it must be ready in the other tab." A list flattens this to "spinner disappears."
- **Interruptions and context switches** — phone calls, switching browser tabs, opening another email mid-flow. These are the conditions under which silent failures happen, but they don't fit the "what does the user do step by step" frame.
- **Consequences that surface later** — "Twenty minutes later her customer asks where the quote is." The bug isn't visible at the moment of failure; it's visible when the consequence catches up.
- **Things the protagonist doesn't notice** — half the bug is the silent failure mode. A list of "what the user does" can't capture inattention.

A user looking at a numbered list can verify the mechanics without ever asking "is this how this actually plays out for the people doing it?"

### The Fix: Scene-by-Scene Narrative with a Cast

Phase 1 now authors a **cast-and-scenes user story** with the user, scene-by-scene. The cast names the protagonist (a real user with a role for bug/feature work; "the developer", "the system", or "the calling service" for refactor/infra). Each scene is a 1–4 sentence-per-step narrative that interleaves protagonist action and system response, calls out what the protagonist doesn't notice, and carries consequences forward.

Pattern: identify the cast, write Scene 1 (main path), present it, iterate until confirmed, then ask whether Scene 2 should cover an interruption / variant load / failure recovery, write that, confirm, repeat up to 4 scenes total. **One scene confirmed at a time.** The constraint forces the planner to pick scenes that actually carry signal — not just enumerate every possible flow.

The narrative goes in a new artifact: `user_story.md`.

### Why Both Narrative AND Structured Flows Exist

The narrative serves humans (alignment, mental-model check, Linear comment). The structured user-flow list serves machines (Phase 4 test architecture agents map flow steps to code paths via mock-boundary analysis — narrative form doesn't decompose cleanly to that).

Rather than ask the user to author both, the planner authors the narrative *with* the user, then mechanically derives the structured flow list into `problem_description.md`. Single source of truth, two formats, no duplicate authoring.

### Post-Implementation Half + IS / IS-NOT (Phase 9)

The full pre/post user story can't be written in Phase 1 — POST needs the approach decided, and "It is NOT" needs the deferred items known. So Phase 9 closes the loop: write a Post Scene N for each Pre Scene N, with the same protagonist, same starting situation, but showing the new behavior. Then write the IS / IS-NOT section sourced from `detailed_plan.md`, `critique_report.md`, `reuse_analysis.md`, and `approach.md`'s out-of-scope items.

The pre/post checkpoint at the end of Phase 9 is the strongest test of whether the plan actually solves the problem. If the user reads the post-scenes and they don't feel right, the plan has drift somewhere upstream — the narrative form makes the drift visible in a way that tasks-list review doesn't.

### Linear Posting (Optional, Best-Effort)

If the user provided a Linear ticket URL during Phase 1, Phase 9 offers to post the full user story as a comment on the ticket. Linear MCP availability is best-effort — if not configured or auth-expired, the planner tells the user where the file lives and skips. No blocking, no auth prompts during finalization.

### Why Not Replace the Whole Phase 1 Q&A

Phase 1 still gathers problem type, summary, success criteria, references, and theories up front. The narrative is *one part* of Phase 1, focused on the user's path through the system. The non-narrative questions (success criteria, references) don't benefit from scenes.

### Trade-offs Accepted

- **Slower Phase 1**: scene-by-scene confirmation roughly 2-3x's Phase 1 time vs. a single user-flow Q&A. The savings show up downstream — plans don't get derailed in Phase 5 or Phase 7 because a missing context surfaced at scene-confirmation time.
- **Cast can feel awkward for refactors**: the generic protagonist fallback ("the developer", "the system") works but is less vivid. Acceptable cost for keeping the same structure across all problem types.
- **Two phases author the same artifact**: `user_story.md` is written by Phase 1 and updated by Phase 9. Each phase only writes the half it has the inputs for. The frontmatter `post_filled_in` flag keeps state explicit.

---

## REM Sleep System (v3.4.0 — New Skill)

### The Problem: Accumulated Weight

Every skill in this plugin produces artifacts, and none of them clean up after themselves. After a few months of real use:

- `_registry.json` gathers completed plans, paused plans, and sessions that never got pruned (session pruning exists but only hits 24-hour-old entries)
- Per-plan folders persist with full scratch pads, findings, and detailed plans even when the plan is long done
- Project-level `lessons.md` grows past the "curate at 200 lines" rule and nobody enforces it
- The knowledge graph stays technically current (commit hashes match after `update-knowledge` runs) while drifting semantically — a module's Purpose paragraph describes what it used to do, not what the code now does
- `/dig` explorations pile up in `explorations/`
- Progress files from interrupted `create-knowledge` or `update-knowledge` runs linger with `in_progress` status

Left alone, the plugin becomes slow to load (registry parsing, cross-plan checks), noisy to search, and quietly inaccurate (stale knowledge misleads planning).

### The Analogy: Sleep

Real brains consolidate memory overnight. Short-term memory moves to long-term; redundant traces merge; abandoned pursuits lose salience; patterns emerge from the day's episodes. REM sleep specifically is when dreaming happens — generative, sometimes weird, noticing things the waking brain didn't.

`/rem` maps this to four phases:

1. **Triage** — the plugin's drowsy survey. Walk every artifact source, categorize into `obvious-dead`, `probably-archive`, `needs-review`, and `dream-seeds`. Touch nothing. Write `_rem_triage.json`.
2. **Prune** — slow-wave sleep analog. Act on the obvious-dead: archive completed/paused plans past retention, old explorations, stale sessions, dangling locks, completed progress files. Safe, low-judgment work.
3. **Consolidate** — memory consolidation. Judgment-heavy merges and promotions, batched for user approval.
4. **Dream** — REM analog. A team of agents with different lenses wanders the repo and each other's observations, producing dream journal entries and action items.

The metaphor isn't load-bearing, but it's useful — it predicts the right sequence (cheap survey, safe cleanup, careful judgment, generative exploration) and names the parts memorably.

### Key Design Decisions

**Bifurcated output: dreams vs action items.**

Early brainstorm had rem producing a single "dream journal." But some findings are urgent ("the hapag-scrapers module is grossly out of date and three recent plans rediscovered this"), while others are leisurely ("there's a naming drift between `getUserData` and `fetchUserData` across four modules"). Bundling them would either make dreams naggy or make urgent items lose context.

Split them:
- **Dreams** live in `dreams/{date}/` — journal entries, no lifecycle, no expiration. Read when curious.
- **Action items** live in `action_items/` — files with frontmatter (priority, type, target, recommended_action, status). They have an inbox (`_index.md`) and a `resolved/` folder. Each item prescribes a specific command (usually `/update-knowledge guided` with suggested `$ARGUMENTS`).

The Weaver (one of the Dream team agents) is the only one authorized to promote a dream finding to an action item. This keeps individual lenses focused on discovery.

**Archive, never delete; registry stubs for findability.**

Rem moves retired artifacts to `.claude/planning-with-james/archive/`. Every move is recorded in `_manifest.json`. The user restores with `mv` — no `/rem restore` command (the code isn't worth it for a rare operation).

The subtle part is plans. `/plan` checks the registry on every invocation to see if the user's request matches existing in-progress work. If rem archived a plan entirely from the registry, that lookup would miss prior work. Solution: **registry stubs**. When a plan is archived, its registry entry is replaced with a slim stub (~5 fields: name, status, archived_at, archive_path, problem_summary). `/plan` can still fuzzy-match against the name and problem summary to offer: "you're planning something similar to SB-1112 (archived) — want to resurrect its findings?" The heavy content (scratch pads, findings) stays in the archive folder and is only read if the user opts in.

**Never archive a plan with unpromoted lessons.**

`go-time` COMPLETE promotes per-plan lessons to project level. Paused or abandoned plans never ran COMPLETE, so their lessons are stuck. Before archiving any such plan, rem sweeps its `lessons.md` and promotes anything that belongs at project level. Silent archiving would lose hard-won context.

**Rem never writes to `knowledge/modules/`.**

One writer per artifact. `create-knowledge` and `update-knowledge` own the knowledge graph. Rem's role is to **observe and recommend**. When the staleness auditor (Consolidate Phase 3) or the Cartographer (Dream Phase 4) finds drift, they produce action items recommending `/update-knowledge guided` with specific context. The user runs the command — rem doesn't.

This mirrors how `/dig` is read-only against the graph. It keeps ownership clean and makes rem safe to run frequently.

**Manual trigger only. No hooks, no scheduling.**

An initial design considered a SessionStart hook nagging when rem hadn't run in N days, or wiring into `go-time COMPLETE`. Both were rejected:
- Hooks would run rem in contexts where the user is focused on something else
- Auto-scheduling competes with the user's mental model of when maintenance happens
- Rem can be heavy (especially Dream), so opportunistic runs might surprise

The user runs `/rem` when they feel the plugin getting heavy. Simple.

**Dream team uses `TeamCreate` for peer messaging.**

Every other multi-agent phase in the plugin uses fire-and-forget + `_result.json` polling, because those agents are doing embarrassingly parallel work (one module, one agent). Dreaming is different: the value is in cross-pollination. An Archaeologist finds an abandoned module and asks the Historian "any plan reference this?" The Historian points to a paused plan, the Archaeologist reads its scratch pad and writes a note neither could have written alone.

`TeamCreate` + `SendMessage` supports this directly. The Team is destroyed after the run. The design-decisions.md already flagged Teams as a deferred future use — Dreams is where it finally earns its place.

**Four cruisers + one weaver.**

The lenses:
- **Archaeologist** — git history, churn, abandonment. What's been dug up and reburied?
- **Cartographer** — knowledge graph vs code. Where does the map lie?
- **Historian** — reads completed + paused + archived plans. What themes keep coming back?
- **Detective** — weirdness hunter. TODOs, orphans, duplicates, config sprawl, Sentry errors.
- **Weaver** — reads the four lens outputs and team transcript. Produces the digest and decides which findings become action items. Can create action items from cross-lens convergences (when two lenses independently noticed the same thing, the signal is real).

Separation of discovery (cruisers) from synthesis (weaver) prevents any single lens from biasing toward action.

**MCP data sources are opt-in and best-effort.**

Detective (and optionally Archaeologist) can pull from Sentry, Vercel, or GCP if those MCPs are configured. `rem-config.json` has a `data_sources` block with `"auto"` / `"on"` / `"off"` per source. `"auto"` means: try, and if the MCP errors or requires reauthorization, log "skipped {source}: unavailable" and move on.

This is important because Sentry's MCP auth prompt currently fires on every use — running rem with forced Sentry access would produce auth prompts mid-dream, defeating the unattended-run design.

**30 days across the board for retention.**

Initial proposal was 90 days for explorations. The user pushed back — "that's pretty long for how fast we move." One retention window (30 days, configurable) applied to completed plans, abandoned paused plans, and explorations makes the system predictable without tuning.

"Last activity" matters more than "created_at". A plan created 60 days ago but whose scratch pad was touched 5 days ago is still active. Rem computes last-activity as the max of `_plan_state.json`'s `last_updated`, registry updates, and the mtime of `context_scratch_pad.md`.

**Batched consolidation approvals.**

When Phase 3 agents produce proposals (lesson merges, promotions, staleness findings), rem groups them by category and asks the user to approve each group as a batch with `AskUserQuestion`. Individual approval is tedious at volume; bulk approve-all-or-nothing is too coarse. Grouped-by-category with "apply all / review each / skip all" per group is the compromise.

### What Rem Does NOT Do

- Does not run on a schedule
- Does not write to `knowledge/modules/`
- Does not delete (except for genuinely ephemeral files like completed progress files)
- Does not take destructive action during consolidation without user approval
- Does not prompt the user for MCP reauthorization mid-run
- Does not archive plans with unpromoted lessons

### Future Considerations

- **Rem-aware `/plan`**: When `/plan` checks the registry, it could also fuzzy-match against archived stubs and say "you're planning something that sounds like SB-1112 (archived). Want to resurrect?" Small addition, big UX win.
- **Tunable dream lenses**: Let `rem-config.json` specify which lenses to spawn. Some users might only want Detective (runtime weirdness) or Cartographer (knowledge audit).
- **Cross-repo dreams**: If the plugin ever supports multiple codebases in one install, dreams could notice patterns across repos.
- **Rem-generated lesson promotions audit**: Weaver could notice "the same lesson has been promoted from three different plans" and merge them.

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

### No Bash for File Modifications

Agents (both orchestrators and subagents) must use the Edit and Write tools for all file modifications. Bash is restricted to git commands and running project tools (linters, test runners).

**The problem**: During `update-knowledge repair` with legacy migration, the orchestrator and subagents were using Bash for-loops and sed commands to batch-update `_discovery.json` entries and module frontmatter. Every Bash command triggers a permission prompt in Claude Code that requires manual user approval. With 80+ modules to classify, the user had to press Enter 80-90 times every 20 minutes -- making it impossible to run unattended.

**Why Edit/Write are different**: Claude Code auto-approves Edit and Write tool calls for project files without requiring manual confirmation. Bash commands go through a stricter permission flow because they can execute arbitrary shell commands. By ensuring all file modifications go through Edit/Write, the entire knowledge indexing and repair pipeline can run unattended.

**Where enforced**: A NON-NEGOTIABLE rule at the top of both `create-knowledge` and `update-knowledge` skills, plus a `## CRITICAL` section in every subagent prompt template. Both layers are needed because the orchestrator can also make this mistake, and subagents don't read the skill's top-level rules.

### Agent Permission Mode (bypassPermissions)

Background agents spawned by the plugin's skills were silently failing. The failure mode:

1. Skills spawn agents with `run_in_background: true`
2. Background agents pre-approve permissions at spawn time
3. Any tool call NOT pre-approved is **auto-denied silently** -- the agent doesn't error or prompt, the tool call just returns a failure and the agent continues
4. The agent burns through its `max_turns` doing research (Read/Glob/Grep work fine) but every Write/Edit call fails silently
5. The agent "completes" having done all its research but written zero output files
6. The orchestrator polls for `_result.json` files that will never exist
7. After 20 minutes (the timeout), the user discovers all agent work was lost

**The fix**: Every agent spawn across all skills must include `mode: "bypassPermissions"`. This skips permission checks entirely so background agents can actually write their output files.

**Why `bypassPermissions` and not other modes**:
- `default`: prompts for permission -- impossible for background agents with no one to approve
- `acceptEdits`: auto-accepts Edit/Write but may still block Bash (needed for git commands)
- `dontAsk`: auto-denies tools not explicitly pre-approved -- the WORST option for background agents
- `plan`: read-only -- agents can't write at all
- `auto`: AI-based classifier, enterprise-only
- `bypassPermissions`: skips all checks -- the only mode that guarantees background agents can write

**Security posture**: The agents are already constrained by their prompts (only write to `.claude/planning-with-james/`, only use Bash for git commands). The permission mode is a safety net that was inadvertently blocking the designed behavior. `bypassPermissions` restores the intended agent capabilities.

**Where enforced**: A NON-NEGOTIABLE rule at the top of every skill that spawns agents (`create-knowledge`, `update-knowledge`, `plan`, `epic`), plus `mode: "bypassPermissions"` on every explicit agent spawn parameter list. Both layers are needed -- the NON-NEGOTIABLE catches any agent spawns that don't have explicit parameter lists (e.g., "same prompt as Standard Mode Phase 2" references).

**Future consideration -- Teams**: Claude Code has an experimental Teams system (TeamCreate) that provides a more robust coordination model: shared task lists, peer-to-peer messaging between agents, and automatic notifications instead of filesystem polling. This could replace the fire-and-forget + poll pattern for parallel agent phases (create-knowledge Phase 2 waves, plan Phase 3 discovery, plan Phase 4 test architecture). Teams would eliminate the polling loop entirely and provide better failure detection. Deferred until the permission fix is validated.

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
