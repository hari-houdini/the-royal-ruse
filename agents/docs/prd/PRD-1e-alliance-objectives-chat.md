# PRD-1e — Alliance Engine, Objectives & Chat

| Field        | Value                                                        |
|--------------|--------------------------------------------------------------|
| Version      | 1.0                                                          |
| Date         | April 2026                                                   |
| Status       | Approved                                                     |
| Sub-Phase    | 1e of 6                                                      |
| Duration     | Weeks 6–7                                                    |
| Depends On   | PRD-1d (Vote Mechanic & Scoring Engine)                      |
| Unlocks      | PRD-1f (Vertical Slice & Deployment)                         |
| Parent PRD   | [PRD-000 — Main](./PRD-000-main.md)                          |

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

The scoring engine from PRD-1d resolves base point mechanics, but three major game systems are missing: alliances (which double objective bonuses when complementary conditions are met), round objectives (each role has a secret win condition beyond just surviving), and in-session text chat (the only communication channel during the discussion phase). Without these, the game loop is incomplete and the vertical slice cannot be demonstrated.

---

## 2. Solution

Implement three systems that integrate into the existing scoring pipeline:

1. **Alliance Engine** — server-side random alliance assignment at round start, complementary condition evaluation at scoring time, blind reveal on the score screen
2. **Objective Tracker** — per-role objective evaluation baked into the `resolve-scoring` Edge Function, with objective bonuses applied before alliance multipliers
3. **Text Chat** — Supabase Realtime broadcast channel per session, phase-locked (disabled during VOTE and SCORING phases), delivered through the `IChatPort` interface already defined in PRD-1a

---

## 3. Design Choices & Justification

### 3.1 Alliance Assignment: Server-Side at Round Start vs On-Demand

| Approach                            | Verdict                                                          |
|-------------------------------------|------------------------------------------------------------------|
| **Server-side at round start (chosen)** | ✅ Assignment is stored in `alliances` table immediately. RLS blocks client reads until round complete. Zero chance of client interference |
| Client-side random                  | ❌ Clients could read their own alliance data from the channel and act on it |
| Assigned on scoring                 | ❌ Late assignment could be gamed — players can't retroactively affect a condition they didn't know about |

Alliances are assigned in the `assign-roles` Edge Function (PRD-1c) immediately after roles are distributed. They are never sent to clients until the `round_complete` Realtime broadcast.

### 3.2 Objective Evaluation: Edge Function vs Client-Side

All objective evaluation is server-side, integrated into `resolve-scoring`. The server has access to all round state (who was accused, what the outcome was, what each player predicted) — the client does not.

Exception: General and Spy submit sealed predictions before discussion opens. These are stored as encrypted server-side records and evaluated at scoring without ever being visible to other clients.

### 3.3 Chat: Build From Scratch vs Third-Party SDK

As decided during design: build text chat using Supabase Realtime. This decision is validated here:

| Criterion          | Supabase Realtime Chat     | Discord API            | Twilio Conversations     |
|--------------------|----------------------------|------------------------|--------------------------|
| Cost at 500 MAU    | ✅ Free tier                | ⚠️ Mobile app unsupported | ❌ Pay per message      |
| Mobile support     | ✅ Full (our own WebSocket) | ❌ Discord Activities = web/desktop only | ✅ But paid |
| Phase-locking      | ✅ Full control             | ❌ Cannot lock Discord channels mid-session | ⚠️ Requires webhook logic |
| Offline support    | ⚠️ N/A (online-only in P1) | ❌ Impossible           | ❌ Requires internet      |
| Implementation time| ✅ 2–3 days                 | ❌ 1–2 weeks + app review | ❌ 1 week + paid account |

**Decision confirmed: Supabase Realtime.** Phase-locking (disabling chat during the VOTE countdown) is a core game mechanic — third-party chat services cannot provide this control.

### 3.4 Prediction Sealing (General, Queen, Spy)

Players who must submit predictions before discussion (General, Spy) and during discussion (Queen) need a way to submit sealed predictions the server stores but does not reveal. Options:

| Approach              | Verdict                                                         |
|-----------------------|-----------------------------------------------------------------|
| **Encrypted server storage** | ✅ Prediction stored in a `round_predictions` table with RLS denying all client reads until round completion |
| Client-holds-prediction | ❌ Client could lie about what they predicted when scoring |
| Hash-commit scheme    | ⚠️ Cryptographically robust but complex to implement for a game at this scale |

