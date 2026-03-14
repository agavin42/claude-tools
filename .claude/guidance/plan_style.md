# Plan Style Guide

Guidelines for creating implementation plans that are clear, reviewable, and architecturally sound.

---

## CRITICAL: Plan Location and Naming

**Plans go in a `plans/` directory** within the relevant area of your project. For monorepos, each service or app may have its own plans folder (e.g., `services/foo/plans/`, `apps/bar/plans/`). For simpler repos, a top-level `plans/` directory works.

**Use descriptive, functional names**:

- `media_uploader_plan.md`
- `streaming_ref_formatting_plan.md`
- `upload_processing_enhancement.md`
- `plan1.md`
- `new_feature.md`
- `temp_plan.md`

---

## CRITICAL: Source Code Study Requirements

**Plans must be grounded in actual source code, not assumptions.** A plan that doesn't deeply understand the existing codebase will fail during implementation.

### Before Writing the Plan

1. **Read existing docs** — Check READMEs, architecture docs, and `CLAUDE.md` for relevant context
2. **Study the source code thoroughly** — This is the most important step:
   - Grep for related classes, functions, and patterns
   - Read the actual implementations, not just interfaces
   - Trace through execution paths for similar operations
   - Understand how data flows through the system
   - Note existing patterns, conventions, and abstractions
3. **Identify integration points** — Where will the new code connect to existing code?

### After Writing the Plan (Verification Pass)

**Do a separate review pass against the source** — Don't skip this:

1. Re-read the source files you'll be modifying
2. Verify your plan's assumptions match reality:
   - Do the classes/methods you plan to call actually exist?
   - Do they have the signatures you assumed?
   - Are there existing patterns you should follow instead of inventing new ones?
3. Check for conflicts with recent changes
4. Update the plan if you find discrepancies

**A plan that hasn't been validated against source is just wishful thinking.**

---

## Purpose

Plans tell a story: what we're building, why, and how. The human reads the story. Claude reads the implementation details.

The top of the plan should read like a clear narrative — someone picking it up cold should understand the problem, the approach, the key decisions, and the architecture without ever reaching a method signature or SQL statement. By the time you get to checklists and code sketches, the plan has already conveyed everything a human reviewer needs.

**In practice: the human stops reading partway through.** That's by design. The conceptual and design sections are for people. The implementation sections are for Claude to execute. Write accordingly — the top half should be your best writing, clear and persuasive. The bottom half should be precise and mechanical.

**Required reading**: [`.claude/guidance/oop_style.md`](oop_style.md) — all class design in plans must follow these OOP principles.

---

## Journalistic Structure: Top-Down Readability

Plans follow the **inverted pyramid** — like a newspaper article, the most important information comes first and detail increases as you read further. A reader should be able to stop at any point and have a complete (if less detailed) understanding.

**Three tiers:**

1. **Conceptual** (everyone reads) — Goals, architecture overview, key decisions, cost/tradeoffs. Written in plain language. No method signatures, no SQL, no code sketches. Someone unfamiliar with the codebase should understand what's being built and why. This is the section that gets shared with leadership or cross-team stakeholders.

2. **Design** (architects read) — How the pieces fit together. Class relationships (diagrams, not method tables), data flow, process sequences, key design decisions with rationale. Technical but focused on _shape_ not _syntax_. A senior engineer should be able to review the architecture and flag concerns without reading implementation details.

3. **Implementation** (Claude executes) — Per-phase checklists, code sketches, file lists, test specifications, migration steps. Precise enough for Claude to implement without ambiguity. Humans rarely read this section — it's the builder's instruction manual.

**How to tell if the ordering is wrong:**

- SQL DDL appears before the reader understands the architecture → wrong
- Method signatures appear before the reader understands what the class _does_ → wrong
- Goals contain implementation steps ("Add a CreationSource enum") instead of outcomes ("Track upload provenance") → wrong level
- A reader has to scroll past code blocks to find out what the plan is _for_ → wrong

**How to tell if the conceptual section is right:**

