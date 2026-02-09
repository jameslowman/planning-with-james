# Planning with James

A Claude Code plugin for deep codebase knowledge and structured planning.

## Get Started

**Install:**
```
/plugin marketplace add jameslowman/planning-with-james
/plugin install planning-with-james@james-plugins
```

**First time only - index your codebase:**
```
/planning-with-james:create-knowledge
```
This takes a while. Go get coffee. It's building a knowledge graph of your entire codebase - every module, every relationship, documented and navigable.

**Plan something:**
```
/planning-with-james:plan
```

It will ask what you're working on. Give it context. Paste a Linear ticket URL, describe the bug, share your theory about what's wrong. Ramble at it. More context is better.

```
User: We need to add UNLOCODE data to the rate search response. Here's the ticket:
https://linear.app/company/issue/SB-1112

The frontend team needs origin and destination UNLOCODEs to render the new
shipping visualization. I think the data is already in the routing tables
somewhere, we just need to surface it through the API.
```

It will start asking questions. It will show you what it found. It will check if its understanding matches yours. This back-and-forth is the point - your knowledge fills gaps that code analysis can't.

**Execute the plan:**
```
/planning-with-james:go-time
```

Works through tasks one at a time. If Claude loses context mid-way, it reads the files and picks up where it left off.

---

## Why This Exists

You're working on a complex feature. You spend 30 minutes explaining the problem to Claude. It produces a plan. You start implementing. Halfway through, Claude hits context limits and forgets everything. You explain it again. The new plan contradicts the old one. You lose an afternoon.

Or worse: Claude produces a plan that *looks* good but is based on stale assumptions. It didn't ask you the right questions. It chose an approach you would have rejected if it had just asked.

This plugin fixes both problems:

1. **Knowledge graph** - Your codebase is indexed to files that survive context loss. When Claude forgets, the files remember.

2. **Real conversations** - Every planning phase stops and asks: "Does this match your understanding? What am I missing?" No more racing through phases to produce an artifact.

---

## Commands

| Command | What it does |
|---------|--------------|
| `/planning-with-james:create-knowledge` | Index your codebase (first time) |
| `/planning-with-james:update-knowledge` | Update after code changes |
| `/planning-with-james:update-knowledge verify` | Check knowledge graph health |
| `/planning-with-james:update-knowledge repair` | Fix any problems |
| `/planning-with-james:query` | Ask about code: how it works, what changes affect, trace flows |
| `/planning-with-james:plan` | Start planning something |
| `/planning-with-james:go-time` | Execute a plan |
| `/planning-with-james:go-time pause` | Pause implementation |
| `/planning-with-james:epic` | Plan something big (weeks/months) |

---

## Philosophy

Cost is not a concern. Depth is the goal.

This plugin uses Opus for analysis. A full knowledge index might consume an entire session. A thorough plan might take an hour of back-and-forth. That's the point.

The time you spend in planning conversations saves days in implementation.

---

## License

MIT