**Decision: Encrypted server storage via RLS.** A `round_predictions` table with RLS `FOR SELECT USING (round_id IN (SELECT id FROM rounds WHERE completed_at IS NOT NULL))` makes predictions unreadable until scoring is complete.

---

## 4. User Stories

1. As all players, I want alliances to be assigned secretly at the start of each round, so that I cannot know in advance whether I have an ally.
2. As a player with an alliance, I want to discover my alliance partner only at the score reveal screen, so that the reveal is a genuine surprise.
3. As a player in a Thief + Jester alliance, I want both our objective bonuses doubled if the Thief escaped AND I was accused (as Jester), so that complementary outcomes are meaningfully rewarded.
4. As a King, I want to earn a +100 objective bonus if I am not accused during Police deliberation, so that surviving without suspicion is actively worth pursuing.
5. As a Queen, I want to submit a secret prediction of who Police will accuse, so that I am rewarded for reading the room correctly.
6. As a General, I want to submit a binary sealed prediction of "Thief caught" or "Thief escaped" before discussion, so that my strategic read of the situation is rewarded.
7. As a Spy, I want to submit a double sealed prediction (who Police will accuse + correct or wrong outcome), so that my intelligence-gathering has the highest potential payoff.
8. As a Jester, I want to earn +200 if Police accuses me but I am not the Thief, so that my goal of looking guilty is directly rewarded.
9. As a Priest, I want to earn +100 if Police correctly catches the Thief, so that justice being served benefits me.
10. As a Prime Minister, I want to earn +150 if Police accuses King or Queen, so that I benefit from royalty's downfall.
11. As a Merchant, I want to earn +75 automatically if any false accusation fine is paid, so that instability profits me without active action.
12. As players with a winning blind alliance, I want both our objective bonuses multiplied, so that hidden coordination is spectacularly rewarded.
13. As a player in a discussion phase, I want to send text messages visible to all players in the session, so that I can participate in the accusation debate.
14. As all players, I want the chat to be disabled during the Police 10-second vote countdown, so that no last-second chat influences the vote.
15. As all players, I want to see who sent each message along with their display name, so that I can track who is saying what.
16. As a player, I want my messages to appear instantly in the chat without perceptible delay, so that the conversation feels natural.
17. As an admin, I want the chat history to be cleared when the session ends, so that no session data is retained beyond its useful life.

---

## 5. Implementation Decisions

### 5.1 New Database Table: round_predictions

```sql
-- Stores sealed predictions submitted by Queen, General, and Spy.
-- RLS prevents reading until round is complete.
CREATE TABLE round_predictions (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  round_id      UUID NOT NULL REFERENCES rounds(id) ON DELETE CASCADE,
  player_id     UUID NOT NULL REFERENCES users(id),
  prediction_type TEXT NOT NULL,  -- 'who_accused' | 'outcome' | 'double'
  prediction_value TEXT NOT NULL, -- Serialised JSON: {"accused_id": "...", "outcome": "caught"}
  submitted_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (round_id, player_id, prediction_type)
);

ALTER TABLE round_predictions ENABLE ROW LEVEL SECURITY;

-- Players can INSERT their own prediction, but never SELECT until round complete
CREATE POLICY "predictions_insert_own" ON round_predictions
  FOR INSERT WITH CHECK (player_id = auth.uid());

CREATE POLICY "predictions_read_post_round" ON round_predictions
  FOR SELECT USING (
    round_id IN (SELECT id FROM rounds WHERE completed_at IS NOT NULL)
  );
```

### 5.2 Module: SubmitPrediction Use Case

