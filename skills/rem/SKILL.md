---
name: rem
description: Sleep-cycle maintenance for the planning plugin. Prunes, archives, consolidates, and dreams — keeps the registry, lessons, knowledge graph, and plan folder lightweight as work accumulates.
disable-model-invocation: true
allowed-tools: Bash, Glob, Grep, Read, Write, Task, Edit, AskUserQuestion
---

# REM: Sleep-Cycle Maintenance

The planning plugin accumulates weight over time: completed plans linger in the registry, lessons pile up without curation, knowledge graph modules describe code that has since evolved, `/dig` explorations pile up, abandoned progress files hang around. Left alone, `planning-with-james` becomes slow to load, noisy to search, and quietly inaccurate.

`/rem` fixes this by running a sleep cycle: triage, prune, consolidate, dream. It mimics what a brain does overnight — archives the obvious dead weight, merges redundant memories, promotes things that got forgotten, and wanders the repo looking for patterns and weirdness the plugin didn't notice during working hours.

The main goal is **consolidation** — keep the plugin heavyweight in capability, light on its feet in operation. Dreams are a bonus: speculative notes and action items the user can act on during planning.

---

## NON-NEGOTIABLES

**1. ALL PATHS ARE RELATIVE TO THE REPO ROOT.** All `.claude/planning-with-james/` paths in this skill are relative to the current working directory, NOT `~/.claude/`. Run `pwd` if unsure.

**2. PROGRESS TRACKING.** After each phase completes, update `_rem_progress.json` BEFORE starting the next. This enables resume after autocompact.

**3. AGENT PERMISSIONS.** Every Task agent spawn MUST include `mode: "bypassPermissions"`. Background agents without this mode have Write/Edit calls silently denied, burning turns with no output.

**4. NO BASH FOR FILE MODIFICATIONS.** Use Read → Edit or Read → Write for all file changes. No `sed`, `awk`, `echo` redirection, heredocs, or `for` loops that write files. Bash is only for `git`, `mv` (archive operations), and project tools. This applies to orchestrator AND subagents — include this rule in every subagent prompt.

**5. REM NEVER WRITES TO `knowledge/modules/`.** The knowledge graph is owned by `create-knowledge` and `update-knowledge`. When rem detects stale or shallow knowledge, it creates an **action item** recommending `/planning-with-james:update-knowledge guided` with specific context. It does not edit the module docs.

**6. NEVER ARCHIVE A PLAN WITH UNPROMOTED LESSONS.** Before archiving any plan, sweep its `lessons.md` — promote anything that belongs at project level, then archive. Silent archiving of unpromoted lessons loses hard-won context.

**7. ARCHIVE, NEVER DELETE.** Every file rem retires moves to `.claude/planning-with-james/archive/`. Every archive action is recorded in `archive/_manifest.json` with original path, archive path, reason, and timestamp. The user restores with `mv` — no `/rem restore` command. (One exception: genuinely ephemeral files like abandoned `_create_progress.json` for a completed creation can be deleted outright, after confirming the completion.)

**8. MCP DATA SOURCES ARE BEST-EFFORT.** If Sentry, Vercel, or GCP MCP tools require reauthorization or are unavailable, skip them gracefully and note in the digest. Never prompt the user mid-run for external auth.

---

## Arguments

`$ARGUMENTS` controls behavior:

| Argument | Behavior |
|----------|----------|
| *(empty)* | Full cycle: triage → prune → consolidate → dream |
| `triage` | Phase 1 only. Report candidates; touch nothing. |
| `prune` | Phases 1–2. Archive obvious-dead; no consolidation or dreaming. |
| `consolidate` | Phases 1, 3. Lesson merges, forgotten-lesson promotions, staleness flags. |
| `dream` | Phases 1, 4. Team wanders repo + MCPs; produces dreams and action items. |
| `status` | Show recent rem activity, open action items, archive stats. No work. |

---

## Step 0: Resume Detection

BEFORE anything else, read `.claude/planning-with-james/_rem_progress.json`.

- **If file exists and `status` is `"in_progress"`**: Resume at the phase indicated. Inform user: "Resuming interrupted rem cycle at {phase}."
- **If file exists and `status` is `"completed"`**: Delete it and proceed normally.
- **If file does not exist**: Proceed normally.

---

## Step 1: Load Config

Read `.claude/planning-with-james/rem-config.json`.

If missing, create it with defaults:

```json
{
  "created_at": "{ISO timestamp}",
  "archive_after_days": 30,
  "abandoned_plan_after_days": 30,
  "exploration_expiry_days": 30,
  "data_sources": {
    "code": true,
    "git_history": true,
    "sentry": "auto",
    "vercel": "auto",
    "gcp": "off"
  },
  "consolidate_batch_granularity": "category",
  "denylist_paths": ["node_modules", "dist", "build", ".next", ".venv", "__pycache__", "target", "vendor", ".turbo"]
}
```

Inform the user on first creation: "Created `rem-config.json` with defaults. Edit to customize retention windows and data sources."

`"auto"` for a data source means: the Dream team tries the MCP, and if it errors or requires auth, logs "skipped {source}: unavailable" and moves on.

---

## Step 2: Route by Argument

Based on `$ARGUMENTS`, skip directly to the appropriate phase(s). All modes except `status` run Phase 1 (triage) first because every downstream phase reads `_rem_triage.json`.

If `status`: see **Status Mode** at the bottom.

Otherwise:
- Write initial progress file
- Run Phase 1 (Triage)
- Run subsequent phases per mode
- Finalize

Progress file format:

```json
{
  "started_at": "{ISO timestamp}",
  "mode": "full | triage | prune | consolidate | dream",
  "phase": "triage | prune | consolidate | dream | completed",
  "phases_complete": [],
  "status": "in_progress"
}
```

