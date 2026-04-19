---
name: write-mock
description: Write a mock (fake) implementation of a Port interface for use in GUT tests. Use whenever a Use Case or Domain Service needs to be tested in isolation from real Supabase, HTTP, or file I/O. Triggers include "mock the X port", "fake adapter for testing", "create a mock", "stub the network port".
---

# Skill: Write a Mock Port

A mock is a controllable fake implementation of a Port interface. It lives in `test/unit/mocks/` and is never shipped to production. Mocks make domain logic testable without a live Supabase connection.

## Before You Start

1. Identify the Port interface being mocked (e.g., `INetworkPort`, `IAuthPort`).
2. Read the Port interface file to see every method signature — the mock must implement all of them.
3. Decide which test-control methods are needed (configure failure, get call count, get last argument).

## Mock Template

```gdscript
# test/unit/mocks/mock_your_port.gd
class_name MockYourPort
extends IYourPort

## Controllable fake implementation of IYourPort.
## Used exclusively in GUT tests — never in production code.
##
## Test controls (call these in before_each to configure behaviour):
##   configure_failure()        — makes the next call return Result.fail()
##   configure_signed_in(id)    — sets an authenticated player id (auth ports)
##   configure_response(data)   — sets the data returned in Result.ok()
##
## Test assertions (call these after act to verify interactions):
##   get_call_count()           — number of times the main method was called
##   get_last_argument()        — last value passed to the main method
##   was_called_with(value)     — true if any call had this argument

# ── Configuration State ───────────────────────────────────────────────────────
var _should_fail: bool = false
var _failure_message: String = "Simulated failure"
var _response_data: Variant = null

# ── Interaction Recording ─────────────────────────────────────────────────────
var _call_count: int = 0
var _call_log: Array[Variant] = []    # Stores all arguments for verification

# ── Test Controls ─────────────────────────────────────────────────────────────

## Makes the next (and all subsequent) calls return Result.fail(message).
func configure_failure(message: String = "Simulated failure") -> void:
    _should_fail = true
    _failure_message = message

## Resets failure configuration — subsequent calls will succeed.
func configure_success() -> void:
    _should_fail = false

## Sets the data value returned inside Result.ok() on success.
func configure_response(data: Variant) -> void:
    _response_data = data

# ── Test Assertions ───────────────────────────────────────────────────────────

## Returns how many times the primary method was called.
func get_call_count() -> int:
    return _call_count

## Returns the argument passed on the most recent call.
func get_last_argument() -> Variant:
    return _call_log.back() if not _call_log.is_empty() else null

## Returns true if any call was made with the given argument.
func was_called_with(value: Variant) -> bool:
    return _call_log.has(value)

## Resets all recorded interactions — call in before_each if reusing across tests.
func reset() -> void:
    _call_count = 0
    _call_log.clear()
    _should_fail = false
    _response_data = null

# ── IYourPort Implementation ──────────────────────────────────────────────────
# Implement every method from the port interface.

func do_thing(param: String) -> Result:
    _call_count += 1
    _call_log.append(param)
    if _should_fail:
        return Result.fail(_failure_message)
    return Result.ok(_response_data)

func get_status() -> bool:
    return not _should_fail
```

## Auth Port Mock (Reference — used in most Use Case tests)

```gdscript
# test/unit/mocks/mock_auth_port.gd
class_name MockAuthPort
extends IAuthPort

var _player_id: String = ""
var _is_authenticated: bool = false
var _sign_in_should_fail: bool = false

func configure_signed_in(player_id: String) -> void:
    _player_id = player_id
    _is_authenticated = true

func configure_signed_out() -> void:
    _player_id = ""
    _is_authenticated = false

func configure_sign_in_failure() -> void:
    _sign_in_should_fail = true

func sign_in_as_guest(display_name: String) -> Result:
    if _sign_in_should_fail:
        return Result.fail("Simulated auth failure")
    _player_id = "mock_guest_001"
    _is_authenticated = false
    signed_in.emit(_player_id, display_name, true)
    return Result.ok({"player_id": _player_id, "display_name": display_name})

func sign_in_with_google() -> Result:
    if _sign_in_should_fail:
        return Result.fail("Simulated Google auth failure")
    _player_id = "mock_google_001"
    _is_authenticated = true
    signed_in.emit(_player_id, "Mock Google User", false)
    return Result.ok({"player_id": _player_id, "display_name": "Mock Google User"})

func sign_out() -> Result:
    _player_id = ""
    _is_authenticated = false
    signed_out.emit()
    return Result.ok()

func get_current_player_id() -> String:
    return _player_id

func is_signed_in() -> bool:
    return _is_authenticated
```

## Network Port Mock (Reference)

```gdscript
# test/unit/mocks/mock_network_port.gd
class_name MockNetworkPort
extends INetworkPort

var _connected: bool = false
var _should_fail: bool = false
var _broadcast_log: Array[Dictionary] = []

func configure_connected() -> void:
    _connected = true

func configure_failure() -> void:
    _should_fail = true

func get_broadcast_log() -> Array[Dictionary]:
    return _broadcast_log

func was_broadcast(event_name: String) -> bool:
    for entry: Dictionary in _broadcast_log:
        if entry.get("event_name") == event_name:
            return true
    return false

func connect_to_session(session_code: String) -> Result:
    if _should_fail:
        return Result.fail("Simulated connection failure")
    _connected = true
    return Result.ok({"session_code": session_code})

func disconnect_from_session() -> void:
    _connected = false

func broadcast_event(event_name: String, payload: Dictionary) -> Result:
    if _should_fail:
        return Result.fail("Simulated broadcast failure")
    _broadcast_log.append({"event_name": event_name, "payload": payload})
    return Result.ok()

func send_private_event(player_id: String, event_name: String, payload: Dictionary) -> Result:
    _broadcast_log.append({"event_name": event_name, "payload": payload, "to": player_id})
    return Result.ok()

func is_connected() -> bool:
    return _connected
```

## Checklist

- [ ] File in `test/unit/mocks/` — never in `src/`
- [ ] Implements ALL methods from the port interface
- [ ] Has test-control methods (configure_*, reset)
- [ ] Records interactions (_call_count, _call_log)
- [ ] Has assertion helpers (get_call_count, was_called_with)
- [ ] Signals are emitted where the real adapter would emit them
- [ ] No real network, file, or Supabase calls anywhere in the mock
