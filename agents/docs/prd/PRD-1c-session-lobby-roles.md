# PRD-1c — Session, Lobby & Role Assignment

| Field        | Value                                              |
|--------------|----------------------------------------------------|
| Version      | 1.0                                                |
| Date         | April 2026                                         |
| Status       | Approved                                           |
| Sub-Phase    | 1c of 6                                            |
| Duration     | Weeks 3–4                                          |
| Depends On   | PRD-1a (scaffold), PRD-1b (Supabase setup)         |
| Unlocks      | PRD-1d (Vote & Scoring)                            |
| Parent PRD   | [PRD-000 — Main](./PRD-000-main.md)                |

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

Players have no way to start or join a game. There is no session creation flow, no lobby, no wait-for-players mechanic, and no role assignment engine. Without these, no game can be played. This sub-phase establishes the complete flow from "I want to play" to "roles are assigned and the first round is ready to begin."

---

## 2. Solution

Implement three use cases — `CreateSession`, `JoinSession`, and `AssignRoles` — backed by their corresponding Edge Functions and UI scenes:

1. **CreateSession**: Admin creates a session, configures rounds and discussion duration, receives a 6-character session code
2. **JoinSession**: Other players enter the code to join the lobby; real-time presence shows who is waiting
3. **AssignRoles**: Once the lobby starts (admin trigger or 60-second timeout), the server assigns roles progressively by player count, generates the Realtime channel structure, and broadcasts role reveals to individual players via private channels

---

## 3. Design Choices & Justification

### 3.1 Session Code Design

| Approach                   | Verdict                                                           |
|----------------------------|-------------------------------------------------------------------|
| **6-char alphanumeric code**| ✅ Chosen. Human-typeable, shareable verbally, 36^6 = ~2B combinations |
| UUID share link            | ❌ Unreadable verbally, poor UX for in-person play                |
| QR code only               | ❌ Cannot share across chat; requires camera permission           |
| Short numeric PIN          | ⚠️ 6-digit: 1M combos — collision risk at scale                  |

Character set: uppercase A–Z plus digits 2–9 (excluding 0, 1, O, I to avoid visual ambiguity). This gives 32^6 = ~1.07B combinations — effectively zero collision risk at 500 MAU.

### 3.2 Role Assignment Algorithm

| Approach                        | Verdict                                                          |
|---------------------------------|------------------------------------------------------------------|
| **Fisher-Yates shuffle on eligible roles** | ✅ Chosen. O(n), provably uniform distribution, server-side only |
| Client-side random assignment   | ❌ Clients could manipulate their role                           |
| Round-robin rotation            | ❌ Predictable — players would know their next role              |
| Weighted random (favour certain roles) | ❌ Creates unfair advantage over sessions              |

The eligible role pool is determined server-side by player count (4 players → 4 roles, 5 players → 5 roles, etc.). The pool is shuffled and roles are assigned 1:1 to players. No role appears twice in the same round.

### 3.3 Lobby Timer: Server-Authoritative vs Client-Authoritative

All timers are **server-authoritative**. The server records the `lobby_started_at` timestamp and broadcasts the remaining seconds via Realtime. Clients display this countdown but cannot affect it. This prevents clock skew between devices from causing inconsistent lobby states.

---

## 4. User Stories

1. As an admin, I want to create a game session, so that I can invite friends to play.
2. As an admin, I want to set the number of rounds (up to 20) before starting, so that I can control how long the game lasts.
3. As an admin, I want to set the discussion duration per round (up to 5 minutes), so that I can tune the game for my group's pace.
4. As an admin, I want to receive a short, shareable session code, so that I can quickly invite others.
5. As a player, I want to enter a session code to join a lobby, so that I can join my friend's game.
6. As a player, I want to see a real-time list of who is in the lobby, so that I know when all my friends have joined.
7. As a player, I want to see my auto-generated display name as a guest, so that I have an identity without creating an account.
8. As an admin, I want to start the game manually when I'm satisfied with the lobby, so that I don't have to wait for the full 60 seconds.
9. As a player, I want the game to start automatically after 60 seconds if the admin doesn't act, so that sessions don't stall indefinitely.
10. As a player, I want the game to wait until at least 4 players have joined before it can start, so that there are always enough players for all mandatory roles.
11. As a player, I want to be notified if a session is full (10 players), so that I know I cannot join.
12. As a player, I want to see my role assignment privately (only me), so that the game remains fair.
13. As all players, I want to see the Police player's identity revealed to everyone simultaneously, so that the discussion phase can begin meaningfully.
14. As a system, I want role assignment to happen server-side, so that no client can manipulate which role they receive.
15. As a system, I want roles to unlock progressively by player count, so that extra roles only appear when there are enough players to fill them.
16. As a system, I want each player's session join to be idempotent, so that refreshing the lobby page doesn't create duplicate entries.