---

# PHASE 1: TRIAGE

**Goal**: Survey the plugin's accumulated state and categorize everything into `obvious-dead`, `probably-archive`, `needs-review`, and `dream-seeds`. No modifications.

Write findings to `.claude/planning-with-james/_rem_triage.json`. Every downstream phase reads this.

## Step 1.1: Survey the Registry

Read `.claude/planning-with-james/plans/_registry.json`.

For each plan, compute:
- **Last activity**: the latest of `_plan_state.json`'s `last_updated`, the plan's registry `current_task` updates, and the mtime of `context_scratch_pad.md`
- **Age**: days since last activity
- **Status**: from registry (`planning`, `planned`, `active`, `paused`, `completed`, `archived`)

Categorize:
- `obvious-dead`: status is `completed` AND age > `archive_after_days`
- `probably-archive`: status is `paused` AND age > `abandoned_plan_after_days`
- `needs-review`: status is `planning` AND age > 30 days (stuck in planning)
- Plans that are `active` or recently `paused`/`completed`: leave alone

Also flag:
- Plans with status `archived` but with no `archive_path` set (inconsistent)
- Plans in registry whose folder doesn't exist
- Folders in `plans/` that aren't in the registry

## Step 1.2: Survey Session Bindings

From the same registry, look at `sessions`. Flag:
- Sessions with `bound_at` older than 24 hours (stale — should be pruned)
- Plans with `locked_by_session` pointing to a session that doesn't exist (dangling lock)

## Step 1.3: Survey Progress Files

Search for progress files in standard locations:
- `.claude/planning-with-james/knowledge/_create_progress.json`
- `.claude/planning-with-james/knowledge/_update_progress.json`
- `.claude/planning-with-james/_rem_progress.json` (from a prior rem — should be `completed`)

For each: read and check `status`. Flag:
- `"completed"` status: safe to delete (or archive)
- `"in_progress"` with `started_at` > 7 days old: likely abandoned, needs user review

## Step 1.4: Survey Explorations

List `.claude/planning-with-james/explorations/*.md`. For each, read frontmatter `timestamp`. Flag:
- Explorations older than `exploration_expiry_days`: obvious-dead
- Explorations referencing modules still in active plans: keep

## Step 1.5: Survey Lessons

Read `.claude/planning-with-james/lessons.md` if it exists. Count lines. Flag:
- File > 200 lines: `needs-review` (candidate for consolidation)
- File exists but has no entries under any section: empty, could be archived

Also scan all plan folders for `lessons.md` files. For each plan being archived (from Step 1.1), check if its `lessons.md` has unpromoted entries — these become a `needs-review` item: "promote before archiving {plan-id}".

## Step 1.6: Survey Knowledge Graph Staleness

Read `.claude/planning-with-james/knowledge/_overview.md` frontmatter — note `last_commit`.

Get current HEAD: `git rev-parse HEAD`.

Get commits since last index: `git rev-list --count {last_commit}..HEAD` (if fails, skip).

For each module in `_discovery.json`:
- Read the module's `_index.md` frontmatter — note `last_commit`
- Get commits touching the module's path: `git log --oneline {module_last_commit}..HEAD -- {module_path} | wc -l`
- Flag as `dream-seed` if > 20 commits have touched the module since its knowledge was updated (semantic drift candidate)

This is the cheap check. The deep staleness check happens in Phase 3 consolidate and in Phase 4 dream (Cartographer).

## Step 1.7: Survey Epic Learnings

If `.claude/planning-with-james/epics/` exists:
- For each epic, read `_epic_state.json` and `learnings.md`
- If epic status is `completed` and no activity in 30+ days: `obvious-dead` for archive
- If `learnings.md` > 300 lines: `needs-review` for consolidation

## Step 1.8: Survey Action Items

If `.claude/planning-with-james/action_items/` exists:
- Count open items (files at top level, excluding `_index.md`)
- Count resolved items (in `resolved/`)
- Flag: items open > 60 days (might be stale themselves — rem should ask if still relevant)

## Step 1.9: Write Triage Report

Create `.claude/planning-with-james/_rem_triage.json`:

```json
{
  "generated_at": "{ISO timestamp}",
  "obvious_dead": {
    "plans_completed": [{"id": "...", "age_days": N, "path": "..."}],
    "plans_abandoned": [...],
    "explorations": [...],
    "stale_sessions": [...],
    "dangling_locks": [...],
    "completed_progress_files": [...]
  },
  "probably_archive": {
    "epic": [...]
  },
  "needs_review": {
    "lessons_bloat": {"path": "...", "lines": N},
    "unpromoted_lessons": [{"plan_id": "...", "entries": N}],
    "stale_in_planning": [...],
    "missing_folders": [...],
    "registry_drift": [...],
    "abandoned_progress_files": [...],
    "old_action_items": [...]
  },
  "dream_seeds": {
    "stale_modules": [{"id": "...", "commits_since_update": N}],
    "empty_modules": [...]
  },
  "stats": {
    "registry_plan_count": N,
    "archive_candidate_count": N,
    "action_items_open": N,
    "knowledge_commits_behind": N
  }
}
```

Update progress: `phase: "triage"` → add to `phases_complete`.

**If mode is `triage`**: print a human-readable summary and stop. Otherwise continue.

---

# PHASE 2: PRUNE

**Goal**: Act on the `obvious-dead` and safe `probably-archive` categories. Archive everything; delete nothing that carries meaning. Leave registry stubs so planning can still discover archived plans by name.

Ensure `.claude/planning-with-james/archive/` exists. Create sub-directories as needed.

## Step 2.1: Sweep Unpromoted Lessons (First, Always)

For each plan slated for archiving, read its `lessons.md`. If there are entries under "User Corrections", "Discoveries", "What Worked", or "Mistakes to Avoid":

