---
epic_id: "{{EPIC_ID}}"
---

# Validation Gates

These are human-driven verification points between sub-plans. The system surfaces
them; you execute and confirm. Do not skip these - they catch problems that
automated tests cannot.

## After Sub-Plan 1: (Name)

Before starting Sub-Plan 2, verify:
- [ ] (gate - e.g., "Deploy to staging, verify functionality")
- [ ] (gate - e.g., "Cost measurement within acceptable range")
- [ ] (gate - e.g., "No regressions in existing system")
- [ ] Epic review completed (`/planning-with-james:epic review`)

## After Sub-Plan 2: (Name)

Before starting Sub-Plan 3, verify:
- [ ] (gate)
- [ ] Epic review completed
