---
created_at: "{{TIMESTAMP}}"
approach: null
status: draft
---

# Detailed Plan: {{PLAN_NAME}}

## Overview

(One paragraph summary of what we're doing and why)

## Architecture Changes

(Any architectural changes required - diagrams welcome)

```
// ASCII diagram or description of architecture changes
```

---

## Implementation Sections

### Section 1: (Name)

**Goal**: (What this section accomplishes)

**Prerequisites**: (What must be done first)

#### Files to Modify

| File | Changes | Complexity |
|------|---------|------------|
| `path/to/file.ts` | (description of changes) | S/M/L |

#### New Files to Create

| File | Purpose |
|------|---------|
| `path/to/new.ts` | (what it does) |

#### Code Patterns

Follow these patterns from existing code:

**Pattern**: (name)
**Reference**: `path/to/example.ts:100-150`

```typescript
// Example code or pseudocode showing the pattern
```

#### Implementation Notes

```typescript
// Pseudocode or actual code snippets
// showing what needs to be implemented
```

#### Edge Cases

| Edge Case | How to Handle |
|-----------|---------------|
| | |

#### Validation

How to verify this section is complete:
- [ ]
- [ ]

---

### Section 2: (Name)

(Repeat structure for each section)

---

## Data Changes

### Database Schema

```sql
-- Schema changes if any
```

### Data Migrations

(Migration strategy and scripts)

### Data Structures

```typescript
// New or modified interfaces/types
```

---

## API Changes

### New Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| | | |

### Modified Endpoints

| Endpoint | Change |
|----------|--------|
| | |

### Breaking Changes

(Any breaking changes and migration path)

---

## Test Strategy

### Unit Tests

| Component | What to Test | File |
|-----------|--------------|------|
| | | |

### Integration Tests

| Flow | What to Test | File |
|------|--------------|------|
| | | |

### Manual Testing

Steps for manual verification:

1.
2.
3.

---

## Migration / Rollout

### Deployment Steps

1.
2.
3.

### Feature Flags

(If using feature flags)

### Rollback Plan

If something goes wrong:

1.
2.

---

## Documentation Updates

| Document | Changes Needed |
|----------|----------------|
| | |

---

## Open Items

(Things still to be figured out during implementation)

- [ ]