1. Read project-level `.claude/planning-with-james/lessons.md` (create with initial structure if it doesn't exist — see `go-time` Phase COMPLETE for the template).
2. For each entry, decide: promote or skip.
   - Promote if: it's about the codebase itself, would help a fresh agent, corrects a common assumption.
   - Skip if: it's specific to this plan's problem, already in the knowledge graph, or too narrow.
3. Rewrite promoted entries for generality (remove plan-specific references).
4. Append to the appropriate project-level section.

This is the same promotion logic as `/go-time`'s COMPLETE step. If the plan already ran COMPLETE, its `lessons.md` should be empty — in that case skip.

Record in the archive manifest: `"lessons_promoted": N` for each plan archived.

## Step 2.2: Archive Completed Plans

For each entry in `triage.obvious_dead.plans_completed`:

1. Determine archive path: `archive/plans/{YYYY-MM}/{plan-id}/`
2. Move the whole plan folder with `git mv` (if tracked) or `mv` (if not):
   ```bash
   mv "{plan_folder}" "{archive_path}"
   ```
3. Update `_registry.json` — replace the plan entry with its archived stub:
   ```json
   "{plan-id}": {
     "name": "{original name}",
     "status": "archived",
     "archived_at": "{ISO timestamp}",
     "archived_from_status": "completed",
     "archive_path": "{archive_path}",
     "problem_summary": "{read from problem_description.md — first paragraph of Summary section}",
     "created_at": "{original}"
   }
   ```
4. Append to `archive/_manifest.json` (see **Archive Manifest Format** below).

Reading `problem_summary`: open `{plan_folder}/problem_description.md`, find the `## Summary` section, take the first paragraph. If missing, fall back to the plan's `name` field.

## Step 2.3: Archive Abandoned (Paused) Plans

Same as Step 2.2, but `archived_from_status: "paused"`. These are recoverable by the user with a simple `mv`, and the registry stub lets `/plan` find them by name.

## Step 2.4: Archive Old Explorations

For each entry in `triage.obvious_dead.explorations`:

1. Move to `archive/explorations/{YYYY-MM}/{filename}`
2. Append to `archive/_manifest.json`

No registry stub — explorations aren't tracked there.

## Step 2.5: Clean Stale Sessions and Dangling Locks

Edit `_registry.json`:
- Remove `sessions` entries with `bound_at` > 24 hours old
- For each plan with `locked_by_session` pointing to a now-removed or nonexistent session: clear `locked_by_session` and `locked_at`, set status to `"paused"` (if it was `"active"`)

No archiving needed — these are ephemeral.

## Step 2.6: Handle Progress Files

For each file in `triage.obvious_dead.completed_progress_files`:
- Move to `archive/progress/{YYYY-MM}_{original_filename}`
- Append to manifest

For each file in `triage.needs_review.abandoned_progress_files` (`in_progress` and > 7 days old):
- Ask user via `AskUserQuestion`: "Found abandoned {type} progress from {date} ({age} days old). The work may have been interrupted. Archive it, or leave it for a potential resume?"
- If archive: move to `archive/progress/`
- If leave: skip

## Step 2.7: Handle Registry Drift

For flags from Step 1.1:
- **Plan in registry but folder missing**: downgrade to archived stub if possible (no archive_path), or prompt to remove the registry entry. Default: prompt.
- **Folder in `plans/` but not in registry**: prompt user — "Found orphan plan folder `{path}`. Archive it, leave it, or register it?" Don't auto-act; this could be a user's in-progress work.

## Step 2.8: Update Progress

Update `_rem_progress.json`: `phase: "prune"` → added to `phases_complete`.

**If mode is `prune`**: print summary, finalize, stop.

---

# PHASE 3: CONSOLIDATE

**Goal**: Merge, reorganize, and promote. Judgment-heavy work that proposes changes and asks the user to approve in batches. Never touches the knowledge graph directly — writes action items instead.

## Step 3.1: Launch Consolidation Agents (Parallel)

Spawn three opus agents in parallel (single message, multiple Task calls). Each:
- `subagent_type`: `"general-purpose"`
- `model`: `"opus"`
- `mode`: `"bypassPermissions"`
- `run_in_background`: `true`
- `max_turns`: `30`

### Agent A: Lessons Consolidator

```
You are consolidating the project-level lessons file for planning-with-james.

## CRITICAL: No Bash for file modifications
Use Edit/Write for file changes. Bash only for git and grep.

## Input
Read: .claude/planning-with-james/lessons.md

## Your Task
1. Identify entries that are:
   - Duplicates (same lesson worded differently in the same or different sections)
   - Stale (reference files, functions, or modules that no longer exist in the codebase — grep to verify)
   - Redundant with the knowledge graph (the lesson describes something now documented in .claude/planning-with-james/knowledge/modules/)
   - Miscategorized (a "Mistake to Avoid" that's really a "Pattern & Convention")

2. For each issue found, propose a specific action:
   - MERGE: combine duplicates into one stronger entry (provide the merged text)
   - REMOVE: drop stale entries (state what no longer exists)
   - RECATEGORIZE: move between sections
   - KEEP AS-IS with note

3. Do NOT apply changes yet. Write proposals to:
   .claude/planning-with-james/_rem_proposals/lessons_consolidation.md

Structure:
```markdown
# Lessons Consolidation Proposals

## Proposal Group 1: {theme}
### Action: MERGE
**Current entries:**
- {quote from lessons.md}
- {quote from lessons.md}

**Proposed merged entry:**
> {new text}

**Rationale**: {why merging helps}

### Action: REMOVE
...
```

4. After writing proposals, write a result file to signal completion:
.claude/planning-with-james/_rem_proposals/lessons_consolidation_result.json
{"agent": "lessons", "proposal_count": N, "summary": "One sentence"}
```

### Agent B: Forgotten-Lesson Promoter

This agent finds lessons in **paused** or **active-but-long-idle** plans that were never promoted because `/go-time` COMPLETE never ran. Phase 2 already handled promotion for archived plans; this agent handles the ones still active in the registry.

```
You are finding lessons in active/paused plans that never got promoted to project level.

## CRITICAL: No Bash for file modifications
Use Edit/Write for file changes.

## Input
- .claude/planning-with-james/plans/_registry.json (find non-archived plans)
- Each plan's lessons.md
- .claude/planning-with-james/lessons.md (project-level — for dedup checks)

## Your Task
For each plan with status "paused" (any age) or "planned"/"active" with last activity > 30 days:
1. Read its lessons.md
2. Identify entries that:
   - Are about the codebase itself (not the plan's specific problem)
   - Would help a fresh agent on a different plan
   - Are not already in project-level lessons.md
3. Propose promotion (rewritten for generality if needed)

Write proposals to:
.claude/planning-with-james/_rem_proposals/forgotten_lessons.md

Structure per proposal:
```markdown
## Proposal N: Promote from {plan-id}
**Source entry**:
> {original text from plan's lessons.md}

**Proposed project-level entry** (rewritten for generality):
> {new text}

**Target section**: Patterns & Conventions | Mistakes to Avoid | What Works Well

**Why promote**: {one sentence}
```

Write result:
.claude/planning-with-james/_rem_proposals/forgotten_lessons_result.json
{"agent": "forgotten_lessons", "proposal_count": N, "summary": "..."}
```

### Agent C: Knowledge Staleness Auditor

```
You are auditing the knowledge graph for semantic drift. You do NOT modify knowledge files — you write action item recommendations.

## CRITICAL: No Bash for file modifications
Use Edit/Write. Bash is for git only.

## CRITICAL: Hands off knowledge/modules/
This agent reads .claude/planning-with-james/knowledge/ but does not write to it. All findings go to action item proposals for the user to address via /planning-with-james:update-knowledge guided.

## Input
- .claude/planning-with-james/knowledge/_overview.md
- .claude/planning-with-james/knowledge/_discovery.json
- .claude/planning-with-james/_rem_triage.json (see dream_seeds.stale_modules)
- For each stale candidate: its _index.md and _refs.json

## Your Task
For each candidate module in triage.dream_seeds.stale_modules:
1. Read the module's _index.md
2. Read a sampling of the module's actual source files (via Glob to list, Read 2-3 representative files)
3. Compare: does the "Purpose" and "Key Files" section still match reality?
4. Classify:
   - FRESH: the knowledge is still accurate, flag was false positive
   - SHALLOW: accurate but thin — would benefit from a deep-dive update
   - DRIFTED: the module now does something materially different from what's documented
   - ORPHANED: the module path no longer exists or has been gutted

For SHALLOW, DRIFTED, and ORPHANED, propose an action item (not a fix). Format below.

Write proposals to:
.claude/planning-with-james/_rem_proposals/knowledge_staleness.md

Structure:
```markdown
## {module-id}: {classification}
**Knowledge says**: {brief summary of current _index.md}
**Code actually shows**: {what source reveals}
**Recommended action item**:
- priority: high | medium | low
- type: knowledge-update
- target: .claude/planning-with-james/knowledge/modules/{module-id}/
- recommended_action: `/planning-with-james:update-knowledge guided` with context "{specific instruction}"
```

Write result:
.claude/planning-with-james/_rem_proposals/knowledge_staleness_result.json
{"agent": "knowledge_staleness", "fresh": N, "shallow": N, "drifted": N, "orphaned": N}
```

## Step 3.2: Wait for Agents

Poll for `_result.json` files. Standard pattern:

```bash
EXPECTED=3; TIMEOUT=1200; ELAPSED=0
PROP_DIR=".claude/planning-with-james/_rem_proposals"
while [ $(find "$PROP_DIR" -name "*_result.json" 2>/dev/null | wc -l) -lt $EXPECTED ] && [ $ELAPSED -lt $TIMEOUT ]; do
  sleep 15; ELAPSED=$((ELAPSED + 15))
done
```

If any agent failed (no result file after timeout), re-launch just that one.

## Step 3.3: Present Proposals to User

Read all three proposal files. Group by category per `consolidate_batch_granularity` config (default `"category"`).

Use `AskUserQuestion` per group. Example groups:

> **Lessons consolidation (8 proposals)**
> - 4 MERGE proposals across 3 themes
> - 2 REMOVE proposals (stale references)
> - 2 RECATEGORIZE proposals
>
> Apply all / Review each / Skip all?

> **Forgotten-lesson promotions (3 proposals)**
> - From SB-1112 (paused): 2 entries worth promoting
> - From auth-refactor (paused): 1 entry
>
> Apply all / Review each / Skip all?

> **Knowledge graph action items (5 findings)**
> Would create 5 action items recommending `/update-knowledge guided`:
> - HIGH: hapag-scrapers (DRIFTED — code diverged significantly)
> - MEDIUM: 2 SHALLOW modules, 1 DRIFTED
> - LOW: 1 ORPHANED
>
> Create all items / Review each / Skip all?

## Step 3.4: Apply Approved Changes

For approved lesson consolidations and promotions: edit `.claude/planning-with-james/lessons.md` using Edit. Never use Bash to modify.

For approved knowledge action items: write each to `.claude/planning-with-james/action_items/{YYYY-MM-DD}_{slug}.md`. See **Action Item Format** below.

## Step 3.5: Clean Up Proposals

After user decisions, archive the proposal files:
- Move `_rem_proposals/*.md` to `archive/proposals/{YYYY-MM}/`
- Delete `_result.json` files
- Delete the `_rem_proposals/` directory if empty

## Step 3.6: Update Progress

`phase: "consolidate"` → added to `phases_complete`.

**If mode is `consolidate`**: finalize, stop.

---

# PHASE 4: DREAM

**Goal**: Let a team of agents cruise the whole repo (code, git history, knowledge graph, plans, MCP data sources) with different lenses. They message each other, build on each other's findings, and produce two outputs: dream journal entries (for leisurely reading) and action items (for urgent attention).

This phase uses the Team pattern: `TeamCreate`, `SendMessage` between members, `TeamDelete` when done. If these tools aren't loaded, use `ToolSearch` with query `select:TeamCreate,SendMessage,TeamDelete` to load their schemas first.

## Step 4.1: Create the Team

```
TeamCreate({team_name: "rem-dream-{YYYY-MM-DD-HHMM}"})
```

Record the team name in `_rem_progress.json` so resume can find it.

## Step 4.2: Prepare Shared Context

Write `dreams/{YYYY-MM-DD}/_shared_context.md` — a lean briefing every team member reads first:

```markdown
# Rem Dream Session — {date}

## What rem just did (triage + prune + consolidate)
- Archived: {N} plans, {N} explorations
- Consolidated: {N} lessons merged, {N} promotions
- Action items created: {N}

## Triage dream_seeds
{copy the dream_seeds section from _rem_triage.json}

## Repo scope for this dream
Root: {pwd}
Denylist (from rem-config): {list}

## Data sources available
- Code: on
- Git history: on
- Sentry: {auto/on/off — noted if unavailable}
- Vercel: {...}
- GCP: {...}

## Your teammates
- archaeologist (git & abandonment)
- cartographer (knowledge graph vs code truth)
- historian (plan archaeology — completed, paused, archived)
- detective (code weirdness)
- weaver (synthesizer — you'll receive from the others)

## Output conventions
- Your lens file: dreams/{date}/{lens}.md
- Send messages to teammates when you find something in their lane
- Flag anything urgent with `**ACTION ITEM CANDIDATE**` inline — Weaver will promote it

## Result signal
When you finish, write:
dreams/{date}/{lens}_result.json
{"lens": "{lens}", "notes_count": N, "action_candidates": N, "summary": "..."}
```

## Step 4.3: Spawn the Four Cruisers

Spawn four agents, all in the team, all background, all opus:

For each:
- `subagent_type`: `"general-purpose"`
- `model`: `"opus"`
- `mode`: `"bypassPermissions"`
- `run_in_background`: `true`
- `max_turns`: `40` (dreaming is exploratory; give them headroom)
- `team_name`: the rem-dream team
- `name`: `"archaeologist"` / `"cartographer"` / `"historian"` / `"detective"` (so teammates can `SendMessage({to: "historian", ...})`)

Launch all four in a single message with four Task calls.

### Cruiser A: Archaeologist

```
You are the Archaeologist on the rem dream team.

## CRITICAL: No Bash for file modifications
Use Edit/Write. Bash for git, grep, ls only.

## Shared context
Read: dreams/{date}/_shared_context.md

## Your lens
Git history, churn patterns, abandonment. You look at what's been dug up, what's been reburied, what layer things came from.

## Things to notice
- Files created in the last 90 days that haven't been imported anywhere (abandoned experiments)
- Modules with high churn but no knowledge graph updates (candidates for guided update)
- Branches merged with "wip", "experiment", "temp" commit messages
- Large commits that did many things (missing abstraction?)
- Files with `_old`, `_v2`, `.bak`, `_deprecated` naming — coexist with non-suffixed versions?
- Knowledge graph modules whose `last_commit` hasn't moved in 60+ days but whose source files have
- Recently-deleted files (check `git log --diff-filter=D --name-only --since="60 days ago"`) — were their callers updated?

## Interaction with teammates
- If you find a module that looks abandoned: SendMessage to historian asking "any plan reference {module}?"
- If a knowledge doc looks wrong: SendMessage to cartographer
- If you find dead code: SendMessage to detective

## Output
Write findings to: dreams/{date}/archaeologist.md

Structure:
```markdown
---
lens: archaeologist
dreamed_at: {ISO timestamp}
notes: N
action_candidates: N
---

# Archaeologist's Notes

## {observation title}
**What I saw**: {specific: files, commits, patterns}
**Why it might matter**: {significance}
**Tags**: [abandonment, churn, missing-abstraction, ...]
**ACTION ITEM CANDIDATE** (only if truly urgent — Weaver will judge)

## {next observation}
...

## Questions I sent to teammates
- "{question}" → {teammate} → {their answer or pending}
```

## Result file
When done:
dreams/{date}/archaeologist_result.json
{"lens": "archaeologist", "notes_count": N, "action_candidates": N, "summary": "..."}
```

### Cruiser B: Cartographer

```
You are the Cartographer on the rem dream team.

## CRITICAL: No Bash for file modifications + No writing to knowledge/modules/
You read the knowledge graph, you never write to it. Findings become action items.
Use Edit/Write for your own lens file.

## Shared context
Read: dreams/{date}/_shared_context.md

## Your lens
Where does the map lie? Cross-reference the knowledge graph against actual code state.

## Things to notice
- Modules whose `_index.md` Purpose section describes behavior the code no longer exhibits
- Orphan detail files (api.md, flows.md, etc.) in `modules/*/` that aren't referenced by the module's `_index.md`
- Modules marked `leaf` that now contain distinct sub-domains (should be containers)
- Modules marked `container` whose children have converged or been removed
- `_refs.json` entries pointing at paths that no longer exist
- Keywords in frontmatter that don't match current code
- Source files not covered by any module's path

## Spot-check method
For each candidate module from triage.dream_seeds.stale_modules, plus any you choose to sample:
1. Read _index.md
2. Glob the module path and read 2-3 representative files
3. Decide: fresh, shallow, drifted, orphaned

## Interaction with teammates
- Drifted modules that changed due to heavy churn: confirm with archaeologist ("did {module} get reshaped in the last 60 days?")
- If you find a pattern the historian should know about (module that keeps surprising plans): SendMessage

## Output
dreams/{date}/cartographer.md

Focus on **where the knowledge graph diverges from reality**. For urgent drifts affecting active planning areas, flag as ACTION ITEM CANDIDATE.

## Result file
dreams/{date}/cartographer_result.json
{"lens": "cartographer", "notes_count": N, "action_candidates": N, "summary": "..."}
```

### Cruiser C: Historian

```
You are the Historian on the rem dream team.

## CRITICAL: No Bash for file modifications
Use Edit/Write. Bash for ls/grep/git.

## Shared context
Read: dreams/{date}/_shared_context.md

## Your lens
Plan archaeology. Read completed, paused, and archived plans. Find recurring themes, recurring surprises, and patterns across plans that nobody has written down yet.

## Sources
- .claude/planning-with-james/plans/{non-archived}/
- .claude/planning-with-james/archive/plans/
- .claude/planning-with-james/epics/

For each, read: problem_description.md (summary + user flows), findings_summary.md, context_scratch_pad.md (especially session log, key decisions, open questions).

## Things to notice
- Surprises documented in scratch pads that keep showing up across plans ("we found that X actually...")
- User corrections that recur (same misunderstanding in multiple plans — maybe a knowledge graph issue)
- Modules that appear as "in scope" in many recent plans (hotspots)
- Modules that haven't been in scope of any plan in 6+ months (cold spots — still relevant?)
- Paused plans that look worth resurrecting given current state
- Abandoned approaches that might now be viable

## Interaction with teammates
- Recurring surprises about a module: SendMessage to cartographer ("does knowledge graph cover {X}? Multiple plans rediscovered it.")
- Hotspot modules with light knowledge docs: SendMessage to cartographer
- Cold spots that look dead: SendMessage to archaeologist ("any git activity on {module} recently?")

## Output
dreams/{date}/historian.md

Focus on **themes across plans** — things that emerge only when you read many plans together.

## Result file
dreams/{date}/historian_result.json
{"lens": "historian", "notes_count": N, "action_candidates": N, "summary": "..."}
```

### Cruiser D: Detective

```
You are the Detective on the rem dream team.

## CRITICAL: No Bash for file modifications
Use Edit/Write. Bash for ripgrep, grep, ls.

## Shared context
Read: dreams/{date}/_shared_context.md

## Your lens
Code weirdness. The stuff that makes engineers say "why is this here?"

## Things to hunt for
- TODO/FIXME/HACK/XXX comments older than 90 days (`git blame` to check age)
- Commented-out code blocks in committed files
- Duplicate implementations of the same concept across modules (date formatters, string normalizers, API clients)
- Config files with keys that are not referenced anywhere
- Feature flags that have been at the same value for 6+ months (candidates for ripping out)
- Functions/classes > 500 lines untouched for 6+ months (complexity debt)
- Naming drift: `getUserData`, `fetchUserData`, `loadUserData`, `retrieveUserData` all used for similar things
- Orphan files: files present but not imported/required/included anywhere in the codebase

## MCP sources (if available)
- Sentry: errors firing > 100x/day for > 14 days without an open plan addressing them
- Vercel: functions with p95 trending upward over 30 days
- GCP: log spam / silent failures

Skip gracefully if a source is unavailable — do not ask for auth.

## Interaction with teammates
- Duplicate implementations: SendMessage to cartographer ("are both modules documented as doing X?")
- Dead code in abandoned modules: SendMessage to archaeologist for confirmation
- Production errors suggesting a plan is needed: SendMessage to historian ("has any plan addressed {error}?")

## Output
dreams/{date}/detective.md

The Detective is the lens most likely to produce action items — real, runtime, happening-now weirdness. Don't oversell — flag only the things you'd actually want to fix this week.

## Result file
dreams/{date}/detective_result.json
{"lens": "detective", "notes_count": N, "action_candidates": N, "summary": "..."}
```

## Step 4.4: Wait for Cruisers

Poll for all four `_result.json` files. Timeout: 1800 seconds (dreaming takes longer than consolidation).

```bash
EXPECTED=4; TIMEOUT=1800; ELAPSED=0
DREAMS_DIR="dreams/{date}"
while [ $(find "$DREAMS_DIR" -name "*_result.json" 2>/dev/null | wc -l) -lt $EXPECTED ] && [ $ELAPSED -lt $TIMEOUT ]; do
  sleep 30; ELAPSED=$((ELAPSED + 30))
done
```

Re-launch any missing cruiser (don't re-run the whole team).

## Step 4.5: Capture the Team Transcript

If the team tool provides a transcript or message log, save it to `dreams/{date}/_transcript.md`. Otherwise, each cruiser's file already includes the "Questions I sent to teammates" section which serves as a partial transcript. This is for future tuning — the user can see what the dreamers discussed.

## Step 4.6: Spawn the Weaver

One final agent, also in the team (so it can SendMessage if it needs to ask a dreamer a clarifying question):

- `subagent_type`: `"general-purpose"`
- `model`: `"opus"`
- `mode`: `"bypassPermissions"`
- `team_name`: same rem-dream team
- `name`: `"weaver"`
- `max_turns`: `30`
- Not background — you need its output in context to finalize

```
You are the Weaver on the rem dream team. The cruisers have finished. Your job is to synthesize their notes into (a) a digest for the user and (b) a set of action items for the ones that need attention now.

## CRITICAL: No Bash for file modifications
Use Edit/Write.

## Read
- dreams/{date}/_shared_context.md (what rem did)
- dreams/{date}/archaeologist.md
- dreams/{date}/cartographer.md
- dreams/{date}/historian.md
- dreams/{date}/detective.md
- dreams/{date}/_transcript.md (if present)

## Your Task

### 1. Write the digest
dreams/{date}/_digest.md

```markdown
---
dreamed_at: {ISO timestamp}
lenses: [archaeologist, cartographer, historian, detective]
action_items_created: N
---

# Dream Digest — {date}

## TL;DR
{3-5 bullet points. The user reads this in 30 seconds and knows whether rem earned its keep.}

## Highlights by lens
### Archaeologist
- {bullet} → [deep note](archaeologist.md#{anchor})
- {bullet} → [deep note](archaeologist.md#{anchor})

### Cartographer
- {bullet} → [link]

### Historian
- {bullet} → [link]

### Detective
- {bullet} → [link]

## Cross-lens convergences
{Where two or more lenses noticed the same thing. This is the highest-signal content — when independent lenses agree, the finding is real.}

- {convergence description — which lenses, what they both saw}

## Action items created
- [{id}]({action_items/...}) — {title} — {priority}
- ...

## Questions for the user
{Things rem noticed but can't judge alone — e.g., "is SB-1130 still relevant or abandoned?"}
```

### 2. Create action items
For each ACTION ITEM CANDIDATE from the lens files, decide: promote or demote to regular dream note.

Promote if:
- High-priority knowledge graph drift affecting an area with active planning
- Production errors firing without a plan addressing them
- Unpromoted critical lesson
- Broken/abandoned infrastructure the team may not know about

Demote if:
- Interesting but not urgent
- Already addressed by another action item
- Speculative — needs human judgment more than action

You may also create action items from CROSS-LENS CONVERGENCES even if no individual lens flagged them — when multiple lenses converge, it's often action-worthy.

Write each action item as a file in:
.claude/planning-with-james/action_items/{YYYY-MM-DD}_{slug}.md

Format per Action Item Format spec (below). Use frontmatter with id, created_by: "rem", priority, type, target, recommended_action, status: "open".

### 3. Update the action items index
Read .claude/planning-with-james/action_items/_index.md (create if missing, see Action Items Index Format).

Add a row for each new action item under the appropriate priority section. Sort by priority then by created date.

### 4. Return result
Return a brief JSON summary (not to a file — conversationally):
{"digest_written": true, "action_items_created": N, "convergences_found": N}
```

## Step 4.7: Tear Down the Team

```
TeamDelete({team_name: "..."})
```

## Step 4.8: Update Progress

`phase: "dream"` → added to `phases_complete`.

---

# FINALIZATION

After the chosen phases complete:

## Update Progress to Complete

Mark `_rem_progress.json`:
```json
{
  "status": "completed",
  "completed_at": "{ISO timestamp}",
  "phases_complete": [...]
}
```

Move the progress file to `archive/rem-runs/{YYYY-MM-DD}_rem_progress.json` so the next run starts clean but the history is preserved.

Do the same for `_rem_triage.json`: move to `archive/rem-runs/{YYYY-MM-DD}_rem_triage.json`.

## Final Summary Output

Print a report:

```
REM Cycle Complete ({mode})
===========================
Triage:
  Plans surveyed:       {N}
  Archive candidates:   {N}
  Dream seeds:          {N}
  Commits behind:       {N}

Prune:
  Plans archived:       {N} ({N} completed, {N} paused)
  Explorations archived:{N}
  Stale sessions:       {N} cleaned
  Dangling locks:       {N} cleared
  Lessons promoted:     {N} (from plans archived before promotion)

Consolidate:
  Lesson merges:        {N}
  Lesson promotions:    {N}
  Staleness action items:{N}

Dream:
  Lenses run:           {N} (archaeologist, cartographer, historian, detective, weaver)
  Dream notes:          {N}
  Action items created: {N} ({N} high, {N} medium, {N} low)
  Cross-lens convergences: {N}
  MCP sources used:     {list, e.g. "sentry (skipped — auth), vercel (on)"}

Output locations:
  Dreams:          .claude/planning-with-james/dreams/{date}/_digest.md
  Action items:    .claude/planning-with-james/action_items/_index.md
  Archive:         .claude/planning-with-james/archive/_manifest.json

Quick links:
  {inline link to digest}
  {inline link to highest-priority action item, if any}
```

---

# STATUS MODE (`$ARGUMENTS` = "status")

No work. Report current state of rem-adjacent artifacts:

1. Read `_rem_progress.json` (or the latest archived run) — when was the last full cycle?
2. Count action items: open, by priority, and oldest-open.
3. Count recent dream sessions (last 90 days) with links to their digests.
4. Read `archive/_manifest.json` — archive volume over time (last run, last month, all-time).
5. Registry health: total plans, archived plans, active plans, paused > 30 days.

Output format:

```
REM Status
==========
Last full cycle:      {date} ({days ago})
Last triage:          {date}
Last dream:           {date}

Action items open:    {N}
  High priority:      {N} (oldest: {date})
  Medium priority:    {N}
  Low priority:       {N}

Recent dreams:
  {date} — [digest]({link}) — {N} notes, {N} action items
  {date} — ...

Archive:
  Plans:              {N} ({size})
  Explorations:       {N}
  Last archive event: {date}

Registry:
  Active plans:       {N}
  Paused plans:       {N} ({N} over 30 days — rem would archive these)
  Archived stubs:     {N}

Next suggestion:
  {e.g., "Run /rem — {N} archive candidates and {N} open action items to consolidate."
   Or "No maintenance needed right now."}
```

---

# APPENDIX: DATA FORMATS

## Archive Manifest Format

`.claude/planning-with-james/archive/_manifest.json`:

```json
{
  "entries": [
    {
      "id": "SB-1112",
      "type": "plan",
      "archived_at": "2026-04-23T10:00:00Z",
      "archived_by": "rem",
      "original_path": ".claude/planning-with-james/plans/SB-1112",
      "archive_path": ".claude/planning-with-james/archive/plans/2026-04/SB-1112/",
      "reason": "completed_30d",
      "age_days_at_archive": 45,
      "metadata": {
        "archived_from_status": "completed",
        "lessons_promoted": 2,
        "problem_summary": "Return UNLOCODEs in rate search response"
      }
    },
    {
      "id": "how-auth-works_2026-01-15",
      "type": "exploration",
      "archived_at": "...",
      "original_path": "...",
      "archive_path": "...",
      "reason": "exploration_30d"
    }
  ]
}
```

Append to `entries` — never mutate existing entries.

## Action Item Format

`.claude/planning-with-james/action_items/{YYYY-MM-DD}_{slug}.md`:

```markdown
---
id: 2026-04-23_hapag-scrapers-knowledge-stale
created_at: 2026-04-23T10:30:00Z
created_by: rem
source: dream | consolidate | triage
priority: high | medium | low
type: knowledge-update | lesson-promotion | code-cleanup | investigation | plan-revive
target: .claude/planning-with-james/knowledge/modules/hapag-scrapers/
recommended_action: /planning-with-james:update-knowledge guided
status: open
tags: [knowledge-drift, hotspot]
---

# {Short descriptive title}

## What rem noticed
{Specific observation — quote code, paths, commits, agent findings}

## Why it matters
{Impact on planning, knowledge accuracy, or code health}

## Recommended action
{Specific command to run, optionally with the $ARGUMENTS text to pass}

**Example**:
```
/planning-with-james:update-knowledge guided

The hapag-scrapers module has had 34 commits since its knowledge
was last indexed. The Cartographer noted the "Purpose" paragraph
still describes the legacy playwright-based scraper, but the code
has moved to puppeteer-stealth with a new entry point.
```

## Related findings
{Links to dream notes, triage flags, or other action items that corroborate}

- dreams/2026-04-23/cartographer.md#hapag-scrapers
- dreams/2026-04-23/archaeologist.md#high-churn-modules
```

User closes an item by moving it to `action_items/resolved/`. Or edits status to `closed` and leaves it.

## Action Items Index Format

`.claude/planning-with-james/action_items/_index.md`:

```markdown
---
last_updated: {ISO timestamp}
open_count: N
high_priority_count: N
---

# Action Items

Open items, sorted by priority then date. Close by moving to `resolved/` or editing the item's `status` field.

## High Priority

| Created | ID | Title | Type | Target |
|---------|-----|-------|------|--------|
| {date} | [{id}]({file}) | {title} | {type} | {target} |

## Medium Priority
{same table}

## Low Priority
{same table}

## Recently Resolved
{optional — last 5 from resolved/, for continuity}
```

---

# EDGE CASES

## Running rem during active planning or go-time
If another session has an active plan (`sessions` has entries < 24h old, plan is `locked_by_session`), rem should:
- Skip archiving that specific plan
- Skip cleaning that session binding
- Everything else proceeds normally
- Note in final summary: "Skipped {N} plans with active sessions."

## Registry has only `archived` stubs
Not a problem. Archived stubs are cheap. Skip them in triage.

## Knowledge graph missing
If `.claude/planning-with-james/knowledge/` doesn't exist:
- Skip staleness auditing (Phase 3 Agent C)
- Cartographer in Phase 4 has nothing to audit — have it note this and contribute weirdness observations instead
- Don't fail; rem is useful even without a knowledge graph

## First-time invocation (no archive folder, no action_items folder)
Create directories as needed. Initialize `archive/_manifest.json` with `{"entries": []}`.

## MCP tools require auth during dream
Each cruiser catches auth errors and notes "skipped {source}: auth required" in their output. Never block the run.

## Resume after interruption
On re-invocation, `_rem_progress.json` with `in_progress` status triggers resume. Skip phases listed in `phases_complete`. The triage file on disk is still valid if written — re-use it.

## Dreaming produces too many action items
If Weaver creates more than 10 action items in a single run, prioritize ruthlessly — demote the bottom of the list to dream notes. Ten is a rough ceiling; the user can't act on more than that.

## User wants to tune dream lenses
Document in `rem-config.json` a `dream.lenses` array (future enhancement). For v1, the four lenses are fixed.

---

# DESIGN NOTES (for future maintainers)

**Why the 4+1 team.** Four lenses cover static (cartographer), temporal (archaeologist), archival (historian), and operational (detective). The weaver is the only member who reads everyone else's work, which keeps synthesis separate from discovery. This prevents any single lens from being biased toward action.

**Why registry stubs.** Archiving a plan entirely from the registry would break `/plan`'s "did we work on something similar?" discovery. A stub keeps findability cheap while the heavy content lives in `archive/`. The registry stays small because stubs are ~5 fields.

**Why action items are a separate concept from dreams.** Dreams are memory (passive, read when curious). Action items are an inbox (active, have a lifecycle). Merging them would either make dreams annoying ("journal that nags") or make action items lose context ("ticket without the story"). Keeping them separate lets each do its job.

**Why rem doesn't touch knowledge graphs directly.** One writer per artifact. `create-knowledge` and `update-knowledge` own the graph. rem observes and recommends. This mirrors how `/dig` is read-only against the graph.

**Why the archive is recoverable with `mv`.** The alternative — a `/rem restore` command — is more code for a rare operation. `mv archive/plans/2026-04/SB-1112 plans/SB-1112` and edit the registry stub back to a full entry. The manifest tells you what to mv. Simpler.
