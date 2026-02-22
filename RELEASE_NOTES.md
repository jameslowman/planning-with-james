# Release Notes

## v2.1.2

### Fix: Context exhaustion from parallel subagents

After switching subagents to `general-purpose` in v2.1.1, their richer output overwhelmed the orchestrator's context window when many agents ran in parallel (e.g., 28 modules in a single wave). The main context hit its limit before autocompact could even run.

**Fixed**: Subagents now run in the background (`run_in_background: true`) and write a small `_result.json` file to disk when done. The orchestrator polls for these files with a single Bash wait loop (20-minute timeout) — zero context cost while waiting. Results are read from disk only after all agents finish.

**Affected skills**: `create-knowledge` (Phase 2 wave agents), `plan` (Phase 3 discovery agents).

---

## v2.1.1

### Bug Fix: Subagents couldn't write files

Subagents in `create-knowledge`, `plan`, and `epic` were spawned as `"Explore"` type, which doesn't have access to Write/Edit tools. This caused all Phase 2 deep-dive agents to fail when writing `_index.md` and `_refs.json`, triggering expensive re-launches as general-purpose agents.

**Fixed**: All agents that need to write files now use `subagent_type: "general-purpose"` with `model: "opus"`. The only skill that correctly uses Explore is `dig`, where agents return findings conversationally without writing.

**Affected skills**: `create-knowledge` (Phase 2 + Phase 3), `plan` (Phase 3 discovery + synthesis), `epic` (review synthesis).

---

## v2.0.0

### Recursive Module Discovery

The knowledge graph now understands hierarchy. Instead of treating every top-level directory as a flat module, the indexer recursively classifies modules as **leaf** (cohesive unit -- gets full documentation) or **container** (distinct sub-domains -- gets a lightweight overview, children become their own modules).

This runs in waves: Phase 1 identifies top-level modules, Phase 2 agents read the code and decide leaf or container, containers produce children for the next wave, repeat until everything is a leaf. Within each wave, agents run in parallel.

**What changed:**

- **`create-knowledge`**: Phase 2 rewritten as iterative wave-based deep dives with leaf/container classification. `_discovery.json` now tracks `module_type`, `parent`, and `children` per module. `_graph.json` includes `contains` edges and hierarchy fields on nodes. `_overview.md` shows hierarchy tree and leaf/container stats.

- **`update-knowledge`**: File-to-module mapping uses deepest match (assigns to the most specific leaf, not the container). Update agents report `structure_changed` when a module's classification should change. Verify mode checks hierarchy consistency (valid parents, no orphans, no circular refs). Repair mode fixes hierarchy problems and can upgrade flat (v1.x) knowledge graphs to hierarchical via legacy migration.

- **`/dig`** (renamed from `/query`): Containers show overview + children list with offer to dive deeper. Leaves with a parent load parent context. Impact traversal follows containment edges (siblings, children of affected containers). Diff impact uses deepest-match logic.

- **`design-decisions.md`**: New section documenting the recursive discovery design, including a worked example.

### Rename: `/query` -> `/dig`

The query skill is now `/planning-with-james:dig`. Shorter, clearer, and better describes what it does -- digging into the knowledge graph to understand code, trace impact, or review changes.

### Upgrading from v1.x

Existing knowledge graphs work as-is. To upgrade to hierarchical format:
```
/planning-with-james:update-knowledge repair
```
This detects the flat format, classifies each module as leaf or container, creates children for containers, and rebuilds the graph with containment edges. No need to re-index from scratch.

---

## v1.1.0

### New: `/query` skill (renamed to `/dig` in v2.0, originally `/explore`)

Query the knowledge graph for everyday work without starting a full planning session. Ask how something works, check the blast radius of a change, trace a flow through the system, or review what your current diff affects.

Four query types:
- **Understanding**: "how does auth work" -- synthesizes module purpose, key files, interfaces, patterns, gotchas
- **Impact**: "what breaks if I change the rate module" -- traverses the dependency graph 2 hops out, builds an impact map
- **Flow**: "trace a request from API to database" -- walks the graph path, describes what enters/happens/exits at each step
- **Diff Impact**: "what's affected by my current changes" -- maps uncommitted changes to modules, classifies interface vs internal changes, identifies downstream risk

Stateless by design: no phases, no state files, no registry entries. Optionally saves findings to `.claude/planning-with-james/explorations/`.

---

## v1.0.0

A Claude Code plugin for deep codebase understanding and structured feature development.

### What it does

Builds a knowledge graph of your codebase so Claude starts from documented understanding instead of rediscovering everything through grep. Plans features through structured conversation with checkpoints, producing artifacts that survive context loss.

### Skills

| Skill | Purpose |
|-------|---------|
| `/create-knowledge` | Index your codebase into a knowledge graph (modules, interfaces, patterns, dependencies) |
| `/update-knowledge` | Incremental updates after commits, with verify and repair modes |
| `/plan` | Structured planning through phases: context, scoping, discovery, approach, detailed plan, tasks |
| `/go-time` | Execute a plan with active task tracking and context resilience |
| `/epic` | Coordinate multiple related plans as a single initiative |

### Key Design Decisions

- **Knowledge-first**: Claude reads the knowledge graph before touching source code
- **Conversation over speed**: Every planning checkpoint invites user input rather than racing ahead
- **Context resilience**: Scratch pads and state files let work survive autocompact
- **Linear updates, parallel creation**: Prevents context exhaustion during knowledge updates
- **Hooks**: Session start warnings for stale knowledge, post-compact reminders for active plans

### Install

```
/plugin marketplace add jameslowman/planning-with-james
/plugin install planning-with-james@james-plugins
```
