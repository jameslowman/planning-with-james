---
name: go-time
description: Execute a plan's tasks with full continuity through context loss. Activate, pause, and resume plans.
disable-model-invocation: true
allowed-tools: Bash, Glob, Grep, Read, Write, Task, Edit, AskUserQuestion
---

# Go Time: Execute the Plan

This skill implements a plan's tasks one at a time with rigorous state tracking. Every action is recorded to files so work survives autocompact, session breaks, and context loss.

**One rule above all: read the files, trust the files, update the files.**

**NON-NEGOTIABLE #1 (RECORD)**: After completing each task, you MUST update `context_scratch_pad.md`, check off the task in `tasks.md`, and update `_plan_state.json` BEFORE starting the next task. This is the RECORD step (Step 4). A task is not done until the record is written. Skipping this step destroys the ability to recover from context loss.

**NON-NEGOTIABLE #2 (KNOWLEDGE-FIRST)**: When you need to understand unfamiliar code during implementation - whether for a task, a deviation, debugging, or research - read the knowledge graph BEFORE exploring source files. The knowledge graph at `.claude/planning-with-james/knowledge/` already contains module boundaries, key files, interfaces, patterns, and gotchas. Starting from source code wastes time rediscovering documented information. Read `_overview.md`, then `_graph.json`, then relevant module `_index.md` files. THEN fill gaps from source. When spawning subagents, include the relevant knowledge file paths in their prompt.

---

## Arguments

`$ARGUMENTS` controls behavior:

| Argument | Behavior |
|----------|----------|
| *(empty)* | Resume active plan, or show available plans to activate |
| `{plan-id}` | Activate and resume a specific plan |
| `pause` | Pause the active plan |
| `status` | Show all plans and their states |

---

## Plan Lifecycle States

Plans move through these states:

```
planning → planned → active ⇄ paused → completed
                       ↑
                       └────────────────────
```

| State | Meaning | Who sets it |
|-------|---------|-------------|
| `planning` | Still in planning phases | `/plan` skill |
| `planned` | Planning complete, ready for implementation | `/plan` skill (Phase 7) |
| `active` | Currently being implemented | `/go-time` on activation |
| `paused` | Implementation paused, safe to do other work | `/go-time pause` or user request |
| `completed` | All tasks done | `/go-time` when final checklist passes |

**Only ONE plan can be `active` at a time.** This is enforced by the registry. When no plan is active, the PreToolUse hook is silent and nothing interferes with other work.

---

## Registry

All plans are tracked in `.claude/planning-with-james/plans/_registry.json`:

```json
{
  "active_plan": null,
  "plans": {
    "SB-1112": {
      "name": "SB-1112 Return UNLOCODEs in RFQ Rate Search",
      "path": "/full/path/to/plan/folder",
      "created_at": "2026-02-04T12:00:00Z",
      "status": "paused",
      "current_task": "2.1"
    }
  }
}
```

---

# STEP 1: ROUTE BY ARGUMENT

## If `$ARGUMENTS` is "pause"

1. Read `.claude/planning-with-james/plans/_registry.json`
2. Find the active plan (where `active_plan` is not null)
3. If no active plan: tell user "No plan is currently active. Nothing to pause."
4. If active plan found:
   a. Read the plan's `context_scratch_pad.md`
   b. Update scratch pad with pause state:
      ```
      ## Current State
      - **Status**: PAUSED
      - **Last completed**: {last checked-off task}
      - **Next task**: {next unchecked task}
      - **Paused at**: {timestamp}
      ```
   c. Update `_plan_state.json`: set `implementation_status` to `"paused"`
   d. Update `_registry.json`: set plan status to `"paused"`, set `active_plan` to `null`
   e. Tell user: "Plan {name} paused at Task {X.Y}. Resume anytime with `/planning-with-james:go-time {plan-id}`"
5. STOP.

## If `$ARGUMENTS` is "status"

1. Read `.claude/planning-with-james/plans/_registry.json`
2. Display all plans in a table:

```
| Plan | Status | Current Task | Created |
|------|--------|--------------|---------|
| SB-1112 | paused | Task 2.1 | 2026-02-04 |
| auth-refactor | planned | - | 2026-02-03 |
| SB-1300 | completed | - | 2026-01-28 |
```

