# PRD-1d — Vote Mechanic & Scoring Engine

| Field        | Value                                                   |
|--------------|---------------------------------------------------------|
| Version      | 1.0                                                     |
| Date         | April 2026                                              |
| Status       | Approved                                                |
| Sub-Phase    | 1d of 6                                                 |
| Duration     | Weeks 5–6                                               |
| Depends On   | PRD-1c (Session, Lobby & Role Assignment)               |
| Unlocks      | PRD-1e (Alliance, Objectives & Chat)                    |
| Parent PRD   | [PRD-000 — Main](./PRD-000-main.md)                     |

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

Roles are assigned but no game resolution exists. Players have no way to accuse each other, no countdown, no scoring, and no way to progress beyond the discussion phase. This sub-phase implements the heart of the game: the Police countdown, the vote mechanic, and the full server-side scoring pipeline for every role and round outcome.

---

## 2. Solution

Implement two use cases — `SubmitVote` and `ResolveScoring` — and the `ScoringEngine` domain service. The client submits the Police's accusation; the Supabase `resolve-scoring` Edge Function performs all score calculations and broadcasts the results. The client is never trusted to calculate scores.

Specific mechanics implemented:
- 10-second Police countdown, server-authoritative
- Auto-escape on timeout
- Thief steal (100 from Police) + false accusation fine (25 from falsely accused)
- Accomplice bonus (+50 on Thief escape, -30 on Thief caught)
- Points may go negative — no floor
- Wanted Brand assignment on Thief escape (cross-round player status)
- Per-role base points awarded after scoring resolution

---

## 3. Design Choices & Justification

### 3.1 Timer Authority — Server vs Client

| Approach               | Verdict                                                                |
|------------------------|------------------------------------------------------------------------|
| **Server-authoritative timer** | ✅ Chosen. Server stores `vote_opened_at` timestamp. Clients display countdown based on `(vote_opened_at + 10s) - now()`. Auto-escape fires from server at T+10s via a scheduled Edge Function invocation |
| Client-authoritative timer | ❌ Clock skew between devices. Fast clients could rush the vote; slow clients could delay it |
| Client consensus timer | ❌ Requires agreement protocol between clients — unnecessary complexity |

The server fires a `vote_timeout` event via Realtime at exactly T+10 seconds. If Police has not voted, this event triggers the auto-escape scoring path.

### 3.2 Negative Points — Floor or No Floor

**Decision: No floor. Points can go negative.** This was explicitly chosen in the design phase. The game's tension comes from the stakes — a Police officer who consistently guesses wrong becomes a liability, which is authentic to the bluffing genre. Removing negative points would reduce the risk of wrong guesses and therefore reduce the incentive for careful deliberation.

### 3.3 Scoring Pipeline: Edge Function vs Database Trigger

| Approach                  | Verdict                                                              |
|---------------------------|----------------------------------------------------------------------|
| **Edge Function (Deno)**  | ✅ Chosen. TypeScript, readable, testable, can call external APIs if needed |
| PostgreSQL trigger (PL/pgSQL) | ⚠️ Viable for simple cases but harder to debug, less expressive for complex bonus stacking |
| Client-side calculation   | ❌ Unacceptable — trivially cheateable                               |

The scoring pipeline is complex enough (base points + objective bonus + alliance multiplier + accomplice mechanic + false accusation fine) that database trigger logic would be brittle. Edge Functions allow clean TypeScript logic that mirrors the GDScript `ScoringEngine` domain service.

---

## 4. User Stories