```gdscript
# res://src/core/use_cases/submit_prediction.gd
class_name SubmitPrediction
extends RefCounted

## Submits a sealed prediction to the server before or during the discussion phase.
## Used by Queen (during discussion), General (before discussion), Spy (before discussion).
## Predictions cannot be changed once submitted.

var _network: INetworkPort
var _auth: IAuthPort

func _init(network: INetworkPort, auth: IAuthPort) -> void:
    _network = network
    _auth = auth

## Submits a prediction.
## [param round_id] UUID of the current round.
## [param prediction_type] One of: "who_accused", "outcome", "double"
## [param prediction_value] Dictionary serialised to JSON server-side.
##   For "who_accused": {"accused_player_id": "uuid"}
##   For "outcome": {"predicted_outcome": "caught" | "escaped"}
##   For "double": {"accused_player_id": "uuid", "predicted_outcome": "caught" | "escaped"}
## Returns Result.ok() if accepted, Result.fail(message) if rejected or already submitted.
func execute(round_id: String, prediction_type: String, prediction_value: Dictionary) -> Result:
    var valid_types: Array[String] = ["who_accused", "outcome", "double"]
    if not valid_types.has(prediction_type):
        return Result.fail("prediction_type must be one of: %s" % ", ".join(valid_types))
    if round_id.is_empty():
        return Result.fail("round_id cannot be empty")
    if prediction_value.is_empty():
        return Result.fail("prediction_value cannot be empty")

    var payload := {
        "round_id": round_id,
        "player_id": _auth.get_current_player_id(),
        "prediction_type": prediction_type,
        "prediction_value": JSON.stringify(prediction_value)
    }
    return await _network.broadcast_event("prediction_submitted", payload)
```

### 5.3 Module: AllianceEvaluator (Domain Service)

**What it is:** A pure domain service that takes a completed round's data and evaluates whether any alliance's complementary condition was met. Returns a list of alliance outcomes with the multiplier flag set.

```gdscript
# res://src/core/domain/alliance_evaluator.gd
class_name AllianceEvaluator
extends RefCounted

## Defines the complementary conditions for each alliance type.
## A condition is a Callable that takes a RoundContext and returns bool.
## RoundContext is a Dictionary with all round outcome data.

## Evaluates all alliances for a completed round.
## [param alliances] Array[Dictionary] each with: {player1_id, player2_id, alliance_type}
## [param round_context] Dictionary with round outcome data (see _build_context below)
## Returns Result.ok(Array[Dictionary]) each with: {player1_id, player2_id, alliance_type, complementary_achieved, multiplier_applied}
func evaluate(alliances: Array[Dictionary], round_context: Dictionary) -> Result:
    if alliances.is_empty():
        return Result.ok([])  # No alliances this round — valid state

    var results: Array[Dictionary] = []
    for alliance: Dictionary in alliances:
        var alliance_type: String = alliance.get("alliance_type", "")
        var achieved: bool = _evaluate_condition(alliance_type, round_context)
        results.append({
            "player1_id": alliance["player1_id"],
            "player2_id": alliance["player2_id"],
            "alliance_type": alliance_type,
            "complementary_achieved": achieved,
            "multiplier_applied": achieved
        })

    return Result.ok(results)

## Evaluates whether a specific alliance type's complementary condition was met.
func _evaluate_condition(alliance_type: String, ctx: Dictionary) -> bool:
    match alliance_type:
        "thief_jester":
            # Thief escaped AND Jester was accused (and Jester is not the Thief)
            return ctx.get("outcome") == "escaped" \
                and ctx.get("jester_player_id", "") == ctx.get("accused_player_id", "") \
                and ctx.get("jester_player_id", "") != ctx.get("thief_player_id", "")

        "thief_spy":
            # Thief escaped AND Spy correctly predicted both outcomes
            return ctx.get("outcome") == "escaped" \
                and ctx.get("spy_double_prediction_correct", false)

        "king_priest":
            # King survived unaccused AND Police caught Thief
            return ctx.get("king_accused", false) == false \
                and ctx.get("outcome") == "caught"

        "queen_prime_minister":
            # Queen's prediction was correct AND PM earned royalty-downfall bonus
            return ctx.get("queen_prediction_correct", false) \
                and ctx.get("pm_royalty_downfall_triggered", false)

        "general_merchant":
            # General's prediction was correct AND Merchant profited from false accusation fine
            return ctx.get("general_prediction_correct", false) \
                and ctx.get("false_accusation_fine_paid", false)

        "spy_general":
            # Both Spy and General predicted correctly in the same round
            return ctx.get("spy_double_prediction_correct", false) \
                and ctx.get("general_prediction_correct", false)

        _:
            push_warning("AllianceEvaluator: unknown alliance_type '%s'" % alliance_type)
            return false
```

### 5.4 Module: ObjectiveEvaluator (Domain Service)

