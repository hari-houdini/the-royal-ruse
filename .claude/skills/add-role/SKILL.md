---
name: add-role
description: Add a new player role to the game. Use when a new role needs to be introduced beyond the existing ten. Triggers include "new role", "add a player role", "new character class", "add a role called X". This skill touches five separate files — follow every step or the role will be incomplete.
---

# Skill: Add a New Player Role

Adding a role is a cross-cutting change. It touches the domain entity, the objective evaluator, the scoring edge function, the alliance evaluator (if the role participates in alliances), and the role assignment edge function. Missing any of these will cause silent bugs at runtime.

## Before You Start

1. Read `docs/tdd/THE_ROYAL_RUSE_TDD.md` Section 5.1 — Role Roster to understand the existing hierarchy.
2. Read `docs/prd/PRD-1c-session-lobby-roles.md` for the role assignment engine.
3. Read `docs/prd/PRD-1e-alliance-objectives-chat.md` for the objective evaluator interface.
4. Decide:
   - **Display name** (Anglicised, title-case)
   - **Base points** (must fit the strict hierarchy — lower than the role above it, higher than the role below)
   - **Min players required** (11 players maximum — must be > 10 if extending beyond the current roster)
   - **Round objective** (Model A — vote-axis only)
   - **Objective bonus** (fixed int, or variable — document the variable logic)
   - **Is mandatory** (only King, Queen, Police, Thief are mandatory — new roles should be false)
   - **Alliance pairings** — does this role participate in any? If yes, use `add-alliance-type` skill after this one.

## Files to Touch (in order)

1. `src/core/domain/entities/role.gd` — add `RoleId` enum value and factory entry
2. `src/core/domain/objective_evaluator.gd` — add objective evaluation case
3. `supabase/functions/assign-roles/index.ts` — add to role definitions array
4. `supabase/functions/resolve-scoring/index.ts` — add objective bonus resolution
5. `test/unit/domain/test_role_assigner.gd` — add player count test
6. `test/unit/domain/test_objective_evaluator.gd` — add objective tests

---

## Step 1 — Add RoleId Enum Value and Factory Entry

```gdscript
# In src/core/domain/entities/role.gd

# Add to the RoleId enum (maintain hierarchy order):
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
    PRIEST          = 9,
    YOUR_NEW_ROLE   = 10,   # ← Add here, continue the sequence
}

# Add to all_roles() static factory (maintain hierarchy order):
static func all_roles() -> Array[Role]:
    var roles: Array[Role] = []
    # ... existing roles ...

    var your_role := Role.new()
    your_role.role_id = RoleId.YOUR_NEW_ROLE
    your_role.display_name = "Your Role Name"
    your_role.base_points = 150          # Must fit hierarchy gap
    your_role.min_players_required = 11  # Next player count threshold
    your_role.objective_description = "One-line description of the round objective"
    your_role.objective_bonus = 120      # Fixed int, or 0 if variable (document below)
    your_role.is_mandatory = false       # Always false for non-core roles
    roles.append(your_role)

    return roles
```

## Step 2 — Add Objective Evaluation Case

```gdscript
# In src/core/domain/objective_evaluator.gd

# In _evaluate_objective() — add a new match case:
func _evaluate_objective(role_id: String, player_id: String, ctx: Dictionary) -> bool:
    match role_id:
        # ... existing cases ...
        "your_new_role":
            # Describe the condition in a comment first, then implement:
            # Objective: [one-line description]
            # True when: [specific condition from round_context]
            return ctx.get("your_condition_key", false)
        _:
            push_warning("ObjectiveEvaluator: unknown role_id '%s'" % role_id)
            return false

# In _calculate_bonus() — add the bonus case:
func _calculate_bonus(role_id: String, player_id: String, ctx: Dictionary) -> int:
    match role_id:
        # ... existing cases ...
        "your_new_role": return 120   # Fixed bonus
        # Or for variable bonus:
        "your_new_role":
            var condition_a: bool = ctx.get("condition_a", false)
            return 200 if condition_a else 100
        _: return 0
```

## Step 3 — Add to assign-roles Edge Function

```typescript
// In supabase/functions/assign-roles/index.ts

const ROLE_DEFINITIONS = [
  // ... existing roles ...
  {
    role_id: "your_new_role",
    base_points: 150,
    min_players: 11,
    is_mandatory: false
  },
]
```

## Step 4 — Add Objective Resolution to resolve-scoring Edge Function

```typescript
// In supabase/functions/resolve-scoring/index.ts
// In the objective evaluation section:

function evaluateObjective(
  roleId: string,
  playerId: string,
  ctx: RoundContext
): { completed: boolean; bonus: number } {
  switch (roleId) {
    // ... existing cases ...
    case "your_new_role": {
      const completed = ctx.yourConditionKey === true
      return { completed, bonus: completed ? 120 : 0 }
    }
    default:
      return { completed: false, bonus: 0 }
  }
}
```

## Step 5 — Add to round_context (if new context key needed)

If the new role's objective depends on data not currently in `round_context`, add the key to the context-building logic in `resolve-scoring/index.ts` and document it in the `round_context` schema at the bottom of `PRD-1e-alliance-objectives-chat.md`.

## Step 6 — Write GUT Tests

```gdscript
# In test/unit/domain/test_role_assigner.gd — add player count test:
func test_eleven_players_includes_your_new_role() -> void:
    # Only if your role unlocks at 11 players
    var players: Array[String] = []
    for i in range(11):
        players.append("p%d" % i)
    var result: Result = assigner.assign(players)
    var role_ids: Array = result.data.map(func(a): return a.role_id)
    assert_has(role_ids, Role.RoleId.YOUR_NEW_ROLE)

# In test/unit/domain/test_objective_evaluator.gd — add objective tests:
func test_your_new_role_objective_met_when_condition_true() -> void:
    var assignments := [{"player_id": "p1", "role_id": "your_new_role", "base_points": 150}]
    var ctx := {"your_condition_key": true}
    var result: Result = evaluator.evaluate(assignments, ctx)
    assert_true(result.data[0]["objective_completed"])
    assert_eq(result.data[0]["objective_bonus"], 120)

func test_your_new_role_objective_not_met_when_condition_false() -> void:
    var assignments := [{"player_id": "p1", "role_id": "your_new_role", "base_points": 150}]
    var ctx := {"your_condition_key": false}
    var result: Result = evaluator.evaluate(assignments, ctx)
    assert_false(result.data[0]["objective_completed"])
    assert_eq(result.data[0]["objective_bonus"], 0)
```

## Checklist

- [ ] `RoleId` enum value added (sequential, never reuse an existing value)
- [ ] Factory entry added to `all_roles()` in correct hierarchy position
- [ ] `base_points` fits the hierarchy gap without colliding with adjacent roles
- [ ] `objective_description` is a single sentence describing the Model A condition
- [ ] `is_mandatory` is `false` (only King/Queen/Police/Thief are mandatory)
- [ ] `_evaluate_objective()` match case added in GDScript
- [ ] `_calculate_bonus()` match case added in GDScript
- [ ] Edge Function `ROLE_DEFINITIONS` array updated
- [ ] Edge Function `evaluateObjective()` updated
- [ ] New `round_context` key documented if one was added
- [ ] GUT tests written and passing for role assignment and objective evaluation
- [ ] Power-ups for this role designed (use `add-powerup` skill — 9 power-ups, 3 per tier)
- [ ] Alliance consideration documented (use `add-alliance-type` if applicable)
