---
plan_name: "{{PLAN_NAME}}"
created_at: "{{TIMESTAMP}}"
test_preference: "{{TEST_PREFERENCE}}"
status: draft
---

# Test Plan: {{PLAN_NAME}}

## Test Preference

{{mock | integration | mixed}}

## Existing Test Landscape

- **Framework**: (jest / vitest / pytest / etc.)
- **Test location pattern**: (e.g., `__tests__/`, `tests/`, co-located `.test.ts`)
- **Mocking patterns**: (how the codebase currently mocks — libraries, conventions)
- **Fixture patterns**: (how test data is set up — factories, fixtures, builders)
- **Relevant existing tests**: (tests that touch the affected area, with file paths)
- **Gap analysis**: (why existing tests don't catch this bug / don't cover this feature)

## Test Scenarios

### Scenario 1: {{Name — derived from user flow}}

**Source**: User Flow 1 from problem_description.md
**Type**: unit | integration

**Tests**:

1. **{{test name}}** — {{plain English description of what this test proves}}
   - **Setup**: {{what data/state/mocks are needed}}
   - **Action**: {{what function/method/endpoint is called, with what arguments}}
   - **Assert**: {{what the expected outcome is — return value, state change, error thrown}}
   - **Mock boundary**: {{what gets mocked and why — e.g., "mock the database layer, test the service logic"}}

2. **{{test name}}** — {{plain English description}}
   - **Setup**: {{...}}
   - **Action**: {{...}}
   - **Assert**: {{...}}
   - **Mock boundary**: {{...}}

### Scenario 2: {{Name}}

**Source**: User Flow 2 from problem_description.md
**Type**: unit | integration

**Tests**:

1. ...

### Edge Case Scenarios

(Tests for boundary conditions, null/empty inputs, concurrent access, error paths discovered during analysis)

1. **{{test name}}** — {{what edge case this covers}}
   - **Setup**: {{...}}
   - **Action**: {{...}}
   - **Assert**: {{...}}

### Regression Guards

(Tests that ensure adjacent behavior — things that currently work — doesn't break)

1. **{{test name}}** — {{what existing behavior this protects}}
   - **Setup**: {{...}}
   - **Action**: {{...}}
   - **Assert**: {{...}}

## Mock Strategy

**Mock boundary**: (which layer is the boundary — e.g., "mock at the database/API client layer, test everything above")
**Data shapes**: (what fabricated data is needed — example payloads, factory patterns)
**Existing fixtures to reuse**: (fixtures/helpers/factories that already exist in the test suite)
**New fixtures needed**: (what needs to be created)

## Files to Create / Modify

| File | Purpose |
|------|---------|
| tests/path/to/test_file | {{what tests go here}} |

## Open Questions

(Things that need user input before tests can be fully specified)

- [ ]
- [ ]
