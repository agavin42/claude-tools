# Bug Plan Style Guide

A variant of [`plan_style.md`](plan_style.md) for bug fixes and targeted improvements. Everything in `plan_style.md` applies unless overridden here. The key difference: bug plans are **diagnosis-first** — they must demonstrate understanding of the problem before proposing a solution.

---

## Header

Every bug plan starts with this header block:

```markdown
# Bug: <Descriptive Title>

**Plan-Created:** YYYY-MM-DD
**Plan-Executed:** YYYY-MM-DD (fill in when execution begins)
**Style:** [bug_plan_style.md](.claude/guidance/bug_plan_style.md)
```

The `Plan-Executed` date stays blank until work starts. Both dates are grepable:

```bash
grep -r "Plan-Created:\|Plan-Executed:" plans/
```

---

## Structure

Bug plans follow this order. Sections 1-7 are the **diagnostic front matter** — they must be coherent as a standalone argument before any implementation details appear.

```
1. Header (dates + style reference)
2. Goals
3. Human Notes (empty by default — reserved for human annotations)
4. Problem Statement
5. Proposed Fix
6. User Experience
7. Reasoning
8. Reproduction Steps
9. Regression Test
10. Implementation Parts (per-phase, each with checklist)
11. Files to Modify
12. Verification
13. Resolved Questions
```

---

## Section 2: Goals

Same as `plan_style.md` — high-level, human-reviewable. For bugs, goals typically look like:

- "Eliminate X failure mode under Y conditions"
- "Ensure Z invariant holds across the A lifecycle"
- "Prevent data loss when B happens concurrently with C"

Not implementation steps. The reader should understand _what success looks like_ in 30 seconds.

---

## Section 3: Human Notes

**Reserved for human-written annotations.** Always present, empty by default. This is where a human reviewer can add context that Claude doesn't have — customer reports, frequency observations, priority overrides, "this bit me last Tuesday", etc.

```markdown
## Human Notes

_(No notes yet.)_
```

When a human adds notes, they replace the placeholder. Claude should **never remove or edit** content in this section — it's human-owned territory. If Claude notices something that seems relevant to existing human notes, mention it in Reasoning or Resolved Questions, not here.

---

## Section 4: Problem Statement

**What is broken, and why does it matter?** This section must be specific and grounded.

```markdown
## Problem Statement

<2-5 sentences describing the observable bug or deficiency. Include:>

- What happens (the symptom)
- When/where it happens (conditions, context)
- What should happen instead (expected behavior)
- Impact (data loss? silent corruption? user-visible error? degraded performance?)
```

**Good problem statements** are falsifiable — someone reading it can say "yes, I can confirm this happens" or "no, that's not what I'm seeing."

