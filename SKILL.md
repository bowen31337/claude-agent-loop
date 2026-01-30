---
name: autonomous-agent-loop
description: "Autonomous AI agent loop with built-in context handoff. Runs Claude Code repeatedly until all PRD items complete. Features: (1) automatic context threshold detection and handoff, (2) complexity-based story decomposition, (3) state persistence via git/files, (4) fresh instance spawning with summary transfer. Use when: starting autonomous development, running ralph-style loops, implementing PRDs autonomously, needing context-aware agent handoffs, or executing multi-story feature development. Triggers on: /autonomous-agent-loop, ralph loop, agent loop, run autonomously, implement prd."
---

# Autonomous Agent Loop

An autonomous coding agent system with built-in context handoff - no external dependencies required.

## Core Concept

Each iteration spawns a **fresh Claude instance** with clean context. Memory persists via:
- Git history (commits from previous iterations)
- `progress.txt` (learnings and handoff summaries)
- `prd.json` (story completion status)
- `handoff.json` (context state for seamless continuation)

## Quick Start

### 1. Generate PRD from Requirements

```
/autonomous-agent-loop analyze [path/to/requirements.md]
```

This analyzes complexity and generates appropriately-sized stories.

### 2. Run the Agent Loop

```bash
./scripts/ralph/loop.sh [max_iterations]
```

The loop handles everything: context monitoring, handoffs, and story execution.

---

## Workflow

### Phase 1: Complexity Analysis

Before generating stories, analyze the PRD/Architecture documents:

```
Complexity Score = (
  functional_requirements × 2 +
  integration_points × 3 +
  ui_components × 1.5 +
  database_changes × 2 +
  external_apis × 3
) / 5
```

**Story Count Guidelines:**

| Complexity Score | Story Count | Iteration Estimate |
|------------------|-------------|-------------------|
| 1-5 (Simple)     | 3-5 stories | 3-5 iterations    |
| 6-15 (Medium)    | 6-12 stories| 6-15 iterations   |
| 16-30 (Complex)  | 13-25 stories| 15-30 iterations |
| 31+ (Enterprise) | 26-50 stories| 30-60 iterations |

### Phase 2: Story Generation

Each story must be completable in ONE context window (~50k tokens of work).

**Right-sized stories:**
- Add a database column and migration
- Create a single UI component
- Implement one API endpoint
- Add a filter/sort feature

**Too large (split these):**
- "Build the dashboard" → Split into data layer, components, layout, interactions
- "Add authentication" → Split into schema, middleware, UI, session handling

### Phase 3: Autonomous Execution

The agent loop:
1. Reads `prd.json` for next story
2. Checks `progress.txt` for codebase patterns
3. Implements the story
4. Runs quality checks
5. Commits if passing
6. Updates status and progress
7. Checks context threshold
8. Hands off if needed, or continues

---

## Context Handoff Protocol

### Automatic Detection

The agent monitors its context usage. When approaching threshold (~80% capacity):

1. **Pause current work** at a safe checkpoint
2. **Generate handoff summary** capturing:
   - Current story progress (% complete)
   - Files being modified
   - Uncommitted changes description
   - Next immediate steps
   - Blockers or decisions needed
3. **Write to handoff.json**
4. **Signal handoff** with `<handoff>CONTEXT_THRESHOLD</handoff>`