```gdscript
# res://src/core/domain/objective_evaluator.gd
class_name ObjectiveEvaluator
extends RefCounted

## Evaluates whether each player met their round objective.
## Returns Result.ok(Array[Dictionary]) each with {player_id, role_id, objective_completed, objective_bonus}
##
## [param role_assignments] Array[Dictionary]: [{player_id, role_id, base_points}]
## [param round_context] Dictionary with all round outcome data
func evaluate(role_assignments: Array[Dictionary], round_context: Dictionary) -> Result:
    if role_assignments.is_empty():
        return Result.fail("role_assignments cannot be empty")

    var results: Array[Dictionary] = []
    for assignment: Dictionary in role_assignments:
        var player_id: String = assignment["player_id"]
        var role_id: String = assignment["role_id"]
        var completed: bool = _evaluate_objective(role_id, player_id, round_context)
        var bonus: int = _calculate_bonus(role_id, player_id, round_context) if completed else 0
        results.append({
            "player_id": player_id,
            "role_id": role_id,
            "objective_completed": completed,
            "objective_bonus": bonus
        })

    return Result.ok(results)

## Returns true if the given player completed their role's objective this round.
func _evaluate_objective(role_id: String, player_id: String, ctx: Dictionary) -> bool:
    match role_id:
        "king":
            # Not accused during police deliberation
            return ctx.get("accused_player_id", "") != player_id

        "queen":
            # Correctly predicted who Police would accuse
            var prediction: String = ctx.get("queen_prediction_accused_id", "")
            return not prediction.is_empty() and prediction == ctx.get("accused_player_id", "")

        "police":
            # Correctly identified Thief
            return ctx.get("outcome", "") == "caught"

        "thief":
            # Escaped — base mechanic, objective is always the escape
            return ctx.get("outcome", "") in ["escaped", "timeout"]

        "prime_minister":
            # Police accused King or Queen
            var accused: String = ctx.get("accused_player_id", "")
            return accused == ctx.get("king_player_id", "") or accused == ctx.get("queen_player_id", "")

        "general":
            # Binary prediction was correct
            return ctx.get("general_prediction_correct", false)

        "spy":
            # At least one prediction was correct (partial bonus handled in _calculate_bonus)
            return ctx.get("spy_who_prediction_correct", false) or ctx.get("spy_outcome_prediction_correct", false)

        "merchant":
            # Any false accusation fine was paid this round
            return ctx.get("false_accusation_fine_paid", false)

        "jester":
            # Accused by Police, but is not the Thief
            var accused: String = ctx.get("accused_player_id", "")
            return accused == player_id and player_id != ctx.get("thief_player_id", "")

        "priest":
            # Police caught the Thief
            return ctx.get("outcome", "") == "caught"

        _:
            push_warning("ObjectiveEvaluator: unknown role_id '%s'" % role_id)
            return false

## Returns the objective bonus for a completed objective.
## Spy has variable bonus depending on how many predictions were correct.
func _calculate_bonus(role_id: String, player_id: String, ctx: Dictionary) -> int:
    match role_id:
        "king":          return 100
        "queen":         return 80
        "police":        return 0    # Core mechanic — no separate bonus
        "thief":         return 0    # Variable — handled by ScoringEngine
        "prime_minister":return 150
        "general":       return 100
        "spy":
            var both_correct: bool = ctx.get("spy_who_prediction_correct", false) and ctx.get("spy_outcome_prediction_correct", false)
            return 150 if both_correct else 75
        "merchant":      return 75
        "jester":        return 200
        "priest":        return 100
        _:               return 0
```

### 5.5 Alliance Multiplier Integration into ScoringEngine

The ScoringEngine (PRD-1d) is extended to accept objective results and alliance outcomes, then apply multipliers:

```gdscript
## ADDITION to ScoringEngine — appended to calculate() after base scoring:

## Applies objective bonuses and alliance multipliers to the computed scores.
## [param scores] Dictionary of player_id → PlayerScore (from PRD-1d calculate())
## [param objective_results] Array[Dictionary] from ObjectiveEvaluator.evaluate()
## [param alliance_results] Array[Dictionary] from AllianceEvaluator.evaluate()
func apply_objectives_and_alliances(
    scores: Dictionary,
    objective_results: Array[Dictionary],
    alliance_results: Array[Dictionary]
) -> void:
    # Step 1: Compute base objective bonuses
    var objective_bonuses: Dictionary = {}  # player_id → bonus int
    for obj: Dictionary in objective_results:
        objective_bonuses[obj["player_id"]] = obj["objective_bonus"]

    # Step 2: For each winning alliance, double both players' objective bonuses
    for alliance: Dictionary in alliance_results:
        if alliance["complementary_achieved"]:
            var p1: String = alliance["player1_id"]
            var p2: String = alliance["player2_id"]
            if objective_bonuses.has(p1):
                objective_bonuses[p1] = objective_bonuses[p1] * 2
            if objective_bonuses.has(p2):
                objective_bonuses[p2] = objective_bonuses[p2] * 2

    # Step 3: Add objective bonuses to total_delta
    for player_id: String in objective_bonuses:
        if scores.has(player_id):
            scores[player_id].total_delta += objective_bonuses[player_id]
```

