# Complexity Analysis Guide

This document details the complexity scoring system used to determine appropriate story counts.

## Scoring Formula

```
Complexity Score = (
  functional_requirements × 2 +
  integration_points × 3 +
  ui_components × 1.5 +
  database_changes × 2 +
  external_apis × 3 +
  authentication_features × 4 +
  file_operations × 1.5 +
  real_time_features × 3
) / 5
```

## Factor Definitions

### Functional Requirements (Weight: 2)
Core features that define what the system does. Look for:
- "The system shall..."
- "Users can..."
- "Must allow..."
- FR-numbered requirements

### Integration Points (Weight: 3)
Connections to other systems or services. Look for:
- "Integrate with..."
- "Connect to..."
- "Third-party service"
- Webhooks, callbacks

### UI Components (Weight: 1.5)
Visual elements users interact with. Look for:
- Buttons, forms, modals
- Pages, screens, views
- Tables, lists, cards
- Interactive elements

### Database Changes (Weight: 2)
Data model modifications. Look for:
- New tables/columns
- Migrations
- Schema changes
- Indexes, constraints

### External APIs (Weight: 3)
API interactions. Look for:
- REST/GraphQL endpoints
- API authentication
- Request/response handling
- Rate limiting concerns

### Authentication Features (Weight: 4)
Security and access control. Look for:
- Login/logout flows
- Password handling
- Session management
- Permissions, roles

### File Operations (Weight: 1.5)
File handling features. Look for:
- Upload/download
- File storage
- Media processing
- Document handling

### Real-time Features (Weight: 3)
Live updates and notifications. Look for:
- WebSockets
- Push notifications
- Live sync
- Streaming data

## Story Count Matrix

| Score Range | Category   | Stories | Iterations | Typical Features |
|-------------|------------|---------|------------|------------------|
| 1-5         | Simple     | 3-5     | 3-5        | Single CRUD, basic UI |
| 6-15        | Medium     | 6-12    | 6-15       | Multi-page app, some integrations |
| 16-30       | Complex    | 13-25   | 15-30      | Full features, auth, real-time |
| 31+         | Enterprise | 26-50   | 30-60      | Platform-level, multi-system |

## Story Sizing Rules

### Maximum Story Size
Each story must fit within ONE context window (~100k tokens total, ~50k for active work).

### Splitting Guidelines

**Too Large → Split By Layer:**
1. Schema/database changes
2. Backend logic/API
3. UI components
4. Integration/wiring

**Too Large → Split By Feature:**
1. Core functionality
2. Secondary features
3. Edge cases
4. Polish/UX

### Size Indicators

**Right-Sized Story:**
- Describable in 2-3 sentences
- Changes 1-5 files
- Single logical unit
- Clear completion criteria

**Too Large Story:**
- Requires multiple paragraphs
- Touches 10+ files
- Multiple features bundled
- Vague completion criteria

## Examples

### Simple (Score: 4)
```
PRD: Add a "mark as favorite" button to items
- 1 UI component
- 1 database column
- 1 API endpoint

Stories:
1. Add favorites column to items table
2. Add favorite toggle button to item card
3. Add favorites filter to list view
```

### Medium (Score: 12)
```
PRD: Add task priority system with filtering
- 3 UI components
- 2 database changes
- 2 API endpoints
- URL state management

Stories:
1. Add priority field to database
2. Display priority badge on task cards
3. Add priority selector to edit modal
4. Implement priority filter dropdown
5. Add sort by priority option
6. Add bulk priority update action
```

### Complex (Score: 24)
```
PRD: Add user authentication with OAuth
- Login/logout UI
- OAuth integration (3 providers)
- Session management
- Role-based access
- Password reset flow

Stories:
1. Add users table and schema
2. Create login page UI
3. Implement Google OAuth
4. Implement GitHub OAuth
5. Implement email/password auth
6. Add session middleware
7. Create protected route wrapper
8. Implement logout flow
9. Add password reset request
10. Add password reset confirmation
11. Create user settings page
12. Add role-based permissions
13. Create admin role assignment
```

## Iteration Estimation

Stories don't complete 1:1 with iterations because:
- Some stories require handoffs
- Quality checks may fail
- Dependencies may block progress

**Iteration multiplier:** 1.0 - 1.5× story count

Simple projects: ~1.0× (linear completion)
Complex projects: ~1.5× (more handoffs, retries)
