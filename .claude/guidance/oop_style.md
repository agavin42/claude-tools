# OOP Style Guide

Principles for clean, maintainable class design in Python and TypeScript.

## Core Philosophy

- **Small logical work units.** Every method does one thing. If you can name what a block does, extract it.
- **No repetition.** Similar code twice → extract. Shared behavior across classes → inheritance or mixins.
- **Self-documenting names.** Good names eliminate comments.

---

## Class Docstrings

Every class needs a docstring explaining **purpose and scope** — what this class is responsible for, and equally important, what it's _not_ responsible for. This helps both humans and AI understand the class's boundaries.

```python
class PromptCompiler:
    """
    Compiles user prompts with @ref syntax into resolved prompts for generation backends.

    Scope:
    - Parses @ref references and resolves them to assets
    - Groups refs by role (identity, style, etc.)
    - Produces ResolvedPrompt with text + resolved refs

    Not responsible for:
    - Actual generation (that's the Generator's job)
    - Persisting results (caller handles storage)
    - Validating that referenced assets exist (assumes valid refs)
    """
```

**What to include:**

- **One-line summary** — What does this class do?
- **Scope** — Key responsibilities (bullet points)
- **Not responsible for** — Boundaries and what's delegated elsewhere
- **Key concepts** — If the class introduces domain terms, define them briefly

**Why this matters for AI:** LLMs benefit enormously from explicit scope statements. Without them, the AI must infer boundaries from implementation — error-prone. Clear docstrings prevent the AI from adding logic that belongs elsewhere.

---

## Method Design

1. **Method size: 5-15 lines.** Predicates: 1-5. State transitions: 5-15. Complex orchestration: extract helpers. Functions/methods over ~50 lines are candidates for splitting.

2. **Extract relentlessly.** When you see a logical unit (validate, transform, save), make it a method.

3. **Predicate prefixes:** `is_` (state), `has_` (existence), `can_` (ability), `should_` (policy), `does_` (action check). All return bool.

4. **Properties for derived state.** Simple computation from other fields → `@property`. Side effects or expensive → method.

5. **Property docstrings required.** Every `@property` must have a one-line docstring explaining what it returns.

6. **Declare all attributes at class level with types.** Initialize in `__init__` (Python) or constructor (TypeScript).

---

## Class Hierarchy

7. **Inheritance for specialization.** Base provides interface + defaults. Subclasses override specific behavior.

8. **Mixins for cross-cutting concerns.** Multiple unrelated classes need same capability → mixin.

9. **State machines with enums.** Explicit states prevent invalid transitions.

10. **Match for multi-branch logic.** Cleaner than if/elif chains when dispatching on type/enum.

---

## Encapsulation

11. **Internal vs public.** `_method()` = implementation detail. `method()` = public API.

12. **Models own their mutations.** Type-specific field logic lives in the model hierarchy via polymorphism, not in callers with if/elif chains. Each class handles its fields and delegates to `super()`.

13. **Methods live with the class they know most about.** Put method on the class whose internals it overlaps most.

14. **Properties hide peculiar logic.** When accessing a value requires any logic — even simple logic — wrap it in a `@property`. This centralizes the "how" and lets callers just use the "what". Especially important for derived state, conditional access, or anything non-obvious.

```python
# Good — logic is hidden, callers don't need to know the rule
@property
def effective_prompt(self) -> str:
    """The prompt to use, falling back to parent if not set."""
    return self.prompt or self.parent.prompt

# Bad — logic leaks to every caller
prompt = node.prompt or node.parent.prompt  # repeated everywhere
```

15. **Wrappers must add abstraction.** A function that just calls a method on its parameter adds nothing — inline it. Valid wrappers add error handling, defaults, or adapt interfaces.

16. **Factory methods for common configurations.** Same constructor pattern repeats → add `for_X()` or `from_X()` factory.

17. **Properties for computed type info.** `asset.effective_type` not `_get_base_type(asset)`. Base returns default; subclasses override.

18. **Never access `_private` from outside.** Add public property or method instead.

19. **NEVER use hasattr() or isinstance() to check class capabilities.** When you need to check if a node has `sources` or `canonicals` or any optional capability, DON'T write `hasattr(node, "sources")` or `isinstance(node, WorkProduct)`. Instead, add a method to the base class that returns a safe default (`[]`, `None`, `False`), and override in subclasses. Example: `node.get_sources()` returns `[]` on BaseNode, returns `self.sources` on WorkProduct. Callers just iterate — no type checking needed.

