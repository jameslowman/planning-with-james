---
name: explore
description: Query the knowledge graph to understand code, trace impact, or review changes. Args: natural language query.
disable-model-invocation: true
allowed-tools: Bash, Glob, Grep, Read, Write, Task, AskUserQuestion
---

# Explore the Knowledge Graph

**NON-NEGOTIABLE: ALL PATHS ARE RELATIVE TO THE REPO ROOT**
All `.claude/planning-with-james/` paths in this skill are relative to the **current working directory** (the repo you're working in), NOT `~/.claude/`. The knowledge graph lives inside the project, not in your home directory. If you're unsure, run `pwd` to confirm you're in the repo root.

**NON-NEGOTIABLE: THIS SKILL ONLY READS THE KNOWLEDGE GRAPH, NEVER UPDATES IT**
If the knowledge graph is missing or stale, inform the user and suggest they run `/planning-with-james:create-knowledge` or `/planning-with-james:update-knowledge`. Do NOT run those skills yourself. Do NOT modify any knowledge files.

**NON-NEGOTIABLE: KNOWLEDGE GRAPH IS YOUR PRIMARY SOURCE**
This skill exists because the knowledge graph already contains deep analysis of module boundaries, key files, public interfaces, internal patterns, cross-module dependencies, and documented gotchas. You MUST answer from the knowledge graph first. Only fall back to source code when the knowledge graph has an explicit gap.

This is a stateless, lightweight skill. No phases, no state files, no registry entries. Answer the user's question and offer to save findings. That's it.

---

## Step 1: Verify Knowledge Graph Exists

Read `.claude/planning-with-james/knowledge/_overview.md`.

- **If missing**: Tell the user there is no knowledge graph for this repo. Suggest running `/planning-with-james:create-knowledge` first. **Stop here.**
- **If present**: Check the `last_commit` in the frontmatter. Compare it to the current HEAD (`git rev-parse HEAD`). If they differ, note this as a caveat: "The knowledge graph was last updated at commit {hash}. There have been changes since. Findings may not reflect recent code." Continue.

---

## Step 2: Handle Empty Arguments

If `$ARGUMENTS` is empty or blank, ask the user what they want to explore using `AskUserQuestion`:

Question: "What do you want to explore? You can ask about how something works, what would be affected by a change, trace a flow through the system, or review your current diff."

Options:
- "How does [something] work?" (Understanding)
- "What gets affected if I change [something]?" (Impact)
- "Trace the flow from [X] to [Y]" (Flow)
- "What's affected by my current changes?" (Diff Impact)

Use their response as the query for the remaining steps.

---

## Step 3: Classify the Query

Classify `$ARGUMENTS` (or the user's response from Step 2) into one of four types:

| Type | Signal Words | Example |
|------|-------------|---------|
| Understanding | "how does", "what is", "explain", "describe", "tell me about" | "how does auth work" |
| Impact | "what gets affected", "what depends on", "blast radius", "what uses", "what breaks" | "what breaks if I change the rate module" |
| Flow | "trace", "flow", "path from X to Y", "lifecycle", "journey" | "trace a request from API to database" |
| Diff Impact | "my changes", "current diff", "review my changes", "what I changed", "uncommitted" | "what's affected by my current changes" |

Default to **Understanding** if the query is ambiguous.

---

## Step 4: Load Knowledge Context (Lean)

For ALL query types, start by reading:
1. `.claude/planning-with-james/knowledge/_overview.md` (module map, system description)
2. `.claude/planning-with-james/knowledge/_discovery.json` (module list with file paths)

Then branch based on query type:

### Understanding / Impact / Flow
3. Read `.claude/planning-with-james/knowledge/_graph.json`
4. Identify the target module(s) from the query. Match against module names and descriptions in `_discovery.json`. If no exact match, search for partial matches and ask the user to confirm if ambiguous.
5. Read the target module(s) `_index.md`: `.claude/planning-with-james/knowledge/modules/{module-id}/_index.md`
6. Read the target module(s) `_refs.json`: `.claude/planning-with-james/knowledge/modules/{module-id}/_refs.json`

### Diff Impact
3. Run `git diff --name-only` and `git diff --cached --name-only` to get all changed files (staged and unstaged).
4. If no files changed: respond "No uncommitted changes found." **Stop here.**
5. Map changed files to modules using `_discovery.json` path entries. Each file should map to at most one module.
6. Read `.claude/planning-with-james/knowledge/_graph.json`
7. Read `_index.md` for each affected module.

---

## Step 5: Execute the Query

### Understanding
Synthesize an explanation from the knowledge graph. Cover:
- **Purpose**: What does this module/component do and why does it exist?
- **Key files**: The important files and what each does.
- **Public interface**: What other modules can call/use.
- **How it connects**: What it depends on and what depends on it (from `_graph.json` edges and `_refs.json`).
- **Patterns and conventions**: Internal patterns, frameworks used, coding conventions.
- **Gotchas**: Known issues, caveats, things that are non-obvious.

Present this as a clear, structured response. Reference specific files and modules.

### Impact
Traverse `_graph.json` edges outward from the target module(s), up to 2 hops:

1. **Direct dependents** (1 hop): modules with edges pointing TO the target. Note the edge type (imports, calls, shares_types, etc.) and what specifically connects them.
2. **Indirect dependents** (2 hops): modules that depend on the direct dependents. Note the propagation path.
3. **Impact assessment**: For each dependent, describe what specifically might break or need changes. Categorize as high/medium/low risk based on coupling strength.

Present as an impact map: target at center, direct dependents first ring, indirect dependents second ring.

### Flow
Find a path through `_graph.json` between the start and end points:

1. Identify the start module and end module from the query.
2. Find the shortest path through `_graph.json` edges.
3. For each module along the path, describe:
   - What enters this module (input format, entry point)
   - What happens inside (transformation, validation, business logic)
   - What exits this module (output format, exit point, next hop)
4. Check `.claude/planning-with-james/knowledge/flows/` for any existing flow documentation that covers this path. If found, reference it.

If no path exists between start and end, say so and suggest alternative paths or explain why they might not be connected.

### Diff Impact
For each changed module:

1. **Classify changes**: Run `git diff` on the specific changed files. Categorize each change as:
   - **Interface change**: public API, exports, types, function signatures, configuration shape
   - **Internal-only**: implementation details, private functions, comments, formatting
2. **Impact traversal**: For modules with interface changes ONLY, traverse `_graph.json` edges outward (1-2 hops) to find affected dependents.
3. **Present findings**:
   - Changed modules and nature of changes (interface vs internal)
   - For interface changes: potentially affected modules and why
   - Suggested test focus: which areas to test based on the blast radius
   - Risk assessment: high/medium/low for each affected area

---

## Step 6: Deeper Exploration (When Needed)

After answering from the knowledge graph, assess whether there are gaps:
- A module referenced in the query has no knowledge entry
- The knowledge entry is thin (missing key sections)
- The query requires detail beyond what the knowledge graph captured

If gaps exist, **offer** to explore deeper:

"The knowledge graph doesn't have detailed information about [specific gap]. Want me to explore the source code to fill this in?"

If the user says yes, spawn an Explore subagent (Opus, one at a time) with this in its prompt:

```
BEFORE exploring source code, read these knowledge files:
- .claude/planning-with-james/knowledge/_overview.md
- .claude/planning-with-james/knowledge/modules/{module}/_index.md
(list the specific module paths relevant to the subagent's scope)
These files contain documented module boundaries, key files, interfaces, patterns, and gotchas. Start there, then explore source code to fill the specific gap: {describe the gap}.
Return your findings conversationally. Do NOT write any files.
```

Present the subagent's findings to the user.

---

## Step 7: Offer to Save

After presenting findings, ask:

"Want me to save these findings to a file?"

If the user says yes:
1. Create the explorations directory if it doesn't exist: `.claude/planning-with-james/explorations/`
2. Generate a filename: `{query-slug}_{YYYY-MM-DD}.md` where query-slug is a short kebab-case version of the query (e.g., `how-auth-works_2026-02-07.md`)
3. Write the file with this structure:

```markdown
---
query: "{original query}"
type: "{Understanding|Impact|Flow|Diff Impact}"
timestamp: "{ISO 8601}"
modules: ["{module-id-1}", "{module-id-2}"]
---

# {Query as title}

{Full findings from Step 5, formatted as markdown}
```

If the user says no, that's fine. Done.

---

## Edge Cases

### Query spans many modules (8+)
If the query touches 8 or more modules, provide a high-level summary first:
- List all affected modules with one-line descriptions
- Identify the 2-3 most critical paths
- Offer to deep-dive into specific paths: "This touches {N} modules. Want me to focus on a specific area?"

### Unknown module reference
If the user's query references something that doesn't match any module in `_discovery.json`:
1. Search `_discovery.json` for partial matches (substring, similar names)
2. If partial matches found: "I couldn't find an exact match for '{query}', but these modules might be what you mean: {list}. Which one?"
3. If no matches: "I couldn't find '{query}' in the knowledge graph. It might be undocumented, or it might go by a different name. Want me to search the source code directly?"

### Stale knowledge
If the knowledge graph is behind HEAD (detected in Step 1), prefix findings with a caveat and suggest running `/planning-with-james:update-knowledge` after exploring. Do not refuse to answer -- stale knowledge is still better than no knowledge.
