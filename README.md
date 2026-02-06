# Planning with James

A Claude Code plugin for people who are tired of shallow plans and lost context.

## The Problem

You're working on a complex feature. You spend 30 minutes explaining the problem to Claude. It produces a plan. You start implementing. Halfway through, Claude hits context limits and forgets everything. You explain it again. The new plan contradicts the old one. You lose an afternoon.

Or worse: Claude produces a plan that *looks* good but is based on stale assumptions. It didn't ask you the right questions. It didn't know that the "1-3 second" API call it found in comments actually takes 35ms now. It chose an approach you would have rejected if it had just asked.

## The Solution

This plugin does two things:

1. **Builds a knowledge graph of your codebase** that survives context loss. Every module, every relationship, documented and navigable. When Claude forgets, the files remember.

2. **Forces real conversations during planning.** No more racing through phases to produce an artifact. Every checkpoint presents findings and asks: "Does this match your understanding? What am I missing?"

## How It Works

### First Time Setup

```
/planning-with-james:create-knowledge
```

This takes a while. It spawns agents to analyze every significant module in your codebase, maps relationships, and writes it all to `.claude/planning-with-james/knowledge/`. The knowledge graph is yours to keep, update, and build on.

### Keeping Knowledge Fresh

```
/planning-with-james:update-knowledge
```

Checks what changed since the last index and updates only the affected modules. Runs automatically through the graph to update anything connected to what changed.

If something's wrong or you want to go deeper:

```
/planning-with-james:update-knowledge the email pipeline is more complex than documented, we need to go deeper on the sync logic
```

It'll ask clarifying questions, then do targeted deep dives.

You can also check the health of your knowledge graph:

```
/planning-with-james:update-knowledge verify
```

And fix any problems it finds:

```
/planning-with-james:update-knowledge repair
```

### Planning a Feature

```
/planning-with-james:plan
```

Seven phases, each with a real conversation:

1. **Context** - What are we solving? (You explain, it confirms understanding)
2. **Scoping** - What's in scope? (It proposes, you correct)
3. **Discovery** - What did we find? (It reports, you validate)
4. **Approach** - How should we do this? (It recommends, you decide)
5. **Detailed Plan** - What exactly will we build? (It specifies, you review)
6. **Task Breakdown** - What are the steps? (It lists, you approve)
7. **Finalize** - Ready to implement

Every phase produces a file. If context is lost, the files remain. The plan folder becomes a complete record of decisions and reasoning.

### Implementing the Plan

```
/planning-with-james:go-time
```

Activates a plan and executes tasks one at a time. After each task:
- Updates a scratch pad (your lifeline through context loss)
- Checks off the task
- Evaluates if anything unexpected happened

If Claude loses context mid-implementation, it reads the files and picks up exactly where it left off.

Pause anytime:
```
/planning-with-james:go-time pause
```

Resume later:
```
/planning-with-james:go-time
```

### Big Initiatives (Epics)

For work that spans weeks or months across many sessions:

```
/planning-with-james:epic
```

Creates a "plan of plans" with a `learnings.md` file that accumulates wisdom across sub-plans. When you finish one phase of a big project, the epic review captures what you learned so the next phase doesn't start from scratch.

## What Makes This Different

**The knowledge graph is the foundation.** Before any planning happens, the system reads what it already knows about your codebase. No more rediscovering the same code patterns every session.

**Conversation over artifacts.** The plan skill doesn't race to produce documents. It stops and asks questions. It presents findings and waits for you to say "that's wrong" or "you missed something." Your knowledge fills the gaps that code analysis can't.

**Files are the source of truth.** Everything important gets written to disk. Context limits become annoying, not catastrophic. You can close your laptop, come back tomorrow, and pick up exactly where you left off.

**Progress survives anything.** Every skill tracks its progress in JSON files. If something crashes halfway through a knowledge update or a long implementation session, re-running the command resumes from where it stopped.

## Installation

In Claude Code, run:

```
/plugin marketplace add jameslowman/planning-with-james
/plugin install planning-with-james@james-plugins
```

That's it. The plugin is now available in all your projects.

## Philosophy

Cost is not a concern. Depth is the goal.

This plugin uses Opus for analysis. A full knowledge index might consume an entire session. A thorough plan might take an hour of back-and-forth. That's the point. Great features aren't built with shallow knowledge.

The time you spend in planning conversations saves days in implementation. The context you preserve across sessions saves hours of re-explanation. The knowledge graph you build compounds over time.

## Files It Creates

```
.claude/planning-with-james/
├── knowledge/           # The knowledge graph
│   ├── _overview.md     # Entry point, stats, module map
│   ├── _graph.json      # Machine-readable relationships
│   ├── _architecture.md # System-wide narrative
│   └── modules/         # Per-module documentation
│       └── {module}/
│           ├── _index.md   # Module overview
│           └── _refs.json  # Dependencies and connections
│
└── plans/
    └── _registry.json   # Tracks all plans and their states
```

Plan folders (location is your choice):
```
{plan-folder}/
├── _plan_state.json        # Current phase, progress
├── context_scratch_pad.md  # Cold restart document
├── problem_description.md  # What we're solving
├── scope.md                # What's in/out
├── findings/               # Discovery results
├── approach.md             # How we'll do it
├── detailed_plan.md        # Full specification
└── tasks.md                # Executable checklist
```

## License

MIT
