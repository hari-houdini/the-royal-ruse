---
name: add-domain-entity
description: Add a new Domain Entity or Value Object to the core domain layer. Use when a new game concept needs to be modelled with identity and behaviour (Entity), or when a typed immutable wrapper is needed (Value Object). Triggers include "new game concept", "add a value object", "model X as a domain object", "new entity".
---

# Skill: Add a Domain Entity or Value Object

**Entity:** Has identity (a unique ID), mutable state, and behaviour. Example: `Player`, `GameSession`, `Round`.
**Value Object:** Immutable, no identity, defined entirely by its values. Example: `Result`, `SessionCode`, `RoleAssignment`.

Domain objects NEVER import from the adapter layer, NEVER call port interfaces, and NEVER depend on Godot scene nodes. They extend `RefCounted` only.

## Before You Start

1. Decide: Entity (has an ID, mutates over time) or Value Object (immutable, no ID)?
2. Confirm it does not already exist in `src/core/domain/entities/` or `src/core/domain/value_objects/`.
3. Identify which Use Cases will need it. Read their PRD sections.
4. If it introduces a new domain event, use `add-gamebus-signal` skill first.

## Steps

### Entity Template

```gdscript
# src/core/domain/entities/your_entity_name.gd
class_name YourEntityName
extends RefCounted

## [One-line description of what this entity represents in the game domain.]

# ── Signals ───────────────────────────────────────────────────────────────────
# Only emit signals if this entity's state change needs to be observed.
# Prefer GameBus signals for cross-system events.

# ── Constants ─────────────────────────────────────────────────────────────────
const MAX_SOMETHING: int = 10
const DEFAULT_VALUE: String = "default"

# ── Properties ────────────────────────────────────────────────────────────────
var entity_id: String          # UUID — always assigned at construction
var display_name: String
var some_count: int
var is_active: bool

# ── Constructor ───────────────────────────────────────────────────────────────

## Creates a new [YourEntityName] with the given identity and initial state.
## [param id] UUID identifying this entity. Must be non-empty.
## [param name] Human-readable display name.
func _init(id: String, name: String) -> void:
    assert(not id.is_empty(), "YourEntityName requires a non-empty id")
    entity_id = id
    display_name = name
    some_count = 0
    is_active = true

# ── Public behaviour ──────────────────────────────────────────────────────────

## [Describe what this mutation does and what invariants it enforces.]
## Returns Result.ok() on success, Result.fail(message) if preconditions not met.
func do_something(value: int) -> Result:
    if value < 0:
        return Result.fail("value cannot be negative")
    some_count += value
    return Result.ok()

## Returns true if this entity is in a valid state for [specific context].
func is_valid_for_something() -> bool:
    return is_active and some_count >= 0
```

### Value Object Template

```gdscript
# src/core/domain/value_objects/your_value_object.gd
class_name YourValueObject
extends RefCounted

## [One-line description. Value objects are immutable — no setters, no mutation.]

# ── Properties (read-only by convention — never mutate after _init) ───────────
var value: String
var secondary: int

# ── Constructor ───────────────────────────────────────────────────────────────

## Creates a [YourValueObject] from the given raw values.
## Raises an assertion error if the values are invalid.
func _init(raw_value: String, secondary_value: int) -> void:
    assert(not raw_value.is_empty(), "YourValueObject.value cannot be empty")
    assert(secondary_value >= 0, "YourValueObject.secondary cannot be negative")
    value = raw_value
    secondary = secondary_value

# ── Derived accessors ─────────────────────────────────────────────────────────

## Returns a human-readable string representation for UI display.
func to_display_string() -> String:
    return "%s (%d)" % [value, secondary]

## Returns true if this value object is equal to another of the same type.
func equals(other: YourValueObject) -> bool:
    return value == other.value and secondary == other.secondary
```

### GUT Tests — Entity

```gdscript
# test/unit/domain/test_your_entity_name.gd
extends GutTest

func test_init_sets_correct_id() -> void:
    var entity := YourEntityName.new("test_id", "Test Name")
    assert_eq(entity.entity_id, "test_id")

func test_init_sets_is_active_true() -> void:
    var entity := YourEntityName.new("test_id", "Test Name")
    assert_true(entity.is_active)

func test_do_something_with_valid_value_succeeds() -> void:
    var entity := YourEntityName.new("test_id", "Test Name")
    var result: Result = entity.do_something(5)
    assert_true(result.success)
    assert_eq(entity.some_count, 5)

func test_do_something_with_negative_value_fails() -> void:
    var entity := YourEntityName.new("test_id", "Test Name")
    var result: Result = entity.do_something(-1)
    assert_false(result.success)
    assert_string_contains(result.error_message, "negative")
```

### GUT Tests — Value Object

```gdscript
# test/unit/domain/test_your_value_object.gd
extends GutTest

func test_valid_construction_succeeds() -> void:
    var vo := YourValueObject.new("valid", 5)
    assert_eq(vo.value, "valid")
    assert_eq(vo.secondary, 5)

func test_equals_returns_true_for_same_values() -> void:
    var a := YourValueObject.new("x", 1)
    var b := YourValueObject.new("x", 1)
    assert_true(a.equals(b))

func test_equals_returns_false_for_different_values() -> void:
    var a := YourValueObject.new("x", 1)
    var b := YourValueObject.new("x", 2)
    assert_false(a.equals(b))
```

### Checklist

- [ ] Placed in correct directory (`entities/` or `value_objects/`)
- [ ] Extends `RefCounted` — not `Node`
- [ ] No imports from adapters, ports, or scenes
- [ ] No Supabase or Godot scene dependencies
- [ ] All properties typed
- [ ] Constructor validates invariants with `assert()`
- [ ] Value objects have no mutation methods after `_init`
- [ ] GUT tests cover construction, valid behaviour, invalid behaviour
- [ ] All tests pass headless
