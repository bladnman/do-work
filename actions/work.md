# Work Action

> **Part of the do-work skill.** Invoked when routing determines the user wants to process the queue. Processes requests from the `do-work/` folder in your project.

An orchestrated build system that processes request files created by the do action. Uses complexity triage to avoid unnecessary overhead - simple requests go straight to implementation, complex ones get planning and exploration first.

## Request Files as Living Logs

**Each request file becomes a complete historical record of the work performed.** As you process a request, you append sections documenting each phase:

1. **Triage** - Route decision and reasoning
2. **Plan** - Either the full plan (Route C) or "Planning not required" (Routes A/B)
3. **Exploration** - Codebase findings (Routes B/C)
4. **Implementation Summary** - What was changed
5. **Testing** - Test results and coverage

This traceability ensures:
- You can review what was planned vs what was done
- Failed requests show exactly where things went wrong
- Triage accuracy can be evaluated over time
- The full context is preserved for future reference

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
├── REQ-018-pending-task.md       # Pending (root = queue)
├── REQ-019-another-task.md
├── user-requests/                 # User Request folders (verbatim input + assets)
│   ├── UR-003/
│   │   ├── input.md              # Verbatim original input
│   │   └── assets/
│   │       └── REQ-017-screenshot.png
│   └── UR-004/
│       └── input.md
├── assets/                        # Legacy assets (pre-UR system)
│   └── CONTEXT-002-old-batch.md   # Legacy context docs
├── working/                       # Currently being processed
│   └── REQ-020-in-progress.md
└── archive/                       # Completed work
    ├── UR-001/                    # Archived as self-contained unit
    │   ├── input.md
    │   ├── REQ-013-feature.md
    │   └── assets/
    │       └── screenshot.png
    ├── UR-002/
    │   ├── input.md
    │   ├── REQ-014-feature.md
    │   └── REQ-015-feature.md
    ├── REQ-010-legacy-task.md     # Legacy REQs (no UR) archive directly
    └── CONTEXT-001-auth-system.md # Legacy context docs archive directly
```

- **Root `do-work/`**: The queue - ONLY pending `REQ-*.md` files live here
- **`user-requests/`**: UR folders with verbatim input and assets per user request
- **`working/`**: File moves here when claimed, prevents double-processing
- **`archive/`**: Completed UR folders (self-contained units) AND legacy REQs/CONTEXT docs
- **`assets/`**: Legacy screenshots and context documents (pre-UR system)

**Immutability rule:** Files in `working/` and `archive/` are locked. Only the work action's own processing pipeline may modify files in `working/` (updating frontmatter, appending workflow sections). The do action and all other external processes must never reach into these folders. If a change request comes in for something already claimed or completed, the do action creates a new addendum REQ in the queue — it never touches the original. See the do action docs for the `addendum_to` field.

**User Request (UR) lifecycle:**
- Created in `do-work/user-requests/` by the do action
- Referenced by REQ files via `user_request: UR-NNN` frontmatter
- When ALL related REQs are complete: UR folder moves from `user-requests/` to `archive/`, with completed REQ files pulled into the UR folder

**Backward compatibility:**
- REQs without `user_request` field (legacy) archive directly to `do-work/archive/` as before
- REQs with `context_ref` (legacy) trigger CONTEXT doc archival when all related REQs complete
- The `do-work/assets/` folder is only used for legacy items — new assets go into UR folders

## Request File Schema

Request files use YAML frontmatter for tracking. Fields are added progressively by the do and work actions.

```yaml
---
# Set by do action when created
id: REQ-001
title: Short descriptive title
status: pending
created_at: 2025-01-26T10:00:00Z
user_request: UR-001  # Links to originating user request (may be absent on legacy REQs)

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
| `user_request` | do | Creation | UR identifier linking to originating user request (absent on legacy REQs) |
| `claimed_at` | work | Claim | ISO timestamp when work began |
| `route` | work | Triage | Complexity route (A/B/C) |
| `completed_at` | work | Completion | ISO timestamp when work finished |
| `commit` | work | Commit | Git commit hash (omitted if not a git repo) |
| `error` | work | Failure | Error description if status is `failed` |

### Status Flow

```
pending (in do-work/, set by do action)
    → claimed (moved to working/, set by work action)
    → [planning] → [exploring] → implementing → testing
    → completed (moved to archive/)
    ↘ failed (moved to archive/ with error)
```

Brackets indicate optional phases based on route.

## Workflow

**CRITICAL: Orchestrator Responsibilities**

