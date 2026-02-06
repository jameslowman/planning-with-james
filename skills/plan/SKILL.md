---
name: plan
description: Plan a feature, bug fix, refactor, or new system with structured phases and checkpoints.
disable-model-invocation: true
allowed-tools: Bash, Glob, Grep, Read, Write, Task, Edit, AskUserQuestion
---

# Plan a Feature

**NON-NEGOTIABLE: ALL PATHS ARE RELATIVE TO THE REPO ROOT**
All `.claude/planning-with-james/` paths in this skill are relative to the **current working directory** (the repo you're working in), NOT `~/.claude/`. The knowledge graph and plans live inside the project, not in your home directory. If you're unsure, run `pwd` to confirm you're in the repo root.

This skill guides you through a structured planning process with multiple phases and checkpoints. It produces a comprehensive plan that can survive context loss and guide implementation.

---

## MANDATORY: Knowledge Graph Before Source Code

This codebase has a knowledge graph that already contains module boundaries, key files, public interfaces, internal patterns, cross-module dependencies, and documented gotchas. **Starting from source code means wasting time rediscovering information that is already written down.**

**YOU MUST read the knowledge graph before using Glob, Grep, or Read on any source file:**

1. Read `.claude/planning-with-james/knowledge/_overview.md`
2. Read `.claude/planning-with-james/knowledge/_graph.json`
3. Read `.claude/planning-with-james/knowledge/modules/{relevant}/_index.md` for each area you're investigating
4. ONLY THEN explore source code to fill specific gaps

This applies to:
- Initial discovery
- Plan amendments and revisions
- Debugging during implementation
- Any research spawned by unexpected findings

**When spawning subagents** (Task tool with Explore or other types), you MUST include in their prompt:
```
BEFORE exploring source code, read these knowledge files:
- .claude/planning-with-james/knowledge/_overview.md
- .claude/planning-with-james/knowledge/modules/{module}/_index.md
(list the specific module paths relevant to the subagent's scope)
These files contain documented module boundaries, key files, interfaces, patterns, and gotchas. Start there.
```

Subagents that skip the knowledge graph will duplicate work and miss documented context.

---

## Phase Overview

```
Phase 0: Setup           → Create folder, copy templates, register plan
Phase 1: Context         → Gather problem description from user
Phase 2: Scoping         → Identify modules, set boundaries
Phase 3: Discovery       → Parallel deep exploration
Phase 4: Approach        → Decide on technical direction
Phase 5: Detailed Plan   → Full implementation specification
Phase 6: Task Breakdown  → Executable checklist with dependencies
Phase 7: Finalize        → Cold start doc ready for implementation
```

Each phase ends with a **checkpoint** - but checkpoints are **adaptive**, not mandatory stops.

---

## Checkpoint Philosophy

**The point of planning is the conversation.**

Every checkpoint is an opportunity to draw information out of the user -- context they haven't thought to share, corrections to your assumptions, better ideas than what you found. The goal isn't to produce a plan artifact quickly; it's to build shared understanding.

**ALWAYS present your findings and invite correction:**
- Phase 1: "Here's my understanding. What did I miss?"
- Phase 2: "Here's what I think is in scope. What else should I consider?"
- Phase 3: "Here's what I discovered. Does this match your understanding?"
- Phase 4: "Here's the approach I recommend. Do you see a better way?"
- Phase 5: "Here's the detailed plan. Any concerns?"
- Phase 6: "Here's the task breakdown. Ready to finalize?"

**The user is your best source of truth.** They know:
- Real performance numbers (not stale estimates from old code)
- Business constraints not visible in code
- What was tried before and why it failed
- Edge cases from production experience
- Their own mental model of how things should work

**Never assume the knowledge graph is complete or correct.** It's a starting point, not ground truth. Discovery agents can report wrong information. Always validate findings with the user.

**Speed comes from clarity, not from skipping steps.** A well-understood problem with aligned mental models executes fast. A misunderstood problem with a rushed plan wastes days.

---

# PHASE 0: SETUP

## Step 1: Verify Knowledge Base

First, verify the knowledge base exists:

```
.claude/planning-with-james/knowledge/
```

If missing, inform user:
> "No knowledge base found. Run `/planning-with-james:create-knowledge` first to index the codebase. Planning without knowledge is like navigating without a map."

Stop execution if knowledge base doesn't exist.

## Step 1b: Check for Epic Context

Check `.claude/planning-with-james/plans/_registry.json` for plans with an `epic` field that have status `"planned"` (created by the epic skill, waiting to be detail-planned).

Also check if there are any active epics in the registry's `epics` section with a sub-plan in `"ready_to_plan"` status.

**If this plan is part of an epic:**

1. Read the epic's `epic_context.md` - understand the full business story
2. Read the epic's `learnings.md` - understand everything learned so far
3. Read the epic's `epic_plan.md` - find this sub-plan's outline
4. Read any `completion_review.md` files from previous sub-plans

This context informs EVERY phase of planning. Include it in:
- Phase 1: The problem description already has context from the epic
- Phase 2: Scoping is guided by the epic outline and learnings
- Phase 3: Discovery agents get epic context in their prompts
- Phase 4: Approach is constrained by architecture decisions in learnings.md
- Phase 5: Detailed plan follows patterns established in learnings.md

**If the plan folder already exists** (created by the epic skill): skip Step 2 (location question) and use the existing folder.

**If not part of an epic:** continue normally.

## Step 2: Ask for Plan Location

Use `AskUserQuestion` to ask:

> "Where should I create the planning folder?
>
> Options:
> 1. `.claude/planning-with-james/plans/{feature-name}/` (recommended - keeps plans with plugin)
> 2. `docs/plans/{feature-name}/` (if you want plans in version control)
> 3. Custom location
>
> What's the feature/bug name? (This becomes the folder name)"

## Step 3: Create Folder Structure

Create the plan folder with all template files:

```
{plan_folder}/
├── _plan_state.json           # Plugin state tracking
├── context_scratch_pad.md     # Cold restart doc (maintained throughout)
├── problem_description.md     # Phase 1 output
├── scope.md                   # Phase 2 output
├── findings/                  # Phase 3 outputs
│   └── (created during discovery)
├── findings_summary.md        # Phase 3 synthesis
├── approach.md                # Phase 4 output
├── detailed_plan.md           # Phase 5 output
└── tasks.md                   # Phase 6 output
```

## Step 4: Initialize Plan State

Create `_plan_state.json`:

```json
{
  "plan_name": "{feature-name}",
  "plan_folder": "{full path}",
  "created_at": "{ISO timestamp}",
  "current_phase": 0,
  "phase_status": {
    "0_setup": "complete",
    "1_context": "pending",
    "2_scoping": "pending",
    "3_discovery": "pending",
    "4_approach": "pending",
    "5_detailed_plan": "pending",
    "6_tasks": "pending",
    "7_finalize": "pending"
  },
  "problem_type": null,
  "in_scope_modules": [],
  "checkpoints_passed": []
}
```

Also register this plan in `.claude/planning-with-james/plans/_registry.json`:

```json
{
  "active_plan": null,
  "plans": {
    "{feature-name}": {
      "name": "{descriptive plan name}",
      "path": "{plan_folder full path}",
      "created_at": "{timestamp}",
      "status": "planning",
      "current_task": null
    }
  }
}
```

If `_registry.json` already exists, merge the new plan into the existing `plans` object. Do not overwrite other plans.

**Plan lifecycle states**: `planning` → `planned` → `active` ⇄ `paused` → `completed`. Only `/go-time` changes status beyond `planned`. See `/planning-with-james:go-time` for details.

## Step 5: Initialize Cold Start Doc

Create initial `context_scratch_pad.md`:

```markdown
---
plan_name: {feature-name}
created_at: {timestamp}
current_phase: 0 - Setup
last_updated: {timestamp}
---

# Context Scratch Pad

This document maintains continuity across sessions. If you're resuming after a context loss, start here.

## Current State
- **Phase**: Setup complete
- **Next**: Phase 1 - Context Gathering

## Plan Documents
- [ ] problem_description.md - Not started
- [ ] scope.md - Not started
- [ ] findings/ - Not started
- [ ] approach.md - Not started
- [ ] detailed_plan.md - Not started
- [ ] tasks.md - Not started

## Key Decisions
(None yet)

## Open Questions
(None yet)

## Session Log
- {timestamp}: Plan initialized
```

## Step 6: Load Planning Context

Read internal planning guidance (these inform your behavior but aren't shown to user):
- How to ask good questions
- What makes a good problem description
- Discovery strategies
- Plan structure best practices

Now proceed to Phase 1.

---

# PHASE 1: CONTEXT GATHERING

**Goal**: Understand what we're solving from the user's perspective.

## Step 1: Initial Questions

Use `AskUserQuestion` with these questions (adapt based on what user already provided):

**Question 1: Problem Type**
> "What type of work is this?
> - Bug fix (something's broken)
> - Feature (new capability)
> - Refactor (improve existing code)
> - New system (build from scratch)"

**Question 2: Description**
> "Describe the problem or goal. Include:
> - What's happening (or should happen)
> - Any error messages, screenshots, or logs
> - Links to tickets (Linear, GitHub, etc.)
> - Your theories about the cause or approach"

**Question 3: Success Criteria**
> "How will we know this is solved? What does success look like?"

## Step 2: Process Additional Context

If user provides:
- **Linear/GitHub URLs**: Fetch and extract relevant details
- **Screenshots**: Analyze them
- **Error logs**: Parse and identify key information
- **Existing code references**: Note them for discovery

## Step 3: Write Problem Description

Create `problem_description.md`:

```markdown
---
problem_type: {bug|feature|refactor|new_system}
created_at: {timestamp}
status: draft
---

# Problem Description

## Type
{Bug Fix | Feature | Refactor | New System}

## Summary
{One paragraph summary of the problem/goal}

## User's Description
{Verbatim or lightly edited user input - preserve their words}

## Symptoms / Current Behavior
{For bugs: what's happening wrong}
{For features: what's missing}

## Desired Behavior / Goals
{What should happen instead}
{What capability should exist}

## User's Theories
{Any hypotheses the user shared about cause or approach}

## External References
- {Linear ticket URL}
- {Screenshot descriptions}
- {Error logs summary}

## Success Criteria
{How we know we're done}

## Open Questions
{Things we need to clarify}
```

## Step 4: Update Cold Start Doc

Update `context_scratch_pad.md`:
- Current phase: 1 - Context Gathering
- Add problem summary
- Note problem type
- Add any open questions

## Step 5: Checkpoint (ALWAYS STOP)

This checkpoint always requires user confirmation. We must verify we understood the problem correctly.

Present the problem description to the user:

> "Here's my understanding of the problem:
>
> **Type**: {type}
> **Summary**: {summary}
> **Success Criteria**: {criteria}
>
> Does this capture it accurately? Should I add or change anything?"

**If user approves**: Update `_plan_state.json` phase to 2, proceed to Scoping
**If user has changes**: Incorporate feedback, re-checkpoint
**If fundamentally wrong**: Loop back, ask more questions

---

# PHASE 2: SCOPING

**Goal**: Identify what's in/out of scope using the knowledge graph.

## Step 1: Consult Knowledge Graph

Based on the problem description, search the knowledge graph:

1. Read `.claude/planning-with-james/knowledge/_overview.md`
2. Read `.claude/planning-with-james/knowledge/_graph.json`
3. Read `.claude/planning-with-james/knowledge/_architecture.md`

Identify:
- Which modules are likely involved (based on keywords, problem type, user mentions)
- What clusters/flows might be affected
- Related modules (from graph edges)

## Step 2: Read Candidate Module Docs

For each potentially-involved module, read:
- `.claude/planning-with-james/knowledge/modules/{module_id}/_index.md`

Build understanding of:
- What each module does
- How they connect
- What's likely in scope vs tangential

## Step 3: Write Scope Document

Create `scope.md`:

```markdown
---
created_at: {timestamp}
status: draft
---

# Scope

## In-Scope Modules

| Module | Why Included | Confidence |
|--------|--------------|------------|
| {module_id} | {rationale from problem + graph} | High/Medium/Low |

## Out-of-Scope

| Module | Why Excluded |
|--------|--------------|
| {module_id} | {rationale} |

## Blast Radius Assessment

{How much of the codebase might this touch?}
{What are the risks of unintended changes?}

## Known Unknowns

Things we need to discover:
- {Question 1}
- {Question 2}

## Discovery Plan

Based on scope, we'll explore these areas in parallel:

| Area | Modules | Focus |
|------|---------|-------|
| {area_name} | module-a, module-b | {what to investigate} |

## Dependencies

External factors that might affect this work:
- {dependency 1}
- {dependency 2}
```

## Step 4: Update Cold Start Doc

Update `context_scratch_pad.md`:
- Current phase: 2 - Scoping
- List in-scope modules
- Note discovery areas planned

## Step 5: Checkpoint (ALWAYS DISCUSS)

Present your scope assessment and invite the user to correct or expand it:

> "Based on the problem and the knowledge graph, here's what I think is in scope:
>
> **In Scope Modules**: {module list with brief rationale for each}
> **Out of Scope**: {what I'm intentionally excluding and why}
> **Known Unknowns**: {questions I plan to investigate in discovery}
> **Discovery Plan**: {areas I'll explore and what I'm looking for}
>
> Does this scope look right? Are there modules I should add or remove? Any constraints or context I should know before I dive into discovery?"

**Wait for the user to respond.** They may:
- Confirm and you proceed
- Add modules you missed ("also check X, it touches this")
- Remove modules ("don't worry about Y, it's unrelated")
- Add context that changes your approach ("X was tried before and failed because...")
- Share performance numbers, business constraints, or other facts not in code

**If user approves**: Update `_plan_state.json`, proceed to Discovery
**If user has changes**: Incorporate, re-present scope, get confirmation
**If scope seems fundamentally wrong**: May need to loop back to Phase 1

---

# PHASE 3: DISCOVERY

**Goal**: Deep exploration of in-scope areas, starting from knowledge graph.

## Step 1: Prepare Discovery Subagents

Based on `scope.md`, identify discovery areas. For each area, spawn a Task agent:
- `subagent_type`: "Explore"
- `model`: "opus"

**Launch in parallel** (single message with multiple Task tool calls).

## Step 2: Discovery Subagent Prompt Template

Each subagent gets this prompt (fill in specifics):

```
You are doing DISCOVERY for the "{plan_name}" plan, focusing on "{area_name}".

## CRITICAL: Knowledge-First Protocol

Start from the knowledge graph, NOT raw code:

1. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_index.md (for each module in your area)
2. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_refs.json
3. Understand what we already know
4. THEN explore code to fill gaps and answer specific questions

## Problem Context
{Paste problem_description.md summary}

## Your Scope
Modules: {list}
Focus: {specific investigation goals}

## Known Unknowns to Investigate
{From scope.md}

## Your Task

1. Start from knowledge docs (read them first!)
2. Identify gaps between knowledge and what we need to know
3. Explore source code to answer:
   - How does this currently work?
   - Where exactly would changes need to happen?
   - What patterns should we follow?
   - What edge cases exist?
   - What could go wrong?

4. Write findings to: {plan_folder}/findings/findings_{area_name}.md

Structure your findings:
```markdown
---
area: {area_name}
modules: {list}
investigated_at: {timestamp}
---

# Findings: {Area Name}

## Summary
{Key takeaways}

## Current State
{How this area currently works}

## Relevant Code Locations
| File | Lines | Purpose |
|------|-------|---------|
| path/to/file.ts | 100-150 | {what this code does} |

## Patterns Observed
{Patterns we should follow}

## Potential Approach
{Initial thoughts on how to make changes}

## Risks / Concerns
{What could go wrong}

## Open Questions
{Things that need more investigation or user input}
```

Be thorough. Note specific file paths and line numbers.
```

## Step 3: Synthesize Findings

After all subagents complete, spawn ONE synthesis agent:

```
You are synthesizing discovery findings for the "{plan_name}" plan.

## Your Task

1. Read all files in {plan_folder}/findings/
2. Read problem_description.md for context
3. Create findings_summary.md

Structure:
```markdown
---
synthesized_at: {timestamp}
areas_covered: {list}
---

# Discovery Summary

## Key Findings

{The most important things we learned}

## Code Change Locations

| Area | Files | Type of Change |
|------|-------|----------------|
| {area} | file1, file2 | {modify/create/delete} |

## Patterns to Follow

{Consolidated patterns from all areas}

## Risks Identified

| Risk | Severity | Mitigation |
|------|----------|------------|
| {risk} | High/Medium/Low | {how to handle} |

## Conflicts / Tensions

{Any conflicts between areas, or surprising dependencies}

## Remaining Questions

{Questions that still need answers}

## Recommendation

{High-level recommendation for approach based on findings}
```
```

## Step 4: Update Cold Start Doc

Update `context_scratch_pad.md`:
- Current phase: 3 - Discovery
- Summary of key findings
- Risks identified
- Remaining questions

## Step 5: Checkpoint (ALWAYS DISCUSS)

Present your discovery findings and validate them with the user:

> "Discovery complete. Here's what I found:
>
> **Key Findings**:
> - {finding 1 with specific details}
> - {finding 2 with specific details}
> - {finding 3 with specific details}
>
> **Performance/Timing** (if relevant): {what I measured or found documented}
>
> **Files to Change**: {list with brief description of each change}
>
> **Patterns I'll Follow**: {existing patterns in the codebase}
>
> **Risks I See**:
> - {risk 1}
> - {risk 2}
>
> **Open Questions**:
> - {question 1}
>
> Does this match your understanding? Did I miss anything important? Are any of these findings wrong or outdated?"

**Wait for the user to respond.** They may:
- Confirm and you proceed
- Correct wrong findings ("that timing is wrong, it's actually X")
- Add context ("we tried that pattern before and it had issues")
- Point to code you missed
- Share knowledge not visible in code (business rules, historical decisions)
- Suggest a better approach than what the findings imply

**If user approves**: Proceed to Phase 4
**If user corrects findings**: Update findings documents, re-present, get confirmation
**If user wants deeper investigation**: Spawn targeted subagents for specific areas
**If findings fundamentally change scope**: May need to revisit Phase 2

---

# PHASE 4: APPROACH

**Goal**: Decide on technical direction before detailed planning.

## Step 1: Generate Options

Based on findings, identify possible approaches:

```
You are deciding on an approach for the "{plan_name}" plan.

Read:
- {plan_folder}/problem_description.md
- {plan_folder}/scope.md
- {plan_folder}/findings_summary.md
- All files in {plan_folder}/findings/

Generate 2-3 viable approaches. For each:
- What would we do?
- Pros and cons
- Risks
- Estimated complexity (S/M/L/XL)
```

## Step 2: Write Approach Document

Create `approach.md`:

```markdown
---
created_at: {timestamp}
status: draft
---

# Approach

## Options Considered

### Option A: {Name}
**Description**: {What we'd do}

**Pros**:
- {pro 1}

**Cons**:
- {con 1}

**Risks**:
- {risk 1}

**Complexity**: {S/M/L/XL}

### Option B: {Name}
...

## Recommended Approach

**Choice**: Option {X}

**Rationale**: {Why this is the best choice}

## Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| {risk} | High/Med/Low | High/Med/Low | {how to handle} |

## Prerequisites

Things that must be true or done before starting:
- {prereq 1}

## Dependencies

External factors we depend on:
- {dependency 1}

## Out of Scope for This Plan

Things we're explicitly NOT doing:
- {exclusion 1}
```

## Step 3: Update Cold Start Doc

Update `context_scratch_pad.md`:
- Current phase: 4 - Approach
- Chosen approach summary
- Key risks

## Step 4: Checkpoint (ALWAYS DISCUSS)

Present your approach analysis and get the user's input:

> "Based on the findings, here are the approaches I considered:
>
> **Option A: {name}**
> - What: {description}
> - Pros: {list}
> - Cons: {list}
> - Complexity: {S/M/L/XL}
>
> **Option B: {name}**
> - What: {description}
> - Pros: {list}
> - Cons: {list}
> - Complexity: {S/M/L/XL}
>
> **My Recommendation**: Option {X}
> **Rationale**: {why this is better than alternatives}
> **Key Risks**: {what could go wrong}
>
> Does this approach make sense? Do you see a better way? Are there constraints or preferences I should factor in?"

**Wait for the user to respond.** They may:
- Confirm and you proceed
- Suggest a different approach entirely ("what about X instead?")
- Add constraints ("we need to support Y which rules out Option A")
- Share past experience ("we tried something similar and it had issues")
- Ask clarifying questions about trade-offs
- Prefer a different option with good reasons

**If user approves**: Proceed to Phase 5
**If user prefers different option**: Update approach.md, confirm, proceed
**If user suggests new approach**: Evaluate it, update approach.md, confirm, proceed
**If new information changes the calculus**: May need to revisit discovery

---

# PHASE 5: DETAILED PLANNING

**Goal**: Full implementation specification.

## Step 1: Generate Detailed Plan

```
You are creating a detailed implementation plan for "{plan_name}".

Read all previous documents:
- problem_description.md
- scope.md
- findings_summary.md
- All findings/*.md
- approach.md

Create detailed_plan.md with complete implementation specification.
```

## Step 2: Write Detailed Plan

Create `detailed_plan.md`:

```markdown
---
created_at: {timestamp}
approach: {chosen approach name}
status: draft
---

# Detailed Plan: {Plan Name}

## Overview

{One paragraph summary of what we're doing and why}

## Architecture Changes

{Any architectural changes required}

## Implementation Sections

### Section 1: {Name}

**Goal**: {what this section accomplishes}

**Files to Modify**:
| File | Changes |
|------|---------|
| path/to/file.ts | {description of changes} |

**New Files to Create**:
| File | Purpose |
|------|---------|
| path/to/new.ts | {what it does} |

**Code Patterns**:
{Patterns to follow, with examples from existing code}

**Implementation Notes**:
```typescript
// Example code or pseudocode
```

**Edge Cases**:
- {edge case 1}: {how to handle}

### Section 2: {Name}
...

## Data Changes

{Any database, schema, or data structure changes}

## API Changes

{Any API contract changes}

## Test Strategy

| Test Type | What to Test | Files |
|-----------|--------------|-------|
| Unit | {what} | {where} |
| Integration | {what} | {where} |

## Migration / Rollout

{How to deploy, any migrations needed, rollback plan}

## Documentation Updates

{Docs that need updating}
```

## Step 3: Update Cold Start Doc

Update `context_scratch_pad.md`:
- Current phase: 5 - Detailed Planning
- Summary of plan sections
- Key implementation notes

## Step 4: Checkpoint (ALWAYS DISCUSS)

Present the detailed plan and invite feedback:

> "Here's the detailed implementation plan:
>
> **Sections**:
> 1. {section name}: {what it accomplishes, key files}
> 2. {section name}: {what it accomplishes, key files}
> ...
>
> **Files to Modify**: {count} ({list the key ones})
> **New Files to Create**: {count} ({list them})
>
> **Key Implementation Details**:
> - {detail 1}
> - {detail 2}
>
> **Edge Cases I'm Handling**:
> - {edge case 1}: {how}
>
> **Test Strategy**: {summary}
>
> **Risks That Emerged**:
> - {any new risks discovered during detailed planning}
>
> Review `detailed_plan.md` for full details. Does this plan look right? Any concerns before I break it into tasks?"

**Wait for the user to respond.** They may:
- Confirm and you proceed
- Spot issues in the plan ("that won't work because...")
- Add requirements ("also need to handle X")
- Question implementation details
- Suggest simplifications

**If user approves**: Proceed to Phase 6
**If user has feedback**: Incorporate into detailed_plan.md, re-present key changes, confirm
**If plan has fundamental issues**: May need to revisit approach

---

# PHASE 6: TASK BREAKDOWN

**Goal**: Convert plan to executable checklist with dependencies.

## Step 1: Generate Task List

```
You are breaking down the detailed plan into executable tasks.

Read:
- detailed_plan.md
- approach.md (for risks/dependencies)

Create a task list that:
- Groups tasks into logical phases
- Identifies dependencies between tasks
- Includes complexity estimates
- Has checkpoints for user review
- Includes research tasks if unknowns remain
```

## Step 2: Write Tasks Document

Create `tasks.md`:

```markdown
---
created_at: {timestamp}
total_tasks: {count}
phases: {count}
status: ready
---

# Task Checklist: {Plan Name}

## Phase 1: {Phase Name}

**Goal**: {what this phase accomplishes}

- [ ] **Task 1.1** (S) - {description}
  - Files: `path/to/file.ts`
  - Notes: {any specific instructions}

- [ ] **Task 1.2** (M) - {description}
  - Depends on: 1.1
  - Files: `path/to/file.ts`

- [ ] **CHECKPOINT 1**: {what to verify}
  - [ ] Verify: {check 1}
  - [ ] Verify: {check 2}
  - [ ] Get user confirmation before proceeding

## Phase 2: {Phase Name}

- [ ] **Task 2.1** (L) - {description}
  - Depends on: Phase 1 complete
  - Research needed: {if any}

...

## Phase N: Testing & Cleanup

- [ ] **Task N.1** (M) - Write tests
- [ ] **Task N.2** (S) - Update documentation
- [ ] **Task N.3** (S) - Code cleanup

## Final Checklist

- [ ] All tests passing
- [ ] Documentation updated
- [ ] Code reviewed
- [ ] Ready for deployment

---

## Complexity Key
- S = Small (< 30 min)
- M = Medium (30 min - 2 hours)
- L = Large (2-4 hours)
- XL = Extra Large (> 4 hours, consider breaking down)

## Dependency Notes

{Any complex dependencies or ordering requirements}
```

## Step 3: Update Cold Start Doc

Update `context_scratch_pad.md`:
- Current phase: 6 - Task Breakdown
- Task summary (phases, counts)
- First tasks to start

## Step 4: Checkpoint (ALWAYS STOP)

Always present the task breakdown to the user. This is the last chance to adjust before implementation.

> "Plan broken into {N} tasks across {M} phases:
>
> **Phase 1**: {name} ({task count} tasks)
> **Phase 2**: {name} ({task count} tasks)
> ...
>
> Ready to finalize?"

**If user approves**: Proceed to Phase 7
**If user has feedback**: Adjust tasks, re-checkpoint

---

# PHASE 7: FINALIZE

**Goal**: Prepare cold start doc and mark planning complete.

## Step 1: Finalize Cold Start Doc

Update `context_scratch_pad.md` to be a complete cold start reference:

```markdown
---
plan_name: {name}
created_at: {timestamp}
current_phase: READY FOR IMPLEMENTATION
last_updated: {timestamp}
---

# Context Scratch Pad: {Plan Name}

## Quick Summary

**Problem**: {one sentence}
**Approach**: {one sentence}
**Scope**: {module list}

## Current State

- **Planning**: COMPLETE
- **Implementation**: NOT STARTED
- **Current Task**: None (ready to begin Phase 1)

## Plan Documents

- [x] problem_description.md - Complete
- [x] scope.md - Complete
- [x] findings/ - Complete ({N} areas investigated)
- [x] findings_summary.md - Complete
- [x] approach.md - Complete
- [x] detailed_plan.md - Complete
- [x] tasks.md - Complete

## Key Decisions

| Decision | Rationale |
|----------|-----------|
| {decision 1} | {why} |

## Risks to Watch

| Risk | Mitigation |
|------|------------|
| {risk 1} | {how to handle} |

## How to Resume

If you're starting fresh or after context loss:

1. Read this document first
2. Read `tasks.md` to see what's next
3. Read `detailed_plan.md` for implementation details
4. Check off tasks as you complete them
5. Update this document with any new findings or decisions

## Session Log

- {timestamp}: Plan initialized
- {timestamp}: Problem description complete
- {timestamp}: Scope defined
- {timestamp}: Discovery complete
- {timestamp}: Approach decided
- {timestamp}: Detailed plan complete
- {timestamp}: Tasks defined
- {timestamp}: PLANNING COMPLETE - Ready for implementation
```

## Step 2: Update Plan State

Update `_plan_state.json`:
- `current_phase`: 7
- All phase statuses: "complete"
- Add final checkpoint timestamp
- Add implementation fields (for go-time):
  ```json
  {
    "implementation_status": "not_started",
    "current_task": null,
    "current_task_description": null,
    "tasks_completed": [],
    "tasks_skipped": [],
    "implementation_started_at": null,
    "last_updated": "{timestamp}"
  }
  ```

Update `.claude/planning-with-james/plans/_registry.json`:
- Plan status: `"planned"` (ready for `/planning-with-james:go-time`)

## Step 3: Final Output

Present completion to user:

> "Planning complete for {plan_name}!
>
> **Documents Created**:
> - problem_description.md
> - scope.md
> - findings_summary.md (+ {N} detailed findings)
> - approach.md
> - detailed_plan.md
> - tasks.md
> - context_scratch_pad.md
>
> **Next Steps**:
> 1. Review tasks.md
> 2. Start with Phase 1, Task 1.1
> 3. Use context_scratch_pad.md if you need to resume after a break
>
> Ready to begin implementation when you are!"

---

## RECOVERY: Resuming a Plan

If this skill is invoked when a plan is already in progress:

1. Check `.claude/planning-with-james/plans/_registry.json` for plans with status `"planning"`
2. Read `_plan_state.json` from that plan folder
3. Read `context_scratch_pad.md` for current state
4. Resume from the current phase

Ask user:
> "Found existing plan in progress: {name}
> Current phase: {phase}
>
> Would you like to:
> 1. Continue this plan
> 2. Start a new plan (the existing one will remain in the registry)"

---

## ADAPTATION: Problem Type Weights

After Phase 1, adjust subsequent phases based on problem type:

| Problem Type | Discovery | Approach | Detailed Plan | Tasks |
|--------------|-----------|----------|---------------|-------|
| Bug | **Heavy** | Light | Light | Light |
| Feature | Medium | **Heavy** | **Heavy** | Medium |
| Refactor | Light | Medium | **Heavy** | **Heavy** |
| New System | **Heavy** | **Heavy** | **Heavy** | Medium |

**Heavy** = More subagents, more detail, more checkpoints
**Light** = Streamlined, fewer questions, faster progression
