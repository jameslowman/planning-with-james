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

You are about to create a comprehensive knowledge base for this codebase. This is a multi-phase process that will deeply analyze the code and create a navigable knowledge graph.

## Step 0: Resume Detection

BEFORE anything else, check for an in-progress creation:

```
Read .claude/planning-with-james/knowledge/_create_progress.json
```

- **If file exists and `status` is `"in_progress"`**:
  Resume from where it stopped based on `phase`:
  - `"discovery"`: Phase 1 didn't complete. Check if `_discovery.json` exists. If yes, skip to Phase 2. If no, re-run Phase 1.
  - `"deep_dives"`: Phase 2 was interrupted. Check which modules already have `_index.md` files. Only process modules that don't. Inform user: "Resuming interrupted knowledge creation. {N} of {M} modules already indexed."
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
      "type": "feature|service|library|config|infrastructure"
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

Be thorough. Identify ALL significant modules. A module is any cohesive unit of functionality - this could be a directory, a service, a feature domain, or a logical grouping.

**After completing Phase 1**, write the progress file:

```json
// .claude/planning-with-james/knowledge/_create_progress.json
{
  "started_at": "{ISO timestamp}",
  "commit": "{current git HEAD}",
  "phase": "deep_dives",
  "total_modules": {count from _discovery.json},
  "modules_indexed": [],
  "status": "in_progress"
}
```

## Phase 2: Deep Dives

Now spawn parallel subagents to deeply analyze each module identified in Phase 1.

For EACH module in `_discovery.json`, spawn a Task agent with:
- `subagent_type`: "Explore"
- `model`: "opus" (we want maximum depth)

**IMPORTANT**: Launch ALL module subagents in a SINGLE message with multiple Task tool calls to run them in parallel.

Each subagent should be given this prompt template (fill in the module details):

```
You are analyzing the "{module_name}" module located at "{module_path}".

Your task is to create comprehensive documentation for this module.

## Investigation Steps

1. Read all significant files in this module
2. Understand the purpose, patterns, and structure
3. Identify:
   - Key files and their purposes
   - Public interfaces/APIs/exports
   - Internal patterns and conventions
   - Dependencies (imports from other modules)
   - External references (what other parts of the codebase might use this)

## Output

Create the following files:

### 1. Module Index: `.claude/planning-with-james/knowledge/modules/{module_id}/_index.md`

```markdown
---
module_id: {module_id}
module_name: {module_name}
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
```

After ALL Phase 2 subagents complete, verify each module has its `_index.md` and `_refs.json` created.

**If resuming Phase 2**: Check which modules already have `_index.md` files. Only launch subagents for modules that are missing. This allows resume after partial completion.

**After Phase 2 is complete**, update the progress file:

```json
{
  "phase": "connectivity",
  "modules_indexed": ["mod-a", "mod-b", "...all completed module IDs"],
  "status": "in_progress"
}
```

## Phase 3: Connectivity Mapping

Now spawn subagents to map the relationships between modules.

Launch a SINGLE Task agent with `subagent_type`: "Explore" to:

1. Read all `_refs.json` files from each module
2. Cross-reference imports and exports
3. Trace API calls and shared types
4. Build the complete relationship graph

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
      "type": "feature|service|library"
    }
  ],
  "edges": [
    {
      "from": "module-a",
      "to": "module-b",
      "type": "imports|calls|shares_types|depends_on",
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

### 2. Create `_architecture.md`:

```markdown
---
last_updated: {ISO timestamp}
last_commit: {git commit hash}
---

# Architecture Overview

## System Diagram

[Text-based description of how components connect]

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

Add the `external_refs` field with modules that reference this one.

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
indexing_completed: true
---

# Codebase Knowledge Base

## Quick Stats
- **Modules indexed**: {count}
- **Last indexed**: {date}
- **Commit**: {short hash}

## Technology Stack
{from _discovery.json}

## Module Map

| Module | Path | Type | Description |
|--------|------|------|-------------|
| [Module Name](modules/module-id/_index.md) | src/path | type | Brief description |

## How to Navigate

1. Start with [Architecture](_architecture.md) for system overview
2. Use [Graph](_graph.json) for relationship queries
3. Dive into specific [modules](modules/) as needed

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
- Total modules indexed
- Total relationships mapped
- Any warnings or areas that need attention
- Suggestions for the user