The work action is an **orchestrator**. You (the orchestrator) are responsible for ALL file management operations. Spawned agents do implementation work but do NOT touch request files or folder structure.

**You must do these yourself (not delegate to agents):**
- Move request file to `working/` folder (Step 2)
- Update frontmatter status fields (Steps 2, 3, 7)
- Write triage/plan/exploration/testing sections to the request file
- Move request file to `archive/` folder (Step 7)
- Create the git commit (Step 8)

**Agents do:**
- Planning (Route C)
- Exploring (Routes B, C)
- Implementation (all routes)
- Writing/running tests

---

### Step 1: Find Next Request

**[Orchestrator action - do this yourself]**

1. **List** (don't read) `REQ-*.md` filenames in `do-work/` folder
2. Sort by filename (REQ-001 before REQ-002)
3. Pick the first one

**Important**: Do NOT read the contents of all request files. Only list filenames to find the next one. You'll read the chosen request in Step 3.

If no request files found, report completion and exit.

### Step 2: Claim the Request

**[Orchestrator action - do this yourself, BEFORE spawning any agents]**

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

**[Orchestrator action - do this yourself]**

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

**Planning:** [Required/Not required]
```

Examples:
- Route A: "Route A - Simple. Bug fix with clear reproduction steps. Planning not required - direct implementation."
- Route B: "Route B - Medium. Clear feature but need to find existing patterns. Planning not required - exploration then implementation."
- Route C: "Route C - Complex. New feature spanning multiple systems. Planning required."

This creates a record for retrospective analysis - you can later review whether triage decisions were appropriate.

**The request file is a living log** - every phase of work gets documented in it so there's full traceability of what happened.

Report the triage decision briefly to the user:
- Route A: "Simple request, going direct to implementation (no planning needed)"
- Route B: "Clear request, exploring codebase first (no planning needed)"
- Route C: "Complex request, starting with planning"

### Step 4: Planning Phase (Route C only, or document skip for Routes A/B)

**[Spawn agent for Route C, or document skip for Routes A/B]**

**For Route C - Full Planning:**

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

**For Routes A and B - Document that planning was skipped:**

Even when planning is not performed, document this in the request file for traceability:

```markdown
## Plan

**Planning not required** - [Route A: Direct implementation / Route B: Exploration-guided implementation]

Rationale: [Brief explanation, e.g., "Simple bug fix with clear scope" or "Clear feature, just need to find existing patterns"]

*Skipped by work action*
```

This preserves the full workflow history - you can always see what the plan said was needed (or why planning was skipped), compare it to what was actually done, and evaluate planning quality over time.

### Step 5: Exploration Phase (Routes B and C)

**[Spawn agent, then orchestrator writes results to file]**

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

**[Spawn agent - agent does the actual code changes]**

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
- If you encounter blockers, document them clearly

Testing requirements:
- If the project has tests, identify tests related to your changes
- Write new tests for new functionality or bug fixes (regression tests)
- Update existing tests if behavior intentionally changed
- Note what tests exist and what new tests may be needed

When complete, provide a summary of:
- What was changed
- What tests exist for this code
- What new tests should be written (if any)
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
- If you encounter blockers, document them clearly

Testing requirements:
- If the project has tests, identify tests related to your changes
- Write new tests for new functionality, following patterns from the codebase context
- For bug fixes, add regression tests that would have caught the bug
- Update existing tests if behavior intentionally changed

When complete, provide a summary of:
- What was changed
- What tests exist for this code
- What new tests were written or need to be written
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
- If you encounter blockers, document them clearly

Testing requirements:
- Identify existing tests related to the changes
- Write new tests for new functionality, following patterns from the codebase context
- Include tests as part of the implementation, not as an afterthought
- For each new component/function/endpoint, include corresponding tests
- Update existing tests if behavior intentionally changed

When complete, provide a summary of:
1. What files were changed/created
2. Any deviations from the plan and why
3. What tests exist and what new tests were written
4. Any follow-up items needed
```

**After implementation agent returns:**
- Capture the summary output
- Update request status to `testing`
- Proceed to testing phase

### Step 6.5: Testing Phase (All routes)

**[Orchestrator runs tests and may spawn agent for new tests]**

Before marking work as complete, verify that tests pass and appropriate test coverage exists.

**1. Detect testing infrastructure:**

Look for common test configurations:
- `package.json` scripts containing "test"
- `jest.config.*`, `vitest.config.*`, `playwright.config.*`
- `pytest.ini`, `pyproject.toml` with pytest section
- `Cargo.toml` (Rust has built-in testing)
- `*_test.go` files (Go has built-in testing)
- `*.spec.*`, `*.test.*` files in the codebase

If no testing infrastructure is detected, skip this phase and proceed to archiving. Note in the implementation summary: "No testing infrastructure detected."

**2. Identify relevant tests:**

Based on what files were modified during implementation:
- Find existing test files that cover the modified code
- Check if the change warrants new tests

**Tests are warranted when:**
- New functionality was added (new functions, components, endpoints)
- Bug fixes that should have regression tests
- Behavioral changes that could break existing functionality
- New edge cases or error handling paths

**Tests may not be needed for:**
- Documentation changes
- Config value changes (unless config affects behavior significantly)
- Pure refactoring with existing test coverage
- Cosmetic/styling changes

**3. Run existing related tests:**

```bash
# Example: JavaScript/TypeScript with Jest
npm test -- --testPathPattern="relevant-pattern"

# Example: Python with pytest
pytest tests/relevant_test.py -v

# Example: Rust
cargo test relevant_module

# Example: Go
go test ./path/to/package -v
```

Run tests that are relevant to the changed code, not the entire test suite (unless the suite is fast).

**4. If tests fail:**

- Do NOT mark the request as complete
- Return to implementation phase to fix the failing tests
- The implementation agent should:
  - Analyze the test failure
  - Fix the implementation OR fix the test if the test was incorrect
  - Re-run the tests

**Loop until tests pass or it becomes clear the request cannot be completed.**

**5. If new tests are needed:**

Spawn a **general-purpose agent** to write tests:

```
The following changes were made for this request:

## Request
[Brief summary]

## Changes Made
[Files created/modified from implementation summary]

## Task
Write appropriate tests for these changes. Follow the existing testing patterns in the codebase.

Guidelines:
- Match the testing style/framework already in use
- Cover the happy path and key edge cases
- For bug fixes, add a regression test that would have caught the bug
- Keep tests focused and readable
- Place test files according to project conventions

After writing tests, run them to verify they pass.
```

**6. Verify all tests pass:**

Run the full relevant test suite one final time:

```bash
# Run tests and capture exit code
npm test  # or pytest, cargo test, go test, etc.
```

**If tests pass:** Update status and proceed to archiving
**If tests fail:** Return to implementation to fix issues

**Write testing results into the request file** (append after Implementation Summary):

```markdown
## Testing

**Tests run:** [command used]
**Result:** ✓ All tests passing (X tests)

**New tests added:**
- tests/new-feature.spec.ts - covers happy path and error cases

**Existing tests verified:**
- tests/related-feature.spec.ts - still passing

*Verified by work action*
```

Or for failures that were resolved:

```markdown
## Testing

**Initial run:** ✗ 2 tests failing
**Issue:** Implementation didn't handle null case
**Fix:** Added null check in handleSubmit()
**Final run:** ✓ All tests passing (X tests)

*Verified by work action*
```

### Step 7: Archive and Continue

**[Orchestrator action - do this yourself, AFTER all agents complete]**

**IMPORTANT: You must perform these file operations yourself. Do not skip them.**

**On success - do ALL of these steps:**

1. **Update the request file frontmatter** (in `do-work/working/`):
```yaml
---
status: completed
claimed_at: 2025-01-26T10:30:00Z
completed_at: 2025-01-26T10:45:00Z
route: B
---
```

2. **Append implementation summary** to the request file (if not already present):
```markdown
## Implementation Summary

[Summary from the implementation agent]

*Completed by work action (Route [A/B/C])*
```

3. **Create archive folder** if it doesn't exist:
```bash
mkdir -p do-work/archive
```

4. **Archive the request file** — behavior depends on whether the REQ has a UR:

**If REQ has `user_request: UR-NNN` (new system):**
   - Read the UR's `input.md` from `do-work/user-requests/UR-NNN/`
   - Check its `requests` array (e.g., `[REQ-018, REQ-019, REQ-020]`)
   - Check if ALL listed REQs are now completed (in `do-work/working/` with `status: completed` or already in `archive/`)
   - If **all REQs complete**:
     - Move the completed REQ file into the UR folder: `mv do-work/working/REQ-XXX.md do-work/user-requests/UR-NNN/`
     - Move any other completed REQs from `do-work/archive/` that belong to this UR into the UR folder
     - Move the entire UR folder to archive: `mv do-work/user-requests/UR-NNN do-work/archive/`
   - If **not all REQs complete**:
     - Move the REQ file directly to archive for now: `mv do-work/working/REQ-XXX.md do-work/archive/`
     - The UR folder stays in `user-requests/` until the last REQ completes, which triggers the full UR archive

**If REQ has `context_ref` (legacy system):**
   - Move REQ to archive: `mv do-work/working/REQ-XXX.md do-work/archive/`
   - Read the CONTEXT document from `do-work/assets/`
   - Check its `requests` array
   - If ALL listed REQs are now in `do-work/archive/`: move the CONTEXT doc to archive too
   - If not: leave CONTEXT doc in `assets/`

**If REQ has neither `user_request` nor `context_ref` (standalone legacy):**
   - Move directly to archive: `mv do-work/working/REQ-XXX.md do-work/archive/`

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

**Planning:** Not required

## Plan

**Planning not required** - Route B: Exploration-guided implementation

Rationale: Clear feature request with well-defined outcome. Just need to discover existing component patterns to match.

*Skipped by work action*

## Exploration

- Found similar component at src/components/ExistingFeature.tsx
- Uses pattern X for state management
- Tests in tests/existing-feature.spec.ts

*Generated by Explore agent*

## Implementation Summary

- Created src/components/NewFeature.tsx
- Added tests in tests/new-feature.spec.ts

*Completed by work action (Route B)*

## Testing

**Tests run:** npm test -- --testPathPattern="new-feature"
**Result:** ✓ All tests passing (4 tests)

**New tests added:**
- tests/new-feature.spec.ts - covers rendering, click handler, edge cases

*Verified by work action*
```

**All routes have a Plan section** - for Route C it contains the actual plan from the Plan agent, for Routes A and B it documents that planning was skipped and why. This ensures full traceability of the workflow decision.

**Timestamps tell the story:**
- `created_at` → `claimed_at`: How long request sat in queue
- `claimed_at` → `completed_at`: How long implementation took
- Route + timestamps: Compare complexity vs actual time spent

**On failure - do ALL of these steps:**

1. **Update frontmatter with error** (in `do-work/working/`):
```yaml
---
status: failed
claimed_at: 2025-01-26T10:30:00Z
route: B
error: "Brief description of what went wrong"
---
```

2. **Create archive folder** if it doesn't exist:
```bash
mkdir -p do-work/archive
```

3. **Move the request file to archive** (failed items are also archived, status tells the story):
```bash
mv do-work/working/REQ-XXX-slug.md do-work/archive/
```
Note: Failed REQs always go directly to `do-work/archive/`, NOT into UR folders. A failed REQ does not count as "complete" for UR archival purposes — the UR folder stays in `user-requests/` until all REQs succeed or the user explicitly archives.

4. **Report the failure to the user**

### Step 8: Commit Changes (Git repos only)

**[Orchestrator action - do this yourself]**

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

**[Orchestrator action - do this yourself]**

After archiving and committing:

1. **Re-check** root `do-work/` folder for `REQ-*.md` files (fresh check, not cached list)
2. If found: Report what was completed, then start Step 1 again
3. If empty: **Run the cleanup action** (see [cleanup action](./cleanup.md)), then report final summary and exit

The cleanup action consolidates the archive — closing any UR folders where all REQs are now complete, moving loose REQ files into their UR folders, and organizing legacy files. This catches any consolidation that was missed during individual request archival.

This fresh check on each loop means newly added requests get picked up automatically.

---

### Orchestrator Checklist (per request)

Use this checklist to ensure you don't skip critical steps:

```
□ Step 1: List REQ-*.md files in do-work/, pick first one
□ Step 2: mkdir -p do-work/working && mv do-work/REQ-XXX.md do-work/working/
□ Step 2: Update frontmatter: status: claimed, claimed_at: <timestamp>
□ Step 3: Read request, decide route (A/B/C), update frontmatter with route
□ Step 3: Append ## Triage section to request file (including Planning status)
□ Step 4: Append ## Plan section (Route C: from Plan agent / Routes A,B: "Planning not required")
□ Step 5: (Routes B,C) Spawn Explore agent, append ## Exploration section
□ Step 6: Spawn implementation agent
□ Step 6.5: Run tests, append ## Testing section
□ Step 7: Update frontmatter: status: completed, completed_at: <timestamp>
□ Step 7: Append ## Implementation Summary section
□ Step 7: Archive REQ (see UR vs legacy archival logic)
□ Step 7: If user_request exists → check if all UR's REQs complete → move UR folder to archive/
□ Step 7: If context_ref exists (legacy) → check if all related REQs archived → move CONTEXT to archive/
□ Step 7: If neither → mv do-work/working/REQ-XXX.md do-work/archive/
□ Step 8: git add -A && git commit (if git repo)
□ Step 9: Check for more requests, loop or exit
□ Step 9: If exiting: Run cleanup action (close completed URs, consolidate loose REQs)
```

**Common mistakes to avoid:**
- Spawning implementation agent without first moving file to `working/`
- Completing implementation without moving file to `archive/`
- Forgetting to update status in frontmatter
- Letting agents handle file management (they shouldn't)
- Forgetting to document planning status for Routes A/B (write "Planning not required")
- Forgetting to check/archive related UR folders or legacy context documents
- Archiving a UR folder before all its REQs are complete

---

## Progress Reporting

Keep the user informed with brief updates:

```
Processing REQ-003-dark-mode.md...
  Triage: Complex (Route C)
  Planning...     [done]
  Exploring...    [done]
  Implementing... [done]
  Testing...      [done] ✓ 12 tests passing
  Archiving...    [done]
  Committing...   [done] → abc1234

Processing REQ-004-fix-typo.md...
  Triage: Simple (Route A)
  Implementing... [done]
  Testing...      [done] ✓ 3 tests passing
  Archiving...    [done]
  Committing...   [done] → def5678

Found 1 more pending request. Continuing...

Processing REQ-005-add-tooltip.md...
  Triage: Simple (Route A)
  Implementing... [done]
  Testing...      [done] ✓ 2 tests passing
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
  Testing...      [done] ✓ 8 tests passing
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

### Tests fail repeatedly
- After 3 attempts to fix failing tests, mark request as `failed`
- Include the test failure details in the error field
- Preserve the implementation work done (it may be correct, tests may need adjustment)
- Note in the request file what tests failed and why fixes didn't work

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
- Allow external modification of files in `working/` or `archive/` — these are immutable to all actions except the work pipeline itself
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

Testing: Running related tests...
  - Added tests/theme-store.spec.ts
  - All 12 tests passing ✓

Archived REQ-003-dark-mode.md
Committed → abc1234

Continuing with REQ-004-fix-submit-crash.md...
Triage: Simple (Route A) - bug fix with clear reproduction

[Spawns implementation agent]
Implementation complete:
  - Fixed null check in src/components/Form.tsx:42

Testing: Running related tests...
  - Added regression test in tests/form.spec.ts
  - All 8 tests passing ✓

Archived REQ-004-fix-submit-crash.md
Committed → def5678

Continuing with REQ-005-change-timeout.md...
Triage: Simple (Route A) - config value change

[Spawns implementation agent]
Implementation complete:
  - Updated API_TIMEOUT in src/config.ts from 30000 to 60000

Testing: No tests needed for config value change

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

**Planning:** Required

## Plan

### Implementation Strategy

1. **Create theme store** (src/stores/theme-store.ts)
   - Use Zustand for state management (matches existing patterns)
   - Store light/dark preference
   - Persist to localStorage

2. **Add useTheme hook** (src/hooks/useTheme.ts)
   - Expose current theme and toggle function
   - Handle system preference detection

3. **Update Tailwind config** (tailwind.config.js)
   - Enable darkMode: 'class'
   - Add dark variants for color palette

4. **Add toggle to settings panel** (src/components/settings/SettingsPanel.tsx)
   - Add ThemeToggle component
   - Position in appearance section

5. **Update key components**
   - Header, sidebar, main content areas
   - Use theme-aware Tailwind classes

### Testing Approach
- Unit tests for theme store
- Component tests for toggle behavior
- Visual regression if available

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

*Completed by work action (Route C)*

## Testing

**Tests run:** npm test
**Result:** ✓ All tests passing (24 tests)

**New tests added:**
- tests/stores/theme-store.spec.ts - store state and persistence
- tests/components/ThemeToggle.spec.ts - toggle behavior

**Existing tests verified:**
- tests/components/SettingsPanel.spec.ts - updated for new toggle

*Verified by work action*
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

**Planning:** Not required

## Plan

**Planning not required** - Route A: Direct implementation

Rationale: Simple config value change with explicit file mentioned. No architectural decisions needed.

*Skipped by work action*

## Implementation Summary

- Updated API_TIMEOUT in src/config.ts from 30000 to 60000

*Completed by work action (Route A)*

## Testing

**Tests run:** N/A
**Result:** Config value change, no tests needed

*Verified by work action*
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
