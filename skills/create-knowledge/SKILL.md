---
name: create-knowledge
description: Generate comprehensive codebase knowledge graph for planning. Run this on first use or for full re-index.
disable-model-invocation: true
allowed-tools: Bash, Glob, Grep, Read, Write, Task, Edit
---

# Create Knowledge Base

**NON-NEGOTIABLE: ALL PATHS ARE RELATIVE TO THE REPO ROOT**
All `.claude/planning-with-james/` paths in this skill are relative to the **current working directory** (the repo you're working in), NOT `~/.claude/`. The knowledge graph lives inside the project, not in your home directory. If you're unsure, run `pwd` to confirm you're in the repo root.

**NON-NEGOTIABLE: PROGRESS TRACKING**
After each phase completes, you MUST update `_create_progress.json` BEFORE starting the next phase. This file enables resume after context loss. If autocompact hits mid-creation, re-invoking this skill will pick up where you left off.

**NON-NEGOTIABLE: AGENT PERMISSIONS**
ALL agents you spawn (Task tool) MUST include `mode: "bypassPermissions"`. Background agents that hit permission prompts will have their Write/Edit calls **silently denied** — they burn through turns doing research but never write output files. The orchestrator then polls for result files that will never exist. This destroys all agent work. Always include this parameter on every agent spawn, background or foreground.

**NON-NEGOTIABLE: NO BASH FOR FILE MODIFICATIONS**
Do NOT use Bash to modify knowledge files. No `sed`, `awk`, `for` loops, `echo` redirection, or `cat` heredocs for writing or editing files. ALWAYS use the Read → Edit or Read → Write tools for file changes. Bash is ONLY permitted for: `git` commands (`git diff`, `git rev-parse`, `git log`) and running project tools (linters, test runners). This applies to you (the orchestrator) AND to all subagents you spawn — include this rule in every subagent prompt. Violating this blocks unattended operation with permission prompts that require manual approval.

You are about to create a comprehensive knowledge base for this codebase. This is a multi-phase process that will deeply analyze the code and create a navigable knowledge graph.

## Step 0: Resume Detection

BEFORE anything else, check for an in-progress creation:

```
Read .claude/planning-with-james/knowledge/_create_progress.json
```

- **If file exists and `status` is `"in_progress"`**:
  Resume from where it stopped based on `phase`:
  - `"discovery"`: Phase 1 didn't complete. Check if `_discovery.json` exists. If yes, skip to Phase 2. If no, re-run Phase 1.
  - `"deep_dives"`: Phase 2 was interrupted. Check `_discovery.json` for modules with `module_type: "pending"` — these still need classification. Also check for modules that have `_index.md` files but whose agent may not have returned (the file exists but the module isn't in `modules_classified`). Resume at the current wave. Inform user: "Resuming interrupted knowledge creation. Wave {N}, {classified} of {total} modules classified."
  - `"connectivity"`: Phase 3 was interrupted. Re-run Phase 3 (it's idempotent).
  - `"finalization"`: Just run the finalization phase.

- **If file exists and `status` is `"completed"`**: Delete it, inform user knowledge base already exists. Suggest update-knowledge instead.

- **If file does not exist**: Proceed normally.

## Setup Phase

First, set up the knowledge directory structure:

1. Create the knowledge directory if it doesn't exist:
   ```
   .claude/planning-with-james/knowledge/
   ```

2. Add `.claude/planning-with-james/` to the project's `.gitignore` if not already present. Create `.gitignore` if it doesn't exist.

3. Create the initial directory structure:
   ```
   .claude/planning-with-james/knowledge/
   ├── _overview.md       (will be written at the end)
   ├── _graph.json        (will be written at the end)
   ├── _architecture.md   (will be written in Phase 3)
   └── modules/           (will be populated in Phase 2)
   ```

## Phase 1: Structural Discovery

Perform a broad analysis of the codebase to understand its structure. Do this yourself (not with subagents).

Investigate:
- Directory structure and organization patterns
- Package files (package.json, requirements.txt, Cargo.toml, go.mod, etc.)
- Entry points (main files, index files, app files)
- Configuration files (env examples, config files, docker files)
- Test structure and patterns
- Documentation that exists

Create a mental map of:
- What are the major modules/domains/features?
- What technology stack is used?
- What are the boundaries between components?
- Are there multiple services/packages/deployments?

**Output of Phase 1**: Create a file `.claude/planning-with-james/knowledge/_discovery.json` with this structure:

```json
{
  "tech_stack": {
    "languages": ["typescript", "python"],
    "frameworks": ["nextjs", "fastapi"],
    "databases": ["postgres"],
    "infrastructure": ["docker", "kubernetes"]
  },
  "modules": [
    {
      "id": "module-slug",
      "name": "Human Readable Name",
      "path": "src/modules/feature",
      "description": "Brief description of what this module does",
      "type": "feature|service|library|config|infrastructure",
      "module_type": "pending",
      "parent": null,
      "children": []
    }
  ],
  "entry_points": [
    {
      "file": "src/index.ts",
      "purpose": "Main application entry"
    }
  ],
  "investigation_notes": "Any important observations about the codebase structure"
}
```

**Identify modules at the highest natural boundary.** Look for top-level domains, services, and packages -- don't try to enumerate every nested directory. If `parsers/` is a top-level domain, list `parsers/` as one module even if it contains sub-directories. Phase 2 agents will read the code and decide whether a module is a leaf (cohesive unit → full documentation) or a container (distinct sub-domains → lightweight overview, children get their own modules). Let Phase 2 handle depth.

The three new fields on each module entry:
- `module_type`: Always `"pending"` in Phase 1. Phase 2 agents will classify as `"leaf"` or `"container"`.
- `parent`: Always `null` for top-level modules discovered in Phase 1. Set to parent module ID for children discovered in Phase 2.
- `children`: Always `[]` initially. Populated when a Phase 2 agent classifies a module as `"container"`.

**After completing Phase 1**, write the progress file:

```json
// .claude/planning-with-james/knowledge/_create_progress.json
{
  "started_at": "{ISO timestamp}",
  "commit": "{current git HEAD}",
  "phase": "deep_dives",
  "total_modules": {count from _discovery.json},
  "modules_indexed": [],
  "modules_classified": [],
  "current_wave": 0,
  "status": "in_progress"
}
```

## Phase 2: Deep Dives (Iterative Waves)

Phase 2 classifies each module as LEAF (cohesive unit → full documentation) or CONTAINER (distinct sub-domains → lightweight overview, children get their own modules). This runs in waves: each wave processes all pending modules in parallel, containers produce children for the next wave, and waves repeat until everything is a leaf.

### Wave Loop

```
while there are modules with module_type: "pending" in _discovery.json:
  1. Collect all pending modules
  2. Launch ALL pending modules as background agents (fire-and-forget)
  3. Wait for all agents to finish by polling for _result.json files on disk
  4. Read each _result.json and process:
     - LEAF: mark as classified, verify files written
     - CONTAINER: mark as classified, add children as new pending modules
  5. Re-launch any agents that failed (missing _result.json after timeout)
  6. Delete _result.json files after processing
  7. Update _discovery.json with results
  8. Update _create_progress.json (increment current_wave, update modules_classified)
  9. Next wave
```

### Subagent Launch

For EACH pending module, spawn a Task agent with:
- `subagent_type`: "general-purpose" (NOT "Explore" — Explore agents cannot write files)
- `model`: "opus" (we want maximum depth)
- `mode`: "bypassPermissions" (agents MUST bypass permission prompts or Write/Edit calls fail silently)
- `run_in_background`: true
- `max_turns`: 30 (background agents need enough turns to read files AND write results)

**IMPORTANT**: Launch ALL pending module subagents for the current wave in a SINGLE message with multiple Task tool calls. All agents run in the background — do NOT use TaskOutput to read their results. Agents write their results to disk.

### Waiting for Agents

After launching all agents, wait for them to complete by polling for `_result.json` files:

```bash
EXPECTED={number of pending modules}; TIMEOUT=1200; ELAPSED=0; KNOWLEDGE=".claude/planning-with-james/knowledge"
while [ $(find "$KNOWLEDGE/modules" -name "_result.json" -newer "$KNOWLEDGE/_create_progress.json" 2>/dev/null | wc -l) -lt $EXPECTED ] && [ $ELAPSED -lt $TIMEOUT ]; do
  sleep 15; ELAPSED=$((ELAPSED + 15))
done
```

This blocks with zero context cost. When it returns, use Glob to find all `modules/*/_result.json` files and Read each one. These are tiny JSON files — negligible context.

**If some agents failed** (fewer `_result.json` files than expected after timeout): identify missing modules and re-launch just those, same fire-and-forget pattern with a fresh wait.

### Subagent Prompt

Use this prompt template (fill in the module details):

```
You are analyzing the "{module_name}" module located at "{module_path}".
{if parent: "This is a child of the '{parent_name}' container module."}

Your task is to create documentation for this module AND decide whether it is a LEAF or CONTAINER.

## CRITICAL: No Bash for file modifications
Use the Write tool to create files and the Edit tool to modify files. Do NOT use Bash commands (sed, awk, for loops, echo) to create or modify files. Bash is only for git commands.

## Investigation Steps

1. Read all significant files in this module's directory
2. Understand the purpose, patterns, and structure
3. Decide: Is this a LEAF or a CONTAINER?

**LEAF** = a cohesive unit of functionality. The code here works together toward one purpose. Sub-directories (if any) are implementation details, not separate domains.

**CONTAINER** = this directory houses multiple distinct sub-domains that deserve their own module documentation. The sub-domains have different purposes, different interfaces, or different domain concepts.

**When in doubt, lean LEAF.** Over-splitting creates busywork. A leaf module can document sub-areas in its prose and detail files. Only classify as CONTAINER when the sub-domains are genuinely distinct.

**Watch for hidden domains in layer-organized code.** Some codebases (especially frontends like Next.js, React, Vue) organize by technical layer first (components/, pages/, hooks/, utils/, lib/) rather than by domain. If the top-level sub-directories are layer-based, look INSIDE them for domain boundaries. A pages/ directory containing quotes/, crm/, shipments/, analytics/ has distinct business domains — the layer organization doesn't make them "implementation details." If the module contains multiple distinct business domains organized under technical layers, classify as CONTAINER and split by domain (not by layer). For example, a Next.js app with pages for quoting, CRM, shipments, and settings should split into domain-based children (quotes, crm, shipments, settings), not layer-based children (pages, components, hooks).

**Size is a signal.** If a module has 50+ files or spans many distinct features/workflows, scrutinize more carefully before classifying as LEAF. Large modules with multiple independent feature areas are likely CONTAINERs even if they share a framework or entry point.

## If LEAF: Create Full Documentation

### 1. Module Index: `.claude/planning-with-james/knowledge/modules/{module_id}/_index.md`

```markdown
---
module_id: {module_id}
module_name: {module_name}
module_type: leaf
parent: {parent_id or null}
path: {module_path}
last_updated: {ISO timestamp}
last_commit: {current git commit hash}
external_refs: []  # Will be filled in Phase 3
internal_deps: []  # Other modules this depends on (by module_id)
keywords: []       # Searchable terms
---

# {module_name}

## Purpose
[2-3 sentences on what this module does and why it exists]

## Key Files
| File | Purpose |
|------|---------|
| path/to/file.ts | Description |

## Public Interface
[What does this module export/expose? APIs, functions, components, types]

## Internal Patterns
[Notable patterns, conventions, or architectural decisions]

## Dependencies
[What other modules does this depend on?]

## Notes for Future Agents
[Anything an agent should know when working with this module]
```

### 2. Detail Files (as needed)

If the module is complex, create additional files in the module folder:
- `api.md` - API endpoints or public functions
- `data-models.md` - Data structures and types
- `flows.md` - Key workflows or processes
- `config.md` - Configuration options

Use your judgment on what detail files are needed.

### 3. References File: `.claude/planning-with-james/knowledge/modules/{module_id}/_refs.json`

```json
{
  "imports_from": ["list", "of", "file", "paths", "imported"],
  "imported_by": [],  // Leave empty, filled in Phase 3
  "calls_to": ["external/api/paths"],
  "shared_types": ["TypeName"],
  "notes": "Any observations about cross-module relationships"
}
```

### 4. Write Result File

After writing all documentation files, write your classification result to:
`.claude/planning-with-james/knowledge/modules/{module_id}/_result.json`

```json
{"module_id": "{module_id}", "type": "leaf"}
```

This file signals to the orchestrator that you are done. Do NOT skip this step.

## If CONTAINER: Create Lightweight Documentation

### 1. Module Index: `.claude/planning-with-james/knowledge/modules/{module_id}/_index.md`

```markdown
---
module_id: {module_id}
module_name: {module_name}
module_type: container
parent: {parent_id or null}
path: {module_path}
children: []  # Will be populated by the orchestrator
last_updated: {ISO timestamp}
last_commit: {current git commit hash}
external_refs: []  # Will be filled in Phase 3
keywords: []       # Searchable terms
---

# {module_name}

## Overview
[2-3 sentences on what this area of the codebase covers]

## Children
[List each child sub-domain you identified with a brief description of what it does]

## Shared Patterns
[Patterns, conventions, or utilities shared across children]

## Common Interface
[If the children share a common interface or are used through a unified entry point, describe it]
```

### 2. References File: `.claude/planning-with-james/knowledge/modules/{module_id}/_refs.json`

```json
{
  "imports_from": ["list", "of", "file", "paths", "imported"],
  "imported_by": [],  // Leave empty, filled in Phase 3
  "calls_to": ["external/api/paths"],
  "shared_types": ["TypeName"],
  "notes": "Any observations about cross-module relationships"
}
```

### 3. Write Result File

After writing all documentation files, write your classification result to:
`.claude/planning-with-james/knowledge/modules/{module_id}/_result.json`

```json
{"module_id": "{module_id}", "type": "container", "children": [{"id": "child-slug", "name": "Child Name", "path": "path/to/child", "description": "Brief description", "type": "feature|service|library|config|infrastructure"}]}
```

Each child should be a distinct sub-domain. Use kebab-case IDs. The child path should be the directory containing that sub-domain's code.

This file signals to the orchestrator that you are done. Do NOT skip this step.
```

### Processing Results

After the wait completes, use Glob to find all `modules/*/_result.json` files. Read each one (they are tiny). Then:

1. **For each LEAF result**: Mark the module as `module_type: "leaf"` in `_discovery.json`. Add to `modules_classified` in progress file. Verify `_index.md` and `_refs.json` were written.

2. **For each CONTAINER result**: Mark the module as `module_type: "container"` in `_discovery.json`. Add returned children as new entries with `module_type: "pending"`, `parent` set to the container's ID, and `children: []`. Update the container's `children` array with the new child IDs. Update the container's `_index.md` frontmatter `children` field. Add container to `modules_classified` in progress file.

3. **Update progress**: Increment `current_wave`, update `modules_classified` list, update `total_modules` count (it grows as containers produce children).

4. **Cleanup**: Delete all `_result.json` files after processing (so the next wave's wait starts clean).

### Resume Handling

If resuming mid-Phase 2:
- Check for unprocessed `_result.json` files: Glob for `modules/*/_result.json`. If any exist, process them first (they are from agents that completed but whose results weren't incorporated before the interruption).
- Read `_discovery.json` for modules with `module_type: "pending"` — these need classification
- Check for orphaned files: modules with `_index.md` on disk but no `_result.json` and not in `modules_classified`. Read the `_index.md` frontmatter to determine if leaf or container, and process accordingly.
- Resume at the next wave with remaining pending modules

### Phase 2 Complete

When no modules have `module_type: "pending"`, Phase 2 is done. Update progress:

```json
{
  "phase": "connectivity",
  "modules_indexed": ["mod-a", "mod-b", "...all leaf and container module IDs"],
  "modules_classified": ["mod-a", "mod-b", "...all classified module IDs"],
  "current_wave": {final wave number},
  "status": "in_progress"
}
```

## Phase 3: Connectivity Mapping

Now spawn subagents to map the relationships between modules.

Launch a SINGLE Task agent with `subagent_type`: "general-purpose", `model`: "sonnet", and `mode`: "bypassPermissions" to:

**CRITICAL**: Include in the agent prompt: "Use the Write and Edit tools for ALL file changes. Do NOT use Bash commands (sed, awk, for loops, echo) to create or modify files. Bash is only for git commands."

1. Read all `_refs.json` files from each module
2. Read `_discovery.json` to get the module hierarchy (parent/children relationships)
3. Cross-reference imports and exports
4. Trace API calls and shared types
5. Build the complete relationship graph, including containment edges from the hierarchy

**Output**:

### 1. Create `_graph.json`:

```json
{
  "generated_at": "ISO timestamp",
  "commit": "git commit hash",
  "nodes": [
    {
      "id": "module-id",
      "name": "Module Name",
      "path": "src/path",
      "type": "feature|service|library",
      "module_type": "leaf|container",
      "parent": null
    }
  ],
  "edges": [
    {
      "from": "module-a",
      "to": "module-b",
      "type": "contains|imports|calls|shares_types|depends_on",
      "description": "Brief description of relationship"
    }
  ],
  "clusters": [
    {
      "name": "Cluster Name",
      "description": "Why these are grouped",
      "modules": ["module-a", "module-b"]
    }
  ]
}
```

**Edge types**:
- `contains`: Container → child. Generated from `_discovery.json` parent/children relationships, not from import analysis.
- `imports`: Module A imports code from module B.
- `calls`: Module A makes runtime calls to module B (API calls, RPC, event emission).
- `shares_types`: Modules share type definitions or data structures.
- `depends_on`: General dependency not captured by the above (configuration, build order, etc.).

**Node fields**:
- `module_type`: `"leaf"` or `"container"` — from `_discovery.json`
- `parent`: Parent module ID or `null` for top-level — from `_discovery.json`

### 2. Create `_architecture.md`:

```markdown
---
last_updated: {ISO timestamp}
last_commit: {git commit hash}
---

# Architecture Overview

## System Diagram

[Text-based description of how components connect]

## Module Hierarchy

[Tree representation of the container/leaf hierarchy. Show which modules contain which children, with brief descriptions.]

## Module Clusters

### {Cluster Name}
- **Modules**: list of modules
- **Purpose**: why they're grouped
- **Interactions**: how they communicate

## Cross-Cutting Concerns

[Authentication, logging, error handling - things that span modules]

## Data Flow

[How data moves through the system]

## Deployment Boundaries

[If there are multiple deployments, services, or environments]
```

### 3. Update each module's `_index.md` frontmatter:

Add the `external_refs` field with modules that reference this one. For container modules, also update the `children` field in frontmatter to include the final list of child module IDs.

**After Phase 3 is complete**, update the progress file:

```json
{
  "phase": "finalization",
  "status": "in_progress"
}
```

## Finalization Phase

Create the `_overview.md` file:

```markdown
---
last_updated: {ISO timestamp}
last_commit: {full git commit hash}
total_modules: {count}
leaf_modules: {count of modules with module_type: "leaf"}
container_modules: {count of modules with module_type: "container"}
max_depth: {deepest nesting level in the hierarchy}
indexing_completed: true
---

# Codebase Knowledge Base

## Quick Stats
- **Modules indexed**: {count} ({leaf_count} leaf, {container_count} container)
- **Max hierarchy depth**: {max_depth}
- **Last indexed**: {date}
- **Commit**: {short hash}

## Technology Stack
{from _discovery.json}

## Module Map

| Module | Path | Type | Description |
|--------|------|------|-------------|
| [Module Name](modules/module-id/_index.md) | src/path | leaf/container | Brief description |

## Module Hierarchy

{Tree representation showing container/leaf nesting. Example:}
{- parsers/ (container)}
{  - air-parser (leaf)}
{  - ocean-fcl/ (container)}
{    - ocean-fcl-email (leaf)}
{    - ocean-fcl-attachment/ (container)}
{      - ocean-fcl-attachment-excel (leaf)}
{      - ocean-fcl-attachment-pdf (leaf)}

## How to Navigate

1. Start with [Architecture](_architecture.md) for system overview
2. Use [Graph](_graph.json) for relationship queries
3. Dive into specific [modules](modules/) as needed
4. Container modules link to their children for drilling down

## Entry Points

{List key entry points from _discovery.json}
```

**After Finalization**, update the progress file:

```json
{
  "phase": "completed",
  "status": "completed",
  "completed_at": "{ISO timestamp}"
}
```

## Completion

After all phases complete, output a summary:
- Total modules indexed (leaf count + container count)
- Max hierarchy depth
- Total waves in Phase 2
- Total relationships mapped (containment edges + cross-cutting edges)
- Any warnings or areas that need attention
- Suggestions for the user
