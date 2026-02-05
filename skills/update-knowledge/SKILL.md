---
name: update-knowledge
description: Incrementally update the knowledge base. Args: empty (standard), text (guided), "verify" (check integrity), "repair" (fix problems).
disable-model-invocation: true
allowed-tools: Bash, Glob, Grep, Read, Write, Task, Edit, AskUserQuestion
---

# Update Knowledge Base

**NON-NEGOTIABLE: PROGRESS TRACKING**
After every module update, you MUST update `_update_progress.json` BEFORE starting the next module. This file is your lifeline through context loss. If you skip this step and autocompact hits, all work since the last progress write is lost.

**NON-NEGOTIABLE: LEAN MAIN CONTEXT**
The orchestrator (you) should hold minimal state. Do NOT read all module `_index.md` files, `_graph.json`, or `_architecture.md` into your context during standard updates. That is wasted context. Subagents read their own knowledge. You read only what you need to plan.

**NON-NEGOTIABLE: LINEAR EXECUTION**
Process modules ONE AT A TIME. Do NOT launch parallel subagents for module updates. Each module update should: spawn agent -> receive brief summary -> update progress -> next. This prevents context exhaustion.

---

## Step 0: Route by Argument

Check `$ARGUMENTS`:

| Argument | Mode |
|----------|------|
| empty | Check for resume, then Standard Mode |
| `verify` | Verify Mode (integrity check, no changes) |
| `repair` | Repair Mode (fix problems found by verify) |
| any other text | Guided Mode (interpret user intent) |

---

## Step 1: Resume Detection (ALL MODES except verify)

BEFORE anything else, check for an in-progress update:

```
Read .claude/planning-with-james/knowledge/_update_progress.json
```

- **If file exists and `status` is `"in_progress"`**: Resume from where it stopped. Skip to the appropriate phase based on `phase` field. Inform user: "Resuming interrupted knowledge update. {N} of {M} modules already processed."
- **If file exists and `status` is `"completed"`**: Delete it, proceed normally.
- **If file does not exist**: Proceed normally.

---

# STANDARD MODE (No Arguments)

## Phase 1: Assess

This phase is LEAN. Read only what's needed to plan the update.

1. Read `.claude/planning-with-james/knowledge/_overview.md` -- extract ONLY:
   - `last_commit` from frontmatter
   - `total_modules` count

2. Get current HEAD:
   ```bash
   git rev-parse HEAD
   ```
   If `last_commit` equals current HEAD, inform user: "Knowledge base is up to date." Stop.

3. Get change summary:
   ```bash
   git diff --stat {last_commit}..HEAD
   ```

4. Read `.claude/planning-with-james/knowledge/_discovery.json` (this is small -- just module IDs and paths).

5. Map changed files to modules using `_discovery.json` paths. Build:
   - `directly_affected_modules`: modules with changed files
   - `new_directories`: directories with changes that don't map to any known module
   - `deleted_directories`: known module paths that no longer exist

6. Write the progress file:

```json
// .claude/planning-with-james/knowledge/_update_progress.json
{
  "started_at": "{ISO timestamp}",
  "from_commit": "{last_commit}",
  "to_commit": "{current HEAD}",
  "phase": "direct_updates",
  "total_commits": {count},
  "modules_to_update": ["mod-a", "mod-b", "mod-c"],
  "modules_completed": {},
  "new_modules_to_create": [],
  "deleted_modules": [],
  "cascade_needed": [],
  "cascade_completed": [],
  "status": "in_progress"
}
```

## Phase 2: Direct Module Updates (LINEAR)

For each module in `modules_to_update` that is NOT in `modules_completed`:

### 2a. Get the changed files for this module:
```bash
git diff --name-status {from_commit}..{to_commit} -- {module_path}
```

### 2b. Spawn ONE Task agent:
- `subagent_type`: "general-purpose"
- `model`: "opus"

**Subagent prompt:**

