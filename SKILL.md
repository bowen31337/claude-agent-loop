---
name: autonomous-agent-loop
description: "Autonomous AI agent loop with built-in context handoff. Runs Claude Code repeatedly until all PRD items complete. Features: (1) automatic context threshold detection and handoff, (2) complexity-based story decomposition, (3) state persistence via git/files, (4) fresh instance spawning with summary transfer. Use when: starting autonomous development, running ralph-style loops, implementing PRDs autonomously, needing context-aware agent handoffs, or executing multi-story feature development. Triggers on: /autonomous-agent-loop, ralph loop, agent loop, run autonomously, implement prd."
---

# Autonomous Agent Loop

An autonomous coding agent system with built-in context handoff - no external dependencies required.

## CRITICAL: Auto-Execute the Loop

**When this skill is invoked to run/start/execute the loop, you MUST automatically run the loop.sh script using Bash. Do NOT just provide instructions for the user to run manually.**

### Execution Steps

1. **Verify prd.json exists** in the project's scripts/ralph/ directory
2. **Calculate max iterations** based on remaining stories (stories × 1.5, minimum 10)
3. **Execute the loop automatically**:

```bash
# Run in background so user can monitor
./scripts/ralph/loop.sh [max_iterations]
```

### Example Auto-Execution

When user says "run the autonomous loop" or "start implementing the PRD":

```
# CORRECT - Execute it:
Bash: ./scripts/ralph/loop.sh 25

# WRONG - Don't just tell user:
"To run the loop: ./scripts/ralph/loop.sh 25"  # NO!
```

---

## Core Concept

Each iteration spawns a **fresh Claude instance** with clean context. Memory persists via:
- Git history (commits from previous iterations)
- `progress.txt` (learnings and handoff summaries)
- `prd.json` (story completion status)
- `handoff.json` (context state for seamless continuation)

---

## Workflow

### Phase 1: Setup (if prd.json doesn't exist)

1. Analyze complexity of requirements/PRD document
2. Generate appropriately-sized user stories
3. Create prd.json with stories
4. Initialize progress.txt

### Phase 2: Execution (Auto-Start)

**Automatically execute the loop** - do not wait for user to run manually:

```bash
./scripts/ralph/loop.sh [calculated_iterations]
```

The loop handles:
1. Reading prd.json for next story
2. Checking progress.txt for codebase patterns
3. Implementing the story
4. Running quality checks
5. Committing if passing
6. Updating status and progress
7. Checking context threshold
8. Handing off if needed, or continuing

---

## Complexity Analysis

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

| Complexity Score | Story Count | Max Iterations |
|------------------|-------------|----------------|
| 1-5 (Simple)     | 3-5 stories | 10             |
| 6-15 (Medium)    | 6-12 stories| 20             |
| 16-30 (Complex)  | 13-25 stories| 40            |
| 31+ (Enterprise) | 26-50 stories| 75            |

---

## Story Sizing

Each story must be completable in ONE context window (~50k tokens of work).

**Right-sized stories:**
- Add a database column and migration
- Create a single UI component
- Implement one API endpoint
- Add a filter/sort feature

**Too large (split these):**
- "Build the dashboard" → Split into data layer, components, layout, interactions
- "Add authentication" → Split into schema, middleware, UI, session handling

---

## Context Handoff Protocol

### Automatic Detection

The agent monitors its context usage. When approaching threshold (~80% capacity):

1. **Pause current work** at a safe checkpoint
2. **Generate handoff summary** capturing current state
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
    "files_modified": ["src/components/TaskEdit.tsx"],
    "uncommitted_changes": "Added priority dropdown UI",
    "next_steps": ["Wire up server action", "Test in browser"]
  },
  "handoff_instruction": "Continue US-003 - dropdown rendered, need save logic"
}
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

## Completion Signals

- `<promise>COMPLETE</promise>` - All stories done, loop exits
- `<handoff>CONTEXT_THRESHOLD</handoff>` - Context filling, spawn fresh instance

---

## References

- See `references/complexity-analysis.md` for detailed scoring
- See `references/handoff-protocol.md` for handoff implementation details
