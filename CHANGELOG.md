# Changelog

What's new, what's better, what's different. Most recent stuff on top.

---

## 0.6.0 — The Bouncer

Working and archive folders are now off-limits. Once a request is claimed by a builder or archived, nobody can reach in and modify it — not even to add "one more thing." If you forgot something, it goes in as a new addendum request that references the original. Clean boundaries, no mid-flight surprises.

- Files in `working/` and `archive/` are now explicitly immutable
- New `addendum_to` frontmatter field for follow-up requests
- Do action checks request location before deciding how to handle duplicates
- Work action reinforces immutability in its folder docs

## 0.5.0 — The Record Keeper

Now you can see what changed and when. Added this very changelog so the project has a memory. CLAUDE.md got updated with rules to keep it honest — every version bump gets a changelog entry, no exceptions.

- Added `CHANGELOG.md` with full retroactive history
- Updated commit workflow: version bump → changelog entry → commit

## 0.4.0 — The Organizer

The archive got a brain. New **cleanup action** automatically tidies your archive at the end of every work loop — closing completed URs, sweeping loose REQs into their folders, and herding legacy files where they belong. Also introduced the **User Request (UR) system** that groups related REQs under a single umbrella, so your work has structure from capture to completion.

- Cleanup action: `do work cleanup` (or automatic after every work loop)
- UR system: related REQs now live under UR folders with shared context
- Routing expanded: cleanup/tidy/consolidate keywords recognized
- Work loop exit now triggers automatic archive consolidation

## 0.3.0 — Self-Aware

The skill learned its own version number. New **version action** lets you check what you're running and whether there's an update upstream. Documentation got a glow-up too.

- Version check: `do work version`
- Update check: `do work check for updates`
- Improved docs across the board

## 0.2.0 — Trust but Verify

Added a **testing phase** to the work loop and clarified what the orchestrator is (and isn't) responsible for. REQs now get validated before they're marked done.

- Testing phase baked into the work loop
- Clearer orchestrator responsibilities
- Better separation of concerns

## 0.1.1 — Typo Patrol

Fixed a username typo in the installation command. Small but important — can't install a skill if the command is wrong.

- Fixed: incorrect username in `npx install-skill` command

## 0.1.0 — Hello, World

The beginning. Core task capture and processing system with do/work routing, REQ file management, and archive workflow.

- Task capture via `do work <description>`
- Work loop processing with `do work run`
- REQ file lifecycle: pending → working → archived
- Git-aware: auto-commits after each completed request
