---
description: PR workflow - branch management, reviews, and GitHub PR operations
allowed-tools: Bash(git *), Bash(gh *), Bash(date *), Bash(pre-commit *), Bash(jq *), Bash(while *), Bash(sleep *), Bash(echo *), Bash(printf *), Read, Write, Edit, Grep, Glob, Task
argument-hint: [subcommand or instructions]
---

## Tool Attribution

When this tool posts reviews to GitHub (`/pr post`), it identifies itself with:

- **Visible footer** in the review body:
  ```
  🔍 Automated review by Claude Code `/pr` · [docs](.claude/commands/pr.md)
  ```
- **Visible tag** on each line comment: `— *Claude Code /pr*`
- **Machine-readable marker** as a hidden HTML comment appended to the review body: `<!-- CLAUDE-REVIEW:v1 ... -->` (used for delta tracking on re-reviews)

To find reviews posted by this tool, search PR review bodies for `"Claude Code /pr"` or the hidden marker `"CLAUDE-REVIEW:v1"`.

---

## Context (Preloaded)

### Metadata

- Branch: !`git branch --show-current 2>/dev/null || echo "(detached or no work tree)"`
- Date: !`date +"%m_%d_%y"`
- Git user: !`git config user.name 2>/dev/null || echo "(unknown)"`
- Uncommitted: !`git status --short 2>/dev/null || echo "(not in a work tree)"`
- Repo: !`gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null || echo "(unknown repo)"`

### GitHub PR

!`gh pr view --json number,url,state,title,mergeable 2>/dev/null || echo "No PR exists for this branch"`

### Branch Delta (vs origin/main)

Commits:
!`git log origin/main..HEAD --oneline 2>/dev/null || echo "(no origin/main or no commits yet)"`

Files changed:
!`git diff --stat origin/main...HEAD 2>/dev/null || echo "(cannot diff against origin/main)"`

---

## Instructions

The user invoked `/pr $ARGUMENTS`

### Subcommand Handling

If `$ARGUMENTS` is a single word matching one of these subcommands, execute that specific workflow. Otherwise, treat `$ARGUMENTS` as general instructions/comments for the review process.

### Remote PR Detection

**IMPORTANT:** Detect remote PR targets in ANY of these formats — the `--pr` flag is optional:

- `/pr review 8209` — bare number after subcommand → remote PR #8209
- `/pr review --pr 8209` — explicit flag → remote PR #8209
- `/pr review https://github.com/owner/repo/pull/8209` — URL → remote PR #8209
- `/pr review user/some-branch` — branch name with `/` → resolve to PR via `gh pr list --head <branch>`

**When a number, URL, or branch name follows a subcommand, ALWAYS treat it as a remote PR target.** Fetch that PR's branch and diff ONLY that PR against main — ignore the current branch's diff entirely.

### Remote PR Support (`--pr`)

Any subcommand that reads a PR can accept `--pr <target>` (or just a bare target after the subcommand) to operate on a remote PR instead of the current branch:

- `--pr 7683` or just `7683` — PR number
- `--pr https://github.com/owner/repo/pull/7683` or just the URL — PR URL (extract number)
- `--pr user/fix_branch_name` or just `user/fix_branch_name` — branch name (resolve to PR number via `gh pr list --head <branch>`)

**When `--pr` is specified, the review operates in remote mode:**

1. **No checkout required** — diff against `origin/<branch>` instead of `HEAD`
2. **Resolve PR metadata:**
   ```bash
   gh pr view <number> --json number,headRefName,baseRefName,title,url
   ```
3. **Fetch the branch:** `git fetch origin <headRefName>`
4. **All diffs use `$REF = origin/<headRefName>`** instead of `HEAD`

**Determine the repo dynamically** using `gh repo view --json nameWithOwner -q .nameWithOwner` and use the result for all `gh api` calls. Store as `$REPO` and use `repos/$REPO/pulls/...` throughout.

---

## Review Pipeline

### Phase 0: Pre-flight Guard (~3 seconds)

Before doing any review work, run a cheap pre-flight check:

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)