**Bad problem statements** are vague ("the system sometimes behaves incorrectly") or conflate symptom with cause ("the race condition in X causes Y" — that's jumping to diagnosis).

---

## Section 5: Proposed Fix

**The basic idea of the fix in plain language.** One paragraph, maybe two. No code yet — this is the elevator pitch.

```markdown
## Proposed Fix

<1-2 paragraphs: What we're going to change and the core insight behind it.
This should be understandable by someone who hasn't read the code.>
```

Example: _"Move the broadcast call to after the database commit, so listeners never see an event for a row that doesn't exist yet. This preserves the existing API contract while eliminating the window where the event arrives before the data is queryable."_

---

## Section 6: User Experience

**What does the user actually see and feel when this bug fires?** This section is written entirely from the client-side perspective — no backend jargon, no line numbers, no code. Imagine you're the person sitting at the keyboard using the product. What happens to _them_?

```markdown
## User Experience

<Describe the user's experience holistically:>

- What were they doing when things went wrong?
- What did they see (or not see)?
- Did anything look broken, or did it silently fail?
- How confused/frustrated would they be?
- Would they know something went wrong, or would they only find out later?
- What's the blast radius — one operation, one session, permanent data loss?
```

This section serves as a **severity gut-check**. A bug that silently drops user messages is worse than one that shows an error toast. A bug that corrupts saved work is worse than one that causes a momentary visual glitch. Writing the user experience forces you to think about whether the fix priority actually matches the user impact.

**Good user experience descriptions** are concrete and empathetic:

_"The user types a message while the agent is working, hits enter, and sees it appear in the chat. The agent finishes its current task but never acknowledges the message. On page refresh, the message is gone — it was never saved. The user has no way to know their input was lost."_

**Bad user experience descriptions** are too technical or too vague:

_"The PendingEvent is deleted before processing completes."_ (that's the problem statement, not the user experience)

_"The user might see an error."_ (what error? when? how bad?)

---

## Section 7: Reasoning

**Why this fix is correct, and why alternatives are worse.** This is the section that prevents cargo-cult fixes. It should make the reader confident the fix addresses root cause, not just symptom.

```markdown
## Reasoning

<Detailed explanation covering:>
- Root cause analysis — what's actually going wrong at the code level
- Why the proposed fix addresses the root cause (not just the symptom)
- What alternatives were considered and why they're inferior
- Any tradeoffs the fix introduces
- Edge cases the fix must handle correctly
```

This section should be the longest of the diagnostic front matter. If you can't write a clear reasoning section, you probably don't understand the bug well enough to fix it. **Go back and read more code.**

---

## Section 8: Reproduction Steps

**The shortest path to making the bug happen.** Two sub-sections: one for manual/empirical reproduction, one for automated.

```markdown
## Reproduction Steps

### Manual Reproduction

<Numbered steps to trigger the bug. Prefer steps that can be
performed directly — API calls, shell commands, test invocations.
If the bug requires timing-dependent conditions, describe the setup
and what to watch for.>

1. ...
2. ...
3. Observe: <what goes wrong>
```

If the bug is non-deterministic or requires infrastructure that isn't locally available, say so explicitly. A reproduction that's honest about its limitations is better than one that pretends to be reliable.

If reproduction requires a running service, external state, or user interaction that can't be performed in isolation, note that clearly: **"Requires running service + database"** or **"Manual browser test only."**

---

## Section 9: Regression Test

**A specific, automatable test that fails before the fix and passes after.** This is the plan's falsifiability criterion — it's how we know the fix actually worked.

````markdown
## Regression Test

### Test Description

<What the test does, in plain language.>

### Test Location

`path/to/test_file.py::test_function_name`

### Expected Behavior

- **Before fix:** <what the test does — fails, wrong output, exception, etc.>
- **After fix:** <what the test should do — passes, correct output, no exception>

### Test Sketch

```python
def test_bug_name_regression():
    """Regression test for <brief description>."""
    # Setup
    ...
    # Action that triggers the bug
    ...
    # Assert the correct behavior
    assert ...
```
````

````

**Not always possible.** Some bugs are inherently hard to test in isolation (race conditions, infrastructure-dependent failures, UI-only issues). When a regression test isn't feasible, say why explicitly:

```markdown
## Regression Test

**Not feasible as unit test** — this bug requires concurrent subscribers
and sub-millisecond timing. Instead, verify via:
- Manual reproduction steps above
- Code review confirming the fix addresses the root cause
- <any partial test that increases confidence>
````

Partial tests that cover _part_ of the fix are still valuable. "We can't test the race, but we can test that the broadcast happens after commit" is better than nothing.

---

## Sections 10-13: Implementation, Files, Verification, Resolved Questions

Follow `plan_style.md` directly. The implementation phases should map clearly back to the proposed fix and reasoning — no mystery steps that aren't justified by the diagnostic sections.

**One addition to Verification:** after implementation, re-run the reproduction steps from Section 8 and confirm the regression test from Section 9 passes. Note the results in the plan:

```markdown
## Verification

### Regression Test Result

- [ ] `test_bug_name_regression` — passes after fix

### Reproduction Re-check

- [ ] Manual reproduction steps no longer trigger the bug
```

---

## Coherence Check

Before submitting a bug plan for review, verify this chain holds:

1. **Problem Statement** describes a real, observable issue
2. **Proposed Fix** directly addresses that issue
3. **User Experience** makes clear how bad this is for real users
4. **Reasoning** explains _why_ the fix is correct (root cause, not symptom)
5. **Reproduction Steps** can trigger the problem described in (1)
6. **Regression Test** will fail due to (1) and pass due to (2)
7. **Implementation** executes exactly what (2) describes, justified by (4)

If any link in this chain is broken — the test doesn't actually test the bug, the fix addresses a symptom not the cause, the reasoning has a gap — **stop and fix the plan before executing.**

---

## Bug Plan Review Checklist

In addition to the checklist in `plan_style.md`:

**Diagnostic Quality:**

- [ ] Problem statement is specific and falsifiable
- [ ] Proposed fix is stated in plain language before any code
- [ ] User experience is written from the user's perspective (no backend jargon)
- [ ] User experience helps calibrate severity — does the fix priority match the impact?
- [ ] Reasoning explains root cause, not just symptom
- [ ] Alternatives were considered (even if briefly dismissed)
- [ ] Reproduction steps are honest about their reliability
- [ ] Human Notes section is present (empty is fine)

**Test Coverage:**

- [ ] Regression test exists, or explicit justification for why not
- [ ] Test sketch is concrete (not just "add a test for this")
- [ ] Before/after behavior is clearly stated

**Coherence:**

- [ ] Problem -> Fix -> Reasoning chain is unbroken
- [ ] Reproduction steps actually trigger the stated problem
- [ ] Regression test actually tests the stated fix
- [ ] Implementation matches the proposed fix (no scope creep)

**Dates:**

- [ ] `Plan-Created:` date is present
- [ ] `Plan-Executed:` field is present (blank until execution)
- [ ] `Style:` line references this guide
