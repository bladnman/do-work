# Version Action

> **Part of the do-work skill.** Handles version reporting and update checks.

**Current version**: 0.9.3

**Upstream**: https://raw.githubusercontent.com/bladnman/do-work/main/SKILL.md

## Responding to Version Requests

When user asks "what version" or "version":
- Report the version shown above

## Responding to Update Checks

When user asks "check for updates", "update", or "is there a newer version":

1. **Fetch upstream**: Use WebFetch to get the raw SKILL.md from the upstream URL above
2. **Extract remote version**: Look for `**Current version**:` in the fetched content
3. **Compare versions**: Use semantic versioning comparison
4. **Report result** using the format below

### Report Format

**If update available** (remote > local):

```
Update available: v{remote} (you have v{local})

To update, run:
npx install-skill bladnman/do-work

Or visit: https://github.com/bladnman/do-work
```

**If up to date** (local >= remote):

```
You're up to date (v{local})
```

**If fetch fails**:

```
Couldn't check for updates.

To manually update, run:
npx install-skill bladnman/do-work

Or visit: https://github.com/bladnman/do-work
```

## Responding to Changelog Requests

When user asks "changelog", "release notes", "what's new", "what's changed", "updates", or "history":

1. **Find the changelog**: Look for `CHANGELOG.md` in the skill's root directory (same level as `SKILL.md`)
2. **Read the file**: Load the full contents
3. **Reverse for terminal reading**: The changelog is written newest-on-top (conventional for file reading). For terminal output, reverse the version sections so the **most recent entries appear at the bottom** — right where the user's eyes are
   - Separate the header (everything before the first `## ` version heading) from the version entries
   - Split version entries at each `## ` heading (each heading + its body is one block)
   - Reverse the order of those blocks
   - Output: header first, then oldest-to-newest entries (so newest lands at the bottom)
4. **Print the result**: Output the reversed changelog directly — no file creation, just terminal output

### Why Reverse?

Changelogs are written newest-first so the file reads well. But in a terminal, the bottom of the output is where the user is looking. Reversing puts the latest changes at the bottom — no scrolling required.

### If No Changelog Exists

If `CHANGELOG.md` is not found in the skill root:

```
No changelog found for this skill.
```
