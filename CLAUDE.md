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
## X.Y.Z — The [Fun Two-Word Name] (YYYY-MM-DD)

[1-2 sentences. Casual, clear, and concise. What changed and why it matters. No fluff, but personality is welcome.]

- [Bullet points for specifics — what was added, changed, or fixed]
```

Rules for changelog entries:
- **Newest on top.** The file reads top-to-bottom as newest-to-oldest.
- **Give every version a name.** A short, fun title after the em dash (e.g., "The Organizer", "Typo Patrol"). This isn't a novel — two or three words max.
- **Date every entry.** Add the date in parentheses after the name, formatted as `(YYYY-MM-DD)`. Use today's date.
- **Lead with the value, not the implementation.** Say what the user gets, not what files changed. "The archive tidies itself now" beats "added cleanup.md".
- **Keep it brief.** One short paragraph + a few bullets. If you're writing more than 5 bullets, you're over-explaining.
- **Match the voice.** Conversational, not corporate. Imagine you're telling a friend what shipped. No jargon walls, no passive voice marathons.
- **Every version gets an entry.** No skipping. Even a patch fix deserves a line.

## Agent Compatibility

This skill is designed to work with **any agentic coding tool**, not just one specific platform. When writing or editing action files:

- **Use generalized language.** Say "use your environment's ask-user prompt/tool" rather than naming a specific tool API. Say "spawn a subagent" rather than referencing a specific tool's agent mechanism.
- **Hint at advanced features, don't require them.** Subagents, multi-agent workflows, and structured question UIs improve results when available. The actions must still work in a single-session tool that reads the markdown as a prompt.
- **Each action file should work as a standalone prompt.** If someone pastes `do.md` into a basic chat interface with file access, the instructions should be clear enough to follow without the SKILL.md routing layer or any skill-runner infrastructure.
- **No tool-specific APIs in action files.** Tool-specific names and APIs belong in the tool's own integration layer, not in the skill's action files. Use platform-specific details only as clearly-labeled examples (e.g., "Example: Claude Code caches images at `~/.claude/...`").
- **Design for the floor, not the ceiling.** The least sophisticated agent that can read/write files and run shell commands should be able to execute these actions correctly. Advanced agents benefit from subagents and parallel execution, but the baseline must work without them.
