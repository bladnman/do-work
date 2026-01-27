# Do Action

> **Part of the do-work skill.** Invoked when routing determines the user is adding a request. Creates a `do-work/` folder in your project for request tracking.

A fast-capture system for turning quick ideas into structured request files. Designed for speed - minimal interaction when intent is clear, human-in-the-loop only when genuinely ambiguous.

## Philosophy

- **Speed over perfection**: This is a rapid capture interface, not a design review
- **Represent, don't expand**: If the user says 5 words, write a 5-word request (with structure). Don't inflate a simple idea into a 400-word PRD
- **The building agent solves technical questions**: You're capturing intent, not making architectural decisions
- **Multiple requests are expected**: Users often say "do X, and also Y, and don't forget Z"
- **Never be lossy**: For complex inputs, preserve ALL detail - don't summarize away requirements

## Simple vs Complex Requests

This skill handles two modes:

| Mode | Input Size | Approach |
|------|------------|----------|
| **Simple** | Short, 1-3 features | Quick capture, lean format |
| **Complex** | Long, detailed, multi-feature | Full preservation with context document |

### Detecting Complex Requests

Treat a request as **complex** when ANY of these apply:

- **Length**: Input is >500 words or feels substantial
- **Detail**: Includes specific requirements, constraints, edge cases, acceptance criteria
- **Breadth**: Describes 3+ distinct features or components
- **Explicit**: User says "spec", "PRD", "detailed requirements", "here's what I need..."
- **Nuance**: Contains "must", "should", "never", "always", specific values, or conditional logic
- **Relationships**: Features depend on each other or have sequencing

**When in doubt, treat it as complex.** Over-preserving is better than losing requirements.

### The Lossy Problem

When splitting a 15-minute detailed spec into 5 requests, information gets lost:
- Shared context disappears
- Cross-feature requirements fall through cracks
- The "why" behind decisions vanishes
- Subtle constraints get summarized away

**Solution: Context Documents**

For complex requests, create a shared context document that preserves the FULL verbatim input, then have each request reference it.

## Request File Location

