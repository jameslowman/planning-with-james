# Release Notes

## v3.4.0

### New: `/rem` — sleep-cycle maintenance

Planning-with-james accumulates weight over time. The registry gathers completed and paused plans that never get cleaned up. `lessons.md` grows past the "curate at 200 lines" threshold and nobody curates it. Knowledge graph modules stay technically current (commit hash matches) while drifting semantically (the "Purpose" paragraph describes code the module used to be). Explorations pile up. Progress files from interrupted runs linger.

`/rem` runs a four-phase sleep cycle to address this:

1. **Triage**: survey registry, plans, lessons, knowledge staleness signals, explorations, epic learnings, and existing action items. Categorize into obvious-dead, probably-archive, needs-review, dream-seeds. Write `_rem_triage.json`.

2. **Prune**: auto-archive completed plans > 30 days, abandoned/paused plans > 30 days, explorations > 30 days, completed progress files. Sweep unpromoted lessons from plans *before* archiving so nothing is lost. Leave registry stubs for archived plans so `/plan` can still find prior work by name. Everything is `mv`-recoverable; the user restores manually.

3. **Consolidate**: three parallel opus agents propose (a) lesson merges and stale-reference removals, (b) forgotten-lesson promotions from paused/abandoned plans, (c) knowledge-graph staleness action items. User approves by batch.

4. **Dream**: a Team of five opus agents — Archaeologist (git/churn), Cartographer (knowledge-vs-code), Historian (plan archaeology), Detective (weirdness hunting), and Weaver (synthesizer) — wanders the repo, messages each other as they find things in each other's lanes, and produces dream notes + action items. Optional MCP data sources (Sentry, Vercel, GCP) are tried if configured; skipped gracefully otherwise.

Sub-commands (`/rem triage`, `/rem prune`, `/rem consolidate`, `/rem dream`, `/rem status`) run individual phases. Default `/rem` runs everything.

**Key invariants**:
- Rem never writes to `knowledge/modules/` — stale modules become action items recommending `/update-knowledge guided`.
- Rem never archives a plan with unpromoted lessons — it promotes first.
- Archive is append-only with a manifest; user restores with `mv`.
- Triggered manually. No hooks, no scheduling.

**New artifacts**:
- `.claude/planning-with-james/rem-config.json` — retention windows, data-source toggles
- `.claude/planning-with-james/dreams/{date}/` — journal entries per sleep session
- `.claude/planning-with-james/action_items/` — inbox with `_index.md` and per-item files
- `.claude/planning-with-james/archive/` — everything rem retires, with `_manifest.json`
- Registry gets `status: "archived"` stubs (5 fields) for findability

---

## v3.3.0

### Optimization: Sonnet for pure aggregation agents

Two agents move from opus to sonnet. Both perform deterministic data aggregation — they read structured inputs that opus subagents already produced and emit structured outputs with a defined schema. Their output quality is bounded by the inputs, not by the aggregating agent's reasoning, so sonnet should be indistinguishable from opus here.

**Changed**:
- `create-knowledge` Phase 3 (connectivity mapping): reads all `_refs.json` files, cross-references imports/exports, builds `_graph.json` and `_architecture.md`.
- `update-knowledge` Phase 4a (graph rebuild): reads all `_refs.json` files and `_discovery.json`, rebuilds `_graph.json` with containment edges.

**Unchanged** (still on opus): all judgment-heavy agents — Phase 2 wave classification (the leaf/container decision sets the graph's quality ceiling), plan Phase 3 discovery, plan Phase 4 test architecture, plan Phase 6 critique and reuse (explicitly designed as "fresh skeptical eyes"), epic review synthesis, and update-knowledge cascade verification (its "no changes needed" answer is the common path — a model that skews toward it would silently rot the graph over time).

**Rationale**: The plugin's original philosophy was "cost is not a concern, depth is the goal." Recent Claude Code limit changes make cost a real factor, so we're selectively downgrading the agents where opus is most obviously overkill — schema-in, schema-out transformations that don't benefit from opus reasoning capacity. Starting conservative and measuring before expanding further.

---

## v3.2.0

### Fix: Background agents silently failing (permission mode)

Background agents spawned by all skills were silently losing their work. The agents would complete their research (Read/Glob/Grep) but every Write/Edit call was auto-denied by the permission system — they'd burn through their turn budget and "complete" without writing any output files. The orchestrator would poll for result files that never existed, eventually timing out after 20 minutes with all agent work lost.

**Root cause**: No agent spawn in any skill specified a `mode` parameter. Background agents pre-approve permissions at spawn time and auto-deny anything not pre-approved. Without an explicit permissive mode, Write/Edit calls failed silently.

**Fixed**: All 22 agent spawn points across 5 skills now include `mode: "bypassPermissions"`. A NON-NEGOTIABLE rule is added to every skill that spawns agents as a safety net for agent spawns that reference other prompts.

**Affected skills**: `create-knowledge` (Phase 2 wave agents, Phase 3 connectivity), `update-knowledge` (all modes — standard, guided, repair), `plan` (Phase 3 discovery, Phase 3 synthesis, Phase 4 test agents, Phase 4 synthesis, Phase 6 stress test agents), `epic` (review synthesis).

---

## v3.0.0

### New: Test-First Architecture

Planning now includes a dedicated Test Architecture phase that maps user flows to concrete test scenarios *before* any implementation begins. Tests are described in plain English so users can review and co-author them.

**What changed:**

- **Phase 1 (Context Gathering)**: New structured questioning extracts step-by-step user flows — reproduction steps for bugs, user journeys for features. These are stored in a new "User Flows" section of `problem_description.md`. For bugs, at least one concrete flow is required.

- **Phase 4 (Test Architecture)** — NEW PHASE: Three parallel background agents investigate the test landscape:
  1. **Test Infrastructure**: framework, fixtures, mocking patterns, CI config
  2. **Existing Coverage**: what tests exist, why they don't catch this bug
  3. **Mock Boundaries**: traces user flows through code paths, recommends where to mock

  For hard-to-track bugs, additional investigation agents are launched. Output is `test_plan.md` — every test described in plain English with setup, action, assertion, and mock boundary.

- **Phase 8 (Task Breakdown)**: Phase 1 of every task list is now "Test Foundation" — writing all tests from `test_plan.md` before implementation. For bugs, the checkpoint verifies tests *fail* (proving the bug is captured).

- **User preferences**: New `.claude/planning-with-james/config.json` supports `test_preference` (`"mock"` / `"integration"` / `"mixed"`, default `"mock"`) and `test_first` (default `true`).

- **Phase renumbering**: Phases 4-7 are now 5-8. The planning system has 9 phases (0-8).

**New template**: `test_plan.md` — plain English test scenarios with mock strategy, gap analysis, and file locations.

**Affected skills**: `plan` (major — new phase, enhanced Phase 1, updated task breakdown), `go-time` (minor — ORIENT reads test_plan.md for test tasks, checkpoints verify test-first compliance).

---

## v2.1.4

### Fix: Background subagents exhausting turns before writing files

Background agents launched in v2.1.2 had no `max_turns` specified. Deep research agents (Opus) would consume their entire default turn budget reading knowledge files and source code, then "complete" without ever reaching the Write tool calls for findings and result files. The orchestrator's polling loop would time out finding 0 result files.

**Fixed**: All background Task agent launches now specify `max_turns: 30`, giving agents enough headroom to read files AND write results.

**Affected skills**: `create-knowledge` (Phase 2 wave agents), `plan` (Phase 3 discovery agents).

---

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