---

## 5. Implementation Decisions

### 5.1 Module: CreateSession Use Case

```gdscript
# res://src/core/use_cases/create_session.gd
class_name CreateSession
extends RefCounted

## Orchestrates the creation of a new game session.
## Calls the INetworkPort to register the session and IStoragePort to persist it.
## Returns the session code on success.

var _network: INetworkPort
var _auth: IAuthPort
var _name_gen: INameGeneratorPort

func _init(network: INetworkPort, auth: IAuthPort, name_gen: INameGeneratorPort) -> void:
    _network = network
    _auth = auth
    _name_gen = name_gen

## Creates a new session with the given configuration.
## [param config] Dictionary with keys: max_rounds (int), discussion_duration_secs (int)
## Returns Result.ok({session_id, session_code}) or Result.fail(message).
func execute(config: Dictionary) -> Result:
    if not _auth.is_signed_in() and _auth.get_current_player_id().is_empty():
        return Result.fail("Player must be authenticated (guest or Google) before creating a session")

    var max_rounds: int = config.get("max_rounds", 5)
    if max_rounds < 1 or max_rounds > GameSession.MAX_ROUNDS:
        return Result.fail("max_rounds must be between 1 and %d" % GameSession.MAX_ROUNDS)

    var discussion_secs: int = config.get("discussion_duration_secs", GameSession.DEFAULT_DISCUSSION_SECS)
    if discussion_secs < 30 or discussion_secs > 300:
        return Result.fail("discussion_duration_secs must be between 30 and 300")

    var session_code: String = _name_gen.generate_session_code()
    var payload := {
        "admin_player_id": _auth.get_current_player_id(),
        "session_code": session_code,
        "max_rounds": max_rounds,
        "discussion_duration_secs": discussion_secs
    }

    var result: Result = await _network.broadcast_event("session_create", payload)
    if not result.success:
        return Result.fail("Failed to register session: %s" % result.error_message)

    return Result.ok({"session_code": session_code})
```

### 5.2 Module: JoinSession Use Case

```gdscript
# res://src/core/use_cases/join_session.gd
class_name JoinSession
extends RefCounted

## Connects a player to an existing session lobby.
## Validates that the session exists, is in LOBBY status, and is not full.

var _network: INetworkPort
var _auth: IAuthPort

func _init(network: INetworkPort, auth: IAuthPort) -> void:
    _network = network
    _auth = auth

## Attempts to join the session identified by session_code.
## [param session_code] 6-character code received from the admin.
## Returns Result.ok({session_id, players: Array}) or Result.fail(message).
func execute(session_code: String) -> Result:
    if session_code.length() != 6:
        return Result.fail("Session code must be exactly 6 characters")

    if _auth.get_current_player_id().is_empty():
        return Result.fail("Player must be authenticated before joining a session")

    var result: Result = await _network.connect_to_session(session_code)
    if not result.success:
        return Result.fail("Session not found or no longer accepting players: %s" % result.error_message)

    var payload := {
        "player_id": _auth.get_current_player_id(),
        "session_code": session_code
    }
    var join_result: Result = await _network.broadcast_event("player_join", payload)
    if not join_result.success:
        return Result.fail("Failed to join session: %s" % join_result.error_message)

    return Result.ok(join_result.data)
```

### 5.3 Module: AssignRoles Use Case

