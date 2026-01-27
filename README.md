# do-work

A task queue skill for Claude Code. Capture requests fast, process them later.

## Installation

```bash
npx add-skill bladnman/do-work
```

## Welcome to your new work loop

This skill gives you a two-phase workflow:

1. **Capture**: Throw ideas, bugs, and feature requests at Claude as they come up. Each one becomes a structured request file in `do-work/`.

2. **Process**: When you're ready, tell Claude to work. It picks up pending requests one by one, triages complexity, and builds until the queue is empty.

The idea: separate *thinking of things* from *doing things*. Capture is instant. Processing is thorough.

## Quick usage

**Add a request:**
```
do work add dark mode to the settings page
```
Creates `do-work/REQ-001-dark-mode.md`

**Add multiple at once:**
```
do work the search is slow, also add an export button, and fix the header alignment
```
Creates three separate request files.

**Process the queue:**
```
do work run
```
Starts the work loop. Claude triages each request by complexity:
- **Simple** (config changes, small fixes) → straight to implementation
- **Medium** (clear goal, unknown location) → explore codebase first
- **Complex** (new features, architectural) → plan, explore, then build

Each completed request gets archived with its implementation notes and a git commit.

## How it works

```
do-work/
├── REQ-001-pending.md      # Queue (pending requests)
├── REQ-002-pending.md
├── working/                 # Currently being processed
│   └── REQ-003-in-progress.md
└── archive/                 # Completed work
    └── REQ-000-done.md
```

Requests move through the folders as they're processed. The archive preserves the full history: original request, triage decision, exploration notes, and implementation summary.

## Designed for Claude Code

This skill is built for [Claude Code](https://github.com/anthropics/claude-code), which has:
- Sub-agent spawning (Plan, Explore, general-purpose builders)
- File editing and bash access
- Git integration for per-request commits

It may work with other AI coding tools that support similar agent patterns.

## The two actions

### Do (capture)

Invoked when you provide descriptive content. Optimized for speed:
- Minimal questions - capture what was said, don't interrogate
- Handles simple one-liners and complex multi-feature specs
- For long inputs, creates a context document preserving the full verbatim text
- Checks for duplicates against existing requests

See [actions/do.md](./actions/do.md) for the full capture logic.

### Work (process)

Invoked when you say "run", "go", "start", or just confirm the prompt. Runs the build loop:
- Triages each request to determine the right amount of planning
- Spawns specialized agents only when needed
- Archives completed work with implementation notes
- Creates atomic git commits per request

See [actions/work.md](./actions/work.md) for the full processing logic.

## License

MIT
