---
name: add-gamebus-signal
description: Add a new domain event signal to GameBus. Use when a new game state change needs to be observable by one or more scenes without tight coupling. Triggers include "new domain event", "add a signal", "broadcast X to all scenes", "scenes need to react to X".
---

# Skill: Add a GameBus Signal

`GameBus` is the global signal hub. Scenes subscribe to it for domain events. Use Cases and adapters emit to it via `GameBus.signal_name.emit(data)`. It is an autoload — always available, always running.

**Rule:** Never emit a GameBus signal from inside an Adapter. Adapters emit their own port signals. The Use Case or scene handler that receives the port signal then emits to GameBus.

## Before You Start

1. Confirm the signal does not already exist in `src/autoloads/game_bus.gd`.
2. Name the signal in past-tense `snake_case` — e.g., `scoring_resolved`, `roles_assigned`, `player_kicked`.
3. Define the typed payload parameters — be specific. `data: Dictionary` is acceptable only if the shape is documented.
4. Identify which Use Case or adapter will emit it and which scenes will subscribe.

## Steps

### Step 1 — Add the Signal to GameBus

```gdscript
# In src/autoloads/game_bus.gd — add below the last existing signal:

## Emitted when [describe the event in one sentence].
## [param your_param] [What this contains and why scenes need it.]
signal your_signal_name(your_param: YourType)
```

**Naming rules:**
- Past tense verb: `scoring_resolved` not `resolve_scoring`
- Noun phrase for state: `player_connection_changed` not `connection_change`
- Never generic: `updated`, `changed`, `event` — always specific

### Step 2 — Emit the Signal

Signals are emitted from the Use Case (after receiving a resolved Result) or from an adapter's signal handler that the Use Case subscribes to:

```gdscript
# In a Use Case or in SceneManager's signal handler:
GameBus.your_signal_name.emit(your_data)

# Example — in scoring resolution handler:
func _on_scoring_edge_function_responded(round_scores: Array[Dictionary]) -> void:
    # Process the response...
    GameBus.scoring_resolved.emit(round_scores)
```

### Step 3 — Subscribe in Scenes

```gdscript
# In any scene that needs to react:
func _ready() -> void:
    GameBus.your_signal_name.connect(_on_your_signal_name)

func _exit_tree() -> void:
    if GameBus.your_signal_name.is_connected(_on_your_signal_name):
        GameBus.your_signal_name.disconnect(_on_your_signal_name)

func _on_your_signal_name(your_param: YourType) -> void:
    # Update UI or trigger further logic
    pass
```

### Step 4 — Write a GUT Test

```gdscript
# test/unit/domain/test_game_bus_signals.gd
extends GutTest

func test_your_signal_name_is_declared() -> void:
    # GUT can verify a signal exists on an object
    assert_has_signal(GameBus, "your_signal_name",
        "GameBus should declare the your_signal_name signal")

func test_your_signal_name_emits_with_correct_type() -> void:
    var received_param: YourType = null
    GameBus.your_signal_name.connect(func(p: YourType) -> void: received_param = p)

    var test_data := YourType.new(...)
    GameBus.your_signal_name.emit(test_data)

    assert_not_null(received_param, "Signal should have fired with a payload")
```

### Signal Catalogue Convention

Every signal added must be documented inline in `game_bus.gd` with:
- A `##` doc comment describing when it fires
- All `[param name]` documentation for each parameter
- A note on which layer emits it (Use Case, adapter handler, SceneManager)

### Checklist

- [ ] Signal declared in `src/autoloads/game_bus.gd` with full `##` doc comment
- [ ] Signal name is past-tense snake_case
- [ ] All parameters are typed (no untyped `data`)
- [ ] Emitter identified and implemented
- [ ] At least one subscriber identified and connected in `_ready()`
- [ ] Subscriber disconnects in `_exit_tree()` to avoid dangling connections
- [ ] GUT test confirms signal is declared on `GameBus`

## Full GameBus Signal Reference

All signals currently declared in `game_bus.gd` — check here before adding to avoid duplicates:

```
session_status_changed(new_status: GameSession.Status)
roles_assigned(assignments: Array[Dictionary])
discussion_opened(duration_secs: int)
vote_resolved(accused_player_id: String, outcome: String)
scoring_resolved(round_scores: Array[Dictionary])
alliance_evaluated(alliance_data: Dictionary)
player_connection_changed(player_id: String, is_online: bool)
round_ended(cumulative_scores: Array[Dictionary])
game_ended(final_scores: Array[Dictionary])
```
