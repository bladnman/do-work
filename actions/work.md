# Work Action

> **Part of the do-work skill.** Invoked when routing determines the user wants to process the queue. Processes requests from the `do-work/` folder in your project.

An orchestrated build system that processes request files created by the do action. Uses complexity triage to avoid unnecessary overhead - simple requests go straight to implementation, complex ones get planning and exploration first.

## Architecture

```
work action (orchestrator - lightweight, stays in loop)
  │
  ├── For each pending request:
  │     │
  │     ├── TRIAGE: Assess complexity (no agent, just read & categorize)
  │     │     │
  │     │     ├── Route A (Simple) ──────────────────┐
  │     │     │   Skip plan/explore, direct to build │
  │     │     │                                      │
  │     │     ├── Route B (Medium) ───────┐          │
  │     │     │   Explore only, then build│          │
  │     │     │                           ▼          │
  │     │     └── Route C (Complex) ──► Plan ──► Explore
  │     │                                            │
  │     │                                            ▼
  │     └─────────────────────────────────────► general-purpose agent
  │                                             (executes implementation)
  │
  └── Loop continues until queue empty
```

The orchestrator stays lightweight - it triages requests and only spawns the agents actually needed.

## Complexity Triage

Before spawning any agents, quickly assess the request to determine the right route.

### Route A: Direct to Builder (Simple)

Skip planning and exploration entirely. The builder agent has these tools available if needed.

**Indicators:**
- Bug fix with clear reproduction steps or error message
- Value, property, or config change
- Adding/removing a single UI element
- Styling or visual tweaks
- Request explicitly names specific files to modify
- Well-specified with obvious scope (under ~50 words, clear outcome)
- Copy/text changes
- Toggling a feature flag or setting

**Examples:**
- "Change the button color from blue to green"
- "Fix the crash when clicking submit with empty form"
- "Add a tooltip to the save button"
- "Update the API timeout from 30s to 60s"
- "Remove the deprecated banner from the header"

### Route B: Explore then Build (Medium)

Skip planning but run exploration. The "what" is clear, but we need to find the "where" or existing patterns.

**Indicators:**
- Clear outcome but unspecified location
- Request mentions "like the existing X" or "match the pattern"
- Need to find similar implementations to follow
- Modifying something that exists but location unknown
- Adding something that should match existing conventions

**Examples:**
- "Add form validation like we have on the login page"
- "Create a new API endpoint following our existing patterns"
- "Add the same loading state we use elsewhere"
- "Fix the styling to match the rest of the settings panel"

### Route C: Full Pipeline (Complex)

Run planning, then exploration, then implementation.

**Indicators:**
- New feature requiring multiple components
- Architectural changes or new patterns
- Ambiguous scope ("improve", "refactor", "optimize")
- Touches multiple systems or layers
- Integration with external services
- Request is long (100+ words) with many requirements
- Introduces new concepts to the codebase
- User explicitly asked for a plan or design

**Examples:**
- "Add user authentication with OAuth"
- "Implement dark mode across the app"
- "Refactor the state management to use Zustand"
- "Add real-time sync between devices"
- "Improve the search performance"

### Triage Decision Flow

```
Read the request file
     │
     ├── Does it name specific files AND have clear changes?
     │     └── Yes → Route A (Direct)
     │
     ├── Is it a bug fix with clear reproduction?
     │     └── Yes → Route A (Direct)
     │
     ├── Is it a simple value/config/copy change?
     │     └── Yes → Route A (Direct)
     │
     ├── Is the outcome clear but location/pattern unknown?
     │     └── Yes → Route B (Explore only)
     │
     ├── Is it ambiguous, multi-system, or architectural?
     │     └── Yes → Route C (Full pipeline)
     │
     └── Default: Route B (Explore only)
         Builder can request planning if needed
```

**When uncertain, prefer Route B.** The builder agent can spawn a Plan agent itself if it determines one is needed. Under-planning is recoverable; over-planning is wasted time.

## Folder Structure

```
do-work/
├── REQ-001-pending-task.md      # Pending (root = queue)
├── REQ-002-another-task.md
├── assets/                       # Screenshots, attachments
│   └── REQ-001-context-1.png
├── working/                      # Currently being processed
│   └── REQ-003-in-progress.md
└── archive/                      # Completed work
    └── REQ-000-finished-task.md
```