3. If there's an active plan, note it.
4. STOP.

## If `$ARGUMENTS` is a plan-id

1. Read `.claude/planning-with-james/plans/_registry.json`
2. Find the specified plan. If not found: "Plan '{plan-id}' not found in registry. Run `/planning-with-james:go-time status` to see available plans."
3. Check plan status:
   - If `planning`: "Plan '{plan-id}' is still in planning phases. Complete planning first with `/planning-with-james:plan`."
   - If `completed`: "Plan '{plan-id}' is already completed."
   - If `active`: It's already active. Proceed to ORIENT.
   - If `planned` or `paused`: Activate it (next step).
4. Check if another plan is already active:
   - If yes: "Plan '{other}' is currently active. Pause it first with `/planning-with-james:go-time pause`, then activate this one."
   - If no: Set this plan to `active` and `active_plan` to its id in the registry.
5. **If this is the first activation** (status was `planned`, not `paused`):
   - Create a branch based on the plan's `problem_type` from `_plan_state.json`:
     - `bug` → `fix/{plan-id}`
     - `feature` → `feat/{plan-id}`
     - `refactor` → `refactor/{plan-id}`
     - `new_system` → `feat/{plan-id}`
     - Other/unknown → ask user for branch name
   - If the user is already on a non-main branch, ask before creating a new one
   - Record the branch name in `_plan_state.json` as `branch`
6. **If resuming** (status was `paused`):
   - Check if we're on the correct branch (from `_plan_state.json`)
   - If not, ask: "Plan {name} was on branch `{branch}`. Switch to it?"
7. Proceed to ORIENT.

## If `$ARGUMENTS` is empty

1. Read `.claude/planning-with-james/plans/_registry.json`
2. If there's an active plan: resume it. Proceed to ORIENT.
3. If no active plan but there are `planned` or `paused` plans:
   - If only one eligible plan: ask "Resume {plan-name}?" with AskUserQuestion
   - If multiple: show them and ask which to activate
4. If no plans exist: "No plans found. Create one with `/planning-with-james:plan`."
5. Once plan is selected, set it to `active` in registry. Proceed to ORIENT.

---

# STEP 2: ORIENT

**This happens before EVERY task.** Not just at the start. Every single time. This is the re-orientation that prevents drift after autocompact.

## Read these files in this order:

### 0. Epic context (if this plan is part of an epic)
Check `_plan_state.json` for an `epic` field. If present:

Read the epic's `learnings.md` - this contains architecture decisions, established patterns, and accumulated context from previous sub-plans. **This is critical.** Decisions locked in by earlier sub-plans constrain what you can do now. Patterns established must be followed for consistency.

Also read `epic_context.md` if this is your first time orienting on this plan (no memory of previous tasks). This gives you the full business story.

### 1. Plan state
Read `{plan_folder}/_plan_state.json`

Know: current task, completed tasks, implementation status.

### 2. Scratch pad
Read `{plan_folder}/context_scratch_pad.md`

Know: what's been done, gotchas encountered, decisions made, current state narrative.

### 3. Task list
Read `{plan_folder}/tasks.md`

Know: which tasks are checked off, what's next, checkpoint requirements.

### 4. Relevant plan section
Read `{plan_folder}/detailed_plan.md` - specifically the section that covers the current task.

Know: exact files to modify, code patterns to follow, edge cases to handle.

## If resuming after a break:

If tasks are already checked off but you don't have memory of doing them:

1. Look at the last checked-off task
2. Verify its work actually exists:
   - Were the files created/modified?
   - Do the changes look correct?
3. If verification passes: proceed to next unchecked task
4. If verification fails: note in scratch pad, re-do that task

## Determine the current task:

Find the first unchecked `- [ ]` task in `tasks.md` that:
- Is not blocked by an unchecked dependency
- Is not a CHECKPOINT line (those are handled separately)

If the next item is a CHECKPOINT: handle it (see CHECKPOINT section below).

---

# STEP 3: EXECUTE