# 1. Is the PR closed or merged?
gh pr view $PR_NUM --json state -q .state 2>/dev/null
# If "CLOSED" or "MERGED" → stop.

# 2. Is the PR a draft?
gh pr view $PR_NUM --json isDraft -q .isDraft 2>/dev/null
# If true → print warning but continue.

# 3. Is the diff empty?
git diff --stat origin/main...${REF:-HEAD} 2>/dev/null
# If empty → stop.
```

For `/pr post` and `/pr repost`, also check:

```bash
# 4. Has Claude already posted a review on this PR?
gh api repos/$REPO/pulls/$PR_NUM/reviews --jq '.[].body' 2>/dev/null | grep -c "Claude Code \`/pr\`"
# If > 0 and this is `post` (not `repost`): warn and stop.
```

### Phase 1: Preparation (~5 seconds)

**1a. Ensure branch state:**

- **Local mode (no `--pr`):** Must be on a feature branch (run `branch` logic if on main). Fetch latest: `git fetch origin main`
- **Remote mode (`--pr`):** Resolve PR number, fetch branch: `git fetch origin main <headRefName>`. Set `$REF = origin/<headRefName>` for all subsequent diff/read operations.

**1b. Gather the diff:**

```bash
# Local mode:
git diff origin/main...HEAD

