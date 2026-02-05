---
created_at: "{{TIMESTAMP}}"
total_tasks: 0
total_phases: 0
status: ready
---

# Task Checklist: {{PLAN_NAME}}

## Progress Summary

| Phase | Status | Tasks | Done |
|-------|--------|-------|------|
| Phase 1 | Not Started | 0 | 0 |
| Total | | 0 | 0 |

---

## Phase 1: (Name)

**Goal**: (What this phase accomplishes)
**Prerequisites**: None

### Tasks

- [ ] **Task 1.1** (S) - (description)
  - Files: `path/to/file.ts`
  - Notes: (any specific instructions)

- [ ] **Task 1.2** (M) - (description)
  - Depends on: 1.1
  - Files: `path/to/file.ts`
  - Notes:

### Phase 1 Checkpoint

Quality gate:
- [ ] Lint passes (full project)
- [ ] Unit tests pass (full suite)
- [ ] Type check passes (if applicable)
- [ ] No debug residue (console.log, print, commented code)
- [ ] Diff review: changes match plan
- [ ] Git commit: `{{COMMIT_TYPE}}({{PLAN_ID}}): Phase 1 - (Name)`

Verification:
- [ ] (domain-specific check 1)
- [ ] (domain-specific check 2)

Decision: continue or pause?

---

## Phase 2: (Name)

**Goal**:
**Prerequisites**: Phase 1 complete

### Tasks

- [ ] **Task 2.1** (L) - (description)
  - Depends on: Phase 1 complete
  - Files:
  - Notes:

### Phase 2 Checkpoint

Quality gate:
- [ ] Lint passes (full project)
- [ ] Unit tests pass (full suite)
- [ ] Type check passes (if applicable)
- [ ] No debug residue
- [ ] Diff review: changes match plan
- [ ] Git commit: `{{COMMIT_TYPE}}({{PLAN_ID}}): Phase 2 - (Name)`

Verification:
- [ ] (domain-specific check)

Decision: continue or pause?

---

## Phase N: Testing & Cleanup

**Goal**: Add tests and verify no regressions
**Prerequisites**: All implementation complete

### Tasks

- [ ] **Task N.1** (M) - Write/update tests
  - Unit tests for:
  - Integration tests for:

- [ ] **Task N.2** (S) - Code cleanup
  - Remove any remaining debug code
  - Clean up comments
  - Verify formatting

### Final Checkpoint

Quality gate:
- [ ] Lint passes (full project)
- [ ] Unit tests pass (full suite, including new tests)
- [ ] Type check passes (if applicable)
- [ ] No debug residue
- [ ] Git commit: `{{COMMIT_TYPE}}({{PLAN_ID}}): Phase N - Testing & cleanup`

Verification:
- [ ] All new tests passing
- [ ] No regressions in existing tests
- [ ] Ready for review

---

## Final Checklist

Before marking complete:

- [ ] All tasks checked off
- [ ] All tests passing
- [ ] All lint clean
- [ ] All commits on feature branch
- [ ] Ready for PR

---

## Complexity Key

- **S** = Small
- **M** = Medium
- **L** = Large
- **XL** = Extra Large (consider breaking down)

## Dependency Notes

(Any complex dependencies or ordering requirements not captured above)
