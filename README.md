# Claude Agent Loop

An autonomous AI coding agent that runs Claude Code repeatedly until all PRD items are complete. Features built-in context handoff for seamless continuation across sessions—no external dependencies required.

![Claude Agent Loop](https://img.shields.io/badge/Claude-Agent%20Loop-blue?style=for-the-badge)
![License](https://img.shields.io/badge/license-MIT-green?style=for-the-badge)

## Overview

Claude Agent Loop is inspired by [Geoffrey Huntley's Ralph pattern](https://ghuntley.com/ralph/) and [snarktank/ralph](https://github.com/snarktank/ralph), but implements its own context management system that works with vanilla Claude Code—no AMP or other external tools required.

### Key Features

- **Autonomous Execution**: Runs until all user stories are complete
- **Built-in Context Handoff**: Self-monitors context usage and hands off to fresh instances
- **Complexity-Based Story Sizing**: Analyzes PRDs to recommend appropriate story counts
- **State Persistence**: Memory persists via git, progress logs, and handoff files
- **Zero External Dependencies**: Works with Claude Code CLI only

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                         loop.sh                              │
│  Spawns Claude instances, detects handoffs, loops until done │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Claude Instance                           │
│  1. Read handoff.json (if exists)                           │
│  2. Read progress.txt patterns                              │
│  3. Pick next story from prd.json                           │
│  4. Implement, test, commit                                 │
│  5. Update status                                           │
│  6. Hand off if context filling → writes handoff.json       │
│  7. Or continue to next story                               │
└─────────────────────────────────────────────────────────────┘
```

### Context Handoff

Instead of relying on external tools for context detection, the agent self-monitors for:
- Response quality degradation
- Difficulty recalling earlier details
- After processing many large files

When approaching ~80% context capacity, it:
1. Commits work at a safe checkpoint
2. Writes `handoff.json` with current state
3. Signals `<handoff>CONTEXT_THRESHOLD</handoff>`
4. Loop spawns fresh instance that continues seamlessly

## Prerequisites

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) (`npm install -g @anthropic-ai/claude-code`)
- `jq` for JSON processing (`brew install jq` on macOS)
- A git repository for your project

## Quick Start

### 1. Copy to Your Project

```bash
# Clone or download
git clone https://github.com/anthropics/claude-agent-loop.git

# Copy to your project
cp -r claude-agent-loop/scripts your-project/scripts/ralph
```

### 2. Create Your PRD

Create a `prd.json` in your ralph directory:

```json
{
  "project": "MyApp",
  "branchName": "ralph/feature-name",
  "description": "Feature description",
  "userStories": [
    {
      "id": "US-001",
      "title": "Add database field",
      "description": "As a developer, I need to store X",
      "acceptanceCriteria": [
        "Add column to table",
        "Migration runs successfully",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

### 3. Run the Loop

```bash
./scripts/ralph/loop.sh [max_iterations]  # Default: 20
```

## Complexity Analysis

Use the analyzer to determine appropriate story counts for your PRD:

```bash
python3 scripts/analyze.py path/to/requirements.md --analyze-only
```

Output:
```
==================================================
  Complexity Analysis Results
==================================================

  Complexity Score: 12.4
  Category: MEDIUM
  Recommended Stories: 6-12
  Estimated Iterations: 6-18

  Factor Breakdown:
    Functional Requirements: 5
    Integration Points: 2
    UI Components: 8
    Database Changes: 3
    ...
==================================================
```

### Complexity Categories

| Score | Category | Stories | Iterations |
|-------|----------|---------|------------|
| 1-5 | Simple | 3-5 | 3-5 |
| 6-15 | Medium | 6-12 | 6-15 |
| 16-30 | Complex | 13-25 | 15-30 |
| 31+ | Enterprise | 26-50 | 30-60 |

## File Structure

```
your-project/
├── scripts/ralph/
│   ├── loop.sh           # Main agent loop
│   ├── CLAUDE.md         # Agent instructions
│   ├── analyze.py        # Complexity analyzer
│   ├── prd.json          # User stories (you create this)
│   ├── progress.txt      # Learning log (auto-generated)
│   └── handoff.json      # Context state (auto-generated)
└── archive/              # Previous run archives
```

## Key Files

| File | Purpose |
|------|---------|
| `loop.sh` | Bash script that spawns Claude instances and manages the loop |
| `CLAUDE.md` | Instructions piped to each Claude instance |
| `analyze.py` | Analyzes PRD complexity and recommends story counts |
| `prd.json` | Your user stories with completion status |
| `progress.txt` | Learnings and patterns for future iterations |
| `handoff.json` | Context state for mid-story handoffs |

## Story Sizing Guidelines

Each story must be completable in ONE context window.

**Right-sized stories:**
- Add a database column and migration
- Create a single UI component
- Implement one API endpoint
- Add a filter/sort feature

**Too large (split these):**
- "Build the entire dashboard"
- "Add authentication"
- "Refactor the API layer"

**Rule of thumb:** If you can't describe the change in 2-3 sentences, split it.

## Handoff Protocol

The `handoff.json` captures precise work state:

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "reason": "context_threshold",
  "current_story": {
    "id": "US-003",
    "title": "Add priority selector",
    "progress_percent": 65
  },
  "work_in_progress": {
    "files_modified": ["src/components/TaskEdit.tsx"],
    "uncommitted_changes": "Added dropdown UI",
    "next_steps": ["Wire up server action", "Test in browser"]
  },
  "handoff_instruction": "Continue US-003 - dropdown rendered, need save logic"
}
```

## Progress Logging

The agent maintains `progress.txt` with:

1. **Codebase Patterns** (top section) - Reusable patterns discovered
2. **Progress Entries** - What was done in each iteration

Example:
```markdown
## Codebase Patterns
- Use server actions for mutations
- Components in src/components/[feature]/
- Use toast() for user feedback

---

## 2024-01-15 10:30 - US-001
- Added priority column to tasks table
- Files: prisma/schema.prisma, migrations/
- **Learnings:** Always use IF NOT EXISTS for migrations
---
```

## Completion Signals

- `<promise>COMPLETE</promise>` - All stories done, loop exits successfully
- `<handoff>CONTEXT_THRESHOLD</handoff>` - Context filling, spawn fresh instance

## Debugging

```bash
# Check story status
cat scripts/ralph/prd.json | jq '.userStories[] | {id, title, passes}'

# View progress and patterns
cat scripts/ralph/progress.txt

# Check for pending handoff
cat scripts/ralph/handoff.json

# View git history
git log --oneline -10
```

## Customization

### Agent Instructions

Edit `CLAUDE.md` to customize:
- Quality check commands for your project
- Commit message format
- Project-specific conventions

### Loop Behavior

Edit `loop.sh` to customize:
- Max iterations (default: 20)
- Sleep duration between iterations
- Archive behavior

## Comparison with Ralph/AMP

| Feature | Ralph + AMP | Claude Agent Loop |
|---------|-------------|-------------------|
| Context Detection | AMP auto-detects | Self-monitoring |
| Dependencies | Requires AMP | Claude Code only |
| Handoff Mechanism | AMP internal | handoff.json file |
| Complexity Analysis | Manual | Built-in analyzer |
| Story Sizing | Manual | Recommended by score |

## License

MIT License - See [LICENSE](LICENSE) for details.

## Credits

- Inspired by [Geoffrey Huntley's Ralph pattern](https://ghuntley.com/ralph/)
- Based on concepts from [snarktank/ralph](https://github.com/snarktank/ralph)
- Built for [Claude Code](https://docs.anthropic.com/en/docs/claude-code)