# Remote mode:
git diff origin/main...origin/<headRefName>
```

**1c. Check for delta review:**

Look for a prior CLAUDE-REVIEW marker in existing PR review bodies:

```bash
gh api repos/$REPO/pulls/$PR_NUM/reviews --jq '.[].body | select(contains("CLAUDE-REVIEW:v1"))' 2>/dev/null || echo "no-prior-review"
```

If a prior review exists, parse its SHA, confirmed findings, and uncertain findings from the marker. Compute the delta diff. Only re-analyze changed/new files. Carry forward unresolved findings from the prior review.

**1d. Group changes:**

Classify each changed file into logical module zones based on the project's directory structure. General guidelines:

| Pattern                 | Zone                            |
| ----------------------- | ------------------------------- |
| Source code directories | By module/feature area          |
| Test files              | With their corresponding source |
| Config / deploy / CI    | infra                           |
| Documentation           | docs                            |

Rules:

- Keep test files with their source (same group)
- Merge groups with <2 files into nearest related zone
- Cap at 5 groups — merge smallest if over
- **Migrations are ALWAYS their own group** (specialized analysis)
- Route + schema/model changes stay together

**1e. Determine PR size class:**

- **SMALL** (1-3 files): Single analysis agent, no coordinator.
- **STANDARD** (4-15 files): Full coordinator pipeline, 2-4 change groups.
- **LARGE** (16+ files): Full coordinator pipeline, 4-5 groups. Note: "Consider splitting this PR."

### Phase 2: Review Pipeline

#### Small Mode

Launch three agents in parallel:

**Agent 1 — Context-aware analysis:** Task with `subagent_type="Explore"`, `model="opus"`.

Provide: the full diff, changed file list, and the analysis rules from the "Analysis Rules" section below. The agent reads context for each file, analyzes across all nine axes, and returns structured findings.

**The agent MUST also determine suggested reviewers:** Run `git shortlog -sn -- <file> | head -5` and `git log --oneline -5 -- <file>` for each changed file to identify top contributors. Produce a Suggested Reviewers section. **Never suggest the PR author as a reviewer.**

**Agent 2 — Diff-only bug sweep:** Task with `subagent_type="Explore"`, `model="opus"`. Reviews the raw diff without reading any surrounding files. See A3c instructions below.

**Agent 3 — CLAUDE.md compliance:** Task with `subagent_type="Explore"`, `model="sonnet"`. Collects applicable CLAUDE.md files and audits the diff against them. See A3b instructions below.

After all three return, merge findings, run adversarial validation on BUG/CRITICAL findings (Step B2), then synthesize (Step C).

#### Standard / Large Mode

Launch a **coordinator subagent** (`subagent_type="general-purpose"`, `model="sonnet"`) with the full pipeline instructions below. The coordinator manages all agents internally.

---

## Review Coordinator Pipeline

**These instructions are for the coordinator subagent. Include them verbatim in the coordinator's prompt — subagents don't inherit parent context.**

You are a code review coordinator. Run a parallel analysis pipeline: launch sub-agents, collect results, synthesize findings, produce a formatted review.

First, read these guidance files for rules to pass to analysis agents:

- `.claude/guidance/review_style.md` — severity, finding format, anti-annoyance rules, grouping, GitHub comment format
- `.claude/guidance/oop_style.md` — style rules for code

### Step A: Parallel Launch

Launch ALL of the following simultaneously. Do not wait between launches.

**A1. Test runners** (background agents):

- **Test runner:** Identify test files for changed source files. Run the project's test suite on relevant files. If no test files found, skip.
- **Lint runner:** Run the project's linter/formatter on changed files.

**A2. Git archaeologist** (background, haiku):

For each changed file, gather:

```bash
git shortlog -sn -- <file> | head -5     # top contributors
git log --oneline -10 -- <file>          # recent history
git log --oneline --since="30 days ago" -- <file>  # recent churn
```

Produce: contributor list, change frequency, recent bug fixes, suggested reviewers. **Never suggest the PR author as a reviewer.**

**A3. Analysis agents** (1 per change group, Opus):

Each agent receives:

- Its group's file list and diff
- The full analysis rules (see below)
- CLAUDE.md and style guides content

**Phase I — Context gathering.** For each file in the group:

1. Read the full file (not just the diff)
2. Read direct callers/callees of changed functions (one level out)
3. If a constant or config value changed, find all consumers

**Phase II — Analysis.** Apply all nine axes from the Analysis Rules below.

**A3b. CLAUDE.md compliance agent** (Sonnet):

1. Collect all CLAUDE.md files in the repo (root + any directory-level ones in changed paths)
2. For each rule in each CLAUDE.md, check the diff for violations
3. Return findings as STYLE severity, citing the specific CLAUDE.md file and rule

**A3c. Diff-only bug sweep agent** (Opus):

Reviews the raw diff without reading surrounding files. Look for:

- Wrong operators (`==` vs `=`, `and` vs `or`, `>` vs `>=`)
- Missing return statements, missing awaits
- Null/None access without guards
- Type mismatches visible in the diff
- Copy-paste errors (duplicated blocks with one not updated)
- Off-by-one errors in slicing, ranging, indexing
- Inverted conditions (negation errors)
- Missing break/continue in loops or match cases
- Variable shadowing
- Unbalanced resource management (open without close)
- String format mismatches
- Internal inconsistency between parallel structures in the diff

**Do NOT read surrounding files.** If you cannot verify from the diff alone, do not flag it.

**A4. Safety deep-dive agent** (conditional, Opus):

**Only launch when the diff removes or modifies behavioral safety code** — error handling, cascade/cleanup logic, validation checks, state machine transitions, retry/timeout mechanisms. Skip for pure additions, docs, config, tests-only, or frontend-only changes.

Trace the full defense chain: what was removed, what replaces it, is there a gap?

### Analysis Rules

Each analysis agent applies these nine axes:

**1. Correctness** — Logic errors, off-by-ones, wrong operators, missing awaits, null access without guards, type mismatches.

**2. Concurrency** — Race conditions, lock ordering, shared mutable state, missing synchronization.

**3. Security** — Injection, auth bypass, secret exposure, OWASP Top 10.

**4. Performance** — N+1 queries, O(n^2) in hot paths, missing indexes, unbounded result sets.

**5. Error handling** — Missing error handlers at boundaries, overly broad catches, swallowed errors, lost tracebacks.

**6. API contracts** — Schema drift, missing validation, backwards-incompatible changes.

**7. Test coverage** — Missing tests for new behavior, edge cases, error paths.

**8. Rollback safety** — Irreversible changes, coordinated deploy requirements.

**9. The 3am Test** — Most likely failure mode in production, blast radius, monitoring coverage, remediation path.

Additional analysis patterns:

**Value/constant propagation** — When ANY constant, config default, or numeric value changes: trace every consumer, verify unit consistency, check arithmetic downstream.

**Exception/error hierarchy** (Python) — `except BaseException` catches `CancelledError`/`KeyboardInterrupt` — almost always wrong. Overly broad `except Exception` that swallows errors in loops. Error re-raising that loses traceback.

**Internal consistency** — When the diff contains multiple representations of the same concept, verify they match.

**Severity classification:**

- **CRITICAL** — will break production, data loss, security hole (blocks merge)
- **BUG** — logic error causing incorrect behavior (blocks merge)
- **PERFORMANCE** — measurable degradation at scale (judgement call)
- **TESTING** — missing or inadequate tests (doesn't block)
- **STYLE** — violates oop_style.md or project conventions (doesn't block)
- **NICE** — improvement suggestion, not a problem (doesn't block)
- **QUESTION** — needs clarification from author (doesn't block)

### Step B: Wait + Collect

Wait for all agents to complete. Merge findings from A3 (context-aware), A3b (CLAUDE.md compliance), and A3c (diff-only). Deduplicate — keep the richer explanation when both flag the same issue.

### Step B2: Adversarial Validation

For every CRITICAL or BUG finding, launch a parallel validation subagent (`subagent_type="Explore"`, `model="opus"`). Each validator gets its own fresh context — no anchoring bias.

**Validator prompt:** "An automated code review flagged this issue. **Your job is to DISPROVE it.** You are a defense attorney for the code:

1. Is there a guard, check, or handler elsewhere that prevents this?
2. Can the problematic input actually reach this code path?
3. Does the type system or a validator guarantee safety?
4. Is there a passing test that exercises this scenario?
5. Is it intentional behavior documented in comments or commit messages?

Return: **CONFIRMED** (with confidence 0-100), **REFUTED** (with evidence), or **UNCERTAIN** (with For/Against evidence and confidence 0-100)."

After validators return:

- **CONFIRMED**: keep, update body with validator context
- **REFUTED**: drop entirely
- **UNCERTAIN ≥ 50**: keep in "Uncertain Findings" section
- **UNCERTAIN < 50**: drop

Skip validation for PERFORMANCE, TESTING, STYLE, NICE, QUESTION findings.

### Step C: Synthesis

Launch one synthesis agent (`model="opus"`). Provide all results: validated findings, unvalidated findings, CLAUDE.md findings, safety analysis, test/lint results, git archaeology.

The synthesizer:

1. **Deduplicates** — merge findings with same root cause
2. **Cross-group analysis** — do changes in group A break assumptions in group B?
3. **The 3am pass** — worst-case production failure? Blast radius? Remediation?
4. **Rollback assessment** — SAFE / CAUTION / IRREVERSIBLE
5. **Prioritizes and numbers** — F001-F999 for confirmed, U001-U999 for uncertain
6. **Determines status** — APPROVED / CHANGES_REQUESTED / NEEDS_WORK / CRITICAL_ISSUES
7. **Writes suggested reviewers** — from git archaeology
8. **Produces the TLDR** — 2-3 sentences
9. **Writes "What This PR Does"** — follow `review_style.md` § "What This PR Does"
10. **Formats uncertain findings** — include For/Against evidence

### Step D: Output + Persist

**D1. Format the review:**

```markdown
# Code Review: [branch-name]

