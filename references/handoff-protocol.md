# Context Handoff Protocol

This document details the handoff mechanism that enables seamless continuation across Claude instances.

## Overview

Unlike AMP's automatic context detection, this protocol relies on the agent self-monitoring context usage and triggering handoffs proactively. This provides similar functionality without external tool dependencies.

## Handoff Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    Instance A (Original)                     │
├─────────────────────────────────────────────────────────────┤
│  1. Start working on story                                  │
│  2. Make progress, commit checkpoints                       │
│  3. Notice context filling (~80%)                           │
│  4. Complete current logical unit                           │
│  5. Write handoff.json                                      │
│  6. Output <handoff>CONTEXT_THRESHOLD</handoff>             │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     Loop Script                              │
├─────────────────────────────────────────────────────────────┤
│  1. Detects handoff signal in output                        │
│  2. Spawns fresh Claude instance                            │
│  3. Instance B reads handoff.json first                     │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                    Instance B (Fresh)                        │
├─────────────────────────────────────────────────────────────┤
│  1. Read handoff.json (understands context)                 │
│  2. Read progress.txt patterns                              │
│  3. Resume from handoff_instruction                         │
│  4. Continue story or start next                            │
│  5. Delete handoff.json when story completes                │
└─────────────────────────────────────────────────────────────┘
```

## Context Threshold Detection

### Self-Monitoring Indicators

The agent should monitor for these signs of context filling:

1. **Response Quality Degradation**
   - Shorter, less detailed responses
   - Missing nuances that were discussed earlier
   - Forgetting file contents recently read

2. **Recall Difficulty**
   - Unable to reference earlier conversation details
   - Needing to re-read files that were just viewed
   - Confusion about the overall task scope

3. **Token Awareness**
   - After processing large files
   - After many tool calls
   - After lengthy implementation discussions

### When to Trigger

**DO trigger handoff when:**
- You've processed 5+ large files in this session
- You notice recall issues with earlier context
- You're about to start a complex sub-task
- The current story is 50%+ complete and complex

**DON'T trigger handoff when:**
- You're in the middle of a logical unit
- Uncommitted changes would be lost
- The current story is nearly complete (<10 min remaining)
- You're about to commit

## Handoff File Format

### handoff.json Schema

```json
{
  "timestamp": "ISO 8601 timestamp",
  "reason": "context_threshold | error | user_request",
  "current_story": {
    "id": "US-XXX",
    "title": "Story title",
    "progress_percent": 0-100,
    "status": "planning | implementing | testing | blocked"
  },
  "work_in_progress": {
    "files_modified": ["array", "of", "file", "paths"],
    "uncommitted_changes": "Description of what's not yet committed",
    "last_completed_step": "What was just finished",
    "next_steps": [
      "Array of specific next actions",
      "In order of execution"
    ]
  },
  "context_learned": [
    "Facts discovered during this session",
    "That aren't in progress.txt yet"
  ],
  "blockers": [
    "Any issues preventing progress",
    "Empty array if none"
  ],
  "handoff_instruction": "Single clear sentence telling next instance what to do"
}
```

### Field Guidelines

**timestamp:** Use ISO 8601 format for debugging/tracking.

**reason:** Why the handoff occurred:
- `context_threshold` - Normal context limit
- `error` - Unrecoverable error encountered
- `user_request` - User requested pause

**current_story:** What's being worked on:
- `progress_percent` - Rough estimate (25%, 50%, 75%, etc.)
- `status` - Current phase of the story

**work_in_progress:** Active work state:
- `files_modified` - Files touched, even if not committed
- `uncommitted_changes` - What would be lost without commit
- `last_completed_step` - Safe checkpoint reached
- `next_steps` - Specific, actionable items in order

**context_learned:** New knowledge this session:
- Only include if not already in progress.txt
- Will be added to Codebase Patterns if reusable

**blockers:** Issues requiring attention:
- Empty if work can continue normally
- Include specific error messages or decisions needed

**handoff_instruction:** Most important field:
- Single, clear directive
- Tells next instance exactly what to do first
- Example: "Continue implementing the save handler in TaskEdit.tsx - dropdown UI is done, need to wire up the server action"

## Safe Checkpoint Guidelines

### What Constitutes a Safe Checkpoint

1. **Code Compiles** - No syntax errors
2. **Tests Pass** - Or were passing before this unit
3. **Logical Unit Complete** - One coherent change
4. **State Captured** - Can be resumed

### Checkpoint Hierarchy

From safest to riskiest:
1. Story fully complete, committed
2. Sub-feature complete, committed
3. Sub-feature complete, uncommitted
4. Mid-implementation, compiles
5. Mid-implementation, broken

**Aim for level 1-3.** If at level 4-5, consider a WIP commit.

### WIP Commits

When handing off mid-implementation:

```bash
git add -A
git commit -m "WIP: [Story ID] - [Description of state]

Handoff state - do not merge until complete.
Files modified: [list]
Next: [immediate next step]"
```

The next instance should:
1. Read the WIP commit message
2. Continue from where it left off
3. Amend or squash the WIP commit when done

## Recovery Scenarios

### Scenario 1: Normal Handoff
1. Instance A completes logical unit
2. Writes handoff.json
3. Instance B picks up, continues story
4. Deletes handoff.json on story complete

### Scenario 2: Mid-Story Handoff
1. Instance A is 60% through story
2. Commits WIP, writes handoff.json
3. Instance B reads handoff, sees WIP
4. Completes story, amends WIP commit
5. Deletes handoff.json

### Scenario 3: Error Recovery
1. Instance A encounters error
2. Writes handoff.json with blocker
3. Instance B reads, attempts to resolve
4. If resolved, continues; if not, escalates

### Scenario 4: Story Boundary
1. Instance A completes story
2. Context still okay but ~70%
3. Proactively hands off before starting next
4. Instance B starts fresh on next story

## Anti-Patterns

### Don't Do This

1. **Handoff Without Commit**
   - Always commit or WIP-commit before handoff
   - Uncommitted work can be lost

2. **Vague Handoff Instructions**
   - Bad: "Continue working on the feature"
   - Good: "Wire up the handleSubmit function to call updatePriority server action"

3. **Waiting Until Exhausted**
   - Hand off at 80%, not 99%
   - Quality degrades rapidly at limit

4. **Multiple Stories Per Handoff**
   - Don't bundle multiple incomplete stories
   - Complete one or hand off cleanly

5. **Skipping Progress Update**
   - Always append to progress.txt
   - Future instances need the learning

## Integration with Progress System

### Before Handoff

1. Update Codebase Patterns if discoveries are reusable
2. Append progress entry for work done
3. Write handoff.json

### After Resuming

1. Read handoff.json first
2. Read Codebase Patterns section
3. Check if patterns from handoff should be promoted
4. Delete handoff.json when story completes
