# Version Action

> **Part of the do-work skill.** Handles version reporting and update checks.

**Current version**: 0.6.0

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