### 5.6 Module: ChatService (Application Service)

```gdscript
# res://src/core/use_cases/chat_service.gd
class_name ChatService
extends RefCounted

## Manages text chat lifecycle for a game session.
## Wraps IChatPort and enforces game-phase chat locking rules.

var _chat: IChatPort
var _session_id: String
var _message_count: int = 0

const MAX_MESSAGES_PER_PHASE: int = 20
const MAX_MESSAGE_LENGTH: int = 200

func _init(chat: IChatPort) -> void:
    _chat = chat

## Joins the chat channel for the given session. Call at session join.
func join(session_id: String) -> Result:
    _session_id = session_id
    _message_count = 0
    return await _chat.join_chat_channel(session_id)

## Sends a text message to the session chat.
## Enforces: not locked, message length, rate limit (20 per discussion phase).
func send(message: String) -> Result:
    if _chat.is_locked():
        return Result.fail("Chat is locked during vote phase")
    if message.strip_edges().is_empty():
        return Result.fail("Message cannot be empty")
    if message.length() > MAX_MESSAGE_LENGTH:
        return Result.fail("Message exceeds %d character limit" % MAX_MESSAGE_LENGTH)
    if _message_count >= MAX_MESSAGES_PER_PHASE:
        return Result.fail("Message limit reached for this discussion phase")
    _message_count += 1
    return await _chat.send_message(_session_id, message)

## Locks chat and resets message counter at start of vote phase.
func on_vote_phase_started() -> void:
    _chat.lock_chat()
    _message_count = 0

## Unlocks chat at start of discussion phase.
func on_discussion_phase_started() -> void:
    _chat.unlock_chat()
    _message_count = 0
```

### 5.7 Alliance Assignment in Edge Function (addition to assign-roles)

```typescript
// Addition to supabase/functions/assign-roles/index.ts

const ALLIANCE_PAIRINGS = [
  { type: "thief_jester",        roles: ["thief", "jester"] },
  { type: "thief_spy",           roles: ["thief", "spy"] },
  { type: "king_priest",         roles: ["king", "priest"] },
  { type: "queen_prime_minister",roles: ["queen", "prime_minister"] },
  { type: "general_merchant",    roles: ["general", "merchant"] },
  { type: "spy_general",         roles: ["spy", "general"] },
]

function assignAlliance(assignments: any[]): { player1_id: string; player2_id: string; alliance_type: string } | null {
  // 60% chance of an alliance existing this round
  if (Math.random() > 0.6) return null

  // Find eligible pairings where both roles exist in this round's assignments
  const assignedRoleIds = assignments.map((a: any) => a.role_id)
  const eligiblePairings = ALLIANCE_PAIRINGS.filter(
    p => p.roles.every(r => assignedRoleIds.includes(r))
  )
  if (eligiblePairings.length === 0) return null

  const chosen = eligiblePairings[Math.floor(Math.random() * eligiblePairings.length)]
  const player1 = assignments.find((a: any) => a.role_id === chosen.roles[0])
  const player2 = assignments.find((a: any) => a.role_id === chosen.roles[1])

  return { player1_id: player1.player_id, player2_id: player2.player_id, alliance_type: chosen.type }
}
```

---

## 6. Testing Decisions

### TDD Focus: AllianceEvaluator and ObjectiveEvaluator

These are pure domain logic classes with no external dependencies — ideal TDD targets.