```gdscript
# res://src/core/use_cases/assign_roles.gd
class_name AssignRoles
extends RefCounted

## Triggers server-side role assignment for the current round.
## This use case sends a request to the assign-roles Edge Function.
## The Edge Function performs the actual shuffle and broadcasts role reveals.
## The client never touches the role assignment algorithm.

var _network: INetworkPort
var _auth: IAuthPort

func _init(network: INetworkPort, auth: IAuthPort) -> void:
    _network = network
    _auth = auth

## Requests role assignment from the server for the given session and round.
## Only the admin may call this. Returns immediately; role assignments arrive
## via GameBus.roles_assigned signal once the Edge Function responds.
## [param session_id] UUID of the current session.
## [param round_number] The round number (1-indexed).
func execute(session_id: String, round_number: int) -> Result:
    if session_id.is_empty():
        return Result.fail("session_id cannot be empty")
    if round_number < 1:
        return Result.fail("round_number must be 1 or greater")

    var payload := {
        "session_id": session_id,
        "round_number": round_number,
        "requesting_player_id": _auth.get_current_player_id()
    }
    return await _network.broadcast_event("assign_roles_request", payload)
```

### 5.4 Module: Role Entity — Full Definition

```gdscript
# res://src/core/domain/entities/role.gd
class_name Role
extends RefCounted

enum RoleId {
    KING            = 0,
    QUEEN           = 1,
    POLICE          = 2,
    THIEF           = 3,
    PRIME_MINISTER  = 4,
    GENERAL         = 5,
    SPY             = 6,
    MERCHANT        = 7,
    JESTER          = 8,
    PRIEST          = 9
}

var role_id: RoleId
var display_name: String
var base_points: int
var min_players_required: int
var objective_description: String
var objective_bonus: int
var is_mandatory: bool    # True for KING, QUEEN, POLICE, THIEF — these cause round pause on disconnect

## Returns the complete, ordered role roster. Order determines assignment priority.
## The first min_players_required roles form the pool for that player count.
static func all_roles() -> Array[Role]:
    var roles: Array[Role] = []

    var king := Role.new()
    king.role_id = RoleId.KING
    king.display_name = "King"
    king.base_points = 1000
    king.min_players_required = 4
    king.objective_description = "Survive without being accused during Police deliberation"
    king.objective_bonus = 100
    king.is_mandatory = true
    roles.append(king)

    var queen := Role.new()
    queen.role_id = RoleId.QUEEN
    queen.display_name = "Queen"
    queen.base_points = 800
    queen.min_players_required = 4
    queen.objective_description = "Secretly predict which player Police will accuse before vote resolves"
    queen.objective_bonus = 80
    queen.is_mandatory = true
    roles.append(queen)

    var police := Role.new()
    police.role_id = RoleId.POLICE
    police.display_name = "Police"
    police.base_points = 100
    police.min_players_required = 4
    police.objective_description = "Correctly identify and accuse the Thief within 10 seconds"
    police.objective_bonus = 0  # Core mechanic — not a bonus
    police.is_mandatory = true
    roles.append(police)

    var thief := Role.new()
    thief.role_id = RoleId.THIEF
    thief.display_name = "Thief"
    thief.base_points = 0
    thief.min_players_required = 4
    thief.objective_description = "Escape Police accusation. Secretly nominate one Accomplice."
    thief.objective_bonus = 0  # Variable — depends on Police outcome
    thief.is_mandatory = true
    roles.append(thief)

    var pm := Role.new()
    pm.role_id = RoleId.PRIME_MINISTER
    pm.display_name = "Prime Minister"
    pm.base_points = 700
    pm.min_players_required = 5
    pm.objective_description = "Benefit from royalty's downfall — Police accuses King or Queen"
    pm.objective_bonus = 150
    pm.is_mandatory = false
    roles.append(pm)

    var general := Role.new()
    general.role_id = RoleId.GENERAL
    general.display_name = "General"
    general.base_points = 600
    general.min_players_required = 6
    general.objective_description = "Predict before discussion: Thief escapes or Thief caught"
    general.objective_bonus = 100
    general.is_mandatory = false
    roles.append(general)

    var spy := Role.new()
    spy.role_id = RoleId.SPY
    spy.display_name = "Spy"
    spy.base_points = 500
    spy.min_players_required = 7
    spy.objective_description = "Double prediction: who Police accuses AND whether correct or wrong"
    spy.objective_bonus = 150  # If both correct; 75 if one correct
    spy.is_mandatory = false
    roles.append(spy)

    var merchant := Role.new()
    merchant.role_id = RoleId.MERCHANT
    merchant.display_name = "Merchant"
    merchant.base_points = 400
    merchant.min_players_required = 8
    merchant.objective_description = "Profit when any player pays a false accusation fine this round"
    merchant.objective_bonus = 75
    merchant.is_mandatory = false
    roles.append(merchant)

    var jester := Role.new()
    jester.role_id = RoleId.JESTER
    jester.display_name = "Jester"
    jester.base_points = 300
    jester.min_players_required = 9
    jester.objective_description = "Get accused by Police without being the Thief"
    jester.objective_bonus = 200
    jester.is_mandatory = false
    roles.append(jester)

    var priest := Role.new()
    priest.role_id = RoleId.PRIEST
    priest.display_name = "Priest"
    priest.base_points = 200
    priest.min_players_required = 10
    priest.objective_description = "Justice bonus — Police correctly catches the Thief"
    priest.objective_bonus = 100
    priest.is_mandatory = false
    roles.append(priest)

    return roles

## Returns only the roles eligible for a given player count.
static func roles_for_player_count(count: int) -> Array[Role]:
    return all_roles().filter(func(r: Role) -> bool: return r.min_players_required <= count)
```

