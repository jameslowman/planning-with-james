---
name: plan
description: Plan a feature, bug fix, refactor, or new system with structured phases and checkpoints.
disable-model-invocation: true
allowed-tools: Bash, Glob, Grep, Read, Write, Task, Edit, AskUserQuestion
---

# Plan a Feature

**NON-NEGOTIABLE: ALL PATHS ARE RELATIVE TO THE REPO ROOT**
All `.claude/planning-with-james/` paths in this skill are relative to the **current working directory** (the repo you're working in), NOT `~/.claude/`. The knowledge graph and plans live inside the project, not in your home directory. If you're unsure, run `pwd` to confirm you're in the repo root.

**NON-NEGOTIABLE: ASK FIRST, WORK LATER**
Your FIRST action must be to ask the user what they want to do. Do NOT read files, check registries, update knowledge, or do any work before asking. The user invoked /plan because they want to start planning with you.

**NON-NEGOTIABLE: THIS SKILL ONLY READS THE KNOWLEDGE GRAPH, NEVER UPDATES IT**
If the knowledge graph is missing or stale, inform the user and suggest they run `/planning-with-james:create-knowledge` or `/planning-with-james:update-knowledge`. Do NOT run those skills yourself.

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

**When spawning subagents** (Task tool), you MUST include in their prompt:
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
Phase 0: Setup              → Create folder, copy templates, register plan
Phase 1: Context            → Gather problem description and user flows
Phase 2: Scoping            → Identify modules, set boundaries
Phase 3: Discovery          → Parallel deep exploration
Phase 4: Test Architecture  → Map user flows to test scenarios
Phase 5: Approach           → Decide on technical direction
Phase 6: Approach Stress Test → Critique the approach + find what already exists
Phase 7: Detailed Plan      → Full implementation specification
Phase 8: Task Breakdown     → Executable checklist with dependencies
Phase 9: Finalize           → Cold start doc ready for implementation
```

Each phase ends with a **checkpoint** - but checkpoints are **adaptive**, not mandatory stops.

---

## Checkpoint Philosophy

**The point of planning is the conversation.**

Every checkpoint is an opportunity to draw information out of the user -- context they haven't thought to share, corrections to your assumptions, better ideas than what you found. The goal isn't to produce a plan artifact quickly; it's to build shared understanding.

**ALWAYS present your findings and invite correction:**
- Phase 1: "Here's my understanding and the user flows I captured. What did I miss?"
- Phase 2: "Here's what I think is in scope. What else should I consider?"
- Phase 3: "Here's what I discovered. Does this match your understanding?"
- Phase 4: "Here are the test scenarios. Do these capture the behavior you described?"
- Phase 5: "Here's the approach I recommend. Do you see a better way?"
- Phase 6: "Here's what a fresh pair of eyes found — flaws in the approach and existing code we should leverage."
- Phase 7: "Here's the detailed plan. Any concerns?"
- Phase 8: "Here's the task breakdown. Ready to finalize?"

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

## Step 1: Ask the User What They Want

**START HERE. Do not read files or check registries first.**

Use `AskUserQuestion` to ask:

> "What would you like to plan?
>
> Give me a brief description of the feature, bug, refactor, or system you want to work on. You can also paste Linear/GitHub links, error messages, or any context you have."

**Wait for the user's response before proceeding.**

## Step 2: Check for Existing Plans

AFTER the user responds, check `.claude/planning-with-james/plans/_registry.json` for any plans with status `"planning"` (in-progress plans).

**If in-progress plans exist**, ask:

> "I found an existing plan in progress: {name} (Phase {N})
>
> Would you like to:
> 1. Continue that plan
> 2. Start a new plan for what you just described"

**If user wants to continue**: Read `context_scratch_pad.md` from that plan folder and resume from the current phase.

**If user wants a new plan** (or no existing plans): Continue to Step 3.

## Step 3: Verify Knowledge Base

Now verify the knowledge base exists:

```
.claude/planning-with-james/knowledge/
```

If missing, inform user:
> "No knowledge base found. Run `/planning-with-james:create-knowledge` first to index the codebase. Planning without knowledge is like navigating without a map."

**Stop execution and wait for user to create knowledge base.** Do NOT run create-knowledge yourself.

## Step 4: Check for Epic Context (Optional)

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
- Phase 5: Approach is constrained by architecture decisions in learnings.md
- Phase 6: Stress test benefits from learnings about established patterns and past failures
- Phase 7: Detailed plan follows patterns established in learnings.md

**If the plan folder already exists** (created by the epic skill): skip Step 5 (location question) and use the existing folder.

**If not part of an epic:** continue normally.

## Step 5: Ask for Plan Location

Use `AskUserQuestion` to ask:

> "Where should I create the planning folder?
>
> Options:
> 1. `.claude/planning-with-james/plans/{feature-name}/` (recommended - keeps plans with plugin)
> 2. `docs/plans/{feature-name}/` (if you want plans in version control)
> 3. Custom location
>
> What's the feature/bug name? (This becomes the folder name)"

## Step 6: Create Folder Structure

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
├── test_plan.md               # Phase 4 output
├── approach.md                # Phase 5 output
├── reuse_analysis.md          # Phase 6 output (reuse agent)
├── critique_report.md         # Phase 6 output (critique agent)
├── detailed_plan.md           # Phase 7 output
├── tasks.md                   # Phase 8 output
└── lessons.md                 # Lessons captured during planning/implementation
```

## Step 7: Initialize Plan State

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
    "4_test_architecture": "pending",
    "5_approach": "pending",
    "6_stress_test": "pending",
    "7_detailed_plan": "pending",
    "8_tasks": "pending",
    "9_finalize": "pending"
  },
  "problem_type": null,
  "in_scope_modules": [],
  "checkpoints_passed": []
}
```

Also register this plan in `.claude/planning-with-james/plans/_registry.json`:

```json
{
  "sessions": {},
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

If `_registry.json` already exists, merge the new plan into the existing `plans` object. Do not overwrite other plans. Do not modify the `sessions` object -- session bindings are managed exclusively by `/go-time`.

**Plan lifecycle states**: `planning` → `planned` → `active` ⇄ `paused` → `completed`. Only `/go-time` changes status beyond `planned`. See `/planning-with-james:go-time` for details.

## Step 8: Initialize Cold Start Doc

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
- [ ] test_plan.md - Not started
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

## Step 9: Load Planning Context

Read internal planning guidance (these inform your behavior but aren't shown to user):
- How to ask good questions
- What makes a good problem description
- Discovery strategies
- Plan structure best practices

**Load project-level lessons**: Read `.claude/planning-with-james/lessons.md` if it exists. These are accumulated lessons from previous plans -- patterns, mistakes to avoid, and techniques that worked. Use them to inform scoping, discovery, and approach phases.

**Load test preferences**: Read `.claude/planning-with-james/config.json` if it exists. This contains user preferences including `test_preference` (`"mock"`, `"integration"`, or `"mixed"` — default `"mock"`) and `test_first` (`true`/`false` — default `true`). These control how the Test Architecture phase (Phase 4) operates. If the config file doesn't exist, use defaults.

Now proceed to Phase 1.

---

# PHASE 1: CONTEXT GATHERING

**Goal**: Understand what we're solving from the user's perspective.

## Step 1: Gather Additional Context

The user already gave you an initial description in Phase 0 Step 1. Now dig deeper with follow-up questions. Use `AskUserQuestion` to ask questions you don't already have answers to:

**If problem type is unclear:**
> "Is this:
> - A bug fix (something's broken)
> - A feature (new capability)
> - A refactor (improve existing code)
> - A new system (build from scratch)"

**If you need more details:**
> "Can you tell me more about:
> - What's happening (or should happen)
> - Any error messages, screenshots, or logs
> - Links to tickets (Linear, GitHub, etc.)
> - Your theories about the cause or approach"

**If success criteria is unclear:**
> "How will we know this is solved? What does success look like?"

**User flow extraction (ALWAYS ask for bugs, STRONGLY encourage for features):**

This is critical for test-first planning. We need step-by-step user flows to build test scenarios from.

**For bugs:**
> "Walk me through exactly what you do to trigger this. Step by step — what do you click or call, what data are you working with, and what do you see at each step? Where does it break?"
>
> "What should happen instead at the point where it fails?"
>
> "Can you reproduce it consistently, or is it intermittent? What conditions make it appear or disappear?"
>
> "When did this last work correctly? Any recent changes that might be related?"

**For bugs: do not proceed past Phase 1 without at least one concrete user flow.** If the user genuinely cannot provide reproduction steps (intermittent, production-only, etc.), note this as a gap and the Test Architecture phase (Phase 4) will escalate its investigation effort accordingly.

**For features:**
> "Walk me through the user journey you're envisioning. Step by step — what does the user do, and what do they see at each step?"
>
> "Are there alternate paths? What happens if the user provides bad input, or skips a step?"
>
> "What's the simplest version of this flow? What's the full version?"

**For refactors:**
> "What behavior should remain exactly the same after the refactor? Walk me through the key user-facing flows that must not change."

**Only ask questions you need answers to.** If the user's initial description already includes detailed flows, don't re-ask — capture what they gave you.

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

## User Flows

{Step-by-step sequences from the user. Capture verbatim or lightly structured.}

### Flow 1: {short name}
1. {step}
2. {step}
3. {step}
4. **Expected**: {what should happen}
5. **Actual**: {what happens instead — for bugs}

### Flow 2: {short name}
1. ...

### Alternate Paths / Edge Cases
{Variations, error conditions, boundary cases}

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
> **User Flows Captured**: {count} ({brief list — e.g., "1. Rate search returns wrong UNLOCODEs, 2. Bulk export times out"})
> **Success Criteria**: {criteria}
>
> Review `problem_description.md` for the full user flows. Do these capture the behavior accurately? Any flows I should add?"

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

**Lesson capture**: If the user's feedback contained a correction to your assumptions, a reusable codebase insight, or context about what has failed before, add an entry to `{plan_folder}/lessons.md` under the appropriate section. Keep entries to 2-3 sentences with enough context to be useful in isolation. Not every correction is a lesson -- only capture insights that would help a fresh agent on a different plan.

---

# PHASE 3: DISCOVERY

**Goal**: Deep exploration of in-scope areas, starting from knowledge graph.

## Step 1: Prepare Discovery Subagents

Based on `scope.md`, identify discovery areas. For each area, spawn a Task agent:
- `subagent_type`: "general-purpose" (NOT "Explore" — Explore agents cannot write files)
- `model`: "opus"
- `run_in_background`: true
- `max_turns`: 30 (background agents need enough turns to read files AND write results)

**Launch ALL discovery agents in a SINGLE message** with multiple Task tool calls. All agents run in the background — do NOT use TaskOutput to read their results. Agents write their results to disk.

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

## Project-Level Lessons
{If .claude/planning-with-james/lessons.md exists and mentions any of the modules
being investigated, include those specific entries here. These are accumulated
insights from previous plans that may affect how you investigate these modules.}

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

5. After writing findings, write a result file to signal completion:
   {plan_folder}/findings/{area_name}_result.json

```json
{"area": "{area_name}", "modules": ["{module-ids}"], "summary": "One sentence summary of key findings"}
```

This file signals to the orchestrator that you are done. Do NOT skip this step.

Be thorough. Note specific file paths and line numbers.
```

## Step 2b: Wait for Discovery Agents

After launching all agents, wait for them to complete by polling for result files:

```bash
EXPECTED={number of discovery areas}; TIMEOUT=1200; ELAPSED=0
while [ $(find "{plan_folder}/findings" -name "*_result.json" 2>/dev/null | wc -l) -lt $EXPECTED ] && [ $ELAPSED -lt $TIMEOUT ]; do
  sleep 15; ELAPSED=$((ELAPSED + 15))
done
```

This blocks with zero context cost. When it returns, use Glob to find all `findings/*_result.json` files and Read each one to verify all areas completed. If any agents failed (missing result file after timeout), re-launch just those.

Delete the `*_result.json` files after confirming all findings are written.

## Step 3: Synthesize Findings

After all discovery agents have completed (confirmed by result files), spawn ONE synthesis agent (`subagent_type`: "general-purpose", `model`: "opus"):

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

**If user approves**: Proceed to Phase 4 (Test Architecture)
**If user corrects findings**: Update findings documents, re-present, get confirmation
**If user wants deeper investigation**: Spawn targeted subagents for specific areas
**If findings fundamentally change scope**: May need to revisit Phase 2

**Lesson capture**: If the user's feedback contained a correction to your assumptions, a reusable codebase insight, or context about what has failed before, add an entry to `{plan_folder}/lessons.md` under the appropriate section. Keep entries to 2-3 sentences with enough context to be useful in isolation. Not every correction is a lesson -- only capture insights that would help a fresh agent on a different plan.

---

# PHASE 4: TEST ARCHITECTURE

**Goal**: Map user flows to concrete test scenarios in plain English, analyze the existing test landscape, and produce a test plan that the user can review and co-author before any implementation begins.

This phase is the bridge between "what should happen" (user flows from Phase 1) and "how do we prove it" (executable tests). The output — `test_plan.md` — is written in plain English so the user can review every test scenario, suggest additions, and catch gaps before a single line of test code is written.

**Test preference**: Use the `test_preference` from `.claude/planning-with-james/config.json` (default: `"mock"`). This affects mock boundary analysis and the test plan's mock strategy section.

## Step 1: Launch Test Discovery Agents

Based on `scope.md` and `findings_summary.md`, spawn background agents to investigate the test landscape. Launch ALL agents in a SINGLE message.

For each agent:
- `subagent_type`: "general-purpose"
- `model`: "opus"
- `run_in_background`: true
- `max_turns`: 30

### Agent 1: Test Infrastructure

```
You are investigating the TEST INFRASTRUCTURE for the "{plan_name}" plan.

## CRITICAL: Knowledge-First Protocol

Start from the knowledge graph:
1. Read: .claude/planning-with-james/knowledge/_overview.md
2. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_index.md (for each in-scope module)
3. THEN explore test files and configuration

## Your Task

Investigate the testing setup for the in-scope modules:
{list modules from scope.md}

Find and document:
1. **Test framework**: What framework is used? (jest, vitest, pytest, mocha, etc.)
2. **Test location**: Where do tests live relative to source? (__tests__/, tests/, co-located .test.ts?)
3. **Configuration**: Test config files (jest.config, vitest.config, pytest.ini, conftest.py, etc.)
4. **Mocking patterns**: How does the codebase mock dependencies? Libraries used (jest.mock, unittest.mock, sinon, etc.), common patterns, mock factories.
5. **Fixture patterns**: How is test data set up? Factories, fixture files, builders, seed data.
6. **Helper utilities**: Shared test helpers, custom assertions, test utilities.
7. **CI/CD integration**: How are tests run in CI? Any relevant scripts or config.

Write findings to: {plan_folder}/findings/findings_test_infrastructure.md

```markdown
---
area: test-infrastructure
investigated_at: {timestamp}
---

# Findings: Test Infrastructure

## Framework & Configuration
{framework, version, config file locations}

## Test Location Patterns
{where tests live, naming conventions}

## Mocking Patterns
{how mocks are done, libraries, examples from existing code}

## Fixture Patterns
{how test data is created, factories, helpers}

## Test Utilities
{shared helpers, custom assertions}

## CI/CD
{how tests run in CI}
```

After writing findings, write a result file:
{plan_folder}/findings/test_infrastructure_result.json
{"area": "test-infrastructure", "summary": "One sentence summary"}
```

### Agent 2: Existing Coverage Analysis

```
You are analyzing EXISTING TEST COVERAGE for the "{plan_name}" plan.

## CRITICAL: Knowledge-First Protocol

Start from the knowledge graph:
1. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_index.md (for each in-scope module)
2. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_refs.json
3. THEN explore test files

## Problem Context
{Paste problem_description.md summary including user flows}

## In-Scope Modules
{list from scope.md}

## Your Task

For each in-scope module, investigate its test coverage:

1. **Find existing tests**: What test files exist for each module? List them with file paths.
2. **Coverage of affected flows**: Do any existing tests touch the user flows described in the problem? Map each user flow step to existing tests (or note the gap).
3. **Gap analysis**: Why don't existing tests catch this bug / cover this feature? Be specific:
   - Are the flows tested at all?
   - Are they tested but with different data that doesn't trigger the issue?
   - Is the mock boundary wrong (mocking out the layer where the bug lives)?
   - Are there missing edge case tests?
   - Is the test asserting the wrong thing?
4. **Reusable assets**: Which existing test fixtures, helpers, factories, or mock setups can be reused for new tests?

Write findings to: {plan_folder}/findings/findings_test_coverage.md

```markdown
---
area: test-coverage
investigated_at: {timestamp}
---

# Findings: Existing Test Coverage

## Module Coverage Map
| Module | Test Files | Tests Count | Relevant to Problem |
|--------|-----------|-------------|---------------------|
| {module} | {files} | {count} | {yes/no — which flows} |

## User Flow Coverage
### Flow 1: {name from problem_description.md}
- Step 1: {covered by test X / NOT COVERED}
- Step 2: {covered / NOT COVERED}
...

### Flow 2: {name}
...

## Gap Analysis
{Why existing tests miss this — be specific and actionable}

## Reusable Test Assets
| Asset | File | Can Be Used For |
|-------|------|-----------------|
| {fixture/helper/factory} | {path} | {what new tests can reuse it for} |
```

After writing findings, write a result file:
{plan_folder}/findings/test_coverage_result.json
{"area": "test-coverage", "summary": "One sentence summary"}
```

### Agent 3: Mock Boundary Analysis

```
You are analyzing MOCK BOUNDARIES for the "{plan_name}" plan.

## CRITICAL: Knowledge-First Protocol

Start from the knowledge graph:
1. Read: .claude/planning-with-james/knowledge/_graph.json
2. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_index.md (for each in-scope module)
3. Read: .claude/planning-with-james/knowledge/modules/{module_id}/_refs.json
4. THEN trace code paths in source

## Problem Context
{Paste problem_description.md summary including user flows}

## Discovery Findings
{Paste or reference findings_summary.md — especially code change locations}

## Test Preference: {mock | integration | mixed}

## Your Task

For each user flow from the problem description, trace the code path through the modules:

1. **Code path trace**: For each flow step, identify the function/method that executes it. Follow the call chain from entry point through to the data layer.
2. **Mock boundary recommendation**: Based on the test preference, identify where mocks should be placed:
   - For "mock": Mock at the data/external service layer. Test all business logic directly.
   - For "integration": Identify what needs real connections vs what can be stubbed.
   - For "mixed": Recommend which scenarios benefit from mocking vs integration.
3. **Data fabrication**: What test data needs to be fabricated? What shapes? What edge cases in the data?
4. **Function signatures**: For each function that will be tested, note its signature, inputs, and expected outputs. This helps write test scenarios in plain English.

Write findings to: {plan_folder}/findings/findings_mock_boundaries.md

```markdown
---
area: mock-boundaries
investigated_at: {timestamp}
---

# Findings: Mock Boundaries

## Code Path Traces

### Flow 1: {name}
| Step | Function | File:Line | Input | Output |
|------|----------|-----------|-------|--------|
| 1 | {function} | {file:line} | {input type} | {output type} |
| 2 | ... | | | |

**Recommended mock boundary**: {where to mock and why}

### Flow 2: {name}
...

## Mock Strategy
**Boundary layer**: {e.g., "mock database calls, test service + handler layers"}
**Libraries to use**: {based on what the codebase already uses}

## Data Fabrication Needs
| Data Shape | Used By | Example | Edge Cases |
|-----------|---------|---------|------------|
| {type} | {function} | {example} | {null, empty, overflow, etc.} |

## Function Signatures for Test Scenarios
| Function | File | Inputs | Expected Output | Side Effects |
|----------|------|--------|-----------------|-------------- |
| {name} | {file} | {params} | {return value} | {db writes, events, etc.} |
```

After writing findings, write a result file:
{plan_folder}/findings/mock_boundaries_result.json
{"area": "mock-boundaries", "summary": "One sentence summary"}
```

### Additional Agents for Hard-to-Track Bugs

If the problem type is `bug` AND the user couldn't provide clean reproduction flows (noted as a gap in `problem_description.md`), launch additional investigation agents:

```
You are investigating a HARD-TO-TRACK BUG for the "{plan_name}" plan.

## Context
{problem description — symptoms, intermittent behavior, conditions}

## Discovery Findings
{relevant findings from Phase 3}

## Your Task

The user cannot reliably reproduce this bug. Your job is to find it:

1. Read the knowledge graph for all in-scope modules
2. Trace suspected code paths based on symptoms
3. Look for:
   - Race conditions, timing dependencies
   - State mutation that depends on execution order
   - Caching that could serve stale data
   - Error handling that silently swallows failures
   - Configuration differences between environments
   - Data-dependent branches (works for most data, fails for edge cases)
4. For each suspected fault location, write a test scenario that WOULD expose it
5. Propose concrete reproduction steps based on your analysis

Write findings to: {plan_folder}/findings/findings_bug_investigation.md
Write result: {plan_folder}/findings/bug_investigation_result.json
```

## Step 2: Wait for Test Discovery Agents

Same polling pattern as Phase 3:

```bash
EXPECTED={number of test discovery agents}; TIMEOUT=1200; ELAPSED=0
while [ $(find "{plan_folder}/findings" -name "*_result.json" -newer "{plan_folder}/findings_summary.md" 2>/dev/null | wc -l) -lt $EXPECTED ] && [ $ELAPSED -lt $TIMEOUT ]; do
  sleep 15; ELAPSED=$((ELAPSED + 15))
done
```

Read all result files after completion. Re-launch any failed agents. Delete result files after processing.

## Step 3: Synthesize into Test Plan

Spawn ONE synthesis agent (`subagent_type`: "general-purpose", `model`: "opus"):

```
You are creating a TEST PLAN for the "{plan_name}" plan.

## CRITICAL: Plain English Test Descriptions
Every test must be described in plain English that a non-engineer could review.
The user will read this document and help refine it before any test code is written.

## Read These Files
- {plan_folder}/problem_description.md (especially User Flows section)
- {plan_folder}/findings_summary.md
- {plan_folder}/findings/findings_test_infrastructure.md
- {plan_folder}/findings/findings_test_coverage.md
- {plan_folder}/findings/findings_mock_boundaries.md
- {plan_folder}/findings/findings_bug_investigation.md (if exists)

## Test Preference: {mock | integration | mixed}

## Your Task

Create {plan_folder}/test_plan.md with this structure:

```markdown
---
plan_name: "{plan_name}"
created_at: "{timestamp}"
test_preference: "{preference}"
status: draft
---

# Test Plan: {Plan Name}

## Test Preference
{mock / integration / mixed — explain what this means for this plan}

## Existing Test Landscape
- **Framework**: {from test infrastructure findings}
- **Patterns**: {how tests are structured}
- **Relevant existing tests**: {list with file paths}
- **Gap analysis**: {why existing tests don't catch this — be specific}

## Test Scenarios

### Scenario 1: {name — derived from User Flow 1}
**Source**: User Flow 1 from problem_description.md
**Type**: {unit | integration}
**Tests**:

1. **{descriptive test name}** — {plain English: "Verify that when a user does X with data Y, the system returns Z"}
   - **Setup**: {what data and mocks are needed, in plain language}
   - **Action**: {what function is called with what arguments}
   - **Assert**: {what we check — return value, state change, error}
   - **Mock boundary**: {what's mocked and why}

2. **{next test}** — {plain English description}
   ...

{Repeat for each user flow}

### Edge Case Scenarios
{Tests for boundary conditions discovered during analysis}

### Regression Guards
{Tests protecting existing behavior that must not change}

## Mock Strategy
{Consolidated from mock boundary findings}

## Files to Create / Modify
| File | Purpose |
|------|---------|
| {test file path} | {what tests go here} |

## Open Questions
{Things needing user input}
```

Be thorough. Every user flow from problem_description.md must map to at least one test scenario. If a flow is complex, break it into multiple tests — one per meaningful assertion.

Return ONLY: {"scenarios": {count}, "tests": {count}, "files": {count}, "summary": "One sentence"}
```

## Step 4: Update Cold Start Doc

Update `context_scratch_pad.md`:
- Current phase: 4 - Test Architecture
- Number of test scenarios
- Key gaps found in existing coverage
- Mock strategy summary

## Step 5: Checkpoint (ALWAYS STOP)

This checkpoint is critical. The user co-authors the test plan.

Present the test plan summary and ask the user to review the full document:

> "Test plan complete. Here's the summary:
>
> **Test Preference**: {mock/integration/mixed}
> **Scenarios**: {N} (mapped from your user flows)
> **Individual Tests**: {M}
> **Test Files to Create/Modify**: {list}
>
> **Why existing tests miss this**: {one-line gap analysis}
>
> **Mock Strategy**: {one-line summary — e.g., "mock at the database layer, test service and handler logic directly"}
>
> Review `test_plan.md` for every test described in plain English. Do these tests capture the behavior you described? Should any be added, changed, or removed?"

**Wait for the user to respond.** They may:
- Confirm the test plan
- Add test scenarios they want covered ("also test what happens when X is null")
- Correct mock boundaries ("don't mock the cache layer, that's where the bug is")
- Simplify tests ("we don't need to test Y, it's already covered elsewhere")
- Add edge cases from production experience
- Question whether a test is testing the right thing

**If user approves**: Update `_plan_state.json` phase to 5, proceed to Approach
**If user has changes**: Incorporate into test_plan.md, re-present key changes, confirm
**If test plan reveals scope issues**: May need to revisit Phase 2

**Lesson capture**: If the user's feedback contained a correction about test patterns, mock boundaries, or coverage expectations, add an entry to `{plan_folder}/lessons.md`.

---

# PHASE 5: APPROACH

**Goal**: Decide on technical direction before detailed planning.

## Step 1: Generate Options

Based on findings and the test plan, identify possible approaches:

```
You are deciding on an approach for the "{plan_name}" plan.

Read:
- {plan_folder}/problem_description.md
- {plan_folder}/scope.md
- {plan_folder}/findings_summary.md
- All files in {plan_folder}/findings/
- {plan_folder}/test_plan.md

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
- Current phase: 5 - Approach
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

**If user approves**: Proceed to Phase 6
**If user prefers different option**: Update approach.md, confirm, proceed
**If user suggests new approach**: Evaluate it, update approach.md, confirm, proceed
**If new information changes the calculus**: May need to revisit discovery

**Lesson capture**: If the user's feedback contained a correction to your assumptions, a reusable codebase insight, or context about what has failed before, add an entry to `{plan_folder}/lessons.md` under the appropriate section. Keep entries to 2-3 sentences with enough context to be useful in isolation. Not every correction is a lesson -- only capture insights that would help a fresh agent on a different plan.

---

# PHASE 6: APPROACH STRESS TEST

**Goal**: Before committing to a detailed plan, stress-test the chosen approach with two parallel agents — one that critiques the approach for flaws, and one that searches for existing code the approach should leverage. This catches both bad assumptions and unnecessary new code before they get baked into the plan.

**Why this phase exists**: Two biases compound in agentic planning:

1. **Confirmation bias**: The agent that chose the approach has investment in it. It won't naturally look for reasons it's wrong. A fresh agent with no stake in the approach will catch assumptions that don't hold, edge cases that break it, and simpler paths that were overlooked.

2. **Build-new bias**: Agentic AI prefers creating new code over leveraging what exists. Discovery (Phase 3) maps the problem area, but runs *before* an approach is decided. Once the approach is locked, nobody asks "given this specific approach, what already exists that serves it?"

Both agents run in parallel — same inputs, different lenses, one checkpoint.

## Step 1: Launch Both Agents in Parallel

Launch two background Opus agents simultaneously with the full approach context.

### Agent A: Approach Critique

```
You are an approach critique agent for plan "{plan_name}".

Your job is to be a skeptical reviewer of the chosen approach. You have NO stake in
this approach — you didn't choose it. Your goal is to find flaws, bad assumptions,
missed edge cases, and simpler alternatives BEFORE the team commits to a detailed plan.

Read these documents first:
- approach.md (the approach you are critiquing)
- findings_summary.md (what was discovered about the problem area)
- All findings/*.md (detailed module-level findings)
- test_plan.md (the test scenarios the approach must satisfy)
- scope.md (which modules are in play)
- problem_description.md (the original problem and user flows)
If the plan is part of an epic, also read the epic learnings.md.

Then investigate with these specific questions:

### 1. Assumption Verification
For each assumption the approach makes (explicit or implicit), verify it against
the actual codebase:
- Does the code actually work the way the approach assumes?
- Are there runtime conditions, configurations, or environments where
  the assumption breaks? (e.g., different user types, account tiers, deployment modes)
- Does the approach assume something is static that is actually dynamic (or vice versa)?
- Are there race conditions, ordering dependencies, or state transitions the approach
  doesn't account for?

### 2. Edge Case Analysis
Walk through each step of the approach and ask "what if...":
- What if the input is unexpected (null, empty, malformed, enormous)?
- What if an external dependency is unavailable or returns something unexpected?
- What if this runs in a different context than the primary one (different user type,
  different environment, different browser, different OS)?
- What if two instances of this run concurrently?
- What if a previous step partially fails?

### 3. Simpler Alternatives
For each significant piece of work the approach proposes, ask:
- Is there a simpler way to achieve the same result?
- Is the approach solving the root cause, or working around a symptom?
- Could a smaller change (config tweak, flag flip, parameter adjustment) fix this
  without the proposed code changes?
- Is the approach over-engineering for the actual scope of the problem?

### 4. Risk Assessment
Evaluate the approach against practical risks:
- What's the blast radius if the approach has a bug? (affects one user? all users? data loss?)
- What are the rollback options if this goes wrong in production?
- Are there breaking changes that affect other consumers of the modified code?
- Does the approach introduce new dependencies or coupling that will be hard to undo?

For each finding, provide:
- **Issue**: What's wrong or risky
- **Evidence**: Code paths, configurations, or scenarios that demonstrate the issue
- **Severity**: BLOCKING (must fix before proceeding), CONCERN (should address in plan),
  or NOTE (worth knowing, not actionable now)
- **Suggestion**: What to do about it (fix the approach, add a step, add a test, accept the risk)

Write your findings to critique_report.md.
```

### Agent B: Reuse Analysis

```
You are a reuse analysis agent for plan "{plan_name}".

Your job is to examine the existing codebase through the lens of the chosen approach
and identify everything that already exists which can serve the implementation — so we
build only what's truly new.

Read these documents first:
- approach.md (the chosen approach — this is your lens)
- findings_summary.md (what was discovered about the problem area)
- All findings/*.md (detailed module-level findings)
- test_plan.md (the test scenarios we need to satisfy)
- scope.md (which modules are in play)

Then investigate the codebase with these specific questions:

### 1. Existing Functions & Utilities
For each capability the approach requires, search for functions that already do it
(or do 80%+ of it). Check:
- Utility modules, helper files, shared libraries
- Similar features elsewhere in the codebase that solved analogous problems
- Base classes, mixins, or HOCs that provide needed behavior
- Built-in framework features that aren't being leveraged

### 2. Patterns Already Established
Identify patterns in the codebase that the approach should follow rather than reinvent:
- How similar features were built (conventions, file organization, naming)
- Existing abstractions the approach can plug into
- Shared infrastructure (event systems, middleware, pipelines) the approach should use
- Configuration or registry patterns that new code should join, not duplicate

### 3. Code That's Almost Right
Look for code that's close to what's needed but currently:
- Failing for a specific reason (fix > rebuild)
- Missing a small extension to cover the new use case
- Doing the right thing but in the wrong place or with the wrong interface
- Handling a subset of what we need (extend > rewrite)

### 4. Reuse Risks
Flag anything that looks reusable but has hidden problems:
- Functions that look right but have subtle behavior differences
- Shared code that would need changes affecting other consumers
- Abstractions that are too rigid to extend without breaking changes
- Technical debt that would make reuse more expensive than building new

For each finding, provide:
- **What exists**: File path, function/class name, what it does
- **How it serves the approach**: Which part of the approach it maps to
- **Recommendation**: REUSE (use as-is), EXTEND (add to it), ADAPT (modify for our needs), or AVOID (looks reusable but isn't)
- **Effort comparison**: Rough comparison of reuse effort vs building new

Write your findings to reuse_analysis.md.
```

## Step 2: Write Output Documents

### Agent A creates `critique_report.md`:

```markdown
---
created_at: {timestamp}
approach: {chosen approach name}
status: draft
---

# Approach Critique: {Plan Name}

## Verdict

{One sentence: Does the approach hold up, need adjustments, or need rethinking?}

**Blocking Issues**: {count}
**Concerns**: {count}
**Notes**: {count}

## Blocking Issues

{Issues that must be resolved before proceeding to detailed planning.
If none: "No blocking issues found."}

### {Issue Title}

- **Issue**: {what's wrong}
- **Evidence**: {code paths, scenarios, or configurations that demonstrate it}
- **Severity**: BLOCKING
- **Suggestion**: {what to do about it}

## Concerns

{Issues that should be addressed in the detailed plan but don't block planning.}

### {Issue Title}

- **Issue**: {what's wrong}
- **Evidence**: {code paths, scenarios, or configurations}
- **Severity**: CONCERN
- **Suggestion**: {what to do about it}

## Notes

{Observations worth knowing but not actionable right now.}

- {note}

## Simpler Alternatives Considered

| Approach Step | Simpler Alternative | Why It Would/Wouldn't Work |
|---------------|---------------------|---------------------------|
| {step} | {alternative} | {analysis} |

## Risk Summary

| Risk | Blast Radius | Rollback Option | Mitigation |
|------|-------------|-----------------|------------|
| {risk} | {who/what is affected} | {how to undo} | {what the plan should include} |
```

### Agent B creates `reuse_analysis.md`:

```markdown
---
created_at: {timestamp}
approach: {chosen approach name}
status: draft
---

# Reuse Analysis: {Plan Name}

## Summary

{One paragraph: how much of the approach is served by existing code vs. truly new work}

**Reuse Score**: {percentage of approach steps served by existing code — rough estimate}

## Reusable Assets

### REUSE (use as-is)

| Asset | Location | Serves | Notes |
|-------|----------|--------|-------|
| {function/class name} | {file:line} | {which part of approach} | {any caveats} |

### EXTEND (add to existing)

| Asset | Location | Serves | Extension Needed |
|-------|----------|--------|------------------|
| {function/class name} | {file:line} | {which part of approach} | {what to add} |

### ADAPT (modify for our needs)

| Asset | Location | Serves | Adaptation Needed | Impact |
|-------|----------|--------|-------------------|--------|
| {function/class name} | {file:line} | {which part of approach} | {what to change} | {other consumers affected?} |

## Patterns to Follow

| Pattern | Example Location | How to Apply |
|---------|------------------|--------------|
| {pattern name} | {file:line} | {how the approach should use this pattern} |

## Build-New List

{Capabilities the approach requires that genuinely have no existing analog — these are the only things that need to be built from scratch}

| Capability | Why New | Approach Step |
|------------|---------|---------------|
| {what} | {why nothing existing serves this} | {which step} |

## Reuse Risks

| Asset | Risk | Recommendation |
|-------|------|----------------|
| {what looks reusable but isn't} | {why} | AVOID — {what to do instead} |

## Impact on Approach

{Any refinements to the approach based on what was found. For example: "Step 3 of the approach calls for a new validation pipeline, but the existing `ValidationChain` in `src/validation/chain.ts` already handles this pattern — the approach should plug into it rather than creating a parallel system."}
```

## Step 3: Resolve Blocking Issues (if any)

If the critique agent found BLOCKING issues:
1. Present them to the user immediately (before the full checkpoint)
2. Determine if the approach needs revision (return to Phase 5) or if the blocking issue can be addressed as an approach amendment
3. If amending: update approach.md with the fix, note the critique finding that prompted it
4. Re-run only the affected agent if the amendment is substantial

## Step 4: Update Cold Start Doc

Update `context_scratch_pad.md`:
- Current phase: 6 - Approach Stress Test
- Critique verdict and any blocking issues
- Reuse score and key assets identified
- Any approach amendments made

## Step 5: Checkpoint (ALWAYS DISCUSS)

Present the combined findings from both agents:

> "Before writing the detailed plan, two agents stress-tested the approach:
>
> **Critique Verdict**: {verdict}
> {If blocking issues were found and resolved, note what was amended}
>
> **Concerns** ({count}):
> - {concern}: {suggestion}
>
> **Simpler alternatives identified**:
> - {any steps where a simpler path exists}
>
> **Reuse Score**: {X}% of the approach is served by existing code
>
> **Reuse as-is** ({count}):
> - {asset}: {what it does, which approach step it serves}
>
> **Extend** ({count}):
> - {asset}: {what to add}
>
> **Build new** ({count}):
> - {capability}: {why nothing existing covers this}
>
> Review `critique_report.md` and `reuse_analysis.md` for full details.
>
> Do the concerns need to be addressed? Did I miss any existing code we should leverage?
> Are there flaws in the approach that the critique agent didn't catch?"

**Wait for the user to respond.** They may:
- Confirm and you proceed
- Raise additional concerns about the approach
- Point out existing code the reuse agent missed
- Disagree with a reuse recommendation
- Share historical context ("we tried that before and it failed because...")
- Request approach revision based on findings

**If user approves**: Proceed to Phase 7
**If user raises new concerns**: Update critique_report.md, determine if approach needs revision
**If user adds reuse targets**: Update reuse_analysis.md, confirm, proceed
**If user flags reuse risks**: Move items from REUSE/EXTEND to AVOID, confirm, proceed
**If critique findings require approach revision**: Return to Phase 5 to amend the approach, then re-run Phase 6

**Lesson capture**: If the user's feedback contained a correction to your assumptions, a reusable codebase insight, or context about what has failed before, add an entry to `{plan_folder}/lessons.md` under the appropriate section. Keep entries to 2-3 sentences with enough context to be useful in isolation. Not every correction is a lesson -- only capture insights that would help a fresh agent on a different plan.

---

# PHASE 7: DETAILED PLANNING

**Goal**: Full implementation specification.

## Step 1: Generate Detailed Plan

```
You are creating a detailed implementation plan for "{plan_name}".

Read all previous documents:
- problem_description.md
- scope.md
- findings_summary.md
- All findings/*.md
- test_plan.md
- approach.md
- critique_report.md
- reuse_analysis.md

Create detailed_plan.md with complete implementation specification.
The plan MUST:
- Address all CONCERN-level findings from the critique report
- Incorporate the reuse analysis — preferring REUSE and EXTEND recommendations
  over building new code wherever the analysis supports it
- Note any BLOCKING findings that were resolved and how
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

See `test_plan.md` for the complete test plan with plain English test scenarios, mock strategy, and file locations. The test plan was reviewed and approved in Phase 4.

## Migration / Rollout

{How to deploy, any migrations needed, rollback plan}

## Documentation Updates

{Docs that need updating}
```

## Step 3: Update Cold Start Doc

Update `context_scratch_pad.md`:
- Current phase: 7 - Detailed Planning
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

**If user approves**: Proceed to Phase 8
**If user has feedback**: Incorporate into detailed_plan.md, re-present key changes, confirm
**If plan has fundamental issues**: May need to revisit approach

**Lesson capture**: If the user's feedback contained a correction to your assumptions, a reusable codebase insight, or context about what has failed before, add an entry to `{plan_folder}/lessons.md` under the appropriate section. Keep entries to 2-3 sentences with enough context to be useful in isolation. Not every correction is a lesson -- only capture insights that would help a fresh agent on a different plan.

---

# PHASE 8: TASK BREAKDOWN

**Goal**: Convert plan to executable checklist with dependencies.

## Step 1: Generate Task List

```
You are breaking down the detailed plan into executable tasks.

Read:
- detailed_plan.md
- approach.md (for risks/dependencies)
- critique_report.md (for risk mitigations and concerns to address)
- reuse_analysis.md (for reuse/extend/build-new decisions)
- test_plan.md (for test-first task ordering)

Create a task list that:
- **Starts with test writing** — Phase 1 of tasks is ALWAYS writing the tests from test_plan.md. For bugs, these tests must fail (proving the bug exists) before implementation begins. For features, these tests describe expected behavior.
- Groups remaining tasks into logical phases after the test foundation
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

## Phase 1: Test Foundation

**Goal**: Write all tests from test_plan.md BEFORE implementation. For bugs, these tests should FAIL (proving the bug exists). For features, these tests describe expected behavior and may need stubs.

- [ ] **Task 1.1** (S) - Set up test scaffolding, fixtures, and mock utilities needed by test_plan.md
  - Files: {from test_plan.md Mock Strategy and Files sections}
  - Notes: Reuse existing fixtures where test_plan.md identifies them

- [ ] **Task 1.2** (M/L) - Write tests for Scenario 1: {name from test_plan.md}
  - Files: {from test_plan.md}
  - Reference: test_plan.md Scenario 1 for plain English test descriptions

- [ ] **Task 1.3** (M/L) - Write tests for Scenario 2: {name from test_plan.md}
  - Files: {from test_plan.md}

{... one task per scenario from test_plan.md ...}

- [ ] **Task 1.N** (S) - Write edge case and regression guard tests
  - Files: {from test_plan.md}

- [ ] **CHECKPOINT 1**: Test foundation complete
  - [ ] Verify: All test scenarios from test_plan.md are implemented
  - [ ] Verify: For bugs — tests FAIL, confirming the bug is captured
  - [ ] Verify: For features — tests describe expected behavior (may fail or use stubs)
  - [ ] Verify: Tests run fast (mocked appropriately per test_plan.md mock strategy)
  - [ ] Get user confirmation before proceeding to implementation

## Phase 2: {Implementation Phase Name}

**Goal**: {what this phase accomplishes}

- [ ] **Task 2.1** (L) - {description}
  - Depends on: Phase 1 complete
  - Files: {from detailed_plan.md}

...

## Phase N: Verification & Cleanup

- [ ] **Task N.1** (M) - Verify all tests from Phase 1 now PASS
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
- Current phase: 8 - Task Breakdown
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

**If user approves**: Proceed to Phase 9
**If user has feedback**: Adjust tasks, re-checkpoint

**Lesson capture**: If the user's feedback contained a correction to your assumptions, a reusable codebase insight, or context about what has failed before, add an entry to `{plan_folder}/lessons.md` under the appropriate section. Keep entries to 2-3 sentences with enough context to be useful in isolation. Not every correction is a lesson -- only capture insights that would help a fresh agent on a different plan.

---

# PHASE 9: FINALIZE

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
- [x] test_plan.md - Complete ({N} test scenarios)
- [x] approach.md - Complete
- [x] critique_report.md - Complete ({N} blocking, {N} concerns)
- [x] reuse_analysis.md - Complete ({X}% reuse score)
- [x] detailed_plan.md - Complete
- [x] tasks.md - Complete
- [x] lessons.md - Complete (captures from planning checkpoints)

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
- {timestamp}: Problem description complete (with user flows)
- {timestamp}: Scope defined
- {timestamp}: Discovery complete
- {timestamp}: Test plan complete ({N} scenarios)
- {timestamp}: Approach decided
- {timestamp}: Approach stress test complete (critique: {verdict}, reuse: {X}%)
- {timestamp}: Detailed plan complete
- {timestamp}: Tasks defined (test-first)
- {timestamp}: PLANNING COMPLETE - Ready for implementation
```

## Step 2: Update Plan State

Update `_plan_state.json`:
- `current_phase`: 9
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
> - problem_description.md (with user flows)
> - scope.md
> - findings_summary.md (+ {N} detailed findings)
> - test_plan.md ({N} test scenarios, {M} individual tests)
> - approach.md
> - critique_report.md ({N} blocking, {N} concerns)
> - reuse_analysis.md ({X}% reuse score)
> - detailed_plan.md
> - tasks.md (test-first: Phase 1 writes all tests before implementation)
> - context_scratch_pad.md
>
> **Next Steps**:
> 1. Review tasks.md
> 2. Start with Phase 1, Task 1.1
> 3. Use context_scratch_pad.md if you need to resume after a break
>
> Ready to begin implementation when you are!"

---

## ADAPTATION: Problem Type Weights

After Phase 1, adjust subsequent phases based on problem type:

| Problem Type | Discovery | Test Architecture | Approach | Stress Test | Detailed Plan | Tasks |
|--------------|-----------|-------------------|----------|-------------|---------------|-------|
| Bug | **Heavy** | **Heavy** | Light | **Heavy** | Light | Light |
| Feature | Medium | Medium | **Heavy** | **Heavy** | **Heavy** | Medium |
| Refactor | Light | Medium | Medium | **Heavy** | **Heavy** | **Heavy** |
| New System | **Heavy** | Medium | **Heavy** | Medium | **Heavy** | Medium |

**Heavy** = More subagents, more detail, more checkpoints
**Light** = Streamlined, fewer questions, faster progression

For Test Architecture specifically:
- **Bug Heavy**: Launch extra investigation agents for hard-to-track bugs. Deep gap analysis on why existing tests miss the failure. Every user flow must have a failing test scenario.
- **Feature/Refactor Medium**: Map user flows to test scenarios. Analyze existing coverage. Regression guards are important for refactors.
- **New System Medium**: Less existing test infrastructure to analyze, but user flow scenarios guide the test foundation.