1. As Police, I want to see a 10-second countdown with all player role cards displayed, so that I can make my accusation decision under time pressure.
2. As Police, I want to tap a player's role card to cast my vote, so that the accusation mechanic is intuitive and unambiguous.
3. As all players, I want to see the Police's vote selection updating in real time, so that the tension of the final seconds is visible to everyone.
4. As all players, I want the round to auto-resolve in the Thief's favour if Police does not vote within 10 seconds, so that a slow or disconnected Police doesn't freeze the game.
5. As Thief, I want to earn 100 points from Police and 25 from the falsely accused player when Police guesses wrong, so that escaping feels genuinely rewarding.
6. As Police, I want to lose my base 100 points to the Thief when I guess incorrectly, so that wrong accusations carry meaningful consequences.
7. As a falsely accused player, I want to lose 25 points when Police incorrectly accuses me, so that being wrongly named has game consequences.
8. As Thief's Accomplice, I want to earn +50 points when the Thief escapes, so that my secret support is rewarded.
9. As Thief's Accomplice, I want to lose 30 points when the Thief is caught, so that backing the wrong person carries risk.
10. As a player who has previously escaped as Thief, I want to see a Wanted Brand on my profile in subsequent rounds, so that other players know I'm a proven Thief.
11. As Police, I want to earn a +50 bonus for catching a Wanted Thief, so that identifying repeat Thieves is worth the extra effort.
12. As a Wanted Thief who escapes again, I want to claim the +50 bounty that Police would have earned, so that repeat escapes are increasingly rewarding.
13. As all players, I want to see a detailed scoring breakdown after each round, so that I understand exactly why my score changed.
14. As a system, I want all scoring to be calculated server-side, so that no client can manipulate the outcome.
15. As a system, I want scoring to be idempotent — calling the scoring endpoint twice for the same round produces the same result, so that network retries don't cause double-scoring.

---

## 5. Implementation Decisions

### 5.1 Module: SubmitVote Use Case

```gdscript
# res://src/core/use_cases/submit_vote.gd
class_name SubmitVote
extends RefCounted

## Sends the Police player's vote to the server.
## Only the Police may call this. The server validates role ownership before accepting.
## Returns immediately; scoring results arrive via GameBus.scoring_resolved signal.

var _network: INetworkPort
var _auth: IAuthPort

func _init(network: INetworkPort, auth: IAuthPort) -> void:
    _network = network
    _auth = auth

## Submits the Police accusation.
## [param round_id] UUID of the current round.
## [param accused_player_id] UUID of the player being accused.
## Returns Result.ok() if the vote was accepted, Result.fail(message) if rejected.
func execute(round_id: String, accused_player_id: String) -> Result:
    if round_id.is_empty():
        return Result.fail("round_id cannot be empty")
    if accused_player_id.is_empty():
        return Result.fail("accused_player_id cannot be empty")
    if accused_player_id == _auth.get_current_player_id():
        return Result.fail("Police cannot accuse themselves")

    var payload := {
        "round_id": round_id,
        "police_player_id": _auth.get_current_player_id(),
        "accused_player_id": accused_player_id
    }
    return await _network.broadcast_event("vote_cast", payload)
```

### 5.2 Module: ScoringEngine (Domain Service)

**What it is:** A pure, stateless GDScript class that takes a `RoundResult` data structure and returns a `ScoringResult` — the full list of point deltas for every player. This is the TDD crown jewel of the entire project: it contains all the scoring math and has zero external dependencies.

**Why mirror it in GDScript when the Edge Function runs in TypeScript?** The GDScript version is TDD'd exhaustively. When you write the TypeScript Edge Function version, you port the tested GDScript logic directly. If a bug is found, it's fixed in GDScript first (with a test) then ported. The GDScript version is the canonical specification.