**One task at a time. No skipping ahead. No batching unless the plan explicitly groups them.**

## For the current task:

### 1. Announce what you're doing
Tell the user briefly: "Starting Task {X.Y}: {description}"

### 2. Read the source files
Read every file mentioned in the task's `Files:` field. Understand the current state before changing anything.

### 3. Knowledge-first for unfamiliar code
If the task involves code you haven't seen in this session, DO NOT jump straight to Glob/Grep on source files. Instead:
1. Read `.claude/planning-with-james/knowledge/modules/{relevant}/_index.md` for the module this task touches
2. The knowledge doc tells you key files, patterns, interfaces, and gotchas
3. Then read the specific source files the knowledge doc points you to
4. Only use Glob/Grep on source if the knowledge graph doesn't cover what you need

### 4. Implement the changes
Follow the detailed plan's code patterns and implementation notes. Make the changes specified.

**Implementation principles:**
- Follow patterns from `detailed_plan.md` exactly
- Match existing code style in the codebase
- Do not over-engineer beyond what the task specifies
- Do not refactor adjacent code unless the task says to
- Do not add comments, docstrings, or type annotations to code you didn't change

### 5. Lint check
After making changes, run the appropriate linter:
- Python: the project's lint command (check for `just lint-python`, `ruff check`, `flake8`, etc.)
- TypeScript/JS: the project's lint command (check for `just lint-js`, `eslint`, `tsc --noEmit`, etc.)
- Other: whatever lint tooling the project uses

If lint fails: fix it before moving on. Note the fix in scratch pad.

### 6. Quick verification
Does the change make sense? Read the modified file to verify. A 5-second sanity check now prevents a 30-minute debugging session later.

---

# STEP 4: RECORD

## THIS IS NOT OPTIONAL. A task is NOT complete until all four record steps are done.

The scratch pad is the single most important file in this system. It is how you recover after autocompact. It is how you know what happened. If you skip this step and context is lost, all implementation progress becomes unverifiable. **Updating the scratch pad IS part of the task, not an afterthought.**

**After every single task, do all four of these before touching anything else:**

### Record 1 of 4: Update `context_scratch_pad.md`

This comes FIRST because it's the most important.

Read the current scratch pad, then update it:

**Add to the session log section:**
```
- [Task X.Y] {what was done}. {any surprises, gotchas, or deviations from plan}
```

**Replace the "Current State" section:**
```
## Current State
- **Status**: ACTIVE
- **Last completed**: Task X.Y - {brief description}
- **Next task**: Task X.Z - {brief description}
- **Implementation phase**: {phase name from tasks.md}
```

### Record 2 of 4: Check off the task in `tasks.md`

Change `- [ ] **Task X.Y**` to `- [x] **Task X.Y**`

### Record 3 of 4: Update `_plan_state.json`

Read the current file, then update:
```json
{
  "current_task": "X.Z",
  "current_task_description": "{next task description}",
  "tasks_completed": ["1.1", "1.2", ..., "X.Y"],
  "last_updated": "{timestamp}"
}
```

### Record 4 of 4: Update `_registry.json`

Update the plan's `current_task` field.

**Only after all four record steps are complete may you proceed to EVALUATE.**

---

# STEP 5: EVALUATE

### Check for surprises:

After each task, briefly assess:

| Question | If Yes |
|----------|--------|
| Did the task work exactly as planned? | Continue to next task |
| Minor surprise but no plan change needed? | Note in scratch pad, continue |
| Discovered a missing step? | Add task to tasks.md, note in scratch pad, continue |
| Task turned out to be unnecessary? | Mark as skipped `- [~]` with note, continue |
| Approach seems wrong? | **STOP. Tell user. Do not continue.** |
| Tests reveal a design flaw? | **STOP. Tell user. Do not continue.** |
| Something contradicts the plan? | **STOP. Tell user. Do not continue.** |

### Then:

- If the next item in tasks.md is a regular task: go back to ORIENT (Step 2)
- If the next item is a CHECKPOINT: handle it (next section)
- If all tasks are done: go to COMPLETE (below)

---

# CHECKPOINTS

When the next item in `tasks.md` is a `CHECKPOINT`:

### 1. Quality gate (run these in order)

**a. Lint check (full project)**
Run the project's lint command for all changed languages. Not just the files you touched - the full project lint. Catch anything your changes broke.

**b. Unit tests (full suite)**
Run the project's full unit test suite. Unit tests should finish in seconds. This catches regressions you didn't expect from your changes. Run ALL of them, not just tests related to your work.

**c. Type check (if applicable)**
For TypeScript projects: `tsc --noEmit` or equivalent. Catches wrong argument types, missing imports, interface mismatches that lint doesn't catch.

**d. Debug residue check**
Before committing, review your changes for:
- `console.log` / `print()` debugging statements
- Commented-out code
- TODO comments you added
- Any temporary workarounds

Remove all of these.

**e. Diff review**
Run `git diff` and review the changes against what the plan specified:
- Did we touch files we shouldn't have?
- Did we miss any files the plan mentioned?
- Do the changes match the plan's intent?

**f. Read-back verification**
Re-read each changed file. Does the code make sense in context? A 30-second re-read catches mistakes that are obvious in hindsight.

### 2. Run domain-specific verification checks
Execute each `- [ ] Verify:` item listed under the checkpoint in tasks.md. These are the plan-specific checks.

### 3. Handle failures
If any quality gate or verification fails:
- Fix the issue
- Re-run the failed check (and any checks that might be affected by the fix)
- Do not proceed past a failed checkpoint

### 4. Git commit
Once all checks pass, commit the phase's work:

```
git add {specific files changed in this phase}
git commit -m "{type}({plan-id}): Phase {N} - {phase name}

{brief summary of what this phase accomplished}

Plan: {plan folder path}"
```

Add specific files by name - do not use `git add .` or `git add -A`.

### 5. Report and ask the user
```
Checkpoint {N} passed: {checkpoint description}

Quality gate:
  [PASS] Lint (full project)
  [PASS] Unit tests ({N} passed, 0 failed)
  [PASS] Type check
  [PASS] No debug residue
  [PASS] Diff matches plan

Verification:
  [PASS] {domain check 1}
  [PASS] {domain check 2}

Committed: {commit hash} on branch {branch name}
```

> Continue to {next phase name}, or pause here?

If user says continue: check off checkpoint items, proceed to next task.
If user says pause: run PAUSE logic.

---

# PAUSE

When the user requests a pause (says "pause", "stop for now", "that's enough", "let's pick this up later", etc.):

1. If currently mid-task, finish and record it first if possible. If not possible, note the incomplete state.

2. Update `context_scratch_pad.md`:
```markdown
## Current State
- **Status**: PAUSED
- **Last completed**: Task X.Y - {description}
- **Next task**: Task X.Z - {description}
- **Paused at**: {timestamp}
- **Notes**: {any context about where things stand, what to watch for on resume}

## How to Resume
Run: /planning-with-james:go-time {plan-id}
The Orient phase will re-read all state files and continue from Task X.Z.
```

3. Update `_plan_state.json`:
   - `implementation_status`: `"paused"`
   - `current_task`: the NEXT task (not the one just completed)

4. Update `_registry.json`:
   - Plan status: `"paused"`
   - `active_plan`: `null`

5. Tell user:
> "Plan {name} paused at Task {X.Y}. Resume anytime with `/planning-with-james:go-time {plan-id}`."
>
> No hooks will fire while the plan is paused. You're free to work on other things.

---

# COMPLETE

When the final checklist in `tasks.md` is reached and all items pass:

1. Run every item in the "Final Checklist" section of tasks.md.

2. Report results:
```
Implementation complete for {plan-name}.

Final checklist:
  [PASS] All tests passing
  [PASS] Lint clean
  [PASS] {other checks}

Tasks completed: {N}/{total}
Tasks skipped: {M} (with reasons noted in scratch pad)
```

3. Update `context_scratch_pad.md`:
```markdown
## Current State
- **Status**: COMPLETED
- **Completed at**: {timestamp}
- **All tasks**: Done

## Summary
{Brief summary of everything that was implemented}
```

4. Update `_plan_state.json`:
   - `implementation_status`: `"completed"`