```
You are updating knowledge for the "{module_name}" module at "{module_path}".

## MANDATORY: Read Existing Knowledge First

Before reading ANY source code:
1. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_index.md
2. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_refs.json
3. Read any existing detail files in that module's knowledge folder

Understand what we already know. Then read the changed source files to see what's different.

## Changes Since Last Index

Commits: {from_commit}..{to_commit}
Files changed in this module:
{file list with add/modify/delete status from git diff --name-status}

## Your Task

1. Read existing knowledge (above)
2. Read the changed source files
3. Compare: what changed vs what we documented?
4. Update the documentation:
   - Update _index.md content to reflect changes
   - Update frontmatter: last_updated (ISO timestamp), last_commit ({to_commit})
   - Update _refs.json if imports/exports/dependencies changed
   - Add/update/remove detail files as needed
5. Write all changes to disk using Write/Edit tools

## CRITICAL: Return Format

After completing all file writes, return ONLY this JSON (nothing else):

{"module_id": "{module_id}", "interface_changed": true|false, "dependencies_changed": true|false, "scope_changed": true|false, "summary": "One sentence describing what changed"}

Keep your response BRIEF. All detailed work goes into the files you write. Your return value is just the summary.
```

### 2c. Record result in progress file:

After receiving the agent's JSON response, update `_update_progress.json`:
- Add module to `modules_completed` with its summary
- If `interface_changed` or `dependencies_changed`, note it for cascade planning

```json
"modules_completed": {
  "mod-a": {
    "interface_changed": false,
    "dependencies_changed": true,
    "summary": "Added new export for rate calculation helper"
  }
}
```

### 2d. Handle new modules

For each entry in `new_modules_to_create`, spawn ONE agent at a time:

```
You are creating knowledge for a NEW module discovered at "{path}".

## Context
This directory appeared in recent commits and doesn't have existing knowledge documentation.

## Reference
Read one existing module's knowledge files for format reference:
- .claude/planning-with-james/knowledge/modules/{any_existing_module_id}/_index.md
- .claude/planning-with-james/knowledge/modules/{any_existing_module_id}/_refs.json

## Your Task
1. Read all significant files at {path}
2. Create: .claude/planning-with-james/knowledge/modules/{new_module_id}/_index.md
3. Create: .claude/planning-with-james/knowledge/modules/{new_module_id}/_refs.json
4. Create detail files if the module is complex

Return ONLY: {"module_id": "{new_module_id}", "summary": "One sentence about this module"}
```

Update progress file after each.

### 2e. Handle deleted modules

For deleted modules, remove their knowledge folders and note in progress file. No subagent needed.

### 2f. Update progress phase:
```json
"phase": "cascade"
```

## Phase 3: Cascade (LINEAR)

Build cascade set from completed module summaries:

1. Read `_update_progress.json` to get completed module summaries
2. Read ONLY the `edges` array from `.claude/planning-with-james/knowledge/_graph.json` (use Grep or targeted read to avoid loading entire file if large)
3. For each module where `interface_changed` or `dependencies_changed`:
   - Find modules connected to it via edges
   - Add to cascade set
4. Remove any modules already in `modules_completed` from cascade set
5. Write cascade set to progress file:
```json
"cascade_needed": ["mod-x", "mod-y"]
```

For each module in `cascade_needed` that is NOT in `cascade_completed`:

Spawn ONE Task agent:
- `subagent_type`: "general-purpose"
- `model`: "opus"

```
You are verifying knowledge for "{module_name}" because a connected module changed.

## MANDATORY: Read Existing Knowledge First
1. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_index.md
2. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_refs.json

## What Changed Upstream
{list changed modules and their one-line summaries from progress file}

## Your Task
1. Read existing knowledge (above)
2. Check if documented dependencies and relationships still match
3. Read source code ONLY if you need to verify a specific relationship
4. Update _index.md and _refs.json if needed
5. Write changes to disk

Return ONLY: {"module_id": "{module_id}", "updated": true|false, "summary": "One sentence or 'no changes needed'"}
```

Update `cascade_completed` in progress file after each.

## Phase 4: Rebuild Graph and Finalize

### 4a. Rebuild graph

Spawn ONE Task agent:

```
You are rebuilding the knowledge graph after updates.

## Your Task
1. Read ALL _refs.json files from .claude/planning-with-james/knowledge/modules/*/
2. Read .claude/planning-with-james/knowledge/_discovery.json

3. Rebuild .claude/planning-with-james/knowledge/_graph.json:
   - Nodes from _discovery.json
   - Edges from cross-referencing all _refs.json files
   - Clusters from logical groupings
   - Update generated_at and commit fields

4. Update .claude/planning-with-james/knowledge/_architecture.md if relationships changed significantly

5. Update .claude/planning-with-james/knowledge/_discovery.json if modules were added/removed

Return ONLY: {"nodes": {count}, "edges": {count}, "clusters": {count}}
```