```gdscript
# res://src/core/domain/scoring_engine.gd
class_name ScoringEngine
extends RefCounted

## Input structure for one round's scoring.
## All fields are required unless marked optional.
class RoundResult:
    extends RefCounted
    var round_id: String
    var thief_player_id: String
    var police_player_id: String
    var accused_player_id: String           # The player Police accused
    var accomplice_player_id: String        # Empty string if Thief nominated no one
    var outcome: String                     # "caught" | "escaped" | "timeout"
    var thief_has_wanted_brand: bool        # True if Thief previously escaped
    var role_assignments: Array[Dictionary] # [{player_id, role_id, base_points}]

## Output structure: one entry per player.
class PlayerScore:
    extends RefCounted
    var player_id: String
    var base_points: int
    var steal_delta: int         # Positive = gained via steal; negative = lost via steal
    var false_accusation_delta: int  # Negative for falsely accused; positive for Thief
    var accomplice_delta: int
    var wanted_brand_delta: int  # Bounty gained or lost
    var total_delta: int         # Sum of all deltas (can be negative)

## Calculates the complete scoring outcome for one round.
## [param round_result] Fully populated RoundResult.
## Returns Result.ok(Array[PlayerScore]) or Result.fail(message).
func calculate(round_result: RoundResult) -> Result:
    if round_result.role_assignments.is_empty():
        return Result.fail("role_assignments cannot be empty")
    if round_result.outcome not in ["caught", "escaped", "timeout"]:
        return Result.fail("outcome must be 'caught', 'escaped', or 'timeout'")

    var scores: Dictionary = {}  # player_id → PlayerScore

    # Initialise all players with base points
    for assignment: Dictionary in round_result.role_assignments:
        var ps := PlayerScore.new()
        ps.player_id = assignment["player_id"]
        ps.base_points = assignment["base_points"]
        ps.steal_delta = 0
        ps.false_accusation_delta = 0
        ps.accomplice_delta = 0
        ps.wanted_brand_delta = 0
        scores[assignment["player_id"]] = ps

    match round_result.outcome:
        "caught":
            _apply_caught_scoring(round_result, scores)
        "escaped", "timeout":
            _apply_escaped_scoring(round_result, scores)

    # Compute total_delta for each player
    for ps: PlayerScore in scores.values():
        ps.total_delta = ps.base_points + ps.steal_delta + ps.false_accusation_delta + ps.accomplice_delta + ps.wanted_brand_delta

    return Result.ok(scores.values())

## Police correctly identified the Thief.
## - Police keeps their 100 base points (no steal penalty)
## - Thief earns 0 base points
## - Accomplice (if any) loses 30 points
## - Wanted Brand: if Thief had brand, Police earns +50 (brand destroyed)
func _apply_caught_scoring(round_result: RoundResult, scores: Dictionary) -> void:
    var accomplice_id: String = round_result.accomplice_player_id

    if not accomplice_id.is_empty() and scores.has(accomplice_id):
        scores[accomplice_id].accomplice_delta = -30

    if round_result.thief_has_wanted_brand:
        if scores.has(round_result.police_player_id):
            scores[round_result.police_player_id].wanted_brand_delta = 50

## Police guessed wrong, or timeout occurred (Thief auto-escapes).
## - Police loses their 100 base points (steal_delta = -100)
## - Thief gains 100 (steal_delta = +100)
## - Falsely accused player loses 25 (false_accusation_delta = -25)
## - Thief also gains 25 from falsely accused (false_accusation_delta = +25)
## - Accomplice (if any) earns +50
## - Wanted Brand: if Thief had brand, Thief claims the +50 bounty instead of Police
func _apply_escaped_scoring(round_result: RoundResult, scores: Dictionary) -> void:
    var thief_id: String = round_result.thief_player_id
    var police_id: String = round_result.police_player_id
    var accused_id: String = round_result.accused_player_id
    var accomplice_id: String = round_result.accomplice_player_id

    # Steal mechanic
    if scores.has(police_id):
        scores[police_id].steal_delta = -100
    if scores.has(thief_id):
        scores[thief_id].steal_delta = 100

    # False accusation fine (only applies if Police voted — not on timeout)
    if round_result.outcome == "escaped" and not accused_id.is_empty() and accused_id != thief_id:
        if scores.has(accused_id):
            scores[accused_id].false_accusation_delta = -25
        if scores.has(thief_id):
            scores[thief_id].false_accusation_delta += 25

    # Accomplice bonus
    if not accomplice_id.is_empty() and scores.has(accomplice_id):
        scores[accomplice_id].accomplice_delta = 50

    # Wanted Brand: Thief claims the bounty on repeat escape
    if round_result.thief_has_wanted_brand and scores.has(thief_id):
        scores[thief_id].wanted_brand_delta = 50
```

