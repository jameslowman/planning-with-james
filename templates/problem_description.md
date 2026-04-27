---
problem_type: null
created_at: "{{TIMESTAMP}}"
status: draft
---

# Problem Description

## Type

(Bug Fix | Feature | Refactor | New System)

## Summary

(One paragraph summary of the problem or goal)

## User's Description

(Verbatim or lightly edited user input - preserve their words and intent)

## Symptoms / Current Behavior

(For bugs: what's happening wrong, error messages, steps to reproduce)
(For features: what's missing, what users are asking for)
(For refactors: what's wrong with current implementation)

## Desired Behavior / Goals

(What should happen instead)
(What capability should exist)
(What improvement we're seeking)

## User Flows

**Source**: derived from `user_story.md` for use by Phase 4 test architecture
agents. Read `user_story.md` for the narrative form (cast, scenes, side effects);
this section is the structured re-cut of the same content. The narrative is
authored scene-by-scene with the user during Phase 1; this section is
mechanically extracted by the planner — do not re-author flows with the user.

### Flow 1: {{scene 1 title}}

1. {{User does X}}
2. {{User sees Y}}
3. {{User does Z}}
- **Expected**: {{what should happen}}
- **Actual**: {{what happens instead — for bugs}}

### Flow 2: {{scene 2 title}}

1. ...

### Alternate Paths / Edge Cases

(Variations on the main flows — different inputs, error conditions, boundary cases extracted from scenes)

- {{description of alternate path}}

## User's Theories

(Any hypotheses the user shared about cause or approach)
(Their intuitions about what might be wrong or how to fix it)

## External References

- Linear/GitHub ticket: (URL)
- Related PRs: (URLs)
- Screenshots: (descriptions)
- Error logs: (summary)
- Slack threads: (summary)

## Success Criteria

(How we know we're done - be specific and measurable)

1.
2.
3.

## Open Questions

(Things we need to clarify before proceeding)

- [ ]
- [ ]
