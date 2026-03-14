# claude-tools

General-purpose style guides and slash commands for Claude Code. These are project-agnostic — install them into any repo or your personal `~/.claude/` directory.

## What's Included

### Style Guides (`.claude/guidance/`)

- **[OOP Style](/.claude/guidance/oop_style.md)** — 53 rules for clean class design in Python and TypeScript. Class docstrings, method sizing, encapsulation, dispatch, data modeling, cross-class architecture.
- **[Plan Style](/.claude/guidance/plan_style.md)** — How to write implementation plans. Journalistic structure (conceptual → design → implementation), source code grounding, checklists, goal audits.
- **[Bug Plan Style](/.claude/guidance/bug_plan_style.md)** — Diagnosis-first variant of plan style for bug fixes. Problem statement, user experience, reasoning, regression tests, coherence checks.
- **[Review Style](/.claude/guidance/review_style.md)** — Voice, severity levels, finding format, anti-annoyance filtering, GitHub comment formatting for code reviews.

### Slash Commands (`.claude/commands/`)

- **[/pr](/.claude/commands/pr.md)** — Full PR workflow: branch management, parallel multi-agent code review with adversarial validation, auto-fix, GitHub posting, CI grinding.
- **[/pc](/.claude/commands/pc.md)** — Plan quality checker: 5-pass parallel analysis (structure, OOP, tests, codebase reality, design intent) with scout/judge architecture.

## Code Style

All code written under these guides should follow:

- **OOP style** — classes have docstrings with scope/boundaries, methods 5-15 lines, models own their mutations, dataclasses not dicts, enums not strings
- **Imports at top** — stdlib → third-party → local
- **No barrel imports** — explicit paths, not `__init__.py` re-exports
- **No hasattr/isinstance** for capability checks — add duck-typed methods to base class
- **Build only what's needed** — no speculative infrastructure

See [how_to_use.md](how_to_use.md) for installation instructions.