- A technical CEO can read it and give useful feedback
- A PM can understand what's changing and why
- An engineer from another team can evaluate whether it affects them
- You can explain the plan by reading the first page aloud

---

## Plan Structure

Every plan should follow this order, progressing from conceptual to concrete:

```
── Conceptual (the story — what and why) ──
1. Title & Context
2. Goals (high-level outcomes, not implementation steps)
3. Architecture Overview (how it works, data flow, key decisions, cost)

── Design (the shape — how pieces fit together) ──
4. New Classes Summary (diagrams, relationships, responsibilities)
5. Process Flow (sequential walkthrough with failure paths)
6. Key Design Decisions (with rationale — why this approach, not that one)

── Implementation (the instructions — for Claude) ──
7. Implementation Parts (per-phase, each with checklist)
8. Files to Modify
9. Verification
10. Goal Audit (when goals are verifiable system properties — skip for exploratory/brainstorm plans)
11. Resolved Questions
```

Not every plan needs all sections — a small refactor may skip the architecture overview. But the ordering should always flow from conceptual to concrete, never the reverse.

**The conceptual sections are the plan.** Everything below them is execution detail. If the conceptual sections don't clearly convey the what/why/how, adding more implementation detail won't help — fix the top, not the bottom.

---

## Section 1: Title & Context

Brief header establishing what this plan continues or relates to. **Always include creation and modification dates** with the grepable labels below.

```markdown
# Feature Name

**Plan-Created:** YYYY-MM-DD
**Plan-Modified:** YYYY-MM-DD
**Style:** [plan_style.md](.claude/guidance/plan_style.md)

**Continuation of `previous_plan.md`** — one-line description of what this adds.

**OOP Style Conformance** (if applicable):

- Rule X: Pattern used
- Rule Y: Pattern used
```

**Date labels are grepable** — find all plans and their dates:

```bash
grep -r "Plan-Created:" plans/
grep -r "Plan-Modified:" plans/
```

---

## Section 2: Goals

High-level goals that a human can review in 30 seconds. These are the "why" — what we're trying to achieve, not how.

```markdown
## Goals

1. **Goal in bold** — one sentence explanation
2. **Another goal** — why this matters
3. **Constraint or rule** — what we're enforcing
```

**Good goals**:

- "Always present a saved WorkProduct to the frontend in a valid state"
- "Track upload provenance — how the WorkProduct was created"

**Bad goals** (too implementation-focused):

- "Add a CreationSource enum to enums.py"
- "Call MediaExtractor.extract() in upload_handler"

---

## Section 3: New Classes Summary

Architectural overview of all new/modified classes. This is the **most important section for human review**.

**All class design must follow `oop_style.md`** — methods belong to their data, models own their mutations, no hasattr/isinstance probing, dataclasses not dicts, etc.

### Class Cluster Diagram

ASCII diagram showing relationships between classes:

````markdown
### Class Cluster Diagram

```
                       Feature Name
  ExistingClass                    NewClass (ABC)
  (existing)                       (new)
       │                                │
       │ creates                        │ factory method
       ▼                                ▼
  ┌─────────────┐              ┌─────────────────────┐
  │  SomeModel  │◄─────────────│  Subclass1          │
  │  (existing) │  relationship│  Subclass2          │
  └─────────────┘              └─────────────────────┘
```
````

### Per-Class Documentation

For each new or significantly modified class:

```markdown
### ClassName (Type)

1-3 sentences: Purpose, scope, and key insight. Note if immutable, transient, etc.

| Method        | Signature                   | Purpose          |
| ------------- | --------------------------- | ---------------- |
| `method_name` | `(params...) -> ReturnType` | One-line purpose |

**Key notes** (if any):

- Important constraints or rules
- What this class does NOT do
```

**Example**:

```markdown
### CreationSource (Enum)

How a WorkProduct was created. **Immutable provenance** — set once at version 1, never changes.

| Value       | Purpose                                             |
| ----------- | --------------------------------------------------- |
| `GENERATED` | AI generation (default for backwards compatibility) |
| `UPLOADED`  | User-uploaded file                                  |

**Set at creation time** — not a state that transitions.
```

