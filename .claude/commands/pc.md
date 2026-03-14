---
description: Plan quality check - systematic review passes before submission
allowed-tools: Task, TaskCreate, TaskUpdate, TaskList, Read, Write, Edit, Glob, Grep, Bash(*)
argument-hint: [plan-name] [pass-name]
---

## Context (Preloaded)

### Plans Directory

!`find . -path "*/plans/*.md" -type f 2>/dev/null | head -10 || echo "(no plans found)"`

---

## Instructions

The user invoked `/pc $ARGUMENTS`

### Argument Parsing

Parse `$ARGUMENTS` to extract:

- **plan**: Path to plan file (optional - defaults to most recently modified)
- **pass**: Specific pass to run (optional - defaults to all passes)

**Valid pass names:** `structure`, `oop`, `tests`, `codebase`, `design`

Legacy aliases (mapped automatically): `style` → `structure`, `goals` → `structure`, `redundancy` → `codebase`

**Examples:**

- `/pc` → all passes on most recent plan
- `/pc artifact_indexing` → all passes on plan matching that name
- `/pc oop` → just OOP pass on most recent plan
- `/pc artifact_indexing oop` → just OOP pass on specific plan

**Plan resolution:**

1. If argument looks like a path (contains `/` or ends in `.md`), use directly
2. If argument is a word, check if it's a pass name first
3. If not a pass name, search for matching plan in `**/plans/` directories
4. If no plan specified, use most recently modified from preloaded context

---

## Question Mode

**Triggered by:** First argument is `q`, `question`, or `questions`

When invoked in question mode, analyze the plan and ask the user clarifying questions. Do NOT run the full quality check.

### What to Look For

1. **Ambiguous requirements** — goals that could be interpreted multiple ways
2. **Missing context** — references to systems/patterns you don't fully understand
3. **Design choices** — places where the plan picks one approach but alternatives exist
4. **Scope boundaries** — what's explicitly in/out of scope that might be wrong
5. **Edge cases** — scenarios the plan doesn't address
6. **Dependencies** — assumptions about existing code that might not hold
7. **Testing gaps** — behaviors that are hard to test or verification is unclear

### Execution

1. Read the plan file thoroughly
2. Identify 3-7 genuine questions (not nitpicks)
3. Present them to the user using `AskUserQuestion` or as a numbered list
4. Wait for answers before proceeding
5. If answers require plan updates, make them

**Do NOT:**

- Ask rhetorical questions
- Ask things you could verify by reading code
- Run the full quality check steps

---

## Execution

### Step 1: Resolve Plan Path and Detect Style

Determine the target plan file. If it doesn't exist, report error and stop.

**After resolving the plan file, read it and check for a `**Style:**` line in the header.** This determines which style guide governs the plan:

- If the `**Style:**` line contains `bug_plan_style.md` → this is a **bug plan**. Read `.claude/guidance/bug_plan_style.md` as the primary style guide (it inherits from `plan_style.md`).
- If the `**Style:**` line contains `plan_style.md` or is absent → this is a **feature plan**. Read `.claude/guidance/plan_style.md` as the style guide.

**Pass the detected style type (`bug` or `feature`) to every subagent prompt** so each pass knows what to check.

### Step 2: Create Tracking Task

Create a task to track the plan check progress:

```
TaskCreate:
  subject: "Plan check: <plan-name>"
  description: |
    Quality check for: <plan-path>

    ## Checklist
    - [ ] Analysis passes (structure, oop, tests, codebase, design)
    - [ ] Consolidation (if needed)
  activeForm: "Checking plan quality"
```

Then mark it in_progress.

### Step 3: Run Passes

**If a specific pass was requested:** Run only that pass as a single subagent using
the prompt from "Pass Definitions" below. Append this to the prompt: "After reporting
findings, **update the plan** to fix any issues you find."

**If running all passes:** Launch the coordinator subagent as described below.

---

## All-Passes Mode: Coordinator

Runs in a Sonnet coordinator subagent to keep the main context small and orchestration fast.

### Coordinator Launch

Launch a single coordinator subagent that manages the full parallel pipeline. **This
keeps all intermediate results (pass findings, triage decisions) out of the main
context — the coordinator accumulates state in its own fresh context window.**

Task configuration:

- `subagent_type="general-purpose"`, `model="sonnet"`

**Include in the coordinator prompt:**

