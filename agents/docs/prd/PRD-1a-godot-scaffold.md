# PRD-1a — Godot Scaffold & CI Pipeline

| Field        | Value                                           |
|--------------|-------------------------------------------------|
| Version      | 1.0                                             |
| Date         | April 2026                                      |
| Status       | Approved                                        |
| Sub-Phase    | 1a of 6                                         |
| Duration     | Weeks 1–2 (parallel with PRD-1b)               |
| Depends On   | Nothing — this is the foundation                |
| Unlocks      | PRD-1c (Session, Lobby & Roles)                 |
| Parent PRD   | [PRD-000 — Main](./PRD-000-main.md)             |

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Solution](#2-solution)
3. [Design Choices & Justification](#3-design-choices--justification)
4. [User Stories](#4-user-stories)
5. [Implementation Decisions](#5-implementation-decisions)
6. [Testing Decisions](#6-testing-decisions)
7. [Out of Scope](#7-out-of-scope)
8. [Further Notes](#8-further-notes)

---

## 1. Problem Statement

There is no Godot project, no folder structure, no architectural skeleton, and no automated testing infrastructure. Every subsequent sub-phase will build on top of what is established here. If this foundation is wrong — wrong architecture pattern, wrong file structure, wrong testing setup — it becomes increasingly expensive to fix as more code accumulates on top of it.

The developer has zero prior Godot experience. The architecture pattern (Hexagonal / Ports and Adapters) must be established by file structure and convention from day one, not retrofitted later.

---

## 2. Solution

Create a Godot 4 project named `the_royal_ruse` with:

1. A **folder structure** that physically enforces the hexagonal architecture — domain, ports, and adapters live in separate directories and may not import from each other in the wrong direction
2. **Stub implementations** of every Port interface and every Domain entity/use case that Phase 1 will need — empty shells with correct signatures, no logic yet
3. **GUT** (Godot Unit Test) installed as an addon, with one passing smoke test to prove the test runner works
4. **GitHub Actions CI** that runs GUT tests on every push to `main` and every pull request
5. A **ServiceLocator autoload** that wires concrete adapters to port interfaces at startup — the single place where the whole application is assembled

---

## 3. Design Choices & Justification

### 3.1 GDScript vs C#

| Criterion            | GDScript                          | C#                                      |
|----------------------|-----------------------------------|-----------------------------------------|
| Learning curve       | ✅ Python-like, beginner-friendly | ❌ Requires .NET SDK, steeper for newcomers |
| Godot integration    | ✅ First-class, native signals    | ⚠️ Requires separate Mono/dotnet build  |
| Export compatibility | ✅ All platforms                  | ⚠️ Web export requires extra WASM step  |
| Performance          | ⚠️ Slower than C#                 | ✅ Faster for heavy computation          |
| Tooling (free)       | ✅ Built-in Godot editor           | ⚠️ Needs VS Code + OmniSharp or Rider   |

**Decision: GDScript.** Performance difference is irrelevant at this game's scale (turn-based social deduction, not a physics simulation). Web export simplicity and zero tooling cost are decisive. C# is reserved as a future option for specific hotspots only if profiling reveals a real bottleneck.

### 3.2 GUT vs Alternatives for Testing

| Framework      | Description                                  | Verdict                        |
|----------------|----------------------------------------------|--------------------------------|
| **GUT**        | Godot Unit Test — purpose-built for GDScript | ✅ Chosen. Largest community, active maintenance, GDScript-native |
| gdUnit4        | Alternative test framework for Godot 4       | ⚠️ Viable but smaller community |
| Raw assert()   | No framework — just assert() calls in scripts| ❌ No test discovery, no reporting, not scalable |
| Pytest (Python)| External test runner for GDScript            | ❌ Requires headless Godot + subprocess orchestration. Overkill. |

**Decision: GUT.** It installs as a Godot addon (no external dependencies), integrates with GitHub Actions via `godot --headless`, and has a mature `GutTest` base class with all standard assertion helpers.

### 3.3 GitHub Actions vs Alternatives for CI

| Tool              | Free Tier                          | Verdict                        |
|-------------------|------------------------------------|--------------------------------|
| **GitHub Actions**| 2,000 min/month (private), unlimited (public) | ✅ Chosen. Native to GitHub, large action ecosystem, `godot-ci` images available |
| GitLab CI         | 400 min/month                      | ❌ Less free time, requires GitLab migration |
| Bitbucket Pipelines| 50 min/month                      | ❌ Insufficient                |
| CircleCI          | 6,000 min/month (free)             | ⚠️ Viable alternative, but `godot-ci` images less maintained there |

**Decision: GitHub Actions.** The `barichello/godot-ci` Docker image provides a ready-made Godot 4 environment for headless test runs and cross-platform exports. Public repos get unlimited minutes — keep the repo public.

### 3.4 Hexagonal Architecture vs Alternatives

| Pattern             | Verdict for this project                                                    |
|---------------------|-----------------------------------------------------------------------------|
| **Hexagonal (Ports & Adapters)** | ✅ Chosen. Isolates domain from Supabase, enabling full TDD on game logic without a live backend |
| MVC                 | ❌ Domain leaks into controllers, Supabase calls bleed everywhere            |
| ECS (Entity Component System) | ⚠️ Godot's node system is already ECS-adjacent. Adding a formal ECS library (like GECS) adds complexity without benefit at this scale |
| Monolith (no pattern) | ❌ Impossible to unit test, guaranteed spaghetti by Phase 2               |

**Decision: Hexagonal.** The architecture enforced here determines testability for the entire project. The main constraint (server-authoritative scoring + swappable online/offline transport) is exactly the problem hexagonal architecture solves.

---

## 4. User Stories

1. As a developer, I want a pre-configured Godot project with a defined folder structure, so that I know exactly where every new file belongs without making ad-hoc decisions.
2. As a developer, I want every port interface stubbed with correct method signatures, so that I can implement use cases against a known API before the adapters are built.
3. As a developer, I want a ServiceLocator autoload that injects dependencies at startup, so that I can swap online/offline adapters without touching game logic.
4. As a developer, I want GUT installed and a working smoke test, so that I can verify the test runner is functional before writing real tests.
5. As a developer, I want GitHub Actions to run GUT tests on every push, so that broken domain logic is caught before it affects other sub-phases.
6. As a developer, I want a GameBus autoload for domain event broadcasting, so that scenes can react to game state changes without tight coupling to use cases.
7. As a developer, I want a Result value object for error handling, so that every use case returns a consistent success/failure structure instead of throwing exceptions.
8. As a developer, I want a clear coding standards document in the repo, so that every file I write is consistent from day one.
9. As a developer, I want the CI pipeline to fail loudly on any GUT test failure, so that regressions are never silently merged.
10. As a developer, I want stub scenes for Lobby, Game, Vote, and Score, so that scene navigation can be built in PRD-1c without starting from a blank canvas.

---

## 5. Implementation Decisions

### 5.1 Project Folder Structure

```
the_royal_ruse/
├── .github/
│   └── workflows/
│       ├── ci.yml                   # Runs GUT tests on push/PR
│       └── export.yml               # Builds web/Android/iOS (PRD-1f)
├── addons/
│   └── gut/                         # GUT test framework (installed via asset library)
├── src/
│   ├── core/
│   │   ├── domain/
│   │   │   ├── entities/            # Pure data + behaviour classes
│   │   │   ├── value_objects/       # Immutable typed wrappers
│   │   │   └── events/              # Domain event data classes
│   │   ├── use_cases/               # Application logic — orchestrates entities via ports
│   │   └── ports/                   # Abstract interfaces (no concrete logic)
│   ├── adapters/
│   │   ├── supabase/                # Online: Supabase Realtime, DB, Auth
│   │   ├── local_wifi/              # Offline: GodotTCPServer (Phase 2)
│   │   ├── guest/                   # Guest auth — ephemeral local identity
│   │   └── local_storage/           # Godot ConfigFile for offline score staging
│   ├── autoloads/
│   │   ├── service_locator.gd       # Wires ports to adapters at startup
│   │   └── game_bus.gd              # Global signal bus for domain events
│   └── scenes/
│       ├── lobby/
│       ├── game/
│       ├── vote/
│       └── score/
└── test/
    ├── unit/
    │   ├── domain/
    │   └── use_cases/
    └── smoke/                       # Minimal scene-level tests (post-scene-build)
```

> **Why separate `src/` from `test/`?** Godot exports only the `src/` directory to production builds. Test files in `test/` are excluded from exports via `.export_exclude` patterns. This keeps the shipped binary clean.

### 5.2 Module: Result Value Object

**What it is:** A tiny class that every use case returns. Instead of returning raw values or throwing exceptions (which GDScript handles poorly), every use case wraps its output in a Result. This gives callers a consistent way to check whether an operation succeeded and why it failed.

**Why:** GDScript has no exception hierarchy. Using a Result type is the industry-standard functional pattern for error handling without exceptions.

```gdscript
# res://src/core/domain/value_objects/result.gd
class_name Result
extends RefCounted

## Wraps any use case return value with a success flag and optional error message.
## Use Result.ok(data) for successes and Result.fail(message) for failures.

var success: bool
var data: Variant
var error_message: String

## Creates a successful Result, optionally wrapping returned data.
static func ok(payload: Variant = null) -> Result:
    var r := Result.new()
    r.success = true
    r.data = payload
    r.error_message = ""
    return r

## Creates a failed Result with a human-readable error message.
static func fail(message: String) -> Result:
    var r := Result.new()
    r.success = false
    r.data = null
    r.error_message = message
    return r
```

### 5.3 Module: Port Interfaces

**What they are:** Abstract base classes that define *what* a dependency can do, without specifying *how*. Use cases only ever call these interfaces — never concrete implementations. GDScript doesn't have formal interfaces, so we use base classes with `push_error()` guards on every method.

**Why:** If use cases import Supabase adapters directly, you cannot test them without a live Supabase connection. Port interfaces allow GUT tests to inject a fake (mock) adapter instead.

```gdscript
# res://src/core/ports/i_network_port.gd
class_name INetworkPort
extends RefCounted

## Emitted when any game event is received from the network layer.
signal game_event_received(event_name: String, payload: Dictionary)

## Emitted when a player joins or leaves the session.
signal presence_changed(player_id: String, is_online: bool)

## Emitted when the connection to the session is lost unexpectedly.
signal connection_lost()

## Connects this client to an existing session channel.
## [param session_code] The 6-character session identifier.
## Returns Result.ok() on success, Result.fail(message) on error.
func connect_to_session(session_code: String) -> Result:
    push_error("INetworkPort.connect_to_session() not implemented")
    return Result.fail("not implemented")

## Disconnects from the current session channel.
func disconnect_from_session() -> void:
    push_error("INetworkPort.disconnect_from_session() not implemented")

## Broadcasts a named event with a payload to all session participants.
## [param event_name] String identifier, e.g. "vote_cast", "round_started"
## [param payload] JSON-serialisable Dictionary
func broadcast_event(event_name: String, payload: Dictionary) -> Result:
    push_error("INetworkPort.broadcast_event() not implemented")
    return Result.fail("not implemented")

## Sends an event to a single player's private channel only.
func send_private_event(player_id: String, event_name: String, payload: Dictionary) -> Result:
    push_error("INetworkPort.send_private_event() not implemented")
    return Result.fail("not implemented")

## Returns true if currently connected to a session channel.
func is_connected() -> bool:
    push_error("INetworkPort.is_connected() not implemented")
    return false
```

```gdscript
# res://src/core/ports/i_auth_port.gd
class_name IAuthPort
extends RefCounted

## Emitted when sign-in completes (Google or guest).
signal signed_in(player_id: String, display_name: String, is_guest: bool)

## Emitted when sign-out completes.
signal signed_out()

## Initiates Google OAuth sign-in flow.
## Returns Result.ok({player_id, display_name}) on success.
func sign_in_with_google() -> Result:
    push_error("IAuthPort.sign_in_with_google() not implemented")
    return Result.fail("not implemented")

## Creates an ephemeral guest session with an auto-generated display name.
## Returns Result.ok({player_id, display_name}) on success.
func sign_in_as_guest(display_name: String) -> Result:
    push_error("IAuthPort.sign_in_as_guest() not implemented")
    return Result.fail("not implemented")

## Signs the current user out. For guests, clears local identity.
func sign_out() -> Result:
    push_error("IAuthPort.sign_out() not implemented")
    return Result.fail("not implemented")

## Returns the currently authenticated player's ID, or empty string if not signed in.
func get_current_player_id() -> String:
    push_error("IAuthPort.get_current_player_id() not implemented")
    return ""

## Returns true if the current user is authenticated via Google (not a guest).
func is_signed_in() -> bool:
    push_error("IAuthPort.is_signed_in() not implemented")
    return false
```

```gdscript
# res://src/core/ports/i_storage_port.gd
class_name IStoragePort
extends RefCounted

## Persists the final score for a player in a completed session.
## Called once per player at game end by the scoring Edge Function response handler.
func save_session_score(session_id: String, player_id: String, cumulative_points: int, final_rank: int) -> Result:
    push_error("IStoragePort.save_session_score() not implemented")
    return Result.fail("not implemented")

## Fetches the cumulative scores for all players in a session.
## Returns Result.ok(Array[Dictionary]) where each dict has player_id and cumulative_points.
func fetch_session_scores(session_id: String) -> Result:
    push_error("IStoragePort.fetch_session_scores() not implemented")
    return Result.fail("not implemented")

## Stages offline round data to local storage for later sync.
## Used only when playing without internet connectivity.
func stage_offline_round(round_data: Dictionary) -> Result:
    push_error("IStoragePort.stage_offline_round() not implemented")
    return Result.fail("not implemented")

## Retrieves all staged offline round data and attempts to sync to the server.
## Returns Result.ok(synced_count) on full sync, Result.fail on partial or full failure.
func sync_offline_data() -> Result:
    push_error("IStoragePort.sync_offline_data() not implemented")
    return Result.fail("not implemented")
```

```gdscript
# res://src/core/ports/i_chat_port.gd
class_name IChatPort
extends RefCounted

## Emitted when a new chat message arrives from any player in the session.
signal message_received(sender_id: String, sender_name: String, message: String, timestamp: int)

## Subscribes to the chat channel for the given session.
func join_chat_channel(session_id: String) -> Result:
    push_error("IChatPort.join_chat_channel() not implemented")
    return Result.fail("not implemented")

## Sends a text message to all players in the current session chat.
func send_message(session_id: String, message: String) -> Result:
    push_error("IChatPort.send_message() not implemented")
    return Result.fail("not implemented")

## Prevents any messages from being sent or received. Used during vote phase.
func lock_chat() -> void:
    push_error("IChatPort.lock_chat() not implemented")

## Re-enables chat. Used when a new discussion phase begins.
func unlock_chat() -> void:
    push_error("IChatPort.unlock_chat() not implemented")

## Returns true if chat is currently in a locked state.
func is_locked() -> bool:
    push_error("IChatPort.is_locked() not implemented")
    return false
```

```gdscript
# res://src/core/ports/i_name_generator_port.gd
class_name INameGeneratorPort
extends RefCounted

## Returns a funny, non-offensive auto-generated display name for a guest player.
## Format: "[Adjective] [Indian Royalty Noun]" — e.g. "Furious Nawab", "Wretched Diwan"
func generate_player_name() -> String:
    push_error("INameGeneratorPort.generate_player_name() not implemented")
    return ""

## Returns a unique 6-character alphanumeric session code. Uppercase, no ambiguous chars.
## Example output: "RUSE42", "GOLD7X"
func generate_session_code() -> String:
    push_error("INameGeneratorPort.generate_session_code() not implemented")
    return ""
```

### 5.4 Module: Domain Entities (Stubs)

**What they are:** Plain GDScript classes representing the core game concepts. At this stage, they are data-holding stubs — properties only, no logic. Logic arrives in PRD-1c through PRD-1e.

```gdscript
# res://src/core/domain/entities/player.gd
class_name Player
extends RefCounted

var player_id: String
var display_name: String
var is_guest: bool
var cumulative_points: int
var has_wanted_brand: bool  # Set when player escapes as Thief; persists across rounds
var is_active: bool         # False when disconnected

func _init(id: String, name: String, guest: bool) -> void:
    player_id = id
    display_name = name
    is_guest = guest
    cumulative_points = 0
    has_wanted_brand = false
    is_active = true
```

```gdscript
# res://src/core/domain/entities/game_session.gd
class_name GameSession
extends RefCounted

enum Status { LOBBY, ROUND_START, ROLE_REVEAL, DISCUSSION, VOTE, SCORING, ROUND_END, GAME_END }

var session_id: String
var session_code: String
var admin_player_id: String
var status: Status
var max_rounds: int
var current_round: int
var discussion_duration_secs: int
var players: Array[Player]

const MIN_PLAYERS: int = 4
const MAX_PLAYERS: int = 10
const MAX_ROUNDS: int = 20
const DEFAULT_DISCUSSION_SECS: int = 120
const LOBBY_TIMEOUT_SECS: int = 60
```

```gdscript
# res://src/core/domain/entities/round.gd
class_name Round
extends RefCounted

var round_id: String
var session_id: String
var round_number: int
var role_assignments: Array[Dictionary]   # [{player_id, role_id, base_points}]
var police_player_id: String
var thief_player_id: String
var accomplice_player_id: String          # Empty until Thief nominates
var accused_player_id: String             # Empty until Police votes
var outcome: String                       # "caught" | "escaped" | "timeout"
```

```gdscript
# res://src/core/domain/entities/role.gd
class_name Role
extends RefCounted

enum RoleId { KING, QUEEN, POLICE, THIEF, PRIME_MINISTER, GENERAL, SPY, MERCHANT, JESTER, PRIEST }

var role_id: RoleId
var display_name: String
var base_points: int
var min_players_required: int
var objective_description: String
var objective_bonus: int

# Static factory — returns all role definitions in hierarchy order
static func all_roles() -> Array[Role]:
    # Populated in PRD-1c
    return []
```

### 5.5 Module: GameBus Autoload

**What it is:** A global signal hub. Scenes subscribe to it to receive domain events without needing a direct reference to use cases. Like a radio station — scenes tune in; use cases broadcast.

**Why:** Godot's signal system requires a direct object reference to connect. GameBus eliminates that coupling. A score scene doesn't need to know about the scoring use case — it just listens to GameBus for `scoring_resolved`.

```gdscript
# res://src/autoloads/game_bus.gd
class_name GameBus
extends Node

## Emitted when the session's status changes (e.g. LOBBY → ROUND_START).
signal session_status_changed(new_status: GameSession.Status)

## Emitted when all role assignments for a round are ready.
signal roles_assigned(assignments: Array[Dictionary])

## Emitted when the discussion phase timer starts.
signal discussion_opened(duration_secs: int)

## Emitted when Police submits their vote or the countdown expires.
signal vote_resolved(accused_player_id: String, outcome: String)

## Emitted when the server returns fully resolved scores for the round.
signal scoring_resolved(round_scores: Array[Dictionary])

## Emitted when alliance evaluation is complete (revealed at score screen).
signal alliance_evaluated(alliance_data: Dictionary)

## Emitted when any player's connection status changes.
signal player_connection_changed(player_id: String, is_online: bool)

## Emitted when a round ends and cumulative scores are updated.
signal round_ended(cumulative_scores: Array[Dictionary])

## Emitted when the final round ends and the game is over.
signal game_ended(final_scores: Array[Dictionary])
```

### 5.6 Module: ServiceLocator Autoload

**What it is:** The single file where concrete adapters are wired to port interfaces. Everything else in the codebase asks ServiceLocator for a port — they never import adapters directly.

**Why:** This is the only place where `online mode vs offline mode` is decided. Swap two lines in this file to switch the entire game between Supabase and local WiFi. Zero other files change.

```gdscript
# res://src/autoloads/service_locator.gd
class_name ServiceLocator
extends Node

var _network_port: INetworkPort
var _auth_port: IAuthPort
var _storage_port: IStoragePort
var _chat_port: IChatPort
var _name_generator_port: INameGeneratorPort

func _ready() -> void:
    _wire_online_adapters()

## Call this at startup to configure for online (Supabase) play.
func _wire_online_adapters() -> void:
    # Adapters implemented in PRD-1b
    pass

## Call this to reconfigure for offline (Local WiFi) play. Phase 2.
func _wire_offline_adapters() -> void:
    pass

## Returns the active network port. Use cases call this — never import adapters directly.
func network() -> INetworkPort:
    return _network_port

func auth() -> IAuthPort:
    return _auth_port

func storage() -> IStoragePort:
    return _storage_port

func chat() -> IChatPort:
    return _chat_port

func name_generator() -> INameGeneratorPort:
    return _name_generator_port
```

### 5.7 Module: GitHub Actions CI Workflow

```yaml
# .github/workflows/ci.yml
name: Run GUT Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    name: GUT Test Suite
    runs-on: ubuntu-latest
    container:
      image: barichello/godot-ci:4.3          # Free, maintained Godot 4 CI image
    steps:
      - uses: actions/checkout@v4
      - name: Run GUT headless
        run: |
          godot --headless \
            --path . \
            -s addons/gut/gut_cmdln.gd \
            -gdir=res://test/ \
            -ginclude_subdirs \
            -gexit
```

**Plain English explanation:** Every time you push code to GitHub, this workflow spins up a Linux virtual machine with Godot pre-installed, runs all your GUT tests without opening a window (headless), and reports pass or fail. If any test fails, GitHub shows a red X on the commit and blocks merging. This is your safety net.

---

## 6. Testing Decisions

### What Makes a Good Test Here

- Tests only the **external behaviour** of a module, not its internal implementation
- A test should still pass after you refactor the internals, as long as the output is the same
- One test per behaviour, not one test per function
- Tests are fast — no network calls, no Godot scene instantiation, no file I/O
- Failures have clear, human-readable error messages

### TDD Red-Green-Refactor Cycle

```
RED   → Write a GUT test that describes the expected behaviour. Run it. It fails (red) because no code exists.
GREEN → Write the minimum GDScript to make the test pass. Run it. It passes (green).
BLUE  → Refactor the code for clarity, performance, or structure. Run tests again. They must still pass.
```

### Modules Under Test in PRD-1a

| Module              | Test Type | Notes                                                 |
|---------------------|-----------|-------------------------------------------------------|
| `Result`            | Unit      | Test ok(), fail(), data access, error_message access  |
| `ServiceLocator`    | Unit      | Test that requesting a port returns a non-null object after wiring |
| `GameBus`           | Unit      | Test that signals are declared and emittable without errors |
| Port interfaces     | Unit      | Smoke test: each `push_error` guard fires when base class method is called directly |

### Example GUT Test (Smoke Test — PRD-1a)

```gdscript
# test/unit/domain/test_result.gd
extends GutTest

func test_ok_result_has_success_true() -> void:
    var result := Result.ok("some_data")
    assert_true(result.success, "Result.ok() should set success to true")

func test_ok_result_holds_data() -> void:
    var result := Result.ok(42)
    assert_eq(result.data, 42, "Result.ok() should hold the provided data")

func test_fail_result_has_success_false() -> void:
    var result := Result.fail("something went wrong")
    assert_false(result.success, "Result.fail() should set success to false")

func test_fail_result_holds_error_message() -> void:
    var result := Result.fail("network timeout")
    assert_eq(result.error_message, "network timeout", "Result.fail() should store the error message")

func test_fail_result_data_is_null() -> void:
    var result := Result.fail("error")
    assert_null(result.data, "Result.fail() data should be null")
```

---

## 7. Out of Scope

- Supabase adapter implementations (PRD-1b)
- Any game logic or scene logic (PRD-1c onwards)
- Asset import pipeline or art (Phase 2)
- The export workflow CI step (PRD-1f)
- Offline / Local WiFi adapter (Phase 2)

---

## 8. Further Notes

### Godot Autoloads Explained (for first-time Godot developers)

In Godot, an **Autoload** is a script that is loaded automatically when the game starts, before any scene, and stays alive for the entire lifetime of the application. It is globally accessible from any script via its registered name.

`ServiceLocator` and `GameBus` are registered as autoloads in `Project → Project Settings → Autoload`. Once registered, any script can call `ServiceLocator.auth()` or emit `GameBus.scoring_resolved.emit(data)` without any import or reference setup.

Think of autoloads as global singletons — the same instance, always available, always running.

### GDScript Type System

GDScript 4 supports static typing. Always use it:

```gdscript
# ✅ Correct
var player_id: String = "abc123"
func get_points() -> int:
    return 0

# ❌ Wrong — never do this
var player_id = "abc123"
func get_points():
    return 0
```

Types are checked at editor time and catch bugs before you even run the game. This is especially important as a first-time GDScript developer.
