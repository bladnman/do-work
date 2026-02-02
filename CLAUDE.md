# Do-Work Skill Project

## Before Every Commit

**Always bump the version in `actions/version.md` before committing.**

The version is on the line that starts with `**Current version**:` - increment it using semver:
- Patch (0.1.0 → 0.1.1): Bug fixes, minor tweaks
- Minor (0.1.0 → 0.2.0): New features, behavior changes
- Major (0.1.0 → 1.0.0): Breaking changes

When in doubt, bump the patch version.

**Immediately after bumping the version, update `CHANGELOG.md`.**

Add a new entry at the top of the file (below the header), following this format:

```markdown
## X.Y.Z — The [Fun Two-Word Name]

[1-2 sentences. Casual, clear, and concise. What changed and why it matters. No fluff, but personality is welcome.]

- [Bullet points for specifics — what was added, changed, or fixed]
```

Rules for changelog entries:
- **Newest on top.** The file reads top-to-bottom as newest-to-oldest.
- **Give every version a name.** A short, fun title after the em dash (e.g., "The Organizer", "Typo Patrol"). This isn't a novel — two or three words max.
- **Lead with the value, not the implementation.** Say what the user gets, not what files changed. "The archive tidies itself now" beats "added cleanup.md".
- **Keep it brief.** One short paragraph + a few bullets. If you're writing more than 5 bullets, you're over-explaining.
- **Match the voice.** Conversational, not corporate. Imagine you're telling a friend what shipped. No jargon walls, no passive voice marathons.
- **Every version gets an entry.** No skipping. Even a patch fix deserves a line.
