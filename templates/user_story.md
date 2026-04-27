---
plan_name: "{{PLAN_NAME}}"
created_at: "{{TIMESTAMP}}"
status: pre_drafted
post_filled_in: false
scene_count: 0
linear_ticket_url: null
---

# User Story: {{PLAN_NAME}}

A scene-by-scene narrative of how the protagonist experiences this work — both today
(pre-implementation) and after the change ships (post-implementation). Authored
scene-by-scene with the user, one confirmation at a time. The narrative is the
source of truth for mental-model alignment; the structured User Flows in
`problem_description.md` are mechanically derived from it.

---

## The Cast

- **{{PROTAGONIST_NAME}}, {{PROTAGONIST_ROLE}}**. (One sentence about how they interact with this system.)
- **{{SUPPORTING_CHARACTER}}** — (one sentence)

---

## PRE-IMPLEMENTATION (today's behavior)

### Scene 1: (title)

1. (Step in narrative form — verbs, side effects, what the protagonist sees and feels)
2. (Step)
...
N. **Expected**: (what should happen at the failure point — for bugs)
N+1. **Actual**: (what happens instead — for bugs)

### Scene 2: (title)

(Same shape. Up to 4 scenes total. Quality over quantity.)

---

## POST-IMPLEMENTATION (after this PR)

(Filled in during Phase 9 Finalize. Mirrors the pre scenes 1:1 — same titles, same
protagonist, same starting situation, but showing the new behavior.)

### Scene 1 (revisited): (same title as pre Scene 1)

(Narrative form. May include brief inline asides about the technical mechanism
where it helps the reader: "Behind the scenes: {one sentence}".)

### Scene 2 (revisited): (same title as pre Scene 2)

...

---

## What the fix IS / IS NOT

(Filled in during Phase 9 Finalize. Sourced from `detailed_plan.md`,
`critique_report.md`, `reuse_analysis.md`, and the Out-of-Scope section of
`approach.md`. Every bullet should reference a file, a behavior, or a
deferred problem with the reason for deferral.)

### It IS

- (Scope — purely client-side / touches schema / new endpoints / etc.)
- (Core mechanism — the load-bearing change in plain language)
- (Approximate code volume — lines per file, files touched)
- (New files vs. modifications)

### It is NOT

- (Concerns from `critique_report.md` deliberately deferred — note why)
- (AVOID entries from `reuse_analysis.md` — things that look similar but won't be touched)
- (Items from `approach.md` Out-of-Scope section)
- (Known limitations the planner and user discussed but chose not to fix)