### 5.3 Module: WantedBrandTracker (Domain Service)

```gdscript
# res://src/core/domain/wanted_brand_tracker.gd
class_name WantedBrandTracker
extends RefCounted

## Determines which players should receive or lose a Wanted Brand after a round.
## Returns a Dictionary: { player_id → bool } where true = has brand, false = brand removed.
##
## Rules:
##   - A player who escapes as Thief GAINS a Wanted Brand.
##   - A player who is caught as Thief LOSES their Wanted Brand (if they had one).
##   - Brand persists until that player is caught as Thief.
##   - Brand is attached to the player, not the role — it follows them across rounds.
##
## [param thief_player_id] UUID of this round's Thief.
## [param outcome] "caught" | "escaped" | "timeout"
## [param current_brand_holders] Array[String] of player_ids currently holding a brand.
## Returns Result.ok(Dictionary) mapping player_id → new brand state.
func evaluate(thief_player_id: String, outcome: String, current_brand_holders: Array[String]) -> Result:
    if thief_player_id.is_empty():
        return Result.fail("thief_player_id cannot be empty")

    var brand_changes: Dictionary = {}

    match outcome:
        "caught":
            # Thief is caught — remove brand if they had one
            if current_brand_holders.has(thief_player_id):
                brand_changes[thief_player_id] = false
        "escaped", "timeout":
            # Thief escapes — grant brand
            if not current_brand_holders.has(thief_player_id):
                brand_changes[thief_player_id] = true
            # Brand remains if they already had one (no change needed)
        _:
            return Result.fail("outcome must be 'caught', 'escaped', or 'timeout'")

    return Result.ok(brand_changes)
```

### 5.4 Edge Function: resolve-scoring (Full Implementation)

```typescript
// supabase/functions/resolve-scoring/index.ts
import { serve } from "https://deno.land/std@0.177.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"

interface PlayerDelta {
  player_id: string
  base_points: number
  steal_delta: number
  false_accusation_delta: number
  accomplice_delta: number
  wanted_brand_delta: number
  total_delta: number
}

serve(async (req: Request) => {
  const { round_id, accused_player_id, police_player_id } = await req.json()
  if (!round_id) return new Response(JSON.stringify({ error: "round_id required" }), { status: 400 })

  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  )

  // Idempotency check — do not score a round twice
  const { data: existingRound } = await supabase
    .from("rounds").select("completed_at, outcome, thief_player_id, accomplice_player_id")
    .eq("id", round_id).single()

  if (existingRound?.completed_at) {
    return new Response(JSON.stringify({ error: "Round already scored" }), { status: 409 })
  }

  const thief_id = existingRound.thief_player_id
  const accomplice_id = existingRound.accomplice_player_id

  // Determine outcome
  const outcome = (accused_player_id === thief_id) ? "caught" : "escaped"

  // Fetch role assignments
  const { data: roleRows } = await supabase.from("round_roles")
    .select("player_id, role_id, base_points").eq("round_id", round_id)

  // Fetch Wanted Brand holders
  const { data: brandHolders } = await supabase.from("users")
    .select("id").eq("has_wanted_brand", true)
  const brandHolderIds = brandHolders?.map((u: any) => u.id) ?? []
  const thiefHasBrand = brandHolderIds.includes(thief_id)

  const deltas: Record<string, PlayerDelta> = {}
  for (const row of roleRows ?? []) {
    deltas[row.player_id] = {
      player_id: row.player_id, base_points: row.base_points,
      steal_delta: 0, false_accusation_delta: 0,
      accomplice_delta: 0, wanted_brand_delta: 0, total_delta: 0
    }
  }

  if (outcome === "caught") {
    if (accomplice_id && deltas[accomplice_id]) deltas[accomplice_id].accomplice_delta = -30
    if (thiefHasBrand && deltas[police_player_id]) deltas[police_player_id].wanted_brand_delta = 50
  } else {
    if (deltas[police_player_id]) deltas[police_player_id].steal_delta = -100
    if (deltas[thief_id]) deltas[thief_id].steal_delta = 100
    if (accused_player_id && accused_player_id !== thief_id) {
      if (deltas[accused_player_id]) deltas[accused_player_id].false_accusation_delta = -25
      if (deltas[thief_id]) deltas[thief_id].false_accusation_delta += 25
    }
    if (accomplice_id && deltas[accomplice_id]) deltas[accomplice_id].accomplice_delta = 50
    if (thiefHasBrand && deltas[thief_id]) deltas[thief_id].wanted_brand_delta = 50
  }

  for (const d of Object.values(deltas)) {
    d.total_delta = d.base_points + d.steal_delta + d.false_accusation_delta + d.accomplice_delta + d.wanted_brand_delta
  }

  // Update round as complete
  await supabase.from("rounds").update({
    outcome, accused_player_id, completed_at: new Date().toISOString()
  }).eq("id", round_id)

  // Update cumulative scores
  for (const d of Object.values(deltas)) {
    await supabase.rpc("increment_score", {
      p_session_id: existingRound.session_id,
      p_player_id: d.player_id,
      p_delta: d.total_delta
    })
  }

  // Update Wanted Brand
  if (outcome === "escaped" && !thiefHasBrand) {
    await supabase.from("users").update({ has_wanted_brand: true }).eq("id", thief_id)
  } else if (outcome === "caught" && thiefHasBrand) {
    await supabase.from("users").update({ has_wanted_brand: false }).eq("id", thief_id)
  }

  const scores = Object.values(deltas)
  return new Response(JSON.stringify({ outcome, scores }), { status: 200 })
})
```

