# Review Style Guide

Voice, severity, finding format, and filtering rules for `/pr review`.

---

## Review Voice

The reviewer is a senior engineer — direct, constructive, specific. Not a linter, not a bureaucrat.

- **State facts, not hedges.** "This is a race condition" not "this might possibly be a concern."
- **Point to exact lines.** Every finding references a file and line number.
- **Suggest exact fixes.** Code snippets when the fix is mechanical. Approach descriptions when it's architectural.
- **Acknowledge good patterns.** If the code does something well, say so briefly. Positive notes help calibrate trust.
- **No sycophantic softening.** Skip "Great work overall!" preambles. Get to the findings.
- **Proportionate depth.** A 3-line typo fix doesn't need a 200-line review. Match review thoroughness to change scope.

---

## The Anti-Annoyance Principle

The goal is a review that programmers **want** to run, not one they dread. **These rules filter style nits and cleanup noise — never bugs.** A confirmed bug is always worth reporting regardless of how minor it seems. For non-bug findings, apply the test: "Would a senior engineer leave this comment on a real PR?" If not, don't include it.

**Do NOT flag:**

- Style preferences not backed by `oop_style.md` or project conventions
- Formatting issues (linters handle these)
- Import ordering (linters handle this)
- TODOs that pre-existed the PR (only flag TODOs introduced in this diff)
- Code in unchanged files (review the diff, not the codebase)
- Theoretical concerns that can't actually happen given the codebase's constraints
- Personal taste ("I'd prefer X over Y") without a guideline backing it
- Trivial naming suggestions unless genuinely confusing
- Missing type annotations on internal code (unless it's a public API)

**DO flag:**

- Bugs that will reach production
- Race conditions and concurrency issues
- Missing error handling on external boundaries
- Security holes (injection, auth bypass, secret exposure)
- Performance problems at scale (N+1 queries, O(n^2) in hot paths)
- Violations of `oop_style.md` that affect maintainability (class sovereignty, hasattr/isinstance)
- Missing tests for non-trivial new behavior
- Integration mismatches (API contract drift, schema staleness)
- Rollback hazards (irreversible migrations, coordinated deploy requirements)
- **Value/constant changes with un-updated consumers** — trace every usage in the codebase
- **Exception hierarchy bugs** — `except BaseException`, overly broad handlers swallowing async errors
- **Config default changes that break existing environments** — local dev, CI, staging
- **Copy-paste divergence** — duplicated code with subtle behavioral differences

---

## Finding Format

Every finding is a structured record (internal format for agent-to-agent communication):

```
ID:         F001 (sequential per review) or U001 (uncertain findings)
Severity:   CRITICAL | BUG | PERFORMANCE | TESTING | STYLE | NICE | QUESTION
Confidence: 0-100 (assigned by adversarial validation for BUG/CRITICAL)
Validation: CONFIRMED | UNCERTAIN (BUG/CRITICAL only — other severities skip validation)
File:       path/to/file.py
Line:       42
Title:      Short description (< 80 chars)
Body:       What's wrong, why it matters, how it could fail
Suggestion: Concrete fix — code snippet or approach description
```

**Confidence scale:**

- **90-100**: Certain — validator confirmed with concrete evidence
- **80-89**: High — validator confirmed, minor ambiguity but strong evidence
- **50-79**: Uncertain — evidence both ways, shown in "Uncertain Findings" section
- **0-49**: Unlikely — suppressed entirely

**IMPORTANT: This format is for internal agent communication only.** When writing findings in a GitHub review body or console output, use the **GitHub Review Body Format** below — never post raw YAML or structured records to GitHub.

**Body writing rules:**

- Lead with what's wrong, not what the code does (the reviewer can read the code)
- Explain the failure mode — when does this break, what's the symptom, who's affected?
- Reference git history if relevant ("This lock was added in commit abc123 to fix a race condition — this change removes it")
- For CRITICAL/BUG, explain blast radius — is it one user, one endpoint, or global?

**Suggestion writing rules:**

- Mechanical fixes get code snippets (add CONCURRENTLY, add a lock, fix the off-by-one)
- Architectural fixes get approach descriptions ("extract this into a separate service that owns the state machine")
- If the fix is ambiguous, say so — better to flag "this needs a fix but I'm not sure which approach" than to suggest the wrong one

---

## Severity Classification

| Severity        | Meaning                                                 | Blocks merge? | Auto-fixable?      |
| --------------- | ------------------------------------------------------- | ------------- | ------------------ |
| **CRITICAL**    | Will break production, data loss, security hole         | Yes           | Usually no (ask)   |
| **BUG**         | Logic error that will cause incorrect behavior          | Yes           | Often yes          |
| **PERFORMANCE** | Will degrade performance measurably at scale            | Judgement     | Often yes          |
| **TESTING**     | Missing or inadequate test coverage                     | No            | Yes (generate)     |
| **STYLE**       | Violates oop_style.md or project conventions            | No            | Yes                |
| **NICE**        | Improvement suggestion, not a problem                   | No            | No (informational) |
| **QUESTION**    | Reviewer doesn't understand intent, needs clarification | No            | No (needs human)   |

**Status determination from findings:**

- **CRITICAL_ISSUES:** Any CRITICAL finding present
- **NEEDS_WORK:** Any BUG finding (no CRITICALs)
- **CHANGES_REQUESTED:** Only PERFORMANCE/TESTING/STYLE findings (no BUGs or CRITICALs)
- **APPROVED:** Only NICE/QUESTION findings, or no findings at all

---

## Grouping Rules

Don't generate a blizzard of micro-comments. Group intelligently:

- **Same bug class, same file:** One finding listing all affected locations. "Off-by-one error at lines 42, 67, and 103" — not three separate findings.
- **Minor style nits:** Lump into one STYLE finding per file. "Minor style: naming at L42, predicate prefix at L67, method length at L103."
- **Same bug class, multiple files:** One finding with a list of locations if the root cause is shared. Separate findings if they need different fixes.
- **Cross-file integration issues:** One finding that names all involved files. "Schema in models.py:42 doesn't match the route handler in routes.py:88."

**Exception:** CRITICAL and BUG findings always get their own entry, even if related. Each needs individual attention and tracking.

---

## Duplication Findings

The #1 problem with AI-written code: reinventing utilities the codebase already has. When the duplication scan finds matches:

- Severity: STYLE (not BUG — the code works, it's just redundant)
- Point to the existing utility with an exact path and function name
- Explain what the new code does that the existing utility already handles
- If the new code does something slightly different, note the delta — maybe the existing utility should be extended instead

---

## History-Informed Review

When git archaeology reveals relevant context, weave it into findings:

- **Recent bug fix in the same area:** "This function was modified 3 days ago (commit abc123) to fix a race condition. The current change removes the lock that fixed it." Severity escalation — what would be a STYLE finding becomes a BUG.
- **High-churn file:** "This file has been modified 8 times in 30 days, all bug fixes. Extra scrutiny warranted — this area is fragile."
- **Original author context:** "This module was designed by @alice as a write-once pipeline. The current changes add mutable state, which may conflict with the original design intent."

---

## Logging Safety Checklist (HAS_LOGGING_CHANGE)

When a PR adds or modifies structured logging, apply these checks:

**1. Crash safety** — Can any bound value blow up?

- `.value` on a potentially-None enum? Guard with `x.value if x else None`
- `str()` on something that might not exist yet (deleted object, unfetched relation)?
- Arithmetic in a bound field that could divide by zero or subtract from None?
- Accessing attributes after a deletion — capture values before deletion

**2. No logic changes hiding** — "Logging only" means logging only.

- `return await f()` → `result = await f(); log; return result` is fine but verify same return value
- Commit/flush ordering unchanged? No commit moved into a branch that was previously shared?
- No new function parameters added (especially dependency injection params that can change auth behavior)
- No new imports that have module-level side effects

**3. No log spam** — Every log should fire at bounded, expected frequency.

- Is the log inside a loop? What bounds the iteration count?
- Is it in a hot path (request handler called thousands of times/sec)?
- Same frequency as the log it replaces, or justified if new?
- Per-item logging in batch operations needs a cap or summary alternative

**4. No PII/secrets** — Check every bound field.

- Emails: user emails are PII. Admin emails are lower risk but still PII. Prefer user_id.
- Tokens: full token values are secrets. Token prefixes are safe.
- Free-text fields (`note`, `description`): could contain anything users typed
- Internal IDs (subscription, customer): identifiers not secrets, generally safe

**5. No risky side-effect queries** — New DB queries purely for log enrichment.

- Does the query run after the operation committed? If it fails, is the operation still successful?
- Is the query bounded? Could it return thousands of rows for per-item logging?
- Could the query fail on a code path where the operation already succeeded, masking the success?

---

## "What This PR Does" Section

Every review includes a **What This PR Does** narrative section. This is the most-read part of the review — optimize for it. Someone reading only this section should understand what merged, why it matters, and why this approach was chosen.

**Tell the full story, not just the diff:**

- **The problem** — What business or operational problem motivated this change? Be specific: data points, failure modes, user impact, frequency.
- **Why the current state is insufficient** — What breaks, degrades, or is painful without this change? What workaround exists and why is it inadequate?
- **The solution** — How does the fix work technically? Walk through the approach at the right altitude — not line-by-line, but enough to understand the architecture.
- **Key design decisions** — Why this approach over alternatives? What tradeoffs were made?
- **Upstream context** — If a Slack thread, analytics investigation, incident, or ticket motivated the work, summarize the key findings. This context is often the most valuable part of the review for future readers.

**Scale to PR size:**

- Small PRs (1-5 files): 2-4 sentences covering problem + approach
- Medium PRs (5-15 files): 1-2 paragraphs
- Large PRs (15+ files): 2-4 paragraphs covering problem, architecture, and decisions

---

## GitHub Review Body Format

When writing findings in the review body (posted to GitHub or printed to console), use markdown — **never raw YAML, code blocks of structured records, or internal agent format.** Findings in code blocks render terribly on GitHub.

Each confirmed finding in the review body should look like:

```
**[F001] BUG: Race condition in heartbeat update** — `services/worker.py:142` · confidence 92

The heartbeat writes to Redis without a lock, but `cleanup_stuck_tasks` reads
the same key. Under load, the cleanup cron could kill an active worker.

**Validation:** Adversarial check confirmed — no lock or guard found anywhere in the call chain.

**Fix:** Use Redis WATCH/MULTI or add the existing `_heartbeat_lock`.
```

Each uncertain finding should look like:

```
**[U001] BUG?: Possible null access on user.email** — `routes/billing.py:55` · confidence 62

`user.email` is accessed without a null check after `get_user()`.

**For:** `get_user()` can return a user with `email=None` for SSO accounts (see `models/user.py:88`).
**Against:** This route is behind the `@require_email_verified` decorator which may guarantee email is present.
```

Rules:

- Bold finding ID, severity, and title on the first line
- File and line after an em dash, in backticks, followed by `· confidence N`
- 2-4 sentences explaining the problem (not in a code block)
- **Validation:** summary of what the adversarial check found (for confirmed BUG/CRITICAL findings)
- **For:** / **Against:** evidence summaries (for uncertain findings only)
- **Fix:** suggestion in bold label
- Separate findings with `---` horizontal rules
- Group by severity (CRITICAL first, then BUG, etc.)
- Uncertain findings go in their own `## Uncertain Findings` section after all confirmed findings

---

## GitHub Line Comment Format

When posting individual line comments to GitHub, keep them concise. The full analysis lives in the review body — line comments are margin notes.

**Standard format (prose fix):**

```
**[F001] CRITICAL: Race condition in heartbeat update** · confidence 95

The heartbeat writes to Redis without a lock, but `cleanup_stuck_tasks` reads
the same key. Under load, the cleanup cron could kill an active worker.

Same class of bug fixed in abc123 (3 days ago) — being reintroduced.

**Fix:** Use Redis WATCH/MULTI or add the existing `_heartbeat_lock`.

— *Claude Code /pr*
```

**With committable suggestion (small, mechanical fixes):**

When the fix is self-contained (1-5 lines, single location, no cascading changes), include a GitHub suggestion block so the author can accept the fix with one click:

````
**[F003] BUG: Missing await on async call** · confidence 98

`fetch_user_data()` is async but called without `await`, so it returns a
coroutine object instead of the actual data.

```suggestion
    user_data = await fetch_user_data(user_id)
```

— *Claude Code /pr*
````

**Rules for suggestion blocks:**

- Only include when committing the suggestion **fully resolves** the finding — no follow-up steps needed
- Max ~5 lines of replacement code
- Must match exact indentation of the existing code
- Use `start_line` + `line` parameters when replacing a range (not just one line)
- Good: missing `await`, wrong operator, null check, off-by-one, wrong constant, missing import
- Bad: architectural changes, multi-file fixes, complex refactors

**General rules:**

- Lead with the finding ID and severity in bold, confidence score after `·`
- 2-4 sentences explaining the problem
- Include a concrete fix (suggestion block for small fixes, prose for larger ones)
- End with `— *Claude Code /pr*` attribution tag
- No pleasantries, no padding — this is a margin note, not a letter
