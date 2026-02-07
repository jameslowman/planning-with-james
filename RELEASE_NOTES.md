# Release Notes

## v1.0.0

A Claude Code plugin for deep codebase understanding and structured feature development.

### What it does

Builds a knowledge graph of your codebase so Claude starts from documented understanding instead of rediscovering everything through grep. Plans features through structured conversation with checkpoints, producing artifacts that survive context loss.

### Skills

| Skill | Purpose |
|-------|---------|
| `/create-knowledge` | Index your codebase into a knowledge graph (modules, interfaces, patterns, dependencies) |
| `/update-knowledge` | Incremental updates after commits, with verify and repair modes |
| `/plan` | Structured planning through phases: context, scoping, discovery, approach, detailed plan, tasks |
| `/go-time` | Execute a plan with active task tracking and context resilience |
| `/epic` | Coordinate multiple related plans as a single initiative |

### Key Design Decisions

- **Knowledge-first**: Claude reads the knowledge graph before touching source code
- **Conversation over speed**: Every planning checkpoint invites user input rather than racing ahead
- **Context resilience**: Scratch pads and state files let work survive autocompact
- **Linear updates, parallel creation**: Prevents context exhaustion during knowledge updates
- **Hooks**: Session start warnings for stale knowledge, post-compact reminders for active plans

### Install

```
/plugin marketplace add jameslowman/planning-with-james
/plugin install planning-with-james@james-plugins
```