**Status:** [APPROVED | CHANGES_REQUESTED | NEEDS_WORK | CRITICAL_ISSUES]
**Reviewed:** [date] | **Files:** N | **Groups:** N | **Duration:** Xm Ys
**Comparing:** [branch] -> main

## TLDR

[2-3 sentences from synthesizer]

## What This PR Does

[Full story: problem, why current state is insufficient, approach, design decisions]

## Suggested Reviewers

- **@name** — rationale

## Changes

### Group 1: [description]

- `path/to/file.py` — What changed (1 sentence)

## Test & Lint Results

[checkmark] N tests passed (M failed) — Xs
[checkmark/warning] Linting status

## Findings

### CRITICAL

**[F001] Title** `file.py:42` · confidence 95

> Body. **Validation:** summary. **Fix:** suggestion.

### BUG / PERFORMANCE / TESTING / STYLE / NICE / QUESTION

...

## Uncertain Findings

**[U001] BUG?: Title** `file.py:55` · confidence 62

> Body. **For:** evidence. **Against:** evidence.

## Safety Analysis

[Only if A4 ran]

## Rollback Assessment

[SAFE/CAUTION/IRREVERSIBLE]

## The 3am Scenario

[Worst-case failure analysis]

## Final Status: [STATUS]

[Summary]