### 5.5 Vote UI — VoteScene Interface Contract

The `VoteScene` subscribes to `GameBus.session_status_changed` and becomes active when status = `VOTE`. It needs to:
- Display a card for each player (role avatar placeholder, role name, player name)
- Show the 10-second countdown prominently
- Only the Police sees an interactive card (others see read-only)
- As Police taps a card, broadcast a preview event so others see the cursor moving in real time
- On confirm (tap again or single tap), call `SubmitVote.execute()`

```gdscript
# res://src/scenes/vote/vote_scene.gd
class_name VoteScene
extends Control

## Signals emitted to parent/GameBus when vote resolves.
signal vote_submitted(accused_player_id: String)

## Populated before scene becomes active.
var round_id: String
var current_player_id: String
var is_police: bool
var players: Array[Dictionary]  # [{player_id, display_name, role_id, role_display_name}]

var _countdown_timer: float = 10.0
var _selected_player_id: String = ""
var _vote_submitted: bool = false

func _process(delta: float) -> void:
    if not is_police or _vote_submitted:
        return
    _countdown_timer -= delta
    _update_countdown_display(_countdown_timer)
    if _countdown_timer <= 0.0:
        _on_timeout()

func on_card_tapped(player_id: String) -> void:
    if not is_police or _vote_submitted:
        return
    _selected_player_id = player_id
    # Broadcast preview (real-time cursor shown to all)
    ServiceLocator.network().broadcast_event("vote_preview", {
        "police_id": current_player_id,
        "hovering_player_id": player_id
    })

func on_card_confirmed(player_id: String) -> void:
    if not is_police or _vote_submitted or player_id.is_empty():
        return
    _vote_submitted = true
    var use_case := SubmitVote.new(ServiceLocator.network(), ServiceLocator.auth())
    use_case.execute(round_id, player_id)
    vote_submitted.emit(player_id)

func _on_timeout() -> void:
    if not _vote_submitted:
        _vote_submitted = true
        # Server fires auto-escape via scheduled Realtime event
        # Client simply stops the timer UI — no action needed
        pass

func _update_countdown_display(remaining: float) -> void:
    pass  # Update UI label — placeholder implementation
```

---

## 6. Testing Decisions