- **Root `do-work/`**: The queue - pending requests live here (created by the do action)
- **`working/`**: File moves here when claimed, prevents double-processing
- **`archive/`**: Completed (or failed) requests with implementation notes
- **`assets/`**: Screenshots and attachments (referenced by requests)

## Request File Schema

Request files use YAML frontmatter for tracking. Fields are added progressively by the do and work actions.

```yaml
---
# Set by do action when created
id: REQ-001
title: Short descriptive title
status: pending
created_at: 2025-01-26T10:00:00Z

# Set by work action when claimed
claimed_at: 2025-01-26T10:30:00Z
route: A | B | C

# Set by work action when finished
completed_at: 2025-01-26T10:45:00Z
status: completed | failed
commit: abc1234  # Git commit hash (if git repo)
error: "Description of failure"  # Only if failed
---
```

### Field Reference

| Field | Set By | When | Description |
|-------|--------|------|-------------|
| `id` | do | Creation | Request identifier (REQ-NNN) |
| `title` | do | Creation | Short title for the request |
| `status` | Both | Throughout | Current state (see flow below) |
| `created_at` | do | Creation | ISO timestamp when request was captured |
| `claimed_at` | work | Claim | ISO timestamp when work began |
| `route` | work | Triage | Complexity route (A/B/C) |
| `completed_at` | work | Completion | ISO timestamp when work finished |
| `commit` | work | Commit | Git commit hash (omitted if not a git repo) |
| `error` | work | Failure | Error description if status is `failed` |

### Status Flow

```
pending (in do-work/, set by do action)
    → claimed (moved to working/, set by work action)
    → [planning] → [exploring] → implementing
    → completed (moved to archive/)
    ↘ failed (moved to archive/ with error)
```

Brackets indicate optional phases based on route.

## Workflow

### Step 1: Find Next Request