20. **General utilities in utils module.** Cross-module standalone functions. Keep small — domain helpers stay in domain.

---

## Single Source of Truth

21. **One function for each important test.** When code needs to check a condition in multiple places, create a single predicate method that is _the_ source of truth for that test. Document what it means in the docstring. All callers use this method — no one reimplements the check inline.

```python
# Good — single source of truth for "ready to render"
def is_ready_for_render(self) -> bool:
    """True if this asset has approved canonicals and valid prompt."""
    return self.has_canonicals() and self.prompt_info.is_valid()

# Bad — logic repeated and potentially inconsistent
if node.canonicals and node.prompt_info and node.prompt_info.text:  # caller 1
if len(node.canonicals) > 0 and node.prompt_info.is_valid():        # caller 2
```

22. **One function for each important state change.** When an object transitions to a significant state (approved, failed, archived, initialized), route _all_ paths through a single method. Even if the transition can be triggered from multiple places, they all call the same function. This method is where you document what the transition means, enforce invariants, and emit signals.

```python
# Good — all approval paths go through here
def mark_approved(self, approved_by: str) -> None:
    """
    Transition to approved state. Sets approval metadata and
    clears any pending review flags. Called from UI approval,
    batch approval, and auto-approval workflows.
    """
    self._status = ApprovalStatus.APPROVED
    self._approved_by = approved_by
    self._approved_at = datetime.now()
    self._clear_review_flags()

# Bad — approval logic scattered
node.status = ApprovalStatus.APPROVED  # in handler A
node.status = ApprovalStatus.APPROVED  # in handler B, forgot metadata
```

23. **Lifecycle phases funnel through canonical methods.** When an object has lifecycle phases (create → validate → process → complete), each phase transition should have one method that _all_ code paths use. Even if "complete" can happen via success, timeout, or cancellation — they all call `_enter_completed_state()` which handles cleanup, logging, and invariants.

```python
# Good — completion always goes through one place
def _enter_completed_state(self, reason: CompletionReason) -> None:
    """Called when processing ends, regardless of how. Handles cleanup."""
    self._cleanup_temp_resources()
    self._status = Status.COMPLETED
    self._completion_reason = reason
    self._completed_at = datetime.now()

def complete_success(self) -> None:
    self._enter_completed_state(CompletionReason.SUCCESS)

def complete_timeout(self) -> None:
    self._enter_completed_state(CompletionReason.TIMEOUT)

def complete_cancelled(self) -> None:
    self._enter_completed_state(CompletionReason.CANCELLED)
```

---

## Convenience Methods

24. **Gateway objects own convenience methods.** When multiple callers construct same helpers to call one method, move that into the gateway they already have. Indicators: repeated helper construction, chained calls, duplicated persistence logic.

25. **Layered convenience: check vs ensure.** Provide `get_X()` (no side effects) and `ensure_X()` (may fetch/persist) variants. Callers pick appropriate level.

26. **Services stay focused, gateways orchestrate.** Service layer = pure I/O. Gateway layer = orchestrates services + persistence. Callers use gateway, ignorant of services.

---

## Extensibility & Dispatch

**Smell**: A central loop/function with `if op == "X": ... elif op == "Y": ...` that grows whenever you add a new type. Adding a new type means modifying the orchestrator.

**Fix**: Each type is a class that knows how to handle itself. Orchestrator asks "what can you do?" and calls standard methods. New type = new class, orchestrator unchanged.

27. **New types = new classes, not modified switch statements.** When adding a new variant means editing a central dispatcher, the design is wrong. Each variant should be its own class with polymorphic methods. The orchestrator loops over objects and calls the same interface on each.

28. **Ask objects, don't inspect them.** Replace `if isinstance(x, Foo)` or `if x.type == "foo"` with a method call. The object knows what it is. Add methods like `x.execute()`, `x.can_poll()`, `x.get_timeout()` that return type-appropriate values.

29. **Isolate new functionality.** Adding a new operation/handler/formatter should require creating one new file with one new class. If you also have to edit 3 other files to "wire it in," the design leaks.

---

## Multi-Component Systems

