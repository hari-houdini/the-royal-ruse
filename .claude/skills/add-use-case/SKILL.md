---
name: add-use-case
description: Add a new Use Case to the domain layer. Use when a new feature needs to be added to the game loop, a new player action needs to be orchestrated, or an existing use case needs to be extended. Triggers include "add a use case", "implement X mechanic", "new feature in game loop", "orchestrate X flow".
---

# Skill: Add a Use Case

A Use Case orchestrates domain entities and domain services through Port interfaces to fulfil one specific player action or system event. It contains no business logic itself — it delegates everything. It never imports an adapter directly.

## Before You Start

1. Read `docs/prd/PRD-1a-godot-scaffold.md` for the port interfaces available.
2. Confirm the use case does not already exist in `src/core/use_cases/`.
3. Identify which Port interfaces it needs — never import adapters.
4. Identify which Domain services or entities it will orchestrate.
5. If the use case needs a new GameBus signal, use the `add-gamebus-signal` skill first.

## Steps

### Step 1 — Write the failing GUT test (TDD — Red)

Create `test/unit/use_cases/test_your_use_case_name.gd` before touching implementation.

```gdscript
# test/unit/use_cases/test_your_use_case_name.gd
extends GutTest

# Declare mock dependencies — inject these into the use case
var _mock_network: MockNetworkPort
var _mock_auth: MockAuthPort
var _use_case: YourUseCaseName

func before_each() -> void:
    _mock_network = MockNetworkPort.new()
    _mock_auth = MockAuthPort.new()
    _mock_auth.configure_signed_in("player_001")
    _use_case = YourUseCaseName.new(_mock_network, _mock_auth)

func test_succeeds_when_valid_input() -> void:
    # Arrange
    var payload := { "key": "value" }
    # Act
    var result: Result = await _use_case.execute(payload)
    # Assert
    assert_true(result.success, "Expected success for valid input")

func test_fails_when_required_field_missing() -> void:
    var result: Result = await _use_case.execute({})
    assert_false(result.success)
    assert_string_contains(result.error_message, "required_field_name")

func test_fails_when_not_authenticated() -> void:
    _mock_auth.configure_signed_out()
    var result: Result = await _use_case.execute({"key": "value"})
    assert_false(result.success)
    assert_string_contains(result.error_message, "authenticated")
```

Run tests: `godot --headless -s addons/gut/gut_cmdln.gd -gdir=res://test/ -ginclude_subdirs -gexit`
Confirm the test fails (red). Proceed to Step 2.

### Step 2 — Implement the Use Case (TDD — Green)

```gdscript
# src/core/use_cases/your_use_case_name.gd
class_name YourUseCaseName
extends RefCounted

## One-line summary of what this use case does.
## [param network] The active network port from ServiceLocator.
## [param auth] The active auth port from ServiceLocator.

var _network: INetworkPort
var _auth: IAuthPort

func _init(network: INetworkPort, auth: IAuthPort) -> void:
    _network = network
    _auth = auth

## Executes the use case.
## [param payload] Dictionary containing required input fields.
## Returns Result.ok(data) on success, Result.fail(message) on validation or network failure.
func execute(payload: Dictionary) -> Result:
    # 1. Validate inputs — always first, before any port calls
    var required_field: String = payload.get("required_field_name", "")
    if required_field.is_empty():
        return Result.fail("required_field_name cannot be empty")

    # 2. Validate auth if player identity is required
    if _auth.get_current_player_id().is_empty():
        return Result.fail("Player must be authenticated before calling YourUseCaseName")

    # 3. Build the intent payload
    var intent := {
        "player_id": _auth.get_current_player_id(),
        "required_field_name": required_field
    }

    # 4. Call port(s) — never call adapters directly
    var result: Result = await _network.broadcast_event("your_event_name", intent)
    if not result.success:
        return Result.fail("Network error: %s" % result.error_message)

    # 5. Return result
    return Result.ok(result.data)
```

Run tests. Confirm they pass (green).

### Step 3 — Wire into ServiceLocator and Scene (TDD — Refactor)

In the Scene that triggers this use case:

```gdscript
# Inside a scene script
func _on_button_pressed() -> void:
    var use_case := YourUseCaseName.new(
        ServiceLocator.network(),
        ServiceLocator.auth()
    )
    var result: Result = await use_case.execute({"required_field_name": _some_value})
    if not result.success:
        _show_error(result.error_message)
        return
    # Handle success
```

### Step 4 — Checklist Before Committing

- [ ] Test file exists in `test/unit/use_cases/`
- [ ] Tests cover: valid input success, each validation failure, auth guard
- [ ] Use case is in `src/core/use_cases/`
- [ ] Uses only port interfaces — no adapter imports
- [ ] Every function has a `##` doc comment
- [ ] All variables are typed
- [ ] Returns `Result` — not raw values
- [ ] `await` used on all async port calls
- [ ] All GUT tests pass headless

## Common Mistakes

- Importing `SupabaseNetworkAdapter` directly — use `INetworkPort` only
- Calculating game state or scores inside the use case — delegate to domain services
- Emitting `GameBus` signals from inside the use case — emit from the scene or the adapter's signal handler
- Forgetting `await` on port methods that return a Result from a network call