5. Update `_registry.json`:
   - Plan status: `"completed"`
   - `active_plan`: `null`

6. Ask user if they want to create a commit or PR.

---

# PLAN DEVIATIONS

Reality doesn't always match the plan. Handle deviations carefully:

## Adding a task

If you discover work that wasn't in the plan:

1. Add the task to `tasks.md` in the appropriate phase, with the next available number
2. Mark dependencies if any
3. Note in scratch pad: "Added Task X.Y: {reason}"
4. Update `total_tasks` in tasks.md frontmatter
5. Continue execution - no need to stop unless the addition is significant

## Skipping a task

If a task turns out to be unnecessary:

1. Change `- [ ]` to `- [~]` in tasks.md
2. Add note: `(Skipped: {reason})`
3. Note in scratch pad
4. Continue to next task

## Changing the approach

If something fundamental is wrong with the plan:

1. **STOP immediately.** Do not try to work around it.
2. Explain to the user:
   - What you discovered
   - Why the current plan won't work
   - What you think should change
3. Wait for user guidance.
4. If user agrees to change:
   a. Update `approach.md` with the new direction
   b. Update `detailed_plan.md` with revised implementation
   c. Update `tasks.md` with revised task list
   d. Update `context_scratch_pad.md` documenting the change and why
   e. Resume execution from the current point
5. If user disagrees: continue with original plan, noting the concern in scratch pad.

---

# AUTOCOMPACT RECOVERY

If you suspect context was compacted (you don't remember recent work, or things feel unfamiliar):

**Do not guess. Do not assume. Read the files.**

1. Read `_registry.json` → find the active plan and its path
2. Read `context_scratch_pad.md` → this is your lifeline. It tells you everything.
3. Read `tasks.md` → see what's checked off vs pending
4. Read `_plan_state.json` → current task, completed list
5. Verify the last completed task's work exists on disk
6. Resume from the current task via the normal ORIENT flow

The scratch pad is specifically designed for this scenario. It contains:
- What was done and in what order
- Gotchas and surprises encountered
- Decisions made during implementation
- The current state and next step

**Trust the scratch pad. It was written by you, moments before the compaction.**

---

# TESTING STRATEGY

## Continuous testing

| When | What | Why |
|------|------|-----|
| After each file edit | Lint check | Catch syntax/style issues immediately |
| After each task | Lint changed files | Verify task didn't break anything |
| At each checkpoint | Run tests listed in checkpoint | Gate progress on correctness |
| After final task | Full relevant test suite | Catch integration issues |

## Writing new tests

When the plan calls for writing tests:

1. **Read existing test patterns first.** Find tests in the same directory or module. Note:
   - Test framework used (pytest, jest, vitest, etc.)
   - Fixture patterns
   - Assertion style
   - Naming conventions
   - Marker/tag conventions
2. Follow those patterns exactly.
3. Run the new tests to verify they pass.
4. Run adjacent existing tests to verify no regressions.

## Test failure protocol

1. **Do NOT move to the next task.** A failing test behind you compounds into a mess ahead.
2. Diagnose the failure. Read the error carefully.
3. Fix the **code**, not the test (unless the test itself is wrong based on updated requirements).
4. Re-run and verify.
5. Note in scratch pad: "Task X.Y: test initially failed because {reason}. Fixed by {change}."
6. If the fix changes the approach or affects other tasks: STOP and evaluate.

---

# INTERACTION PATTERNS

## When to talk to the user

| Situation | Action |
|-----------|--------|
| Starting a new task | Brief announcement: "Starting Task X.Y: {description}" |
| Task completed normally | No announcement needed (they can see the file changes) |
| Checkpoint reached | Full report with verification results |
| Something unexpected | Explain and ask for guidance |
| Approach needs changing | Stop and discuss |
| All tasks done | Completion report |

## When NOT to talk to the user

- Don't narrate every file read or lint run
- Don't ask permission for things the plan already specifies
- Don't seek approval for individual code changes (that's what the plan was for)
- Don't report task completion unless at a checkpoint

The plan was approved. Execute it. Report at checkpoints. Stop if something is wrong.