30. **Gateway pattern for subsystem coordination.** When multiple components must work together (registry, connection manager, event router), create a gateway class that owns them all. Gateway provides unified introspection (`get_status()`) and lifecycle management. Callers use gateway, ignorant of subsystems.

31. **Immutable snapshots for introspection.** Never expose mutable internals. Return frozen `*Info` dataclasses from `to_info()` methods. Caller gets a snapshot they can safely pass around without affecting system state.

32. **Callbacks for cross-component communication.** When component A needs to notify component B of events (connection added → update project count), use callback functions set at initialization. Avoids tight coupling and circular dependencies.

33. **Secondary indexes for efficient lookups.** When you need fast access by multiple keys (connections by session_id, by project, by user), maintain secondary index dicts alongside primary storage. Update all indexes atomically in add/remove operations.

34. **Eviction protection via state checks.** When resources can be evicted/unloaded but shouldn't be while in use, add `is_evictable` property that checks active usage (connection count, pending operations). Eviction code checks this before removal.

---

## Documentation & Comments

35. **Field docs above, not bunched.** Document each field with a comment directly above it, not in a big block at class top.

```python
# Good
class Asset:
    # Unique identifier, e.g., "a1b2c3d4"
    id: str
    # Human-readable name, e.g., "gandalf_the_grey"
    name: str

# Bad — bunched at top
class Asset:
    # id: unique identifier
    # name: human-readable name
    id: str
    name: str
```

36. **Identifiers must be crystal clear.** Always include examples in comments for id/name fields. Distinguish between node identity (`id`), ref paths (`name`), and parsed user input. Ambiguous identifiers cause bugs.

37. **Derive, don't store redundantly.** If a value can be computed from other fields, compute it. Don't store both `width`, `height`, and `aspect_ratio` — derive aspect_ratio. Exception: expensive computations that are accessed frequently.

---

## Data Modeling

38. **Dataclasses, not ad-hoc dicts.** When passing structured data between functions or modules, use proper dataclasses (or typed structures), not dictionaries. Dicts used as structs are a code smell — they lack type safety, can't be validated, and obscure the data contract. If you find yourself returning `{"id": x, "name": y, "status": z}`, define a class for it.

39. **Enums for constrained values.** Use enums instead of strings for fixed sets (`AspectRatio`, `Resolution`, not `"16:9"`). Catches typos at definition time.

40. **Models can have methods.** Pure data transforms are fine on models — computing derived values, validating state, formatting for display. But **no I/O in models** — no file access, no network calls, no database queries. Models are data + logic about that data.

41. **Separate concerns into proper abstractions.** When code is doing detailed inspection of data structures (parsing dicts, checking types, formatting variants), ask: does this logic belong here, or does it belong with the data? Extract variant logic into classes with duck-typed methods; core code just calls those methods.

---

## Code Organization

42. **No barrel imports.** Don't use `__init__.py` to re-export everything. Use explicit imports: `from mypackage.models.user import User`, not `from mypackage.models import User`.

43. **Imports at top of file.** All imports go at the top, organized as: stdlib → third-party → local. The ONLY exception is imports inside functions to avoid circular dependencies — and those MUST have a comment explaining why.

44. **Build only what's needed.** Leave out unused features we don't have a complete design for. Add TODO comments for future work. Don't build speculative infrastructure.

45. **Minimal accessors.** Only add properties/getters that are actively used or very likely to be used. Don't add `get_foo()` just because `foo` exists.

---

## Model vs. Service Boundary

46. **Model methods**: state transitions (`mark_completed`), derived values (`effective_prompt`), validation (`is_valid`), display formatting. All operate on model's own fields.

47. **Service functions**: external API calls, multi-service orchestration, retry/timeout logic, anything that needs mocking in tests. Lives in separate service/ops modules.

**Test**: Does it need anything outside the model's data? No → model method. Yes → service function.

---

## Cross-Class Design

The preceding sections address individual class design. These address the spaces _between_ classes — how they compose into coherent architecture.

48. **State has one owner.** Every piece of mutable state has exactly one class or module that owns it. Other code reads via queries and mutates via the owner's methods. When the same state is managed from N places with N different mutation patterns, extract into one owner with a clear query/mutation API.