### 4b. Update overview

Update `_overview.md` yourself (this is small):
- `last_updated`: current ISO timestamp
- `last_commit`: current HEAD
- Update module table if modules added/removed
- Update stats

### 4c. Mark complete

Update `_update_progress.json`:
```json
"phase": "completed",
"status": "completed",
"completed_at": "{ISO timestamp}"
```

### 4d. Output summary

```
Knowledge Base Updated (Standard Mode)
======================================
Previous commit: {from_commit}
Current commit:  {to_commit}
Commits processed: {count}

Directly updated: {module list}
New modules:      {list or "none"}
Deleted modules:  {list or "none"}
Cascade updated:  {module list}
Graph rebuilt:    {nodes} nodes, {edges} edges

Total modules touched: {count}
```

---

# GUIDED MODE (With Context)

When the user provides context via `$ARGUMENTS`, interpret their intent and execute targeted updates.

## Phase 1: Lean Context Load

Read ONLY:
1. `_overview.md` -- module map and stats
2. `_discovery.json` -- module IDs and paths
3. Search `_graph.json` for modules matching the user's topic (use Grep with module names, not full file read)

Do NOT read all module `_index.md` files. Only read the specific modules relevant to the user's request.

## Phase 2: Interpret Context

Read the user's input: `$ARGUMENTS`

With the module map loaded, identify which modules are relevant and what the user wants:

| User Intent | Indicators | Action |
|-------------|------------|--------|
| Deep dive | "go deeper on X", "more detail on X" | Re-index module(s) with higher depth |
| Correction | "X is wrong", "actually X does Y" | Update module(s) with corrections |
| Business context | "X exists because", "the reason for X" | Add context to documentation |
| Relationship fix | "X and Y are connected", "X calls Y" | Update relationships and cascade |
| New area | "you missed the X system" | Create new module documentation |
| Cross-cutting | "the X pipeline", "how X flows" | Multi-module deep dive |

## Phase 3: Clarification

Ask 1-3 targeted questions using `AskUserQuestion` based on intent and existing knowledge.

For the relevant modules, NOW read their `_index.md` files (just those, not all) so you can ask informed questions:
- "Based on our knowledge, {module} connects to {A}, {B}. Should I deep-dive those too?"
- "I see we already documented {existing knowledge}. Focus on {gaps} or re-examine everything?"

Wait for user response.

## Phase 4: Write Progress File

Write `_update_progress.json` with the plan:

```json
{
  "started_at": "{ISO timestamp}",
  "mode": "guided",
  "intent": "{interpreted intent}",
  "phase": "guided_updates",
  "modules_to_update": ["mod-a", "mod-b"],
  "modules_completed": {},
  "cascade_needed": [],
  "cascade_completed": [],
  "status": "in_progress"
}
```

## Phase 5: Execute (LINEAR, one module at a time)

For each target module, spawn ONE Task agent with appropriate prompt based on intent type:

### Deep Dive (single module):
```
You are doing a DEEP DIVE on "{module_name}" at "{module_path}".

## MANDATORY: Start From Existing Knowledge
1. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_index.md
2. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_refs.json
3. Read any existing detail files in the module's knowledge folder
4. UNDERSTAND what we already documented
5. IDENTIFY gaps and areas needing more depth
6. THEN explore source code to fill gaps

DO NOT rediscover what we already know. Build upon it.

## User Context
{$ARGUMENTS}

## User Clarifications
{responses from Phase 3}

## Your Task
Go significantly deeper than the original indexing, starting from existing knowledge:

1. Review existing documentation
2. Identify what's missing or shallow
3. Read source files to fill gaps:
   - All public and significant private interfaces
   - Internal architecture and patterns
   - Edge cases and error handling
   - Configuration, performance, security considerations
4. Create additional detail files as needed:
   - api.md, internals.md, flows.md, edge-cases.md
5. Update _index.md with richer overview (preserve accurate existing content)
6. Update _refs.json with more detailed relationship info

Write all changes to disk. Be thorough. Cost is not a concern.

Return ONLY: {"module_id": "{module_id}", "interface_changed": true|false, "dependencies_changed": true|false, "detail_files_created": ["list"], "summary": "One sentence"}
```