```gdscript
# test/unit/domain/test_alliance_evaluator.gd
extends GutTest

var evaluator: AllianceEvaluator

func before_each() -> void:
    evaluator = AllianceEvaluator.new()

func test_thief_jester_alliance_achieved_when_thief_escapes_and_jester_accused() -> void:
    var alliances := [{"player1_id": "thief", "player2_id": "jester", "alliance_type": "thief_jester"}]
    var ctx := {"outcome": "escaped", "accused_player_id": "jester", "thief_player_id": "thief", "jester_player_id": "jester"}
    var result: Result = evaluator.evaluate(alliances, ctx)
    assert_true(result.success)
    assert_true(result.data[0]["complementary_achieved"])

func test_thief_jester_not_achieved_when_thief_caught() -> void:
    var alliances := [{"player1_id": "thief", "player2_id": "jester", "alliance_type": "thief_jester"}]
    var ctx := {"outcome": "caught", "accused_player_id": "thief", "thief_player_id": "thief", "jester_player_id": "jester"}
    var result: Result = evaluator.evaluate(alliances, ctx)
    assert_false(result.data[0]["complementary_achieved"])

func test_thief_jester_not_achieved_when_jester_not_accused() -> void:
    var alliances := [{"player1_id": "thief", "player2_id": "jester", "alliance_type": "thief_jester"}]
    var ctx := {"outcome": "escaped", "accused_player_id": "king", "thief_player_id": "thief", "jester_player_id": "jester"}
    var result: Result = evaluator.evaluate(alliances, ctx)
    assert_false(result.data[0]["complementary_achieved"])

func test_empty_alliances_returns_ok_empty_array() -> void:
    var result: Result = evaluator.evaluate([], {})
    assert_true(result.success)
    assert_eq(result.data.size(), 0)

func test_unknown_alliance_type_returns_false_gracefully() -> void:
    var alliances := [{"player1_id": "p1", "player2_id": "p2", "alliance_type": "invalid_type"}]
    var result: Result = evaluator.evaluate(alliances, {})
    assert_true(result.success)  # Does not crash
    assert_false(result.data[0]["complementary_achieved"])
```

```gdscript
# test/unit/domain/test_objective_evaluator.gd
extends GutTest

var evaluator: ObjectiveEvaluator

func before_each() -> void:
    evaluator = ObjectiveEvaluator.new()

func test_king_objective_met_when_not_accused() -> void:
    var assignments := [{"player_id": "king", "role_id": "king", "base_points": 1000}]
    var ctx := {"accused_player_id": "police"}
    var result: Result = evaluator.evaluate(assignments, ctx)
    assert_true(result.data[0]["objective_completed"])
    assert_eq(result.data[0]["objective_bonus"], 100)

func test_king_objective_not_met_when_accused() -> void:
    var assignments := [{"player_id": "king", "role_id": "king", "base_points": 1000}]
    var ctx := {"accused_player_id": "king"}
    var result: Result = evaluator.evaluate(assignments, ctx)
    assert_false(result.data[0]["objective_completed"])
    assert_eq(result.data[0]["objective_bonus"], 0)

func test_jester_objective_met_when_accused_but_not_thief() -> void:
    var assignments := [{"player_id": "jester", "role_id": "jester", "base_points": 300}]
    var ctx := {"accused_player_id": "jester", "thief_player_id": "thief"}
    var result: Result = evaluator.evaluate(assignments, ctx)
    assert_true(result.data[0]["objective_completed"])
    assert_eq(result.data[0]["objective_bonus"], 200)

func test_jester_objective_not_met_when_not_accused() -> void:
    var assignments := [{"player_id": "jester", "role_id": "jester", "base_points": 300}]
    var ctx := {"accused_player_id": "king", "thief_player_id": "thief"}
    var result: Result = evaluator.evaluate(assignments, ctx)
    assert_false(result.data[0]["objective_completed"])

func test_spy_gets_150_bonus_when_both_predictions_correct() -> void:
    var assignments := [{"player_id": "spy", "role_id": "spy", "base_points": 500}]
    var ctx := {"spy_who_prediction_correct": true, "spy_outcome_prediction_correct": true}
    var result: Result = evaluator.evaluate(assignments, ctx)
    assert_eq(result.data[0]["objective_bonus"], 150)

func test_spy_gets_75_bonus_when_one_prediction_correct() -> void:
    var assignments := [{"player_id": "spy", "role_id": "spy", "base_points": 500}]
    var ctx := {"spy_who_prediction_correct": true, "spy_outcome_prediction_correct": false}
    var result: Result = evaluator.evaluate(assignments, ctx)
    assert_eq(result.data[0]["objective_bonus"], 75)

func test_merchant_earns_bonus_when_false_accusation_fine_paid() -> void:
    var assignments := [{"player_id": "merchant", "role_id": "merchant", "base_points": 400}]
    var ctx := {"false_accusation_fine_paid": true}
    var result: Result = evaluator.evaluate(assignments, ctx)
    assert_true(result.data[0]["objective_completed"])
    assert_eq(result.data[0]["objective_bonus"], 75)
```