### Handoff Summary Format

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "reason": "context_threshold",
  "current_story": {
    "id": "US-003",
    "title": "Add priority selector",
    "progress_percent": 65,
    "status": "implementing"
  },
  "work_in_progress": {
    "files_modified": ["src/components/TaskEdit.tsx", "src/actions/tasks.ts"],
    "uncommitted_changes": "Added priority dropdown UI, server action pending",
    "last_completed_step": "Created PrioritySelect component",
    "next_steps": [
      "Wire up server action to save priority",
      "Add optimistic update",
      "Test in browser"
    ]
  },
  "context_learned": [
    "This codebase uses server actions pattern",
    "Components are in src/components/[feature]/",
    "Use toast() for user feedback"
  ],
  "blockers": [],
  "handoff_instruction": "Continue implementing US-003 - priority dropdown is rendered, need to connect save logic"
}
```

### Fresh Instance Continuation

When a new instance starts after handoff:

1. Read `handoff.json` first
2. Read `progress.txt` Codebase Patterns section
3. Resume from `handoff_instruction`
4. Complete the in-progress story
5. Continue with remaining stories

---

## Agent Instructions Template

When running as the autonomous agent, follow this protocol:

### On Startup

1. Check for `handoff.json` - if exists, this is a continuation
2. Read `progress.txt` Codebase Patterns section
3. Read `prd.json` for story status
4. If handoff exists: resume from handoff instruction
5. If no handoff: pick highest priority story where `passes: false`

### During Implementation

1. Work on ONE story at a time
2. Monitor your context usage (note when responses feel constrained)
3. Commit early and often (each logical unit of work)
4. Update progress.txt with learnings

### Context Threshold Detection

Watch for these signs you're approaching context limits:
- Difficulty recalling earlier conversation details
- Responses becoming shorter or less detailed
- Needing to re-read files you recently viewed
- Feeling "fuzzy" about the overall task

When detected:
1. Stop at the nearest safe checkpoint
2. Commit any complete work
3. Write handoff.json with current state
4. Output `<handoff>CONTEXT_THRESHOLD</handoff>`

### On Story Completion

1. Run quality checks (typecheck, lint, test)
2. Commit with message: `feat: [Story ID] - [Story Title]`
3. Update prd.json: set `passes: true`
4. Append to progress.txt
5. Check if ALL stories complete → output `<promise>COMPLETE</promise>`
6. Otherwise, continue to next story (if context permits)

### Progress Report Format

Append to progress.txt:

```
## [Date/Time] - [Story ID]
- What was implemented
- Files changed
- **Learnings:**
  - Patterns discovered
  - Gotchas encountered
  - Useful context for future iterations
---
```

---

## File Structure

```
project/
├── scripts/ralph/
│   ├── loop.sh           # Main agent loop script
│   ├── CLAUDE.md         # Agent instructions (auto-loaded)
│   └── analyze.py        # Complexity analyzer
├── prd.json              # Story definitions and status
├── progress.txt          # Learnings and progress log
├── handoff.json          # Context handoff state (when needed)
└── archive/              # Previous run archives
```

---

## Commands

### Analyze Requirements

Analyzes PRD/Architecture docs and generates sized stories:

```
/autonomous-agent-loop analyze path/to/prd.md [path/to/architecture.md]
```

### Convert Existing PRD

Converts a PRD markdown file to prd.json:

```
/autonomous-agent-loop convert path/to/prd.md
```

### Run Loop

Starts the autonomous agent loop:

```bash
./scripts/ralph/loop.sh [max_iterations]  # Default: 20
```

### Check Status

```bash
cat prd.json | jq '.userStories[] | {id, title, passes}'
```

---

## PRD JSON Format

```json
{
  "project": "ProjectName",
  "branchName": "ralph/feature-name",
  "description": "Feature description",
  "complexity": {
    "score": 12,
    "category": "medium",
    "estimated_iterations": 15
  },
  "userStories": [
    {
      "id": "US-001",
      "title": "Story title",
      "description": "As a [user], I want [feature] so that [benefit]",
      "acceptanceCriteria": [
        "Specific verifiable criterion",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

---

## Best Practices

### Story Sizing

- Each story: 1 context window of work
- If you cannot describe the change in 2-3 sentences, split it
- Dependencies flow forward (schema → backend → frontend)

### Quality Gates

Every story must include:
- `"Typecheck passes"` in acceptance criteria
- `"Tests pass"` for logic-heavy stories
- `"Verify in browser"` for UI stories

### Learning Persistence

The Codebase Patterns section in progress.txt is critical:
- Add patterns that apply across stories
- Include gotchas specific to this codebase
- Note file locations and conventions

### Handoff Discipline

- Don't wait until context is exhausted
- Hand off at logical checkpoints
- Include enough detail for seamless continuation
- Commit work-in-progress to a WIP commit if needed

---

## Troubleshooting

### Agent stuck in loop

Check `progress.txt` for repeated errors. Common causes:
- Failing quality checks (fix the code, not the checks)
- Missing dependencies
- Incorrect file paths

### Handoff not working

Ensure `handoff.json` is being written correctly. The next instance must read it on startup.

### Stories too large

Re-analyze with stricter splitting:
```
/autonomous-agent-loop analyze --max-story-size small path/to/prd.md
```

---

## References

- See `references/complexity-analysis.md` for detailed scoring
- See `references/handoff-protocol.md` for handoff implementation details
