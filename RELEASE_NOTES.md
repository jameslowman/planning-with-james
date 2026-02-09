# Release Notes

## v1.1.0

### New: `/query` skill (originally `/explore`, renamed to avoid collision with Claude Code's built-in Explore agent)

Query the knowledge graph for everyday work without starting a full planning session. Ask how something works, check the blast radius of a change, trace a flow through the system, or review what your current diff affects.

Four query types:
- **Understanding**: "how does auth work" -- synthesizes module purpose, key files, interfaces, patterns, gotchas
- **Impact**: "what breaks if I change the rate module" -- traverses the dependency graph 2 hops out, builds an impact map
- **Flow**: "trace a request from API to database" -- walks the graph path, describes what enters/happens/exits at each step
- **Diff Impact**: "what's affected by my current changes" -- maps uncommitted changes to modules, classifies interface vs internal changes, identifies downstream risk

Stateless by design: no phases, no state files, no registry entries. Optionally saves findings to `.claude/planning-with-james/explorations/`.

---

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