---

🔍 Automated review by Claude Code `/pr` · [docs](.claude/commands/pr.md)
```

**D2. Persist findings:**

Write structured findings to `.claude/review-state/<branch-name>.json`.

**D3. Return** the formatted review text as your response.

---

## Subcommand: `rereview`

Same as `review` but **always fresh** — ignore any prior CLAUDE-REVIEW markers and any local review-state files.

---

## Subcommand: `fix`

Read structured findings from the last review and apply fixes.

**Usage:**

- `/pr fix` — fix all auto-fixable findings
- `/pr fix F001 F003` — fix specific findings by ID

**Step 1:** Read `.claude/review-state/<branch-name>.json`. If not found, tell user to run `/pr review` first.

**Step 2:** Triage each finding:

- **Auto-fix:** BUG with concrete suggestions, STYLE, TESTING ("add test"), PERFORMANCE with mechanical fixes
- **Ask first:** CRITICAL findings, ambiguous/architectural fixes
- **Skip:** QUESTION (present to human), NICE (informational)

**Step 3:** Apply fixes. Work in file order to minimize line-number drift.

**Step 4:** After fixes: run tests, run linter, commit, update review-state JSON, reply to GitHub comments if review was posted.

**Step 5:** Print summary of what was fixed, skipped, or needs human input.

---

## Subcommand: `resolve`

Fetch GitHub PR review comments from human reviewers and address each one.

**Step 1: Fetch comments**

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
PR_NUM=$(gh pr view --json number -q .number)

gh api repos/$REPO/pulls/$PR_NUM/comments --paginate
gh api repos/$REPO/issues/$PR_NUM/comments --paginate
gh api repos/$REPO/pulls/$PR_NUM/reviews --paginate
```

**Step 2: Filter noise** — skip bot comments, already-resolved, self-notes, pure approvals.

**Step 3: Triage** — Fix (safe, clear) / Discuss (complex, risky) / Dismiss (outdated). Present to user for confirmation.

**Step 4: Execute** — apply fixes, reply to comments via `gh api`.

**Step 5: Commit and push.**

---

## Subcommand: `post` (alias: `comment`)

Run review if needed, then post to GitHub as a proper review.

**Step 1:** Run `review` if not done yet.

**Step 2:** Ensure PR exists and push local commits.

**Step 3:** Use `COMMENT` as the review event by default.

**Step 4:** Post review body as GitHub review. Body is the executive summary — confirmed findings go as line comments, not in the body. Uncertain findings ARE in the body (need For/Against context).

**Step 5:** Post line comments for unresolved findings. Include committable suggestion blocks for small mechanical fixes (1-5 lines). Every line comment ends with `— *Claude Code /pr*`.

**Step 6:** Append CLAUDE-REVIEW delta tracking marker to review body.

```bash
gh api repos/$REPO/pulls/$PR_NUM/reviews \
  -X POST \
  -f event="COMMENT" \
  -f body="<review body>"
```

---

## Subcommand: `repost`

Delete prior `/pr` reviews, then post fresh.

1. Find prior reviews by searching for the attribution footer
2. Collapse old reviews in `<details>` blocks
3. Delete prior line comments tagged with `— *Claude Code /pr*`
4. Run full `post` logic

---

## Subcommand: `branch`

Ensure on a feature branch. If on `main`:

1. Check diff/status for clues about the work
2. Create `{prefix}/description_mm_dd_yy` where prefix = git user's first name lowercased
3. If unclear, **ask the user**

---

## Subcommand: `pr`

