---
name: add-alliance-type
description: Add a new alliance pairing type to the blind alliance system. Use when two roles should have a complementary condition that doubles their objective bonuses when both outcomes co-occur in the same round. Triggers include "new alliance", "new alliance pairing", "add alliance between X and Y", "X and Y should be allied".
---

# Skill: Add a New Alliance Type

Alliances are assigned server-side at round start and revealed at scoring. Neither player knows they are allied until the score reveal. A new alliance type requires a complementary condition — a boolean expression over the round outcome that must be true for both players to receive the multiplier.

## Before You Start

1. Read `docs/tdd/THE_ROYAL_RUSE_TDD.md` Section 5.3 — Alliance System.
2. Read `docs/prd/PRD-1e-alliance-objectives-chat.md` for the `AllianceEvaluator` interface.
3. Decide:
   - **Which two roles** are allied (must both exist in the role roster)
   - **The complementary condition** — what must both roles' round outcomes be, simultaneously, for the alliance to trigger?
   - **The alliance type key** — `snake_case`, format: `role1_role2` (alphabetical order)
4. Confirm the condition uses only keys already present in `round_context` — or plan to add new context keys.

## Files to Touch

1. `src/core/domain/alliance_evaluator.gd` — add condition match case
2. `supabase/functions/assign-roles/index.ts` — add pairing to the pool
3. `test/unit/domain/test_alliance_evaluator.gd` — add condition tests

---

## Step 1 — Add Condition to AllianceEvaluator

```gdscript
# In src/core/domain/alliance_evaluator.gd

func _evaluate_condition(alliance_type: String, ctx: Dictionary) -> bool:
    match alliance_type:
        # ... existing cases ...

        "role_a_role_b":
            # Alliance: Role A + Role B
            # Complementary condition: [describe in plain English]
            # Role A completes when: [condition from round_context]
            # Role B completes when: [condition from round_context]
            # Both must be true simultaneously.
            return ctx.get("role_a_condition_key", false) \
                and ctx.get("role_b_condition_key", false)

        _:
            push_warning("AllianceEvaluator: unknown alliance_type '%s'" % alliance_type)
            return false
```

**Naming convention for alliance_type key:**
- Use alphabetical role order: `general_merchant` not `merchant_general`
- Use underscores: `king_priest` not `king-priest`
- Never use display names — use the `role_id` string (e.g., `prime_minister` not `pm`)

## Step 2 — Add New round_context Keys (if needed)

If the condition depends on a fact not already in `round_context`, add it:

```gdscript
# In AllianceEvaluator, document the new key in the class doc comment:
## round_context expected keys (add new ones here):
##   ...existing keys...
##   role_a_condition_key : bool — true when [describe when this is true]
##   role_b_condition_key : bool — true when [describe when this is true]
```

And add to the Edge Function context builder:

```typescript
// In supabase/functions/resolve-scoring/index.ts
// In the buildRoundContext() function:
const ctx: RoundContext = {
  // ... existing keys ...
  role_a_condition_key: computeRoleACondition(round, roleAssignments),
  role_b_condition_key: computeRoleBCondition(round, roleAssignments),
}
```

## Step 3 — Add Pairing to the Edge Function Pool

```typescript
// In supabase/functions/assign-roles/index.ts

const ALLIANCE_PAIRINGS = [
  // ... existing pairings ...
  {
    type: "role_a_role_b",
    roles: ["role_a", "role_b"]   // Must match role_id strings exactly
  },
]
```

The `assignAlliance()` function automatically picks from eligible pairings based on which roles exist in the current round. No other changes needed in the Edge Function.

## Step 4 — Write GUT Tests

```gdscript
# In test/unit/domain/test_alliance_evaluator.gd

func test_role_a_role_b_alliance_achieved_when_both_conditions_met() -> void:
    var alliances := [{
        "player1_id": "role_a_player",
        "player2_id": "role_b_player",
        "alliance_type": "role_a_role_b"
    }]
    var ctx := {
        "role_a_condition_key": true,
        "role_b_condition_key": true
    }
    var result: Result = evaluator.evaluate(alliances, ctx)
    assert_true(result.success)
    assert_true(result.data[0]["complementary_achieved"],
        "Alliance should trigger when both conditions are true")

func test_role_a_role_b_not_achieved_when_only_role_a_condition_true() -> void:
    var alliances := [{
        "player1_id": "role_a_player",
        "player2_id": "role_b_player",
        "alliance_type": "role_a_role_b"
    }]
    var ctx := {
        "role_a_condition_key": true,
        "role_b_condition_key": false   # Role B didn't complete
    }
    var result: Result = evaluator.evaluate(alliances, ctx)
    assert_false(result.data[0]["complementary_achieved"],
        "Alliance should NOT trigger when only one condition is met")

func test_role_a_role_b_not_achieved_when_only_role_b_condition_true() -> void:
    var alliances := [{
        "player1_id": "role_a_player",
        "player2_id": "role_b_player",
        "alliance_type": "role_a_role_b"
    }]
    var ctx := {
        "role_a_condition_key": false,
        "role_b_condition_key": true
    }
    var result: Result = evaluator.evaluate(alliances, ctx)
    assert_false(result.data[0]["complementary_achieved"])

func test_role_a_role_b_not_achieved_when_neither_condition_true() -> void:
    var alliances := [{
        "player1_id": "role_a_player",
        "player2_id": "role_b_player",
        "alliance_type": "role_a_role_b"
    }]
    var ctx := {
        "role_a_condition_key": false,
        "role_b_condition_key": false
    }
    var result: Result = evaluator.evaluate(alliances, ctx)
    assert_false(result.data[0]["complementary_achieved"])
```

## Checklist

- [ ] Alliance type key uses alphabetical role order in `snake_case`
- [ ] Complementary condition documented in plain English above the `match` case
- [ ] Condition uses only keys present in `round_context` (or new keys documented)
- [ ] New `round_context` keys added to Edge Function `buildRoundContext()` if needed
- [ ] New `round_context` keys documented in PRD-1e schema section
- [ ] `ALLIANCE_PAIRINGS` array updated in `assign-roles` Edge Function
- [ ] `role_id` strings in the Edge Function array match exactly (case-sensitive)
- [ ] GUT tests cover: both conditions true (trigger), each condition alone (no trigger), neither true (no trigger)
- [ ] All GUT tests passing headless

## Existing Alliance Type Reference

| Type                   | Role A          | Role B          | Condition                                              |
|------------------------|-----------------|-----------------|--------------------------------------------------------|
| `thief_jester`         | Thief           | Jester          | Thief escapes AND Jester is accused as non-Thief       |
| `thief_spy`            | Spy             | Thief           | Thief escapes AND Spy correctly predicts both outcomes |
| `king_priest`          | King            | Priest          | King survives unaccused AND Police catches Thief       |
| `prime_minister_queen` | Prime Minister  | Queen           | PM royalty-downfall triggered AND Queen predicts correctly |
| `general_merchant`     | General         | Merchant        | General predicts correctly AND false accusation fine paid |
| `general_spy`          | General         | Spy             | Both General and Spy predict correctly                 |
