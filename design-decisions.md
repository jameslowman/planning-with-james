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

The planning skill guides users through a structured 7-phase process:

```
Phase 0: Setup              → Create folder, copy templates, register plan
Phase 1: Context            → Gather problem description and user flows
Phase 2: Scoping            → Identify modules, set boundaries (knowledge-first)
Phase 3: Discovery          → Parallel deep exploration (Opus subagents)
Phase 4: Test Architecture  → Map user flows to test scenarios
Phase 5: Approach           → Decide on technical direction
Phase 6: Detailed Plan      → Full implementation specification
Phase 7: Task Breakdown     → Executable checklist with dependencies
Phase 8: Finalize           → Cold start doc ready for implementation
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
- Phase 6 (detailed plan) - "Any concerns before I break this into tasks?"
- Phase 7 (task breakdown) - Last chance before implementation

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
| `planned` | Planning complete, ready for implementation | `/plan` skill (Phase 8) |
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

Phase 7 (Task Breakdown, formerly Phase 6) now mandates that Phase 1 of every task list is "Test Foundation" -- writing all tests from `test_plan.md` before any implementation begins:

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

| Before | After |
|--------|-------|
| Phase 4: Approach | Phase 5: Approach |
| Phase 5: Detailed Plan | Phase 6: Detailed Plan |
| Phase 6: Task Breakdown | Phase 7: Task Breakdown |
| Phase 7: Finalize | Phase 8: Finalize |

### Problem Type Adaptation

Test Architecture weight varies by problem type:

| Type | Weight | Why |
|------|--------|-----|
| Bug | **Heavy** | Extra agents for hard-to-track bugs. Deep gap analysis. Every flow must have a failing test. |
| Feature | Medium | Map user journey flows to test scenarios. Establish the test foundation. |
| Refactor | Medium | Regression guards are critical -- prove existing behavior doesn't change. |
| New System | Medium | Less existing test infrastructure to analyze, but flows guide the test foundation. |

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
