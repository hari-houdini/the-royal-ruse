---
name: add-port-and-adapter
description: Add a new Port interface and one or more concrete Adapter implementations. Use when introducing a new external dependency (a new service, transport layer, or I/O concern) that domain logic needs to communicate with. Triggers include "new external dependency", "add support for X service", "add offline support for X", "integrate with Y API".
---

# Skill: Add a Port Interface and Adapter

A Port is an abstract interface that defines *what* the domain can do with an external concern. An Adapter is the concrete implementation that fulfils that contract using a real service (Supabase, local WiFi, file system, etc.).

The domain layer never knows which adapter is active. `ServiceLocator` wires them at startup.

## Before You Start

1. Confirm the concern does not already fit into an existing Port (`INetworkPort`, `IAuthPort`, `IStoragePort`, `IChatPort`, `INameGeneratorPort`).
2. Name the port in the form `IThingPort` — present tense, noun, interface prefix `I`.
3. Identify which signals the domain needs to react to (these go on the Port).
4. Identify which adapters you need now (online) vs later (offline/Phase 2).

## Steps

### Step 1 — Define the Port Interface

```gdscript
# src/core/ports/i_your_thing_port.gd
class_name IYourThingPort
extends RefCounted

## Abstract port interface for [describe the external concern].
## All concrete adapters MUST extend this class and override every method.

# ── Signals ───────────────────────────────────────────────────────────────────
## Emitted when [describe when this signal fires].
signal thing_happened(data: Dictionary)

## Emitted when [describe the error signal].
signal thing_failed(reason: String)

# ── Methods ───────────────────────────────────────────────────────────────────

## [One-line description of what this method achieves.]
## [param param_name] Description.
## Returns Result.ok(data) on success, Result.fail(message) on error.
func do_thing(param_name: String) -> Result:
    push_error("IYourThingPort.do_thing() not implemented by %s" % get_class())
    return Result.fail("not implemented")

## [Second method description.]
func get_status() -> bool:
    push_error("IYourThingPort.get_status() not implemented by %s" % get_class())
    return false
```

### Step 2 — Write the Mock (for TDD)

Create the mock before the real adapter. Tests use the mock; real code uses the adapter.

```gdscript
# test/unit/mocks/mock_your_thing_port.gd
class_name MockYourThingPort
extends IYourThingPort

## Controllable fake implementation of IYourThingPort for use in GUT tests.

var _should_fail: bool = false
var _last_param: String = ""
var _call_count: int = 0

# ── Test controls ─────────────────────────────────────────────────────────────

func configure_failure() -> void:
    _should_fail = true

func get_last_param() -> String:
    return _last_param

func get_call_count() -> int:
    return _call_count

# ── IYourThingPort implementation ─────────────────────────────────────────────

func do_thing(param_name: String) -> Result:
    _call_count += 1
    _last_param = param_name
    if _should_fail:
        return Result.fail("Simulated failure in MockYourThingPort")
    return Result.ok({"param": param_name})

func get_status() -> bool:
    return not _should_fail
```

### Step 3 — Implement the Online Adapter

```gdscript
# src/adapters/supabase/supabase_your_thing_adapter.gd
class_name SupabaseYourThingAdapter
extends IYourThingPort

## Concrete implementation of IYourThingPort using [Supabase / HTTP / etc.].

# ── Private state ─────────────────────────────────────────────────────────────
var _is_ready: bool = false

# ── IYourThingPort implementation ─────────────────────────────────────────────

func do_thing(param_name: String) -> Result:
    if param_name.is_empty():
        return Result.fail("param_name cannot be empty")

    # Make the actual external call here (HTTP, WebSocket, etc.)
    # On success:
    thing_happened.emit({"param": param_name})
    return Result.ok()

    # On failure:
    # thing_failed.emit("reason")
    # return Result.fail("reason")

func get_status() -> bool:
    return _is_ready
```

### Step 4 — Register in ServiceLocator

```gdscript
# In src/autoloads/service_locator.gd — add the new port field and wiring:

var _your_thing_port: IYourThingPort   # Add field

func _wire_online_adapters() -> void:
    # ... existing adapters ...
    _your_thing_port = SupabaseYourThingAdapter.new()

## Returns the active YourThing port.
func your_thing() -> IYourThingPort:
    return _your_thing_port
```

### Step 5 — Write GUT Tests for the Port Interface Base Class

```gdscript
# test/unit/domain/test_i_your_thing_port.gd
extends GutTest

func test_base_class_do_thing_fails_with_not_implemented() -> void:
    var port := IYourThingPort.new()
    var result: Result = await port.do_thing("test")
    assert_false(result.success)
    assert_string_contains(result.error_message, "not implemented")

func test_mock_succeeds_by_default() -> void:
    var mock := MockYourThingPort.new()
    var result: Result = await mock.do_thing("test_param")
    assert_true(result.success)

func test_mock_fails_when_configured() -> void:
    var mock := MockYourThingPort.new()
    mock.configure_failure()
    var result: Result = await mock.do_thing("test_param")
    assert_false(result.success)

func test_mock_records_last_param() -> void:
    var mock := MockYourThingPort.new()
    await mock.do_thing("hello_world")
    assert_eq(mock.get_last_param(), "hello_world")
```

### Step 6 — Checklist Before Committing

- [ ] Port interface in `src/core/ports/i_your_thing_port.gd`
- [ ] Port class_name prefixed with `I`
- [ ] Every method has `push_error("...not implemented...")` guard
- [ ] Every method returns `Result`
- [ ] Mock in `test/unit/mocks/mock_your_thing_port.gd`
- [ ] Mock implements every method on the port
- [ ] Mock has test-control methods (configure_failure, get_last_param, etc.)
- [ ] Adapter in `src/adapters/[service]/`
- [ ] ServiceLocator updated with new field and accessor
- [ ] `_wire_offline_adapters()` has a stub or Phase 2 note
- [ ] GUT tests pass for mock and base class

## Offline Adapter Stub (Phase 2 placeholder)

```gdscript
# src/adapters/local_wifi/local_wifi_your_thing_adapter.gd
class_name LocalWiFiYourThingAdapter
extends IYourThingPort

## Phase 2 stub — offline implementation of IYourThingPort.
## Returns Result.fail until implemented.

func do_thing(param_name: String) -> Result:
    return Result.fail("LocalWiFiYourThingAdapter not yet implemented (Phase 2)")

func get_status() -> bool:
    return false
```