```gdscript
# test/unit/use_cases/test_chat_service.gd
extends GutTest

var chat_service: ChatService
var mock_chat: MockChatPort  # Defined in test/unit/mocks/

func before_each() -> void:
    mock_chat = MockChatPort.new()
    chat_service = ChatService.new(mock_chat)
    await chat_service.join("session_001")

func test_send_message_succeeds_when_chat_unlocked() -> void:
    var result: Result = await chat_service.send("Hello everyone")
    assert_true(result.success)

func test_send_fails_when_chat_locked() -> void:
    chat_service.on_vote_phase_started()
    var result: Result = await chat_service.send("Last second plea")
    assert_false(result.success)
    assert_string_contains(result.error_message, "locked")

func test_send_fails_when_message_too_long() -> void:
    var long_msg: String = "x".repeat(201)
    var result: Result = await chat_service.send(long_msg)
    assert_false(result.success)
    assert_string_contains(result.error_message, "character limit")

func test_send_fails_when_rate_limit_exceeded() -> void:
    for i: int in range(ChatService.MAX_MESSAGES_PER_PHASE):
        await chat_service.send("message %d" % i)
    var result: Result = await chat_service.send("one too many")
    assert_false(result.success)
    assert_string_contains(result.error_message, "limit reached")

func test_rate_limit_resets_on_new_discussion_phase() -> void:
    for i: int in range(ChatService.MAX_MESSAGES_PER_PHASE):
        await chat_service.send("message %d" % i)
    chat_service.on_vote_phase_started()
    chat_service.on_discussion_phase_started()  # New discussion phase
    var result: Result = await chat_service.send("fresh start")
    assert_true(result.success)
```

### Modules Under Test

| Module                 | Test Type | Key Behaviours                                                 |
|------------------------|-----------|----------------------------------------------------------------|
| `AllianceEvaluator`    | Unit TDD  | All 6 alliance types, boundary cases, unknown type graceful handling |
| `ObjectiveEvaluator`   | Unit TDD  | All 10 roles, Spy variable bonus, empty assignments            |
| `ChatService`          | Unit TDD  | Lock/unlock, rate limit, length limit, reset on phase change   |
| `SubmitPrediction`     | Unit TDD  | Invalid type rejected, empty round_id rejected, mock network called |
| ScoringEngine extension| Unit TDD  | Alliance multiplier doubles objective bonus, stacking with base scores |

---

## 7. Out of Scope

- Power-up effect integration into scoring (Phase 2)
- Voice chat (Phase 3)
- Chat message search or history UI (Phase 2)
- Model B objectives (chat-axis) — explicitly Phase 2
- Alliance pairing probabilities tuning (post-beta feedback)
- Admin mute functionality in chat UI (backend supports; UI Phase 2)

---

## 8. Further Notes

### Round Context Dictionary — Full Schema

The `round_context` Dictionary passed to both `AllianceEvaluator` and `ObjectiveEvaluator` is constructed by the `resolve-scoring` Edge Function. For reference, its complete expected keys are:

```
outcome                         : String   — "caught" | "escaped" | "timeout"
thief_player_id                 : String   — UUID
police_player_id                : String   — UUID
accused_player_id               : String   — UUID (empty on timeout)
accomplice_player_id            : String   — UUID (empty if no nomination)
king_player_id                  : String   — UUID
queen_player_id                 : String   — UUID
jester_player_id                : String   — UUID (empty if not in this round)
false_accusation_fine_paid      : bool     — true if any player paid the 25-point fine
pm_royalty_downfall_triggered   : bool     — true if King or Queen was accused
queen_prediction_accused_id     : String   — UUID from sealed prediction (empty if not submitted)
queen_prediction_correct        : bool
general_prediction_correct      : bool
spy_who_prediction_correct      : bool
spy_outcome_prediction_correct  : bool
spy_double_prediction_correct   : bool     — true only if BOTH spy predictions correct
king_accused                    : bool     — true if accused_player_id == king_player_id
```

This is constructed once per scoring run and passed immutably to evaluators. No evaluator modifies it.