---

## Section 4: Process Flow

Sequential walkthrough of how the code executes. Use numbered steps with nested details. This shows the "how" at a readable level.

````markdown
## Process Flow

### Canonical Methods (if applicable)

All state changes go through these methods — no direct field assignment:

```python
obj.transition_a(...)  # → STATE_A + journaled
obj.transition_b(...)  # → STATE_B + journaled
```
````

### Flow

```
1. Entry point
   └── What happens first
   └── Can FAIL → what happens on failure

2. Next step
   └── extractor = SomeClass.factory_method(type)  ← FACTORY METHOD
   └── Details of what this does

3. External service call
   └── await service.do_thing(obj)  ← SINGLE PATHWAY (already exists)
   └── Can FAIL → obj.mark_failed(error)  ← CANONICAL METHOD

4. Final step
   └── obj.mark_completed()  ← CANONICAL METHOD
```

**Key principles**:

- Bullet points summarizing the important patterns
- What the human reviewer should verify

````

---

## Section 5: Implementation Parts

Detailed implementation guidance, organized by phase or logical grouping. This is where the "how" gets specific. **Each phase MUST have a checklist** — concrete steps that can be checked off during execution.

```markdown
## Part A: First Logical Unit

### Checklist
- [ ] Create `path/to/file.py` with ClassName
- [ ] Implement `method_name()` with signature X
- [ ] Add unit tests in `test_file.py` (list specific test cases)
- [ ] Update imports in caller modules
- [ ] Update docs if this phase changes documented behavior

### Design Decision (if non-obvious)

Explanation of why we chose this approach.

### Code Structure

```python
class Example:
    """Docstring with Scope and Not responsible for."""

    def method(self, ...) -> ReturnType:
        """What this does."""
        # Implementation sketch
````

### Notes

- Important implementation details
- Edge cases to handle

````

---

## Section 6: Files to Modify

Clear list of what changes where:

```markdown
## Files to Modify/Create

### New Files
- `path/to/new.py` — One-line description

### Modified Files
- `path/to/existing.py` — What changes (e.g., "Add CreationSource enum")
````

---

## Section 7: Verification

How to verify the implementation is correct. **Every plan must address tests and docs.**

```markdown
## Verification

1. **Unit tests**: What new tests to add, what existing tests to update
2. **Test coverage**: Ensure all new code paths have tests — no untested branches
3. **Integration test**: End-to-end scenario if applicable
4. **Documentation**: Which existing docs need updates (CLAUDE.md, arch docs, usage docs)
5. **Backwards compatibility**: What to check, migration path if needed
```

**Tests are not optional.** If a phase adds logic, it adds tests. List specific test cases, not just "add tests".

---

## Section 8: Goal Audit (when applicable)

Some plans have concrete, testable goals (reliability invariants, performance targets, security
properties). For these plans, add a post-implementation audit step that traces the actual code
against each goal.

**When to include:** Plans with goals that are verifiable properties of the system — "never lose
data", "handle deploys seamlessly", "sub-100ms latency". These need proof, not just tests passing.

**When to skip:** Exploratory plans, brainstorming docs, refactors with no behavioral goals,
plans where the goals are just "clean up X" or "add feature Y." If the goals are satisfied by
the tests passing, a separate audit adds nothing.

```markdown
## Goal Audit

After implementation is complete, trace the code to verify each goal:

| Goal                | How to verify                                  | Code path                     | Confirmed? |
| ------------------- | ---------------------------------------------- | ----------------------------- | ---------- |
| Never lose X        | Trace the failure scenario through actual code | `file.py:123` → `file.py:456` | ✅ / ❌    |
| Handle Y seamlessly | Run test scenario Z, check logs                | Integration test result       | ✅ / ❌    |

For each goal:

1. Identify the specific code path that satisfies it
2. Identify the scenario that would violate it
3. Trace that scenario through the implementation
4. Confirm the goal holds or document why it doesn't
```

This is the "measure twice" step — it catches gaps that unit tests miss because it
focuses on system-level properties, not individual function behavior.

---

## Section 9: Resolved Questions

Track design decisions made during planning:

```markdown
## Resolved Questions

1. ~~**Question?**~~ **RESOLVED**: Answer and rationale.
2. ~~**Another question?**~~ **RESOLVED**: Decision made.
```

---

## Extracting Design from Existing Code

This same structure can document existing subsystems. When reverse-engineering:

1. **Goals**: Infer from class docstrings and usage patterns
2. **Class Summary**: Document actual classes with their interfaces
3. **Process Flow**: Trace through a typical execution path
4. **Implementation Parts**: Document non-obvious patterns

Use this prompt:

```
Examine [subsystem/feature] and create a design document following plan_style.md:
1. What are the high-level goals this code achieves?
2. What classes are involved and how do they relate?
3. What's the process flow for [typical operation]?
4. What patterns or decisions should future maintainers understand?
```

---

## Style Notes

- **Be precise about ownership**: "X does NOT handle Y (that's Z's job)"
- **Mark immutability**: "set once at creation, never changes"
- **Flag shared code**: "← SHARED (path/to/module.py)"
- **Note canonical methods**: "← CANONICAL METHOD" for single-source-of-truth functions
- **Cross-reference OOP rules**: "OOP rule 35: dataclasses not dicts"

---

## Plan Review Checklist

Before submitting a plan for review:

**Location & Naming:**

- [ ] Plan is saved in a `plans/` directory
- [ ] Filename is descriptive (e.g., `media_uploader_plan.md`, not `plan1.md`)
- [ ] `Plan-Created:` and `Plan-Modified:` dates are present at the top

**Source Code Study:**

- [ ] Relevant source files have been read and understood
- [ ] Existing patterns and abstractions are identified and followed
- [ ] Integration points are verified against actual code
- [ ] Post-write verification pass completed — plan matches source reality

**Content Quality:**

- [ ] Goals are high-level and human-reviewable (not implementation steps)
- [ ] Class design follows `oop_style.md` principles
- [ ] Class diagram shows relationships, not just a list
- [ ] Each class has purpose, scope, and interface documented
- [ ] Process flow is sequential and shows failure paths
- [ ] Canonical methods and shared code are clearly marked
- [ ] Each implementation phase has a concrete checklist
- [ ] Tests specified per phase (specific test cases, not just "add tests")
- [ ] Docs to update are identified
- [ ] Files to modify is complete
- [ ] Resolved questions capture design decisions
- [ ] Goal audit included (if goals are verifiable system properties — skip for exploratory plans)

---

## Executing Plans

When asked to execute a plan, follow this workflow:

### 1. Execute All Phases by Default

**Default behavior: execute the entire plan in one shot.** Don't pause between phases unless:

- The plan explicitly notes a stopping point (e.g., "Stop here for review")
- User requests phase-by-phase execution
- User asks to execute only up to a certain phase

If none of these apply, just keep going until the plan is complete.

### 2. Track Progress with Tasks

Use Claude's task system to track plan execution:

1. **At start**: Create a task for the plan execution:

   ```
   TaskCreate:
     subject: "Execute: <plan-name>"
     description: "Implementing <plan-path>"
     activeForm: "Executing plan"
   ```

   Then mark it `in_progress`.

2. **During execution**:
   - Work through checklist items in order
   - Check off items in the plan as you complete them
   - If you hit a blocker, note it and ask the user

3. **At completion**: Mark the task `completed`.

**Why this matters**: If execution is interrupted, the task remains `in_progress`.
Running `TaskList` shows incomplete work, making it easy to resume.

### 3. Completion Criteria

**A plan is complete when ALL checklist items are done:**

1. Every `- [ ]` must become `- [x]`
2. "Tests pass" is NOT sufficient — all plan items must be done
3. If you want to skip items, STOP and ask the user first
4. Don't declare victory while checklist items remain
