---
name: add-scene
description: Add a new Godot scene with its accompanying GDScript controller. Use when a new game screen or UI phase is needed. Triggers include "new screen", "new UI scene", "add a scene for X", "create the vote screen", "new lobby UI".
---

# Skill: Add a Godot Scene

Scenes are the outermost layer of the architecture. They subscribe to `GameBus` signals to receive domain state, call Use Cases to send player intents, and navigate via `SceneManager`. They never import adapters, never calculate game state, and never hardcode file paths.

## Before You Start

1. Confirm which `GameBus` signals this scene needs to subscribe to.
2. Confirm which Use Cases this scene will invoke.
3. Decide if a new `GameBus` signal is needed — if yes, use `add-gamebus-signal` skill first.
4. Register the scene in `SceneManager.SCENE_PATHS` before navigating to it.

## Steps

### Step 1 — Register in SceneManager First

```gdscript
# In src/autoloads/scene_manager.gd — add the entry before creating the scene file:
const SCENE_PATHS: Dictionary = {
    # ... existing entries ...
    "your_scene": "res://src/scenes/your_scene/your_scene.tscn",
}
```

### Step 2 — Create the Scene Script

```gdscript
# src/scenes/your_scene/your_scene.gd
class_name YourScene
extends Control

## [One-line description of what this scene represents in the game flow.]
## Subscribes to: [list GameBus signals]
## Invokes: [list Use Cases]
## Navigates to: [list target scenes via SceneManager]

# ── State ─────────────────────────────────────────────────────────────────────
# Data this scene needs to display. Populated via GameBus signal handlers.
var _session_id: String = ""
var _players: Array[Dictionary] = []
var _is_admin: bool = false

# ── Lifecycle ─────────────────────────────────────────────────────────────────

func _ready() -> void:
    _connect_game_bus_signals()
    _build_placeholder_ui()

func _exit_tree() -> void:
    # Disconnect signals if needed to prevent memory leaks
    _disconnect_game_bus_signals()

# ── GameBus Signal Connections ────────────────────────────────────────────────

func _connect_game_bus_signals() -> void:
    GameBus.session_status_changed.connect(_on_session_status_changed)
    # Add other subscriptions here

func _disconnect_game_bus_signals() -> void:
    if GameBus.session_status_changed.is_connected(_on_session_status_changed):
        GameBus.session_status_changed.disconnect(_on_session_status_changed)

# ── GameBus Signal Handlers ───────────────────────────────────────────────────

func _on_session_status_changed(new_status: GameSession.Status) -> void:
    # React to relevant status changes only
    if new_status == GameSession.Status.YOUR_RELEVANT_STATUS:
        _update_display()

# ── User Intent Handlers ──────────────────────────────────────────────────────
# These are called by UI interactions (button presses, card taps, etc.)

func _on_action_button_pressed() -> void:
    var use_case := YourUseCaseName.new(
        ServiceLocator.network(),
        ServiceLocator.auth()
    )
    var result: Result = await use_case.execute({"key": _some_value})
    if not result.success:
        _show_error_message(result.error_message)
        return
    # Success — GameBus will fire and trigger the next transition

# ── Display Updates ───────────────────────────────────────────────────────────

func _update_display() -> void:
    pass  # Update labels, lists, timers based on current state

func _show_error_message(message: String) -> void:
    pass  # Display error to player — placeholder

# ── Placeholder UI ────────────────────────────────────────────────────────────
# Phase 1: Godot default Control theme — no art assets, functional only.
# Phase 2: This method is replaced with the illustrated UI.

func _build_placeholder_ui() -> void:
    # Build UI programmatically for Phase 1 placeholder.
    # In Phase 2, this is replaced by the .tscn scene tree.
    var title := Label.new()
    title.text = "YOUR SCENE NAME"
    add_child(title)
```

### Step 3 — Create the .tscn File

In the Godot editor:
1. Create a new scene (`Control` as root node)
2. Set the root node's script to `your_scene.gd`
3. Save as `src/scenes/your_scene/your_scene.tscn`
4. In Phase 1 — the `.tscn` may be nearly empty if `_build_placeholder_ui()` constructs everything programmatically

### Step 4 — Trigger Navigation To This Scene

Navigation always goes through `SceneManager`:

```gdscript
# From another scene or from SceneManager's GameBus handler:
SceneManager.go_to("your_scene")   # ✅

get_tree().change_scene_to_file("res://src/scenes/your_scene/your_scene.tscn")  # ❌ Never
```

If this scene is entered in response to a `GameBus.session_status_changed` signal, add the case to `SceneManager._on_session_status_changed()`:

```gdscript
func _on_session_status_changed(new_status: GameSession.Status) -> void:
    match new_status:
        # ... existing cases ...
        GameSession.Status.YOUR_STATUS:
            go_to("your_scene")
```

### Checklist

- [ ] Scene registered in `SceneManager.SCENE_PATHS` before any navigation
- [ ] Script extends `Control` (or appropriate Control subclass)
- [ ] `_connect_game_bus_signals()` called in `_ready()`
- [ ] `_disconnect_game_bus_signals()` called in `_exit_tree()`
- [ ] No adapter imports — only Use Cases and GameBus
- [ ] No hardcoded scene paths
- [ ] All Use Case calls `await`ed and `Result` checked
- [ ] `_build_placeholder_ui()` method exists for Phase 1
- [ ] `.tscn` file saved in `src/scenes/your_scene/`

## Signal Subscription Pattern (Reference)

```gdscript
# Subscribe once in _ready — pattern for all scenes
func _ready() -> void:
    GameBus.scoring_resolved.connect(_on_scoring_resolved)
    GameBus.roles_assigned.connect(_on_roles_assigned)

func _on_scoring_resolved(round_scores: Array[Dictionary]) -> void:
    for score_data: Dictionary in round_scores:
        _update_player_score_display(score_data)

func _on_roles_assigned(assignments: Array[Dictionary]) -> void:
    # Find this player's role
    var my_id: String = ServiceLocator.auth().get_current_player_id()
    for assignment: Dictionary in assignments:
        if assignment["player_id"] == my_id:
            _show_role_card(assignment)
            return
```
