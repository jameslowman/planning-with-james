---
name: epic
description: Create and manage multi-plan epics that span weeks or months. Coordinate sub-plans, carry context across sessions, and run epic reviews.
disable-model-invocation: true
allowed-tools: Bash, Glob, Grep, Read, Write, Task, Edit, AskUserQuestion
---

# Epic: Plan of Plans

**NON-NEGOTIABLE: ALL PATHS ARE RELATIVE TO THE REPO ROOT**
All `.claude/planning-with-james/` paths in this skill are relative to the **current working directory** (the repo you're working in), NOT `~/.claude/`. The knowledge graph and plans live inside the project, not in your home directory. If you're unsure, run `pwd` to confirm you're in the repo root.

An epic is a large initiative that breaks down into multiple sequential plans, each with its own discovery, planning, implementation, and validation. Epics solve the problem of carrying context across dozens or hundreds of Claude Code sessions over weeks or months.

**The core mechanism is `learnings.md`** - a curated, growing document that accumulates architectural decisions, discoveries, and insights from every completed sub-plan. Any fresh session reads this file and immediately has the full history.

---

## MANDATORY: Knowledge Graph Before Source Code

When exploring code for any reason during epic creation or review, you MUST read the knowledge graph first:

1. Read `.claude/planning-with-james/knowledge/_overview.md`
2. Read `.claude/planning-with-james/knowledge/_graph.json`
3. Read relevant module `_index.md` files
4. THEN explore source code for specific gaps

When spawning subagents, ALWAYS include relevant knowledge file paths in their prompts.

---

## Arguments

`$ARGUMENTS` controls behavior:

| Argument | Behavior |
|----------|----------|
| *(empty)* | Show existing epics, or start creating a new one |
| `review` | Run epic review after a sub-plan completes |
| `status` | Show all epics and their sub-plan progress |
| `{epic-id}` | Show details for a specific epic |

---

## Epic Structure

```
{epic-folder}/
├── _epic_state.json           # Epic metadata, sub-plan tracking
├── epic_context.md            # Business story, goals, constraints, history
├── epic_plan.md               # Overall shape - all sub-plans outlined
├── learnings.md               # THE carried context file - grows with each review
├── validation_gates.md        # What to verify between sub-plans
└── sub-plans/
    ├── 01-{name}/
    │   ├── completion_review.md   # Written during epic review
    │   └── (all normal plan files once /plan runs)
    ├── 02-{name}/
    │   └── (outlined only until it's next)
    └── ...
```

---

# CREATING AN EPIC

## If `$ARGUMENTS` is empty and no epics exist (or user wants a new one):

### Step 1: Verify Knowledge Base

Check for `.claude/planning-with-james/knowledge/`. If missing:
> "No knowledge base found. Run `/planning-with-james:create-knowledge` first. Epics require deep codebase knowledge."

Stop if missing.

### Step 2: Gather Epic Context

Use `AskUserQuestion` to understand the initiative. Ask:

**Question 1**: "Describe the full initiative. What are you trying to accomplish and why? Include the business context, history, and what's been tried before."

**Question 2**: "What are the major phases or milestones you see? Don't worry about getting them perfect - we'll refine these."

**Question 3**: "What constraints exist? Budget, timeline, dependencies on other teams, production safety concerns?"

If the user provides a long description as $ARGUMENTS or in conversation, parse it instead of asking questions that are already answered.

### Step 3: Research via Knowledge Graph

Read the knowledge graph to understand the technical landscape:

1. Read `_overview.md` and `_architecture.md`
2. Read `_graph.json` to understand module relationships
3. Read `_index.md` for every module the user's description touches
4. Identify which modules are involved across the full initiative

### Step 4: Break Down into Sub-Plans

Based on the user's description and the knowledge graph, identify the natural sub-plans. Each sub-plan should be:

- **Independently implementable** - can be completed, tested, and deployed on its own
- **Sequentially dependent** - later plans build on earlier ones
- **Scope-bounded** - fits in a single plan/go-time cycle (days to ~2 weeks of work)

For each sub-plan, write a high-level outline:
- What it accomplishes
- Which modules it touches
- Key technical challenges
- What it depends on from previous sub-plans
- What downstream sub-plans depend on from it

**Detail the first sub-plan.** Give it enough structure that `/plan` can use it as a starting point. Outline the rest.

### Step 5: Define Validation Gates

Between each sub-plan, define what must be verified before moving on. These are typically:
- Deploy and verify in staging/production
- Live testing with real data
- Performance measurement
- User acceptance
- Cost verification
- Integration testing with other systems

Validation gates are human-driven. The system surfaces them; the human executes and confirms.

### Step 6: Ask for Epic Location

Use `AskUserQuestion`:
> "Where should I create the epic folder?
>
> Options:
> 1. `.claude/planning-with-james/epics/{epic-name}/` (recommended)
> 2. Custom location
>
> What's the epic name? (Short identifier, e.g., 'passive-v2')"

### Step 7: Write Epic Documents

Create the folder structure and write:

**`_epic_state.json`:**
```json
{
  "epic_id": "{epic-id}",
  "epic_name": "{descriptive name}",
  "epic_folder": "{full path}",
  "created_at": "{timestamp}",
  "status": "active",
  "sub_plans": [
    {
      "id": "01-{name}",
      "name": "{descriptive name}",
      "status": "ready_to_plan",
      "plan_path": "{epic_folder}/sub-plans/01-{name}"
    },
    {
      "id": "02-{name}",
      "name": "{descriptive name}",
      "status": "outlined",
      "plan_path": null
    }
  ],
  "current_sub_plan": "01-{name}",
  "reviews_completed": [],
  "last_updated": "{timestamp}"
}
```

**`epic_context.md`:**
```markdown
---
epic_id: {id}
created_at: {timestamp}
---

# Epic Context: {Name}

## Business Story

{The user's full description - preserve their words. This is the "why" that every
sub-plan's agents need to understand.}

## Goals

{What success looks like for the full initiative}

## Constraints

{Budget, timeline, technical constraints, team dependencies}

## History

{What was tried before, what failed, why. This prevents repeating mistakes.}

## Technical Landscape

{Summary from knowledge graph - which modules are involved, how they connect,
key architectural considerations}
```

**`epic_plan.md`:**
```markdown
---
epic_id: {id}
created_at: {timestamp}
last_reviewed: {timestamp}
---

# Epic Plan: {Name}

## Overview

{One paragraph summary of the full initiative}

## Sub-Plans

### 1. {Name} [READY TO PLAN]

**Goal**: {what this accomplishes}
**Modules**: {which modules it touches}
**Key Challenges**: {technical challenges}
**Depends On**: Nothing (first sub-plan)
**Downstream Impact**: {what later plans need from this}

**Detailed Outline**:
{Enough detail for /plan to use as a starting point. Include:
- Key areas to investigate
- Likely approach direction
- Files/modules likely involved
- Known risks}

### 2. {Name} [OUTLINE]

**Goal**: {what this accomplishes}
**Modules**: {which modules it touches}
**Key Challenges**: {technical challenges}
**Depends On**: Sub-Plan 1 ({specific deliverables})
**Downstream Impact**: {what later plans need from this}

**Rough Outline**:
{High-level description. Will be detailed after Sub-Plan 1 completes
and epic review runs.}

### 3. {Name} [OUTLINE]
...

## Sequence Rationale

{Why this ordering? What drives the dependencies?}
```

**`learnings.md`:**
```markdown
---
epic_id: {id}
created_at: {timestamp}
last_updated: {timestamp}
entries: 0
---

# Epic Learnings: {Name}

**Read this file before starting any sub-plan.** It contains accumulated context
from all completed sub-plans. This is how you understand months of work in minutes.

## Architecture Decisions (Locked In)

{Decisions that have been made and cannot be revisited. Include rationale.}

(None yet - this section grows with each epic review)

## Discoveries That Changed the Plan

{Things we learned that altered downstream sub-plans}

(None yet)

## Assumptions Proved Wrong

{Things we assumed that turned out to be false}

(None yet)

## Technical Debt Introduced (Deliberate)

{Shortcuts taken deliberately, with justification and cleanup plan}

(None yet)

## Patterns Established

{Patterns that downstream sub-plans should follow for consistency}

(None yet)

## Risks: Materialized vs Avoided

| Risk | Status | Notes |
|------|--------|-------|
(None yet)
```

**`validation_gates.md`:**
```markdown
---
epic_id: {id}
---

# Validation Gates

## After Sub-Plan 1: {Name}

Before starting Sub-Plan 2, verify:
- [ ] {gate 1 - e.g., "Deploy to staging, verify emails flowing"}
- [ ] {gate 2 - e.g., "Cost per email below $X threshold"}
- [ ] {gate 3}
- [ ] Epic review completed

## After Sub-Plan 2: {Name}

Before starting Sub-Plan 3, verify:
- [ ] {gate 1}
- [ ] {gate 2}
- [ ] Epic review completed

...
```

### Step 8: Create First Sub-Plan Folder

Create `{epic_folder}/sub-plans/01-{name}/` with a minimal `_plan_state.json`:

```json
{
  "plan_name": "01-{name}",
  "plan_folder": "{path}",
  "created_at": "{timestamp}",
  "current_phase": 0,
  "phase_status": {},
  "epic": {
    "epic_id": "{epic-id}",
    "epic_folder": "{epic folder path}",
    "sub_plan_order": 1,
    "sub_plan_total": {N}
  }
}
```

### Step 9: Register in Registry

Update `.claude/planning-with-james/plans/_registry.json`:

Add the first sub-plan:
```json
{
  "01-{name}": {
    "name": "{descriptive name}",
    "path": "{sub-plan folder path}",
    "created_at": "{timestamp}",
    "status": "planned",
    "current_task": null,
    "epic": "{epic-id}"
  }
}
```

Add the epic to the registry (create `epics` section if needed):
```json
{
  "epics": {
    "{epic-id}": {
      "name": "{descriptive name}",
      "path": "{epic folder path}",
      "created_at": "{timestamp}",
      "status": "active",
      "current_sub_plan": "01-{name}",
      "total_sub_plans": {N},
      "completed_sub_plans": 0
    }
  }
}
```

### Step 10: Present to User

> "Epic created: {name}
>
> **Sub-Plans** ({N} total):
> 1. {name} [READY TO PLAN] - {one line}
> 2. {name} [OUTLINE] - {one line}
> 3. {name} [OUTLINE] - {one line}
> ...
>
> **Next step**: Run `/planning-with-james:plan` to detail-plan the first sub-plan.
> The plan skill will automatically read the epic context and use the outline as a starting point.
>
> **Validation gates** are defined between each sub-plan. After completing each one, run
> `/planning-with-james:epic review` to capture learnings and prepare the next sub-plan."

---

# EPIC REVIEW

**Trigger**: `/planning-with-james:epic review`

This is the critical process that carries context forward. Run this after a sub-plan completes (implementation done via `/go-time`) and any validation gates have been passed by the user.

## Step 1: Identify What to Review

Read `.claude/planning-with-james/plans/_registry.json` to find:
- Which epic has a recently completed sub-plan
- If ambiguous, ask the user which epic/sub-plan to review

Read the epic's `_epic_state.json` to confirm current state.

## Step 2: Read Everything from the Completed Sub-Plan

Read ALL of these files from the completed sub-plan's folder:
- `context_scratch_pad.md` - The narrative of what happened
- `problem_description.md` - What we set out to do
- `scope.md` - What was in/out of scope
- `findings_summary.md` - What discovery revealed
- `approach.md` - What approach was chosen and why
- `detailed_plan.md` - What was planned
- `tasks.md` - What was actually done (checked off items, any added/skipped tasks)

Also read:
- The epic's current `learnings.md`
- The epic's current `epic_plan.md`
- The validation gate results (ask user what passed/failed)

## Step 3: Synthesize via Opus Agent

Spawn a Task agent (model: opus) to synthesize the review:

```
You are conducting an EPIC REVIEW for the "{epic_name}" epic.
Sub-plan "{sub_plan_name}" has been completed.

Your job is to extract everything that matters for future sub-plans from this
completed work. Think of yourself as a historian writing for someone who has
zero memory of what happened.

## Read These Files First:
{list all files from Step 2}

## Current Epic Learnings:
{paste or reference learnings.md}

## Remaining Sub-Plans (from epic_plan.md):
{paste the outlines of remaining sub-plans}

## Your Task:

Write TWO documents:

### 1. completion_review.md for this sub-plan:

```markdown
---
sub_plan: {id}
reviewed_at: {timestamp}
---

# Completion Review: {Name}

## What Was Accomplished
{Concrete deliverables - what exists now that didn't before}

## Key Decisions Made
{Every significant decision, with rationale. These may be locked in.}

## What Was Discovered During Implementation
{Surprises, things that were different than expected}

## Deviations from Original Plan
{What changed and why - added tasks, skipped tasks, approach changes}

## Impact on Downstream Sub-Plans
{Specific, actionable impacts. "Sub-Plan 2 should know that..." format}

## Technical Debt Introduced
{Any deliberate shortcuts, with justification}

## Code Locations
{Key files created/modified, for reference by future sub-plans}
```

### 2. Updated learnings.md sections:

For each section of learnings.md, provide the NEW entries to add
(do not repeat existing entries). Only add genuinely important insights
that future sub-plans need to know. Be selective - this document must
stay readable across months of work.

Focus on:
- Architecture decisions that are now locked in
- Assumptions that proved wrong
- Patterns established that must be followed
- Discoveries that change the shape of downstream sub-plans
```

## Step 4: Write the Completion Review

Save the completion review to `{sub-plan-folder}/completion_review.md`

## Step 5: Update Learnings

Read the current `learnings.md`, then append the new entries from the synthesis agent into the appropriate sections. Do not rewrite existing entries - add to them.

**Keep learnings.md curated.** This file will be read by every future session. If it grows too long, it becomes useless. Each entry should be:
- Specific and actionable (not vague observations)
- Contextualized (why does this matter for future work)
- Concise (one paragraph max per entry)

## Step 6: Re-evaluate Remaining Sub-Plans

Read the updated `learnings.md` and the remaining sub-plan outlines in `epic_plan.md`.

Ask:
- Does the next sub-plan's outline still make sense given what we learned?
- Do any sub-plans need to be added, removed, or reordered?
- Have the scope or goals of remaining sub-plans shifted?

Update `epic_plan.md` with revised outlines. Mark changes clearly:
```markdown
### 2. Pipeline Rebuild [OUTLINE - REVISED after Sub-Plan 1]

**Changes from original outline**:
- Originally planned PostgreSQL storage, now using {X} based on Sub-Plan 1 findings
- Added requirement for {Y} based on cost analysis
```

## Step 7: Update Epic State

Update `_epic_state.json`:
- Mark completed sub-plan as `"status": "completed"`
- Set next sub-plan to `"status": "ready_to_plan"`
- Update `current_sub_plan`
- Add to `reviews_completed`

Check off the completed sub-plan's validation gate in `validation_gates.md`.

## Step 8: Prepare Next Sub-Plan

Create the next sub-plan's folder: `{epic_folder}/sub-plans/{NN}-{name}/`

Create a minimal `_plan_state.json` with the epic reference (same as Step 8 in creation).

Register the next sub-plan in `_registry.json` with status `"planned"` and the `epic` field.

## Step 9: Present Review to User

> "Epic review complete for {sub-plan-name}.
>
> **Key Learnings Added**:
> - {learning 1}
> - {learning 2}
>
> **Changes to Remaining Sub-Plans**:
> - {change 1, or "No changes needed"}
>
> **Next Sub-Plan**: {name}
> **Validation Gate**: {what needs to be verified before starting}
>
> When the validation gate is satisfied, run `/planning-with-james:plan`
> to detail-plan the next sub-plan. It will automatically read the epic context
> and all accumulated learnings."

---

# EPIC STATUS

**Trigger**: `/planning-with-james:epic status`

1. Read all epics from the registry
2. For each epic, read `_epic_state.json`
3. Display:

```
Epic: Passive V2 - Smart Email Pipeline
Status: In Progress (2/4 sub-plans complete)

| # | Sub-Plan | Status | Reviewed |
|---|----------|--------|----------|
| 1 | Webhook Discovery | completed | yes |
| 2 | Pipeline Rebuild | active (Task 2.3) | - |
| 3 | Pre-processing | outlined | - |
| 4 | Dead Code Cleanup | outlined | - |

Next validation gate: After Pipeline Rebuild - deploy to staging, verify email flow
```

---

# EPIC DETAILS

**Trigger**: `/planning-with-james:epic {epic-id}`

1. Read `_epic_state.json`
2. Read `learnings.md`
3. Read `epic_plan.md`
4. Present a comprehensive summary:
   - Current state
   - Key learnings so far
   - Next sub-plan outline
   - Upcoming validation gates

---

# RESUMING AN EPIC

If the user invokes `/planning-with-james:epic` with no arguments and epics exist:

1. Show existing epics
2. Ask: "Continue with an existing epic, or create a new one?"
3. If continuing: show the epic's current state and what action is needed next
   - If a sub-plan needs reviewing: suggest `/planning-with-james:epic review`
   - If a sub-plan needs planning: suggest `/planning-with-james:plan`
   - If a sub-plan needs implementing: suggest `/planning-with-james:go-time {plan-id}`
   - If a validation gate needs checking: show the gate requirements

---

# EPIC COMPLETION

When all sub-plans are completed and reviewed:

1. Read all `completion_review.md` files
2. Read final `learnings.md`
3. Write a final epic summary in `epic_context.md` (append a "## Completion" section)
4. Update `_epic_state.json`: status → `"completed"`
5. Update registry

> "Epic complete: {name}
>
> **Duration**: {first created} to {now}
> **Sub-Plans Completed**: {N}
> **Key Outcomes**: {summary from learnings.md}
>
> All learnings preserved in {epic_folder}/learnings.md for future reference."