Ensure a GitHub PR exists:

1. Ensure on a branch (run `branch` if on main)
2. If PR exists, report it
3. If no PR: push and create draft via `gh pr create --draft`
4. Report the PR URL

---

## Subcommand: `commit`

1. Ensure on a branch
2. If uncommitted changes: read diff, stage, generate message, commit
3. If clean, inform user

---

## Subcommand: `merge`

1. Ensure on branch (not main), ensure PR exists
2. Check mergeable status
3. If mergeable: `gh pr merge --merge`
4. After merge, switch to main and pull

---

## Subcommand: `main`

Switch to main and update: `git checkout main && git pull origin main`

---

## Subcommand: `status`

Print concise branch state summary. Read-only.

```
Branch: <name>
Uncommitted: [clean | N files]
PR Status: [none | PR #N: <title>]
Branch vs main: [N commits ahead | up to date]
```

---

## Subcommand: `review --quick`

Single-agent fast review. One Sonnet agent does a single-pass review across all nine axes. Less depth, broader coverage. Still writes to review-state JSON.

---

## Subcommand: `grind`

Keep churning CI failures until all checks pass. Merges main first, then iterates: poll checks, diagnose failures, fix, push, repeat.

**Key principles:**

- Merge main first (most CI failures come from being behind main)
- Minimal fixes only
- Never force-push
- Max 5 iterations — if same check fails twice with same error, stop and ask
- Skip unrelated failures (flaky tests, infra) — note and move on
- Batch related fixes in a single commit

**Polling CI:**

```bash
while true; do
  ci_out=$(gh pr checks --json name,bucket 2>&1)
  pending_n=$(echo "$ci_out" | jq '[.[] | select(.bucket == "pending")] | length')
  fail_n=$(echo "$ci_out" | jq '[.[] | select(.bucket == "fail")] | length')
  pass_n=$(echo "$ci_out" | jq '[.[] | select(.bucket == "pass")] | length')
  echo "$(date '+%H:%M:%S') — checks: $pass_n pass, $pending_n pending, $fail_n fail"
  if [ "$pending_n" -eq 0 ] 2>/dev/null; then break; fi
  sleep 10
done
```

---

## Subcommand: `help`

```
/pr Subcommands:

  (no args)  Same as 'review' - comprehensive parallel code review
  status     Quick summary of branch/PR state (read-only)
  branch     Ensure on feature branch (creates one if on main)
  review     Full parallel review (coordinator + Opus analysis)
  review --quick  Single Sonnet agent, fast but less thorough
  review --pr N   Review someone else's PR (by number, URL, or branch)
  rereview   Fresh review from scratch (ignores prior state)
  fix        Auto-fix findings from last review
  fix F001   Fix specific findings by ID
  resolve    Fetch GitHub comments, triage, fix/reply to each
  resolve --dry  Show triage without making changes
  pr         Ensure GitHub PR exists for this branch
  post       Run review + post to GitHub as proper review (alias: comment)
  post --pr N  Post review to someone else's PR
  repost     Delete prior /pr reviews, then post fresh
  commit     Stage and commit current changes
  grind      Keep churning CI failures until all checks pass
  merge      Merge PR into main
  main       Switch to main and pull latest
  help       Show this help

Remote PR support:
  --pr 7683              Review/post by PR number
  --pr <url>             Review/post by GitHub PR URL
  --pr user/branch-name  Review/post by branch name

Lifecycle (own PR):
  review → fix → review → post → [humans review] → resolve → push

Lifecycle (someone else's PR):
  review --pr N → post --pr N
```

---

## Key Principles

**Workflow:**

- Always check branch state before operations
- When uncertain about branch names or next steps, **ask the user**

**Review Quality:**

- Follow `.claude/guidance/review_style.md` for voice, severity, and formatting
- Follow `.claude/guidance/oop_style.md` for style checks
- Be specific — point to exact lines, suggest exact fixes
- Be proportionate — small changes get light reviews
- Organize findings for execution — group by file, order by line
- The anti-annoyance principle: every finding must pass the "would a senior engineer say this?" test