All request files go in the project's `do-work/` folder:
- `do-work/` at the project root
- Create the folder if it doesn't exist
- `do-work/assets/` for screenshots and files (can be auto-copied from Claude's image cache)

This folder name reflects the **do-work pattern** - requests created by the do action are processed by the work action.

## File Naming Convention

**Request files:** `REQ-[number]-[slug].md`
- `REQ-001-dark-mode.md`
- `REQ-002-export-to-pdf.md`

**Context documents (for complex requests):** `CONTEXT-[number]-[slug].md`
- `CONTEXT-001-auth-system.md`
- `CONTEXT-002-dashboard-redesign.md`

**Context document lifecycle:**
- Live in root `do-work/` as **reference documents** (not actionable work items)
- Are **not** picked up by the work action (it only processes `REQ-*.md`)
- Archive to `do-work/archive/` when **all related REQs** are complete
- The `requests` field in the context frontmatter tracks which REQs belong to it

To get the next number, check existing `REQ-*.md` files in `do-work/`, `do-work/working/`, and `do-work/archive/`, then increment from the highest. Context documents use their own numbering sequence.

## Request File Format

### Simple Request Format

For quick captures - short, single-feature requests.

```markdown
---
id: REQ-001
title: Brief descriptive title
status: pending
created_at: 2025-01-26T10:00:00Z
---

# [Brief Title]

## What
[1-3 sentences describing what is being requested]

## Why (if provided)
[User's stated reasoning - ONLY if they gave one. Omit section if not provided]

## Context
[Any additional context, constraints, or details the user mentioned]

## Assets
[Detailed description of any screenshots, or links to manually-saved files]

---
*Source: [original verbatim request]*
```

Keep it lean. If the user said "add dark mode", the What section might just be "Add a dark mode toggle to the app." That's fine.

### Complex Request Format

For detailed, multi-feature, or nuanced requests. Uses additional fields and references a context document.

```markdown
---
id: REQ-005
title: OAuth login flow
status: pending
created_at: 2025-01-26T10:00:00Z
context_ref: CONTEXT-001-auth-system.md
related: [REQ-006, REQ-007, REQ-008]
batch: auth-system
---

# OAuth Login Flow

## What
Implement OAuth-based authentication with Google and GitHub providers.

## Detailed Requirements
[Extract ALL requirements from the original input that apply to THIS feature specifically.
DO NOT SUMMARIZE. Include everything the user said about this feature.]

- Must support Google OAuth 2.0 with PKCE
- Must support GitHub OAuth
- Redirect to /dashboard after successful login
- Store refresh tokens encrypted at rest (user said "tokens must be encrypted")
- Handle token expiration gracefully - refresh automatically if possible
- Show loading spinner during OAuth redirect
- Handle OAuth errors with user-friendly messages, not technical errors
- "Remember me" checkbox to extend session duration
- Log authentication events for security audit

## Constraints
[Any limitations or restrictions mentioned]

- Must work with existing session management system
- Cannot use third-party auth services (user explicitly said "no Auth0 or similar")
- Must support existing users migrating from password auth

## Dependencies
[What this feature needs or what needs it]

- Depends on: REQ-007 (session management) - needs session system first
- Blocks: REQ-006 (user profile) - profile needs auth to exist

## Open Questions
[Ambiguities that the builder should clarify or decide]

- Should OAuth be the only login method, or alongside password?
- What happens if user's email from OAuth doesn't match existing account?

## Full Context
See [CONTEXT-001-auth-system.md](./CONTEXT-001-auth-system.md) for complete requirements discussion.

---
*Source: See context document for full verbatim input*
```

### Context Document Format

Created when processing complex requests. Preserves the FULL verbatim input.

```markdown
---
id: CONTEXT-001
title: Authentication System Requirements
created_at: 2025-01-26T10:00:00Z
requests: [REQ-005, REQ-006, REQ-007, REQ-008]
word_count: 1847
---

# Authentication System Requirements

## Summary
[2-3 sentence overview of what this context covers]

User described a complete authentication system including OAuth login, user profiles,
session management, and password reset flows. Emphasized security requirements and
integration with existing systems.

## Extracted Requests

| ID | Title | Summary |
|----|-------|---------|
| REQ-005 | OAuth login flow | Google/GitHub OAuth with PKCE |
| REQ-006 | User profile management | Profile editing, avatar upload |
| REQ-007 | Session management | Token storage, expiration, refresh |
| REQ-008 | Password reset flow | Email-based reset with expiry |

## Full Verbatim Input

[THE COMPLETE, UNEDITED INPUT FROM THE USER]

[Paste the entire 15 minutes of transcribed text here, exactly as received.
Do not summarize, edit, or clean up. This is the source of truth.]

[If the input was very long, it's okay - include all of it. The builder
can read this if they need context that wasn't captured in the individual
request files.]

---
*Captured: 2025-01-26T10:00:00Z*
```

### Frontmatter Fields

**Simple requests:**

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Request identifier (REQ-NNN) |
| `title` | Yes | Short title |
| `status` | Yes | Always `pending` when created |
| `created_at` | Yes | ISO 8601 timestamp |

**Complex requests (additional fields):**

| Field | Required | Description |
|-------|----------|-------------|
| `context_ref` | Yes | Filename of the context document |
| `related` | If applicable | Array of related REQ IDs |
| `batch` | If applicable | Batch name grouping related requests |

**Context documents:**

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Context identifier (CONTEXT-NNN) |
| `title` | Yes | Descriptive title for the batch |
| `created_at` | Yes | ISO 8601 timestamp |
| `requests` | Yes | Array of REQ IDs extracted from this context |
| `word_count` | Yes | Word count of original input (for reference) |

The work action adds additional fields (`claimed_at`, `route`, `completed_at`) as it processes each request.

## Workflow

### Step 1: Parse the Request

Read the user's input. Look for:
- Single request vs multiple requests
- Clear intent vs ambiguous intent
- Any provided context (screenshots, references to existing behavior)

**Multiple request indicators:**
- "and also", "oh and", "plus", "additionally"
- Comma-separated lists
- Numbered items
- Distinct topics in the same message

### Step 1.5: Assess Complexity

Determine if this is a simple or complex request.

**Complex indicators** (any of these → complex mode):
- Input is >500 words
- 3+ distinct features or components
- Detailed requirements, constraints, or acceptance criteria
- User says "spec", "PRD", "detailed", "requirements"
- Contains specific values, conditions, or edge cases
- Features have dependencies or sequencing
- Multiple stakeholders or user types mentioned

**Simple indicators** (all of these → simple mode):
- Short, focused input (<200 words)
- 1-2 features max
- No detailed constraints
- Quick idea capture

**When uncertain → treat as complex.** It's better to over-preserve than lose requirements.

### Step 2: Check for Existing Requests

Read all files in `do-work/` folder. For each parsed request:
- Does a similar request already exist?
- Is this an enhancement to an existing request?
- Is this a duplicate?

**If duplicate/similar found:**
- If clearly the same thing: Tell user, don't create new file
- If similar but potentially different: Use AskUserQuestion to clarify
- If enhancement: Ask if they want to update the existing request

**Updating an existing request (addendum, not rewrite):**

Don't rework the original content - that's lossy. Instead, append an Addendum section:

```markdown
## Addendum (2025-01-27)

User added: "dark mode should also affect the sidebar"

- Sidebar must also respect dark mode theme
- [any other new requirements extracted from the update]
```

This preserves the original request intact. The building agent reconciles the original + addendum during planning - it can see what came first and what was added later.

### Step 3: Clarify Only If Needed

Use AskUserQuestion ONLY when:
- Request is genuinely ambiguous (could mean two very different things)
- Similar request exists and it's unclear if this is new or a revision
- User's intent could conflict with existing requests

**DO NOT ask questions when:**
- You can reasonably infer the intent
- The request is simple and clear
- Technical implementation is unclear (that's for the building agent)

Example good question:
```
You said "fix the search". I found an existing request about search performance (REQ-003).
Did you mean:
- Update that request (search performance)
- Something different (describe a new search issue)
```

Example unnecessary question (don't do this):
```
You want dark mode. Should it:
- Use system preferences
- Have its own toggle
- Support scheduled switching
```
^ These are implementation details for the building agent.

### Step 4: Handle Screenshots

**Image Cache Discovery**: Claude Code caches images shared in conversation at `~/.claude/image-cache/[session-uuid]/[number].png`. You can read these with the Read tool and copy them to project folders.

If the user passes a screenshot:

1. **Find the cached image**: List `~/.claude/image-cache/` to find recent session folders, then check for `.png` files inside
2. **Verify it's the right image**: Use the Read tool on the `.png` file to visually confirm it matches what the user shared
3. **Copy to project assets**: `cp ~/.claude/image-cache/[uuid]/[n].png do-work/assets/REQ-[num]-[slug].png`
4. **Reference in request file**: Point to `do-work/assets/REQ-[num]-[slug].png` in the Assets section
5. **Still write a description**: Even with the file saved, include a thorough text description for searchability and context

**Finding the right image:**
```bash
# List recent image cache sessions
find ~/.claude/image-cache -name "*.png" -exec ls -la {} \;

# Images are numbered in order shared (1.png, 2.png, etc.)
```

**Fallback** if cache is empty or images can't be found:
- Write a detailed description as the primary record
- User can manually save to `do-work/assets/` if needed

When describing screenshots, be thorough - this is the primary record:
- **What it shows**: UI state, screen area, dialog, error message, etc.
- **Visual layout**: Where elements are positioned relative to each other
- **All visible text**: Labels, buttons, headings, error text - quote exactly
- **What the user is pointing out**: Infer from context what's relevant
- **UI state**: Selected items, toggle states, hover states, etc.
- **Problems visible**: Misalignment, missing labels, redundant elements

Example good description:
```
Screenshot shows settings panel with:
- "Featured Items" section heading (bold, ~14px)
- "Show Featured Item" label with toggle switch (enabled/blue) on same line
- Dropdown below showing "1 hour" - NO LABEL explaining what this controls
- "Refresh Featured Items" button at bottom
Problem: The dropdown's purpose is unclear without a label.
```

Example bad description:
```
Screenshot of settings with a dropdown.
```

### Step 5: Write Request Files

#### Simple Mode

For each distinct request:
1. Determine the next REQ number (check `do-work/`, `working/`, and `archive/` for highest existing number)
2. Create a slug from the request (lowercase, hyphens, 3-4 words max)
3. Generate the current ISO 8601 timestamp for `created_at`
4. Write the file using the **simple request format**
5. Keep the original verbatim request in the Source field

#### Complex Mode

For complex, multi-feature requests:

**1. Create the Context Document first:**
```yaml
---
id: CONTEXT-001
title: [Descriptive name for this batch]
created_at: 2025-01-26T10:00:00Z
requests: []  # Will fill in after creating requests
word_count: [count words in original input]
---
```

- Include a brief summary (2-3 sentences)
- Leave the `requests` array empty initially
- **Paste the FULL verbatim input** in the "Full Verbatim Input" section
- Do NOT edit, clean up, or summarize the verbatim input

**2. Identify all distinct features/requests:**
- Read through the input carefully
- List each distinct feature, component, or deliverable
- Note dependencies between them
- Note shared constraints or requirements

**3. Create individual request files:**

For EACH identified feature:
1. Determine the next REQ number
2. Create using the **complex request format**
3. Set `context_ref` to the context document filename
4. Set `related` to other REQ IDs in this batch
5. Set `batch` to a short name for this group

**Critical: Detailed Requirements section**

This is where lossiness happens. For each request:
- Extract EVERY requirement from the original input that applies to THIS feature
- **DO NOT SUMMARIZE** - use the user's words or close paraphrases
- Include specific values, constraints, conditions
- Include edge cases and error handling requirements
- Include "must", "should", "never" statements
- If something might apply, include it (let builder filter)

**4. Update the Context Document:**
- Go back and fill in the `requests` array with all created REQ IDs
- Update the "Extracted Requests" table

**5. Verify completeness:**
- Re-read the original input
- Check that every requirement appears in at least one request file
- If something doesn't fit any request, either:
  - Create another request for it, OR
  - Add it to the most relevant request's "Context" section

**Frontmatter examples:**

Simple:
```yaml
---
id: REQ-004
title: Add keyboard shortcuts
status: pending
created_at: 2025-01-26T14:30:00Z
---
```

Complex:
```yaml
---
id: REQ-012
title: OAuth login flow
status: pending
created_at: 2025-01-26T14:30:00Z
context_ref: CONTEXT-003-auth-system.md
related: [REQ-013, REQ-014, REQ-015]
batch: auth-system
---
```

### Step 6: Report Back

After creating files, give a brief summary:
- What files were created
- Any existing requests that were updated
- Any requests that were skipped (duplicates)

Keep it concise. Don't explain your reasoning unless asked.

## Examples

### Example 1: Simple Single Request

```
User: do work add keyboard shortcuts

Claude: Created REQ-004-keyboard-shortcuts.md
```

### Example 2: Multiple Requests

```
User: do work add dark mode, also the search feels slow, and we need an export button

Claude: Created:
- REQ-005-dark-mode.md
- REQ-006-search-performance.md
- REQ-007-export-button.md
```

### Example 3: Potential Duplicate

```
User: do work fix the lag when typing

Claude: [Uses AskUserQuestion]
I found an existing request REQ-006 about search performance that mentions typing lag.
Is this:
- The same issue (I'll note you reported it again)
- A different typing lag issue (I'll create a new request)
```

### Example 4: Request with Screenshot

```
User: do work when I click here [screenshot] nothing happens

Claude: Created REQ-008-button-click-fix.md
Screenshot saved to do-work/assets/REQ-008-button-click-fix.png
```

(Request file includes path to saved screenshot plus detailed description)

### Example 5: Enhancement (Addendum)

```
User: do work dark mode should also affect the sidebar

Claude: [Checks existing]
Found REQ-005-dark-mode.md. Added addendum for sidebar requirement.
```

(The original request content stays intact. An Addendum section is appended with the new requirement and timestamp.)

### Example 6: Complex Multi-Feature Request

```
User: do work [15 minutes of detailed requirements about an authentication system
including OAuth, user profiles, session management, password reset,
security requirements, migration from old system, etc.]

Claude: This is a complex request with multiple features. Creating context document
and individual requests...

Created:
- CONTEXT-001-auth-system.md (full verbatim input preserved, 1847 words)
- REQ-010-oauth-login.md (references context, related to REQ-011,012,013)
- REQ-011-user-profiles.md (references context, depends on REQ-010)
- REQ-012-session-management.md (references context, blocks REQ-010)
- REQ-013-password-reset.md (references context, alternative to REQ-010)

All requests reference the context document for full requirements.
```

### Example 7: Detecting Complexity

```
User: do work I need a new dashboard. It should have a summary card at the top
showing total users, active sessions, and revenue. Below that, a chart
showing trends over the last 30 days - make sure it's interactive so users
can hover for details. The chart should support different time ranges.
On the right side, show a feed of recent activity. Users should be able to
filter the activity by type. Oh and the whole thing needs to work on mobile,
so the layout should stack vertically on small screens. Also add a date
range picker that affects all the widgets.

Claude: [Detects: multiple components, specific requirements, responsive design
constraints, interactive features → Complex mode]

Created:
- CONTEXT-002-dashboard.md (preserves full input)
- REQ-014-dashboard-summary-cards.md
- REQ-015-dashboard-trend-chart.md
- REQ-016-dashboard-activity-feed.md
- REQ-017-dashboard-date-picker.md
- REQ-018-dashboard-responsive-layout.md

Each request includes full detailed requirements for that component.
```

## What NOT To Do

- Don't ask about implementation details
- Don't create multi-page PRDs for **simple** requests
- Don't refuse to create a request because it's "too vague" - capture what was said
- Don't merge obviously distinct requests into one file
- Don't make the user wait for questions you could answer yourself
- Don't add requirements the user didn't mention

**For complex requests specifically:**
- **Don't summarize detailed requirements** - preserve the user's exact words
- **Don't skip the context document** - it's the source of truth
- **Don't drop edge cases or constraints** - if the user mentioned it, capture it
- **Don't assume something is obvious** - write it down explicitly
- **Don't split related requirements** across unrelated requests - keep context together
- **Don't "clean up" the verbatim input** - typos, rambling, and all

## Edge Cases

**User says something is broken but doesn't say what:**
Create the request anyway with what they said. The building agent or a follow-up conversation can clarify.

**User references something from earlier conversation:**
Include that context in the request file.

**User gives a very long, detailed request:**
Switch to complex mode. Create a context document with the full verbatim input. Split into multiple requests, each referencing the context.

**Request seems impossible or contradictory:**
Not your call. Capture it. The building agent will figure it out or ask. For complex requests, note contradictions in the "Open Questions" section.

**Requirement applies to multiple features:**
Include it in ALL relevant request files. Duplication is better than losing it.

**User changes their mind mid-request:**
Capture the final decision, but note the evolution in the context document. The builder might need to understand why.

**Features have circular dependencies:**
Note the dependencies in both request files. The builder will work out the sequencing.

**Some requirements are vague, others specific:**
Capture both. Vague ones go in as-is; the builder can ask for clarification. Don't try to make vague requirements more specific.

**User mentions something once in passing:**
Capture it. If they said "oh and it should probably work offline too" - that's a requirement, even if brief.