### Cross-Cutting Deep Dive:

For cross-cutting requests (pipelines, flows), process each module in the flow ONE AT A TIME, giving each agent shared context about the pipeline:

```
You are doing a DEEP DIVE on "{module_name}" as part of a cross-cutting exploration of the "{pipeline_name}".

## MANDATORY: Start From Existing Knowledge
1. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_index.md
2. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_refs.json
3. Read existing detail files

## Pipeline Context
This module is part of the {pipeline_name} which flows through:
{ordered list of modules}

Your module's role:
- Receives from: {upstream modules}
- Sends to: {downstream modules}

## Summaries from Previously Analyzed Pipeline Modules
{summaries from modules already completed in this session}

## User Context
{$ARGUMENTS}

## Your Task
Deep dive with special attention to:
1. Connections to upstream/downstream modules
2. Data transformations that occur here
3. Error handling in the pipeline flow
4. Configuration that affects pipeline behavior

Create a flows.md if it doesn't exist. Update _index.md and _refs.json.
Write all changes to disk.

Return ONLY: {"module_id": "{module_id}", "interface_changed": true|false, "dependencies_changed": true|false, "summary": "One sentence about this module's role in the pipeline"}
```

After ALL pipeline modules complete, spawn ONE MORE agent to create a flow document:

```
You are creating a cross-cutting flow document for "{pipeline_name}".

## Your Task
1. Read the updated _index.md for each module: {list}
2. Create: .claude/planning-with-james/knowledge/flows/{pipeline_name}.md

Structure it as:
- Overview (end-to-end description)
- Flow diagram (text-based)
- Step-by-step through each module
- Error handling across the pipeline
- Configuration options
- Common issues

3. Update _architecture.md to reference this flow document

Return ONLY: {"flow_file": "flows/{pipeline_name}.md", "summary": "One sentence"}
```

### Correction, Business Context, Relationship Fix, New Area:

Spawn ONE agent per module, with a prompt tailored to the intent (correction, context, etc.). Each agent:
1. Reads existing knowledge first
2. Makes targeted updates
3. Writes to disk
4. Returns brief JSON summary

Update progress file after each.

## Phase 6: Cascade and Finalize

Same as Standard Mode Phases 3 and 4. Linear cascade, rebuild graph, update overview, mark complete.

Output summary:
```
Knowledge Base Updated (Guided Mode)
====================================
Intent: {interpreted intent}
Target: {module or area}

Modules updated: {list}
Cascade updated: {list}
Graph rebuilt: yes

Summary: {description of changes}
```

---

# VERIFY MODE (`$ARGUMENTS` = "verify")

This mode checks knowledge base integrity without making changes. No subagents needed -- this runs entirely in the main context.

## Checks

1. Read `_overview.md` frontmatter (`last_commit`, `total_modules`)
2. Read `_discovery.json` (module list)
3. Read `_graph.json` (nodes, edges)

Run these checks:

### Check 1: Module Folder Completeness
For each module in `_discovery.json`:
- Does `.claude/planning-with-james/knowledge/modules/{module_id}/` exist?
- Does it contain `_index.md`?
- Does it contain `_refs.json`?

### Check 2: Staleness
For each module with an `_index.md`:
- Read ONLY the frontmatter (first ~10 lines)
- Does `last_commit` match `_overview.md`'s `last_commit`?
- If not, module is stale

### Check 3: Codebase Alignment
For each module in `_discovery.json`:
- Does the `path` still exist in the codebase? (quick `ls` check)
- If not, module knowledge is orphaned

### Check 4: Graph Consistency
For each edge in `_graph.json`:
- Do both `from` and `to` modules exist in the nodes list?
- Do both modules have knowledge folders?

### Check 5: Missing Modules
Scan the codebase for significant directories that SHOULD be modules but aren't in `_discovery.json`. This is a heuristic check:
- Look at top-level src directories
- Check for directories with their own package.json, __init__.py, etc.
- Compare against known modules

### Check 6: In-Progress Detection
- Does `_update_progress.json` exist with status "in_progress"?
- If so, report it as an interrupted update

## Output