### 5.5 Module: RoleAssigner (Domain Service)

**What it is:** A pure, stateless domain service that takes a list of players and returns a shuffled role assignment. This is what GUT tests against — pure input/output, no network, no Supabase.

**Why separate from the Edge Function?** The Edge Function calls this same logic in TypeScript. Having a GDScript mirror lets us TDD the algorithm locally and be confident the TypeScript version is equivalent.

```gdscript
# res://src/core/domain/role_assigner.gd
class_name RoleAssigner
extends RefCounted

## Assigns roles to players using a Fisher-Yates shuffle.
## [param players] Array of player_id strings. Length must be 4–10.
## Returns Result.ok(Array[Dictionary]) where each dict is {player_id, role_id, base_points, is_mandatory}
## Returns Result.fail(message) if player count is out of range.
func assign(player_ids: Array[String]) -> Result:
    var count: int = player_ids.size()
    if count < GameSession.MIN_PLAYERS:
        return Result.fail("Minimum %d players required. Got %d." % [GameSession.MIN_PLAYERS, count])
    if count > GameSession.MAX_PLAYERS:
        return Result.fail("Maximum %d players allowed. Got %d." % [GameSession.MAX_PLAYERS, count])

    var eligible_roles: Array[Role] = Role.roles_for_player_count(count)
    var shuffled_roles: Array[Role] = _fisher_yates_shuffle(eligible_roles)

    var assignments: Array[Dictionary] = []
    for i: int in range(count):
        assignments.append({
            "player_id": player_ids[i],
            "role_id": shuffled_roles[i].role_id,
            "role_display_name": shuffled_roles[i].display_name,
            "base_points": shuffled_roles[i].base_points,
            "is_mandatory": shuffled_roles[i].is_mandatory
        })

    return Result.ok(assignments)

## Fisher-Yates shuffle — provably uniform distribution.
## Each element has exactly 1/n chance of landing in any position.
func _fisher_yates_shuffle(roles: Array[Role]) -> Array[Role]:
    var arr: Array[Role] = roles.duplicate()
    var n: int = arr.size()
    for i: int in range(n - 1, 0, -1):
        var j: int = randi() % (i + 1)
        var temp: Role = arr[i]
        arr[i] = arr[j]
        arr[j] = temp
    return arr
```

### 5.6 Module: NameGenerator