### ScoringEngine — Primary TDD Target

The ScoringEngine is the highest-value TDD target in the entire project. Every scoring scenario is a test case.

```gdscript
# test/unit/domain/test_scoring_engine.gd
extends GutTest

var engine: ScoringEngine

func before_each() -> void:
    engine = ScoringEngine.new()

func _make_round(outcome: String, thief_accused: bool = false, has_brand: bool = false) -> ScoringEngine.RoundResult:
    var r := ScoringEngine.RoundResult.new()
    r.round_id = "test_round"
    r.thief_player_id = "thief"
    r.police_player_id = "police"
    r.accused_player_id = "thief" if thief_accused else "king"
    r.accomplice_player_id = "queen"
    r.outcome = outcome
    r.thief_has_wanted_brand = has_brand
    r.role_assignments = [
        {"player_id": "king",   "role_id": "king",   "base_points": 1000},
        {"player_id": "queen",  "role_id": "queen",  "base_points": 800},
        {"player_id": "police", "role_id": "police", "base_points": 100},
        {"player_id": "thief",  "role_id": "thief",  "base_points": 0},
    ]
    return r

func _find_score(scores: Array, player_id: String) -> ScoringEngine.PlayerScore:
    for s in scores:
        if s.player_id == player_id:
            return s
    return null

func test_police_earns_base_points_when_correct() -> void:
    var r := _make_round("caught", true)
    var result: Result = engine.calculate(r)
    assert_true(result.success)
    var police_score: ScoringEngine.PlayerScore = _find_score(result.data, "police")
    assert_eq(police_score.total_delta, 100)

func test_thief_earns_zero_when_caught() -> void:
    var r := _make_round("caught", true)
    var result: Result = engine.calculate(r)
    var thief_score: ScoringEngine.PlayerScore = _find_score(result.data, "thief")
    assert_eq(thief_score.total_delta, 0)

func test_accomplice_loses_30_when_thief_caught() -> void:
    var r := _make_round("caught", true)
    var result: Result = engine.calculate(r)
    var queen_score: ScoringEngine.PlayerScore = _find_score(result.data, "queen")
    assert_eq(queen_score.accomplice_delta, -30)

func test_police_loses_100_when_wrong() -> void:
    var r := _make_round("escaped", false)
    var result: Result = engine.calculate(r)
    var police_score: ScoringEngine.PlayerScore = _find_score(result.data, "police")
    assert_eq(police_score.steal_delta, -100)

func test_thief_steals_100_on_escape() -> void:
    var r := _make_round("escaped", false)
    var result: Result = engine.calculate(r)
    var thief_score: ScoringEngine.PlayerScore = _find_score(result.data, "thief")
    assert_eq(thief_score.steal_delta, 100)

func test_falsely_accused_loses_25() -> void:
    var r := _make_round("escaped", false)  # King is accused instead of Thief
    var result: Result = engine.calculate(r)
    var king_score: ScoringEngine.PlayerScore = _find_score(result.data, "king")
    assert_eq(king_score.false_accusation_delta, -25)

func test_thief_gains_25_from_false_accusation() -> void:
    var r := _make_round("escaped", false)
    var result: Result = engine.calculate(r)
    var thief_score: ScoringEngine.PlayerScore = _find_score(result.data, "thief")
    assert_eq(thief_score.false_accusation_delta, 25)

func test_accomplice_earns_50_on_thief_escape() -> void:
    var r := _make_round("escaped", false)
    var result: Result = engine.calculate(r)
    var queen_score: ScoringEngine.PlayerScore = _find_score(result.data, "queen")
    assert_eq(queen_score.accomplice_delta, 50)

func test_score_can_go_negative() -> void:
    # Police has 100 base points, loses 100 via steal = 0. Also falsely accused: -25 from fine is on accused not police.
    # Test that a player with 0 base points who loses points goes negative.
    var r := ScoringEngine.RoundResult.new()
    r.round_id = "neg_test"
    r.thief_player_id = "thief"
    r.police_player_id = "police"
    r.accused_player_id = "thief"  # caught — so accomplice loses 30
    r.accomplice_player_id = "thief"  # Edge case: thief is their own accomplice? Server prevents this, but engine handles it gracefully
    r.outcome = "caught"
    r.thief_has_wanted_brand = false
    r.role_assignments = [
        {"player_id": "police", "role_id": "police", "base_points": 100},
        {"player_id": "thief",  "role_id": "thief",  "base_points": 0},
    ]
    # Remove accomplice from this edge case
    r.accomplice_player_id = ""
    var result: Result = engine.calculate(r)
    assert_true(result.success)

func test_wanted_brand_gives_police_50_bonus_on_catch() -> void:
    var r := _make_round("caught", true, true)  # Thief has brand, police catches them
    var result: Result = engine.calculate(r)
    var police_score: ScoringEngine.PlayerScore = _find_score(result.data, "police")
    assert_eq(police_score.wanted_brand_delta, 50)
    assert_eq(police_score.total_delta, 150)  # 100 base + 50 brand bonus

func test_wanted_thief_claims_bounty_on_escape() -> void:
    var r := _make_round("escaped", false, true)  # Thief has brand, escapes
    var result: Result = engine.calculate(r)
    var thief_score: ScoringEngine.PlayerScore = _find_score(result.data, "thief")
    assert_eq(thief_score.wanted_brand_delta, 50)

func test_invalid_outcome_returns_failure() -> void:
    var r := _make_round("INVALID", false)
    var result: Result = engine.calculate(r)
    assert_false(result.success)

func test_empty_role_assignments_returns_failure() -> void:
    var r := ScoringEngine.RoundResult.new()
    r.outcome = "caught"
    r.role_assignments = []
    var result: Result = engine.calculate(r)
    assert_false(result.success)
```