```
Knowledge Base Integrity Report
===============================
Last indexed commit: {hash}
Current HEAD:        {hash}
Commits behind:      {count}

Module Coverage:
  Total modules:     {count}
  Complete:          {count} (have _index.md + _refs.json)
  Incomplete:        {list} (missing files)

Staleness:
  Current:           {count}
  Stale:             {list with their last_commit}

Codebase Alignment:
  Aligned:           {count}
  Orphaned:          {list} (knowledge exists, code doesn't)
  Missing:           {list} (code exists, knowledge doesn't)

Graph Integrity:
  Nodes:             {count}
  Edges:             {count}
  Broken edges:      {list}

Interrupted Updates:
  {status of _update_progress.json if exists}

Recommendation:
  {one of: "Knowledge base is healthy", "Run repair to fix {N} issues", "Run standard update to catch up on {N} commits", etc.}
```

---

# REPAIR MODE (`$ARGUMENTS` = "repair")

This mode fixes problems found by verify. It runs verify first, then addresses each problem.

## Step 1: Run Verify

Execute all verify checks. Collect problems into categories.

## Step 2: Write Repair Progress

```json
// _update_progress.json
{
  "started_at": "{ISO timestamp}",
  "mode": "repair",
  "phase": "repair",
  "problems_found": {
    "incomplete_modules": ["mod-a"],
    "stale_modules": ["mod-b", "mod-c"],
    "orphaned_modules": ["mod-d"],
    "missing_modules": ["path/to/new-area"],
    "broken_edges": [{"from": "x", "to": "y"}]
  },
  "problems_fixed": {},
  "status": "in_progress"
}
```

## Step 3: Fix Problems (LINEAR, one at a time)

Process in this order:

### 3a. Delete orphaned knowledge
For each orphaned module (knowledge exists, code doesn't):
- Delete the module's knowledge folder
- Remove from _discovery.json
- Update progress file
No subagent needed.

### 3b. Fix incomplete modules
For each module missing `_index.md` or `_refs.json`:
- Spawn ONE agent to re-index the module (same prompt as Standard Mode Phase 2)
- Update progress file after completion

### 3c. Fix stale modules
For each stale module:
- Get changed files: `git diff --name-status {module_last_commit}..HEAD -- {module_path}`
- If no files changed in this module, just update the frontmatter commit hash
- If files changed, spawn ONE agent to update (same as Standard Mode Phase 2)
- Update progress file after each

### 3d. Create missing modules
For each codebase directory that should be a module:
- Spawn ONE agent to create knowledge (same prompt as new module creation)
- Update progress file after each

### 3e. Fix graph
After all module fixes complete:
- Spawn ONE agent to rebuild `_graph.json` from all `_refs.json` files
- Update `_discovery.json`
- Update `_overview.md`

## Step 4: Finalize

Mark progress as completed. Output:

```
Knowledge Base Repair Complete
==============================
Problems found:    {total count}
Problems fixed:    {total count}

Orphaned modules removed: {list or "none"}
Incomplete modules fixed:  {list or "none"}
Stale modules refreshed:  {list or "none"}
New modules created:       {list or "none"}
Graph rebuilt:             yes/no

Knowledge base is now consistent with commit {HEAD}.
```

---

## Edge Cases

### Autocompact recovery
If autocompact hits during any phase:
- The `_update_progress.json` file persists on disk
- Re-invoking `/planning-with-james:update-knowledge` detects it in Step 1
- Resumes from where it left off
- No work is lost because each module writes to disk as it completes

### Very large change sets (50+ modules affected)
Inform the user: "This update affects {N} modules. Processing linearly. This will take a while but ensures nothing is lost."
Proceed normally. The linear approach handles any size.

### No changes but user wants refresh
If `last_commit` matches HEAD but user insists on update:
- Suggest: "Knowledge base is current. Use `verify` to check integrity, or provide context for a guided deep-dive."

### Guided mode with ambiguous module reference
If the user mentions something that doesn't map to a known module:
1. Search `_discovery.json` for partial matches
2. Ask: "I'm not sure which module you mean. Did you mean {A}, {B}, or something else?"

### Conflicting information in guided mode
If the user's correction contradicts the code:
1. Point this out: "You said X, but the code shows Y. Should I document your intent, the current behavior, or both?"
2. Act on their response

### Interrupted repair
Repair mode uses the same progress file pattern. If interrupted:
- Resume picks up the problem list and continues fixing unfixed problems
- Already-fixed problems are not re-processed