1. The plan file path
2. The detected plan style (`bug` or `feature`)
3. ALL instructions from the **Coordinator Pipeline** section below
4. ALL prompts from the **Pass Definitions** section below (subagents can't inherit context)
5. The **Design Scout Prompt** and **Design Judge Prompt** from below

The coordinator returns the final summary text. After receiving it:

1. Update the tracking task description to check off completed items
2. Mark the task as completed
3. Print the coordinator's summary to the user

---

## Coordinator Pipeline

**These instructions are for the coordinator subagent. Include them verbatim in the
coordinator's prompt — subagents don't inherit parent context.**

You are a plan quality check coordinator. Run a parallel analysis pipeline: launch
sub-agents, collect results, triage findings, optionally consolidate, and return a
summary.

### Step A: Parallel Launch

Launch ALL of the following simultaneously in ONE message to maximize parallelism:

**A1. Fast analysis passes** (4 background agents):

Launch each as a **background** Task agent (`run_in_background=true`):

| Pass      | Model    | Subagent Type     |
| --------- | -------- | ----------------- |
| Structure | `haiku`  | `general-purpose` |
| OOP       | `sonnet` | `general-purpose` |
| Tests     | `haiku`  | `general-purpose` |
| Codebase  | `sonnet` | `general-purpose` |

Use the pass prompts from the Pass Definitions section. Each pass is **read-only** —
it analyzes and reports findings but does NOT edit the plan.

**A2. Design scout** (1 foreground agent):

Launch one Task with `subagent_type="Explore"`, `model="sonnet"`.

Use the **Design Scout Prompt** below. The scout gathers codebase context relevant to
the plan's design — reads architecture docs, traces callers/callees, identifies
implicit contracts. Returns a structured context brief. Does NOT evaluate design
compliance.

### Step B: Design Judge

After the design scout returns with its context brief, launch the design judge:

- `subagent_type="Explore"`, `model="opus"`

Use the **Design Judge Prompt** below, inserting the scout's context brief. The judge
evaluates design compliance using the pre-gathered context. Pure reasoning — no
codebase exploration needed.

### Step C: Collect Results

After the design judge returns, collect all 4 background task results (from Step A1).
Use TaskOutput to retrieve each background agent's result.

### Step D: Triage + Conditional Consolidation

Scan each pass's output plus the design judge output. Classify each as:

- **clean** — "no issues found" or only trivial observations
- **has findings** — specific issues, gaps, or violations reported

**If all passes clean:** Skip consolidation. Return: "All 5 passes clean — no changes
needed."

**If 1+ passes have findings:** Launch the consolidation subagent.

**Consolidation subagent:**

- `subagent_type="general-purpose"`, `model="sonnet"`

Prompt:

```
You are consolidating plan quality check results and applying fixes.

Plan file: <plan-path>
Plan style: <bug or feature>
Style guide: `.claude/guidance/<bug_plan_style.md or plan_style.md>`

## Findings from Analysis Passes

### Structure
<paste structure pass output>

### OOP
<paste oop pass output>

### Tests
<paste tests pass output>

### Codebase
<paste codebase pass output>

### Design Intent
<paste design judge output>

## Your Task

1. Read the plan file and the applicable style guide
2. Review ALL findings from the 5 analysis passes above
3. Apply all necessary changes to the plan in one coherent edit:
   - Fix structural issues and close goal-implementation gaps (structure)
   - Fix OOP style violations (oop)
   - Add missing test specifications (tests)
   - Add "Reuse Existing Infrastructure" section if redundancy was found,
     and correct wrong assumptions about existing code (codebase)
   - Address design-intent gaps — missing invariants, violated conventions,
     overlooked interactions with neighboring subsystems (design)
4. **If plan style is `bug`:** Verify the diagnostic coherence chain holds
   (Problem → Fix → Reasoning → Repro → Test → Implementation). Fix any broken links.
5. Resolve any conflicts between pass recommendations (design findings take priority
   over cosmetic fixes)
6. Do a final coherence check — verify the plan reads well after all edits
7. Report: summary of all changes made, and whether the plan is ready for submission
```

### Step E: Return Summary

Return the final summary to the main agent:

- If consolidation ran: the consolidation subagent's report
- If all clean: "All 5 passes clean — no changes needed"

---

## Pass Definitions

Each prompt below is for Phase 1 (read-only analysis mode). When running a single
pass via argument, append the edit instruction noted in Step 3.

### Pass 1: Structure (`structure`)

**Model:** `haiku`

Combines style/formatting checks with goal-implementation alignment. Both are
plan-text-only analysis — no codebase searching needed.

**Subagent prompt:**

```
You are analyzing plan quality. Run the STRUCTURE pass only.
Report findings — do NOT edit the plan.

Plan file: <plan-path>
Plan style: <bug or feature>

## Part A: Style & Formatting

Read `.claude/guidance/plan_style.md` (structure sections only), then verify the plan has:
- Plan saved in a `plans/` directory
- `Plan-Created:` date present
- `Style:` line referencing its style guide
- High-level goals section (not implementation steps)
- Class diagrams showing relationships (if new/modified classes exist)
- Sequential process flow with failure paths
- Concrete checklist per phase
- Specific tests per phase

**If plan style is `bug`:** Also read `.claude/guidance/bug_plan_style.md` and verify:
- `Plan-Executed:` field present (blank is fine)
- Problem Statement section exists and is specific/falsifiable
- Proposed Fix section exists (plain language, before any code)
- Reasoning section exists with root cause analysis
- Reproduction Steps section exists (honest about reliability)
- Regression Test section exists (or explicit justification for why not)
- Sections appear in the correct order per bug_plan_style.md

## Part B: Goal-Implementation Alignment

Read the plan's Goals section, then for each stated goal verify:
- Does the design/implementation address this goal?
- Are there gaps where a goal is stated but not implemented?
- Are there implementation details that don't serve any stated goal?

**If plan style is `bug`:** Also verify the diagnostic coherence chain
(see `.claude/guidance/bug_plan_style.md` Coherence Check):
1. Problem Statement describes a real, observable issue
2. Proposed Fix directly addresses that issue
3. Reasoning explains *why* the fix is correct (root cause, not symptom)
4. Reproduction Steps can trigger the problem in (1)
5. Regression Test will fail due to (1) and pass due to (2)
6. Implementation executes what (2) describes, justified by (3)

Flag any broken links in the chain.

## Report

List findings from both parts. Be specific — cite section names, goal names,
and what's missing or misaligned.
```

### Pass 2: OOP Style (`oop`)

**Model:** `sonnet`

**Subagent prompt:**

```
You are analyzing plan quality. Run the OOP STYLE pass only.
Report findings — do NOT edit the plan.

Plan file: <plan-path>
Plan style: <bug or feature>

Read `.claude/guidance/oop_style.md` in full, then read the plan file.

For each class/method in the plan, verify:
- Methods belong to their data (not caller classes)
- No hasattr/isinstance probing for capabilities
- Dataclasses not ad-hoc dicts
- Method sizes reasonable (5-15 lines target)
- Models own their mutations

Report what you checked and list any OOP style violations.
Be specific — cite class/method names and which principle is violated.
```

### Pass 3: Tests (`tests`)

**Model:** `haiku`

**Subagent prompt:**

```
You are analyzing plan quality. Run the TEST COVERAGE pass only.
Report findings — do NOT edit the plan.

Plan file: <plan-path>
Plan style: <bug or feature>

For each implementation phase in the plan, verify:
- Every new class/method has corresponding test cases specified
- Edge cases and error paths are tested
- Tests are specific (not just "add tests")
- Tests actually verify the stated goals

**If plan style is `bug`:** Also verify the Regression Test section
(see `.claude/guidance/bug_plan_style.md`):
- A specific regression test is defined with test location and before/after behavior
- The test sketch is concrete enough to implement
- The test actually targets the bug described in the Problem Statement
- OR: an explicit justification is given for why a regression test isn't feasible,
  with partial tests or alternative verification described

Report what you checked and list any missing test specifications.
Be specific — cite which classes/methods lack tests and suggest what should be tested.
```

### Pass 4: Codebase Reality (`codebase`)

**Model:** `sonnet`

Combines codebase verification (do referenced things exist?) with redundancy
detection (does equivalent code already exist?). Both require searching the
codebase, so a single agent avoids duplicate grep work.

**Subagent prompt:**

```
You are analyzing plan quality. Run the CODEBASE REALITY pass only.
Report findings — do NOT edit the plan.

Plan file: <plan-path>
Plan style: <bug or feature>

## Part A: Verification

For each class/method the plan references or extends:
- Grep/read the actual source file
- Verify it exists with expected signature
- Verify existing patterns are followed, not reinvented
- Check that assumptions about existing code are correct

## Part B: Redundancy Detection

**Goal: Identify any proposed code that already exists in the codebase.**

Read the plan and list ALL proposed new functions, classes, and methods.

For each, search the codebase:

**By name similarity:** Search for similar function/class names in the relevant
directories.

**By purpose (most duplications are same purpose, different name):**

| If plan proposes... | Search for... |
|---------------------|---------------|
| Downloading/fetching files | `download`, `fetch`, `urllib`, `requests.get` |
| File output/temp files | `tempfile`, `NamedTemporaryFile`, `tmp` |
| Data validation | `validate_`, `is_valid`, `check_` |
| API clients | `client`, `api`, `service` |
| Caching | `cache`, `memoize`, `lru_cache` |
| Retry logic | `retry`, `backoff`, `attempt` |

Check shared/utils modules and service-specific utils.

## Report

**Part A:** List what you verified and any incorrect assumptions or mismatched
signatures. Cite file:line for each verification.

**Part B:** List what new code is proposed, what existing code serves the same
purpose (with file paths), and specific recommendations to reuse vs. reimplement.
```

### Pass 5: Design Intent (`design`)

**Model:** `opus`

Used for **single-pass mode only** (`/pc design`). When running all passes, the
coordinator uses the split scout/judge pipeline instead (see Coordinator Pipeline).

**Subagent prompt:**

```
You are analyzing plan quality. Run the DESIGN INTENT pass only.
Report findings — do NOT edit the plan.

Plan file: <plan-path>
Plan style: <bug or feature>

## Your Job

You are the reviewer who asks "yes, but have you considered how this affects X?"
Your job is NOT to check syntax, style, or whether the code compiles. Other passes
handle that. Your job is to determine whether this plan **fits the design of the
system it's changing** — whether it respects the invariants, conventions, and
architectural intent that make the system coherent.

## Step 1: Understand the Plan

Read the plan file thoroughly. Identify:
- What specific code changes are proposed (files, classes, methods, behavior changes)
- What subsystem(s) the changes live in
- What the plan is trying to achieve (goals/problem statement)

## Step 2: Targeted Blast Radius Check

Focus your codebase exploration on what's **directly relevant** to the changes:

1. **Read relevant architecture docs** for the subsystem being changed.

2. **Trace direct callers and callees** — grep for the functions/methods being
   changed. Check one level out: who calls them, what do they call.

3. **Check for implicit contracts** — look for code that assumes the current behavior.

## Step 3: Evaluate Design Compliance

For the interactions found in Step 2, check:

1. **Invariant preservation** — Does the plan maintain the invariants that neighboring
   code depends on?

2. **Convention consistency** — Does the plan follow the same patterns used elsewhere
   for the same kind of change?

3. **Missing interactions** — Are there subsystems that should be updated alongside
   this change but aren't mentioned?

4. **Failure mode coherence** — Does the error handling match the system's conventions?

## Step 4: Report

### Subsystems Examined
List the architecture docs read and code areas explored (briefly).

### Design Compliance Findings
For each finding:
- **What the plan does** (specific change)
- **What the system expects** (invariant, convention, or implicit contract)
- **Whether they align** (compliant, gap, or conflict)
- **If gap/conflict: what's missing** (specific suggestion)

### Verdict
One of:
- **Compliant** — no issues found.
- **Minor gaps** — mostly sound but overlooks N small interactions. List them.
- **Significant concern** — may violate an architectural invariant or miss a critical
  interaction. Explain what and why.

Be specific. Cite file paths, function names, and architecture doc sections.
Do not flag cosmetic issues — focus exclusively on whether the change fits the
broader system design.
```

---

## Design Scout Prompt

Used by the coordinator in all-passes mode. The scout gathers codebase context for
the design judge — pure exploration, no evaluation.

**Model:** `sonnet` | **Subagent type:** `Explore`

```
You are the design scout for a plan quality check. Your job is to gather codebase
context that the design judge will use to evaluate design compliance.

Do NOT evaluate the plan. Do NOT report findings. Just gather context and return
a structured brief.

Plan file: <plan-path>
Plan style: <bug or feature>

## Step 1: Understand the Plan

Read the plan file thoroughly. Identify:
- What specific code changes are proposed (files, classes, methods, behavior changes)
- What subsystem(s) the changes live in
- What the plan is trying to achieve (goals/problem statement)

## Step 2: Gather Context

1. **Architecture docs:** Read relevant docs for the subsystem being changed.
   Summarize the key design invariants and conventions described.

2. **Callers and callees:** Grep for the functions/methods being changed. For each,
   note who calls it and what it calls (one level out). Include file paths and
   brief descriptions.

3. **Implicit contracts:** Look for code that assumes the current behavior of the
   things being changed. Note any patterns like "X always happens after Y" or
   "Z is always non-null at this point."

4. **Neighboring subsystems:** Identify subsystems that interact with the changed
   code and might be affected.

## Step 3: Return Context Brief

Return a structured brief with these sections:

### Plan Summary
What the plan proposes (2-3 sentences).

### Subsystems Involved
Which subsystems are being changed, with file paths.

### Architecture Doc Excerpts
Key invariants and conventions from the relevant docs.

### Caller/Callee Map
For each changed function/method:
- Who calls it (file:function)
- What it calls (file:function)

### Implicit Contracts
Any assumptions neighboring code makes about the current behavior.

### Neighboring Subsystems
What else might be affected and why.

Keep the brief focused and factual. The judge will use it for reasoning.
```

---

## Design Judge Prompt

Used by the coordinator in all-passes mode. The judge evaluates design compliance
using the scout's pre-gathered context — pure reasoning, no codebase exploration.

**Model:** `opus` | **Subagent type:** `Explore`

```
You are the design judge for a plan quality check. You evaluate whether a plan
fits the design of the system it's changing.

Report findings — do NOT edit the plan.

Plan file: <plan-path>
Plan style: <bug or feature>

## Pre-Gathered Context

The design scout has already explored the codebase. Here is the context brief:

<insert scout's context brief here>

## Your Job

You are the reviewer who asks "yes, but have you considered how this affects X?"
Your job is NOT to check syntax, style, or whether the code compiles. Other passes
handle that. Your job is to determine whether this plan **fits the design of the
system it's changing** — whether it respects the invariants, conventions, and
architectural intent that make the system coherent.

Using the context brief above (you may also read the plan file and any files
referenced in the brief if you need more detail):

## Step 1: Evaluate Design Compliance

For each interaction identified by the scout, check:

1. **Invariant preservation** — Does the plan maintain the invariants that neighboring
   code depends on?

2. **Convention consistency** — Does the plan follow the same patterns used elsewhere
   for the same kind of change?

3. **Missing interactions** — Are there subsystems that should be updated alongside
   this change but aren't mentioned?

4. **Failure mode coherence** — Does the error handling match the system's conventions?

## Step 2: Report

### Subsystems Examined
List the architecture docs and code areas from the scout's brief.

### Design Compliance Findings
For each finding:
- **What the plan does** (specific change)
- **What the system expects** (invariant, convention, or implicit contract)
- **Whether they align** (compliant, gap, or conflict)
- **If gap/conflict: what's missing** (specific suggestion)

### Verdict
One of:
- **Compliant** — no issues found.
- **Minor gaps** — mostly sound but overlooks N small interactions. List them.
- **Significant concern** — may violate an architectural invariant or miss a critical
  interaction. Explain what and why.

Be specific. Cite file paths, function names, and architecture doc sections.
Do not flag cosmetic issues — focus exclusively on whether the change fits the
broader system design.
```

---

## Execution Flow

1. **Resolve plan** → report which plan you're checking
2. **Create tracking task** → TaskCreate with checklist, then TaskUpdate to in_progress
3. **If single pass requested:**
   - Launch one subagent (with edit permission)
   - Mark task completed
   - Report results
4. **If all passes:**
   a. Announce: "Running plan quality check..."
   b. Launch Sonnet coordinator with plan details + full pipeline instructions
   c. Coordinator runs pipeline internally (parallel passes → design judge → triage → consolidation)
   d. Receive summary from coordinator
   e. Mark task completed
   f. Report summary to user

**IMPORTANT**: Always complete the task tracking. If the command is interrupted, the
task remains in_progress — this makes it easy to see incomplete work via TaskList.

---

## Key Principles

- **Coordinator pattern**: A Sonnet coordinator manages the full pipeline — launching passes, collecting results, triaging, consolidating. All intermediate results stay in the coordinator's context, keeping the main context clean and orchestration fast.
- **Design pass split**: Codebase exploration (Sonnet scout) and design reasoning (Opus judge) run as separate agents. The scout runs in parallel with fast passes; the judge gets pre-gathered context so Opus focuses purely on high-value reasoning.
- **Parallel analysis, conditional consolidation**: Analysis passes are read-only and run simultaneously. Only the consolidation pass edits the plan — and only when issues are found. Clean plans skip straight to the report.
- **Model-appropriate allocation**: Haiku for structural checks, Sonnet for exploration/reasoning/orchestration, Opus for design judgment only.
- **Merged passes reduce overhead**: Structure (style+goals) and codebase (verification+redundancy) each combine complementary checks that share the same input source, avoiding duplicate work.
- **Fresh context**: Each subagent gets focused attention without context pollution.
- **Be specific**: Report exactly what was checked and changed.