### Modules Under Test

| Module                | Test Type | Key Scenarios                                                  |
|-----------------------|-----------|----------------------------------------------------------------|
| `ScoringEngine`       | Unit TDD  | All 15 test cases above, plus edge cases (empty accomplice, timeout) |
| `WantedBrandTracker`  | Unit TDD  | Brand granted on escape, removed on catch, no change if already branded |
| `SubmitVote`          | Unit TDD  | Self-accusation rejected, empty round_id rejected, mock network called |

---

## 7. Out of Scope

- Alliance multipliers on objective bonuses (PRD-1e)
- Power-up effect application to scoring (Phase 2)
- Objective bonus calculation (PRD-1e)
- Disconnection handling during vote phase (covered in PRD-1c reconnection flow)
- VoteScene full UI implementation (placeholder scene in this phase)

---

## 8. Further Notes

### Idempotency in Edge Functions

The `resolve-scoring` Edge Function checks `completed_at IS NOT NULL` before processing. If called twice with the same `round_id`, it returns `409 Conflict`. This protects against:
- Network retries from the client
- Double-taps on the vote card
- Race conditions if two events trigger scoring simultaneously

Idempotency is a standard backend engineering practice. Every state-mutating endpoint should have it.

### PostgreSQL RPC for Score Increment

Rather than `UPDATE scores SET cumulative_points = cumulative_points + delta`, use a PostgreSQL function (RPC) to atomically increment. This prevents race conditions when multiple rounds complete in quick succession:

```sql
CREATE OR REPLACE FUNCTION increment_score(
  p_session_id UUID,
  p_player_id UUID,
  p_delta INT
) RETURNS VOID AS $$
BEGIN
  INSERT INTO scores (session_id, player_id, cumulative_points)
  VALUES (p_session_id, p_player_id, p_delta)
  ON CONFLICT (session_id, player_id)
  DO UPDATE SET
    cumulative_points = scores.cumulative_points + p_delta,
    recorded_at = now();
END;
$$ LANGUAGE plpgsql;
```

Add a unique constraint: `UNIQUE (session_id, player_id)` on the `scores` table to make the upsert work correctly.