49. **Parse at the boundary, not at a distance.** Raw data (dicts, JSON, untyped structures) should be parsed into typed objects at the module boundary where it enters the system. Downstream code works with typed objects only. If you find raw-data parsing scattered across files, add a typed wrapper at the entry point.

50. **One concern, one home.** Each cross-cutting concern (selection, navigation, context menus, error handling) has one authoritative module. All code touching that concern routes through it. Finding the same concern reimplemented across N files with N different behaviors means a module is missing.

51. **Interfaces reflect the caller's model, not the implementation.** A module's public API should mirror how callers think about the problem. If using the API requires knowing internal structure (which dict keys exist, which fields are populated at which lifecycle phase, internal class hierarchy), the abstraction is leaking.

52. **Shared shape → shared base.** When peer types share fields, methods, or behavioral patterns but don't share a base class, they need one. The test: if you're writing `isinstance` or `if type ==` to dispatch different behavior across peer types, they should have a common interface with polymorphic methods instead.

53. **Same concept, same behavior.** When the same concept (displaying an icon, formatting a reference, handling an error) appears in multiple places, it should work the same way everywhere. Different code is fine; different _behavior_ for the same concept confuses users and creates bugs.

---

## Language-Specific Notes

### Python

- Use `msgspec.Struct`, `@dataclass`, or Pydantic models — not dicts
- Imports: stdlib → third-party → local
- Use `match` statements for enum/type dispatch (Python 3.10+)

### TypeScript

- Use explicit function components, not `React.FC`
- Use interfaces/types, not `any`
- Prefer `const` arrow functions for React components
- Use discriminated unions for state machines

---

## Checklist

### Class Structure

- Do classes have docstrings explaining purpose, scope, and boundaries?
- Do all `@property` methods have one-line docstrings?
- Are all class attributes declared with types?
- Is shared behavior in base classes or mixins?

### Methods

- Can each method be described in one short phrase?
- Are methods under 15 lines? (50 max for complex orchestration)
- Do predicates use `is_/has_/can_/should_/does_` prefixes?
- Is any code repeated? Extract it.

### Encapsulation

- Do models handle their own field mutations via polymorphism?
- Do methods live with the class whose internals they know?
- Do properties hide peculiar/conditional logic from callers?
- Do wrappers add abstraction, not just delegate?
- Do config objects have factory methods for common patterns?
- Is computed type info exposed as properties?
- Are cross-class accesses using public methods only?
- **Is there any `hasattr()` or `isinstance()` to check class capabilities? REMOVE IT.** Add duck-typed methods to base class instead.

### Single Source of Truth

- Is each important test a single predicate method (not logic repeated inline)?
- Do all paths to a state change go through one canonical method?
- Do lifecycle phase transitions funnel through canonical methods?
- Are state transition methods documented with what they mean and who calls them?

### Documentation & Comments

- Are field docs above each field, not bunched at top?
- Do id/name fields have example values in comments?
- Is derived data computed, not stored redundantly?

### Data Modeling

- Are structured return values dataclasses, not dicts?
- Are constrained values enums, not strings?
- Do models have methods for pure transforms (no I/O)?
- Is data inspection logic in the data class, not callers?

### Code Organization

- Are imports at top of file (stdlib → third-party → local)?
- Are there no barrel imports via `__init__.py`?
- Are unused features left out (with TODOs for future)?
- Are there no speculative accessors?

### Extensibility & Dispatch

- Can you add a new type/variant by creating one new class (no editing central dispatcher)?
- Do objects know how to handle themselves (not inspected externally with isinstance/type checks)?
- Is new functionality isolated to new files (not scattered edits to wire things in)?

### Cross-Class Design

- Does each piece of mutable state have exactly one owner module/class?
- Is raw data parsed at module boundaries (not scattered across consumers)?
- Does each cross-cutting concern have one authoritative module?
- Do module APIs reflect the caller's mental model (not implementation structure)?
- Do peer types with shared behavior share a base class or interface?
- Is the same concept handled consistently across all contexts?

### Multi-Component Systems

- Are general utilities in a utils module?
- Do gateways own convenience methods for repeated caller patterns?
- Are there both check and ensure variants where needed?
- Do multi-component systems have a gateway class for coordination?
- Are mutable internals exposed only via immutable `*Info` snapshots?
- Are secondary indexes maintained for multi-key lookups?