```gdscript
# res://src/adapters/name_generator/wordlist_name_generator.gd
class_name WordlistNameGenerator
extends INameGeneratorPort

const ADJECTIVES: Array[String] = [
    "Furious", "Wretched", "Cursed", "Doomed", "Reckless",
    "Treacherous", "Scheming", "Chaotic", "Dishonoured", "Brazen",
    "Ruthless", "Cunning", "Devious", "Disgraced", "Audacious"
]

const NOUNS: Array[String] = [
    "Nawab", "Diwan", "Maharani", "Kotwal", "Sipahi",
    "Vakeel", "Daroga", "Zamindar", "Mirza", "Thakur",
    "Sardar", "Havildar", "Munshi", "Rajput", "Rani"
]

const CODE_CHARS: String = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"  # No 0,1,O,I

func generate_player_name() -> String:
    var adj: String = ADJECTIVES[randi() % ADJECTIVES.size()]
    var noun: String = NOUNS[randi() % NOUNS.size()]
    return "%s %s" % [adj, noun]

func generate_session_code() -> String:
    var code: String = ""
    for i: int in range(6):
        code += CODE_CHARS[randi() % CODE_CHARS.length()]
    return code
```

### 5.7 Edge Function: assign-roles

```typescript
// supabase/functions/assign-roles/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"

const ROLE_DEFINITIONS = [
  { role_id: "king",           base_points: 1000, min_players: 4, is_mandatory: true },
  { role_id: "queen",          base_points: 800,  min_players: 4, is_mandatory: true },
  { role_id: "police",         base_points: 100,  min_players: 4, is_mandatory: true },
  { role_id: "thief",          base_points: 0,    min_players: 4, is_mandatory: true },
  { role_id: "prime_minister", base_points: 700,  min_players: 5, is_mandatory: false },
  { role_id: "general",        base_points: 600,  min_players: 6, is_mandatory: false },
  { role_id: "spy",            base_points: 500,  min_players: 7, is_mandatory: false },
  { role_id: "merchant",       base_points: 400,  min_players: 8, is_mandatory: false },
  { role_id: "jester",         base_points: 300,  min_players: 9, is_mandatory: false },
  { role_id: "priest",         base_points: 200,  min_players: 10, is_mandatory: false },
]

function fisherYatesShuffle<T>(arr: T[]): T[] {
  const a = [...arr]
  for (let i = a.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [a[i], a[j]] = [a[j], a[i]]
  }
  return a
}

serve(async (req: Request) => {
  const { session_id, round_number } = await req.json()
  if (!session_id || !round_number) {
    return new Response(JSON.stringify({ error: "Missing required fields" }), { status: 400 })
  }

  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  )

  // Fetch active players in this session
  const { data: players, error } = await supabase
    .from("session_players")
    .select("user_id")
    .eq("session_id", session_id)
    .eq("is_active", true)

  if (error || !players) {
    return new Response(JSON.stringify({ error: "Failed to fetch players" }), { status: 500 })
  }

  const count = players.length
  const eligibleRoles = ROLE_DEFINITIONS.filter(r => r.min_players <= count)
  const shuffledRoles = fisherYatesShuffle(eligibleRoles)

  const assignments = players.map((p, i) => ({
    player_id: p.user_id,
    ...shuffledRoles[i]
  }))

  // Insert round and role assignments
  const { data: round } = await supabase
    .from("rounds")
    .insert({ session_id, round_number })
    .select()
    .single()

  const policeAssignment = assignments.find(a => a.role_id === "police")

  // Insert round_roles
  await supabase.from("round_roles").insert(
    assignments.map(a => ({
      round_id: round.id,
      player_id: a.player_id,
      role_id: a.role_id,
      base_points: a.base_points
    }))
  )

  // Broadcast police identity to all (public)
  // Broadcast individual roles to each player via private channel
  // (Realtime channel broadcast handled here)

  return new Response(JSON.stringify({
    round_id: round.id,
    assignments,
    police_player_id: policeAssignment?.player_id
  }), { status: 200 })
})
```

---

## 6. Testing Decisions

### TDD Focus: RoleAssigner

The `RoleAssigner` domain service is the highest-value TDD target in this sub-phase. It is pure input/output logic with no external dependencies.