1. **List** (don't read) `REQ-*.md` filenames in `do-work/` folder
2. Sort by filename (REQ-001 before REQ-002)
3. Pick the first one

**Important**: Do NOT read the contents of all request files. Only list filenames to find the next one. You'll read the chosen request in Step 3.

If no request files found, report completion and exit.

### Step 2: Claim the Request

1. Create `do-work/working/` folder if it doesn't exist
2. Move the request file from `do-work/` to `do-work/working/`
3. Update the frontmatter:

```yaml
---
status: claimed
claimed_at: 2025-01-26T10:30:00Z
---
```

### Step 3: Triage

Read the request content and apply the triage decision flow. Update frontmatter with the chosen route:

```yaml
---
status: claimed
claimed_at: 2025-01-26T10:30:00Z
route: B
---
```

**Write the triage decision into the request file** (append after the original content):

```markdown
---

## Triage

**Route: [A/B/C]** - [Simple/Medium/Complex]

**Reasoning:** [1-2 sentences explaining why this route was chosen]

Examples:
- "Route A - Bug fix with clear reproduction steps and specific file mentioned"
- "Route B - Clear feature request but need to find existing patterns to follow"
- "Route C - New feature spanning multiple systems, requires architectural planning"
```

This creates a record for retrospective analysis - you can later review whether triage decisions were appropriate.

Report the triage decision briefly to the user:
- Route A: "Simple request, going direct to implementation"
- Route B: "Clear request, exploring codebase first"
- Route C: "Complex request, starting with planning"

### Step 4: Planning Phase (Route C only)

Spawn a **Plan agent** with this prompt structure:

```
You are planning the implementation for this request:

---
[Full content of the request file]
---

Project context:
- This is a [describe project type based on package.json/Cargo.toml]
- Key directories: [list from exploring project structure]

Create a detailed implementation plan that includes:
1. What files need to be created or modified
2. The order of changes (dependencies between steps)
3. Any architectural decisions needed
4. Testing approach

Be specific about file paths and function names where possible.
```

**After Plan agent returns:**
- Update request status to `exploring`
- Store the plan output for the next phase
- **Write the plan into the request file** (append after Triage section):

```markdown
## Plan

[Full output from the Plan agent]

*Generated by Plan agent*
```

This preserves the planning work for retrospective review - you can always see what the plan said was needed, compare it to what was actually done, and evaluate planning quality over time.

### Step 5: Exploration Phase (Routes B and C)

Spawn an **Explore agent** with this prompt:

**For Route C (has plan):**
```
Based on this implementation plan:

---
[Plan from previous phase]
---

For this request:
[Brief summary of the request]

Find and read the relevant files that will need to be modified or that contain patterns we should follow. Focus on:
1. Files explicitly mentioned in the plan
2. Similar existing implementations we should match
3. Type definitions and interfaces we'll need
4. Test files that show testing patterns

Return a summary of what you found, including:
- Key code patterns to follow
- Existing types/interfaces to use
- Any concerns or blockers discovered
```

**For Route B (no plan):**
```
For this request:

---
[Full content of request file]
---

Find the relevant files and patterns needed to implement this. Focus on:
1. Where this change should be made
2. Similar existing implementations to follow
3. Related types/interfaces
4. Testing patterns if applicable

Return a summary of what you found with specific file paths and code patterns.
```

**After Explore agent returns:**
- Update request status to `implementing`
- Store the exploration output for the next phase
- **Write exploration findings into the request file** (append after Plan section if present, or after Triage):

```markdown
## Exploration

[Summary output from the Explore agent - key files, patterns found, concerns]

*Generated by Explore agent*
```

### Step 6: Implementation Phase (All routes)

Spawn a **general-purpose agent** with context appropriate to the route:

**Route A (direct):**
```
You are implementing this request:

## Request
[Full content of request file]

## Instructions

Implement this change. You have full access to Edit, Write, Bash, and can spawn Explore or Plan agents if you discover you need more context.

Key guidelines:
- This was triaged as a simple request - aim for a focused, minimal change
- If you find the request is more complex than expected, you can use Plan or Explore agents
- Run tests after making changes if the project has tests
- If you encounter blockers, document them clearly

When complete, provide a summary of what was changed.
```

**Route B (explored):**
```
You are implementing this request:

## Request
[Full content of request file]

## Codebase Context
[Output from Explore agent]

## Instructions

Implement the changes using the patterns and locations identified above. You have full access to Edit, Write, and Bash tools.

Key guidelines:
- Follow existing code patterns identified in the codebase context
- Make minimal, focused changes
- Run tests after making changes if the project has tests
- If you encounter blockers, document them clearly

When complete, provide a summary of what was changed.
```

**Route C (planned + explored):**
```
You are implementing this request:

## Original Request
[Full content of request file]

## Implementation Plan
[Output from Plan agent]

## Codebase Context
[Output from Explore agent]

## Instructions

Implement the changes according to the plan. You have full access to Edit, Write, and Bash tools.

Key guidelines:
- Follow existing code patterns identified in the codebase context
- Make minimal, focused changes
- Run tests after making changes if the project has tests
- If you encounter blockers, document them clearly

When complete, provide a summary of:
1. What files were changed/created
2. Any deviations from the plan and why
3. Test results if applicable
4. Any follow-up items needed
```

**After implementation agent returns:**
- Capture the summary output
- Update request status to `completed` or `failed`

### Step 7: Archive and Continue

**On success:**

1. Update request file frontmatter:
```yaml
---
status: completed
claimed_at: 2025-01-26T10:30:00Z
completed_at: 2025-01-26T10:45:00Z
route: B
---
```

2. Append implementation summary to the request file:
```markdown
## Implementation Summary

[Summary from the implementation agent]

*Completed by work action (Route [A/B/C])*
```

**Complete request file structure after archival:**

```markdown
---
id: REQ-007
title: Add user avatar component
status: completed
created_at: 2025-01-26T09:30:00Z
claimed_at: 2025-01-26T11:00:00Z
route: B
completed_at: 2025-01-26T11:08:00Z
commit: a1b2c3d
---

# Add User Avatar Component

## What
[Original request]

## Context
[Original context]

---

## Triage

**Route: B** - Medium

**Reasoning:** Clear feature but need to find existing component patterns.

## Exploration

- Found similar component at src/components/ExistingFeature.tsx
- Uses pattern X for state management
- Tests in tests/existing-feature.spec.ts

*Generated by Explore agent*

## Implementation Summary

- Created src/components/NewFeature.tsx
- Added tests in tests/new-feature.spec.ts
- All tests passing

*Completed by work action (Route B)*
```

For Route C requests, the Plan section appears between Triage and Exploration. For Route A, only Triage and Implementation Summary appear.

**Timestamps tell the story:**
- `created_at` → `claimed_at`: How long request sat in queue
- `claimed_at` → `completed_at`: How long implementation took
- Route + timestamps: Compare complexity vs actual time spent

3. Create `do-work/archive/` if it doesn't exist
4. Move file from `do-work/working/` to `do-work/archive/`
5. **If this REQ has a `context_ref`**: Check if all related REQs are now archived. If so, archive the CONTEXT file too.

**On failure:**

1. Update frontmatter with error:
```yaml
---
status: failed
claimed_at: 2025-01-26T10:30:00Z
route: B
error: "Brief description of what went wrong"
---
```

2. Move file to `do-work/archive/` (failed items are also archived, status tells the story)
3. Report the failure to the user

### Step 8: Commit Changes (Git repos only)

If the project is Git-backed, create a **single commit** containing all changes made for this request. This provides a clean rollback/cherry-pick surface per request.

**Check for Git:**
```bash
git rev-parse --git-dir 2>/dev/null
```

If this fails (not a Git repo), skip this step entirely.

**Stage and commit (single operation):**
```bash
# Stage ALL current changes - code, request file, assets
git add -A

# Single commit with structured message
git commit -m "$(cat <<'EOF'
[REQ-003] Dark Mode (Route C)

Implements: do-work/archive/REQ-003-dark-mode.md

- Created src/stores/theme-store.ts
- Modified src/components/settings/SettingsPanel.tsx
- Updated tailwind.config.js

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

**Commit message format:**
```
[{id}] {title} (Route {route})

Implements: do-work/archive/{filename}

{implementation summary as bullet points}

Co-Authored-By: Claude <noreply@anthropic.com>
```

**Important commit rules:**
- **ONE commit per request** - do not analyze files individually or create multiple commits
- **Stage everything** with `git add -A` - includes code changes, archived request file, and any assets
- **No pre-commit hook bypass** - if hooks fail, fix the issue and retry
- **Failed requests get committed too** - the archived request with `status: failed` documents what was attempted

**If commit fails:**
- Report the error to the user
- Do NOT retry with different commit strategies
- Continue to next request (changes remain uncommitted but archived)

### Step 9: Loop or Exit

After archiving and committing:

1. **Re-check** root `do-work/` folder for `REQ-*.md` files (fresh check, not cached list)
2. If found: Report what was completed, then start Step 1 again
3. If empty: Report final summary and exit

This fresh check on each loop means newly added requests get picked up automatically.

## Progress Reporting

Keep the user informed with brief updates:

```
Processing REQ-003-dark-mode.md...
  Triage: Complex (Route C)
  Planning...     [done]
  Exploring...    [done]
  Implementing... [done]
  Archiving...    [done]
  Committing...   [done] → abc1234

Processing REQ-004-fix-typo.md...
  Triage: Simple (Route A)
  Implementing... [done]
  Archiving...    [done]
  Committing...   [done] → def5678

Found 1 more pending request. Continuing...

Processing REQ-005-add-tooltip.md...
  Triage: Simple (Route A)
  Implementing... [done]
  Archiving...    [done]
  Committing...   [done] → ghi9012

All 3 requests completed:
  - REQ-003 (Route C) → abc1234
  - REQ-004 (Route A) → def5678
  - REQ-005 (Route A) → ghi9012
```

For non-Git projects, the commit step is skipped:
```
Processing REQ-003-dark-mode.md...
  Triage: Complex (Route C)
  Planning...     [done]
  Exploring...    [done]
  Implementing... [done]
  Archiving...    [done]
  (not a git repo, skipping commit)

Completed.
```

## Error Handling

### Plan agent fails (Route C)
- Mark request as `failed` with error
- Continue to next request (don't block the queue)

### Explore agent fails (Routes B, C)
- Proceed to implementation anyway with reduced context
- Note the limitation in the implementation prompt
- Builder can explore on its own if needed

### Implementation agent fails
- Mark request as `failed`
- Preserve any plan and exploration outputs in the request file for retry

### Commit fails
- Report the error to the user (usually pre-commit hook failure)
- Do NOT retry with `--no-verify` or alternative strategies
- Continue to next request - changes remain uncommitted but are archived
- User can manually fix and commit later

### Unrecoverable error
- Stop the loop
- Report clearly what happened
- Leave queue state intact for manual recovery

## Commands

### `do work`
Process all pending requests in order.

### `do work REQ-005` (future enhancement)
Process a specific request by number, regardless of status.

### `do work --dry-run` (future enhancement)
Show what would be processed and their triage routes without making changes.

## What This Action Does NOT Do

- Create new request files (use the do action for that)
- Make architectural decisions beyond what's in the request
- Run without user being present (this is supervised automation)
- Modify requests that are already `completed` or `in_progress` by another agent
- Override triage decisions mid-flight (complete the request, then retry if needed)

## Example Session

```
User: do work

Claude: Checking do-work/ folder...
Found 3 pending requests. Starting with REQ-003-dark-mode.md...
Triage: Complex (Route C) - new feature across multiple components

[Spawns Plan agent]
Planning complete. Key steps:
  1. Add theme context provider
  2. Create useTheme hook
  3. Add toggle to settings
  4. Update Tailwind config

[Spawns Explore agent]
Found relevant patterns:
  - Existing context in src/stores/
  - Settings panel at src/components/settings/
  - Tailwind config at tailwind.config.js

[Spawns implementation agent]
Implementation complete:
  - Created src/stores/theme-store.ts
  - Modified src/components/settings/SettingsPanel.tsx
  - Updated tailwind.config.js
  - Tests passing

Archived REQ-003-dark-mode.md
Committed → abc1234

Continuing with REQ-004-fix-submit-crash.md...
Triage: Simple (Route A) - bug fix with clear reproduction

[Spawns implementation agent]
Implementation complete:
  - Fixed null check in src/components/Form.tsx:42
  - Added test case

Archived REQ-004-fix-submit-crash.md
Committed → def5678

Continuing with REQ-005-change-timeout.md...
Triage: Simple (Route A) - config value change

[Spawns implementation agent]
Implementation complete:
  - Updated API_TIMEOUT in src/config.ts from 30000 to 60000

Archived REQ-005-change-timeout.md
Committed → ghi9012

All 3 requests completed:
  - REQ-003: Route C (plan + explore + build) → abc1234
  - REQ-004: Route A (direct build) → def5678
  - REQ-005: Route A (direct build) → ghi9012
```

## Retrospective Value

After running the work action, archived request files contain their full history:

**REQ-003-dark-mode.md (Route C):**
```markdown
---
id: REQ-003
title: Dark Mode
status: completed
created_at: 2025-01-26T09:00:00Z
claimed_at: 2025-01-26T10:30:00Z
route: C
completed_at: 2025-01-26T10:52:00Z
commit: abc1234
---

# Dark Mode

## What
Implement dark mode across the app.

---

## Triage

**Route: C** - Complex

**Reasoning:** New feature requiring theme system, multiple component updates, and Tailwind configuration changes.

## Plan

1. Create theme store with light/dark state
2. Add useTheme hook for components
3. Update Tailwind config for dark variants
4. Add toggle to settings panel
5. Update key components to use theme-aware classes

*Generated by Plan agent*

## Exploration

- Existing stores in src/stores/ use Zustand pattern
- Settings panel at src/components/settings/SettingsPanel.tsx
- Tailwind config supports darkMode: 'class'

*Generated by Explore agent*

## Implementation Summary

- Created src/stores/theme-store.ts
- Modified 4 components for dark mode support
- Updated tailwind.config.js
- All tests passing

*Completed by work action (Route C)*
```

**REQ-005-change-timeout.md (Route A):**
```markdown
---
id: REQ-005
title: Change API Timeout
status: completed
created_at: 2025-01-26T09:15:00Z
claimed_at: 2025-01-26T10:55:00Z
route: A
completed_at: 2025-01-26T10:56:00Z
commit: ghi9012
---

# Change API Timeout

## What
Update the API timeout from 30s to 60s.

---

## Triage

**Route: A** - Simple

**Reasoning:** Single config value change, file explicitly mentioned in request.

## Implementation Summary

- Updated API_TIMEOUT in src/config.ts from 30000 to 60000

*Completed by work action (Route A)*
```

This lets you:
- Review what planning recommended vs what was actually done
- Identify requests where triage was wrong (simple request that needed planning)
- Track patterns in request complexity over time
- Debug failed requests by seeing what context was gathered
- Analyze throughput: timestamps show queue wait time and implementation time
- Calibrate triage: compare route assignment to actual time spent (Route A should be fast)

**Git integration benefits (when applicable):**
- **Rollback**: `git revert <commit>` undoes one complete request
- **Cherry-pick**: `git cherry-pick <commit>` pulls a specific fix to another branch
- **Bisect**: Find which request introduced a bug
- **Blame**: Commit message links to full request documentation
