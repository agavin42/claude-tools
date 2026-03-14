# How to Use

This repo contains general-purpose style guides and slash commands for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). You can install them globally (applies to all projects) or per-project.

## Option 1: Global Install (recommended)

Symlink the guidance and commands into your personal `~/.claude/` directory so they're available in every project.

```bash
# Clone the repo
git clone git@github.com:agavin42/claude-tools.git ~/src/claude-tools

# Symlink guidance files
mkdir -p ~/.claude/guidance
ln -sf ~/src/claude-tools/.claude/guidance/oop_style.md ~/.claude/guidance/oop_style.md
ln -sf ~/src/claude-tools/.claude/guidance/plan_style.md ~/.claude/guidance/plan_style.md
ln -sf ~/src/claude-tools/.claude/guidance/bug_plan_style.md ~/.claude/guidance/bug_plan_style.md
ln -sf ~/src/claude-tools/.claude/guidance/review_style.md ~/.claude/guidance/review_style.md

# Symlink slash commands
mkdir -p ~/.claude/commands
ln -sf ~/src/claude-tools/.claude/commands/pr.md ~/.claude/commands/pr.md
ln -sf ~/src/claude-tools/.claude/commands/pc.md ~/.claude/commands/pc.md
```

After this, `/pr` and `/pc` are available in any Claude Code session, and the style guides are always loaded.

To update, just `git pull` in the repo — symlinks pick up changes automatically.

## Option 2: Per-Project Install

Copy (or symlink) into a specific project's `.claude/` directory:

```bash
# From your project root
mkdir -p .claude/guidance .claude/commands

# Copy files (snapshot — won't auto-update)
cp ~/src/claude-tools/.claude/guidance/*.md .claude/guidance/
cp ~/src/claude-tools/.claude/commands/*.md .claude/commands/

# Or symlink (auto-updates on git pull)
ln -sf ~/src/claude-tools/.claude/guidance/oop_style.md .claude/guidance/oop_style.md
ln -sf ~/src/claude-tools/.claude/guidance/plan_style.md .claude/guidance/plan_style.md
ln -sf ~/src/claude-tools/.claude/guidance/bug_plan_style.md .claude/guidance/bug_plan_style.md
ln -sf ~/src/claude-tools/.claude/guidance/review_style.md .claude/guidance/review_style.md
ln -sf ~/src/claude-tools/.claude/commands/pr.md .claude/commands/pr.md
ln -sf ~/src/claude-tools/.claude/commands/pc.md .claude/commands/pc.md
```

## Option 3: Reference from CLAUDE.md

If you just want the style guides (not the slash commands), reference them from your project's `CLAUDE.md`:

```markdown
## Code Style

Follow the OOP style guide at `.claude/guidance/oop_style.md` for all class design.
Follow the plan style guide at `.claude/guidance/plan_style.md` for implementation plans.
```

Claude will read these files when they're referenced.

## What You Get

### Style Guides

| File                | What it does                                                                                  |
| ------------------- | --------------------------------------------------------------------------------------------- |
| `oop_style.md`      | 53 rules for class design — docstrings, method sizing, encapsulation, dispatch, data modeling |
| `plan_style.md`     | How to write implementation plans — journalistic structure, source grounding, checklists      |
| `bug_plan_style.md` | Bug fix plans — diagnosis-first, coherence checks, regression tests                           |
| `review_style.md`   | Code review voice, severity levels, finding format, anti-annoyance rules                      |

### Slash Commands

| Command      | What it does                                                               |
| ------------ | -------------------------------------------------------------------------- |
| `/pr`        | Full PR workflow — review, fix, post to GitHub, resolve comments, grind CI |
| `/pr review` | Multi-agent parallel code review with adversarial validation               |
| `/pr post`   | Post review to GitHub as proper review with line comments                  |
| `/pr fix`    | Auto-fix findings from the last review                                     |
| `/pr grind`  | Keep fixing CI failures until all checks pass                              |
| `/pc`        | Plan quality checker — 5-pass parallel analysis                            |
| `/pc oop`    | Single-pass OOP style check on a plan                                      |
| `/pc design` | Single-pass design intent check (Opus)                                     |

## Keeping Up to Date

If you used symlinks:

```bash
cd ~/src/claude-tools && git pull
```

That's it — all symlinked projects pick up changes immediately.

## Customization

These are starting points. Fork the repo and customize:

- Add project-specific rules to `oop_style.md`
- Add your CI commands to `pr.md`'s test runner and lint runner sections
- Add project-specific zone classifications to `pr.md`'s change grouping
- Adjust model allocations in `pc.md` based on your budget/speed preferences