```gdscript
# test/unit/domain/test_role_assigner.gd
extends GutTest

var assigner: RoleAssigner

func before_each() -> void:
    assigner = RoleAssigner.new()

func test_four_players_get_exactly_four_roles() -> void:
    var players: Array[String] = ["p1", "p2", "p3", "p4"]
    var result: Result = assigner.assign(players)
    assert_true(result.success)
    assert_eq(result.data.size(), 4)

func test_four_players_only_get_mandatory_roles() -> void:
    var players: Array[String] = ["p1", "p2", "p3", "p4"]
    var result: Result = assigner.assign(players)
    var role_ids: Array = result.data.map(func(a): return a.role_id)
    assert_has(role_ids, Role.RoleId.KING)
    assert_has(role_ids, Role.RoleId.QUEEN)
    assert_has(role_ids, Role.RoleId.POLICE)
    assert_has(role_ids, Role.RoleId.THIEF)

func test_five_players_includes_prime_minister() -> void:
    var players: Array[String] = ["p1", "p2", "p3", "p4", "p5"]
    var result: Result = assigner.assign(players)
    var role_ids: Array = result.data.map(func(a): return a.role_id)
    assert_has(role_ids, Role.RoleId.PRIME_MINISTER)

func test_no_player_gets_duplicate_role() -> void:
    var players: Array[String] = ["p1", "p2", "p3", "p4", "p5", "p6"]
    var result: Result = assigner.assign(players)
    var role_ids: Array = result.data.map(func(a): return a.role_id)
    var unique_ids := {}
    for r in role_ids:
        assert_false(unique_ids.has(r), "Role %s assigned more than once" % r)
        unique_ids[r] = true

func test_too_few_players_returns_failure() -> void:
    var players: Array[String] = ["p1", "p2", "p3"]
    var result: Result = assigner.assign(players)
    assert_false(result.success)
    assert_string_contains(result.error_message, "Minimum")

func test_too_many_players_returns_failure() -> void:
    var players: Array[String] = ["p1","p2","p3","p4","p5","p6","p7","p8","p9","p10","p11"]
    var result: Result = assigner.assign(players)
    assert_false(result.success)
    assert_string_contains(result.error_message, "Maximum")

func test_assign_uses_all_eligible_roles_for_ten_players() -> void:
    var players: Array[String] = ["p1","p2","p3","p4","p5","p6","p7","p8","p9","p10"]
    var result: Result = assigner.assign(players)
    assert_eq(result.data.size(), 10)
```

### Modules Under Test

| Module           | Test Type | Key Behaviours Tested                                |
|------------------|-----------|------------------------------------------------------|
| `RoleAssigner`   | Unit TDD  | Correct role count, no duplicates, boundary errors, progressive unlock |
| `CreateSession`  | Unit TDD  | Validation of max_rounds range, discussion_secs range, auth guard |
| `JoinSession`    | Unit TDD  | Session code length validation, auth guard           |
| `NameGenerator`  | Unit      | Output is non-empty string, session code is 6 chars, no ambiguous chars |
| `Role.all_roles` | Unit      | All 10 roles present, correct base_points, correct min_players |

---

## 7. Out of Scope

- Vote mechanic and scoring (PRD-1d)
- Alliance assignment (PRD-1e)
- Power-up shop during role reveal (Phase 2)
- Admin kick functionality beyond lobby (Phase 2)
- Reconnection flow during a round (PRD-1d)
- Placeholder UI art and animations (Phase 2)

---

## 8. Further Notes

### Fisher-Yates Shuffle Explained (Plain English)

Imagine you have 6 cards face-down. Fisher-Yates works like this: pick any card from the full deck and put it in position 6. Then pick any card from the remaining 5 and put it in position 5. Repeat until done. Every possible ordering has exactly the same probability — it's mathematically proven to be fair. This is why it's used in games, cryptography, and statistics.

The naïve alternative (`sort by random number`) has subtle biases. Fisher-Yates does not. Always use Fisher-Yates for game role assignment.

### Session State Machine

The session `status` column in Supabase is the single source of truth. The client never sets status directly — it comes from a Realtime broadcast after an Edge Function updates it:

```
LOBBY → ROUND_START → ROLE_REVEAL → DISCUSSION → VOTE → SCORING → ROUND_END
  ↑                                                                      |
  └──────────────── (if rounds remain, loop back to ROUND_START) ────────┘
                                                                         ↓
                                                                    GAME_END
```

GameBus emits `session_status_changed` whenever the server pushes a new status. All scenes subscribe to this signal and transition accordingly.
