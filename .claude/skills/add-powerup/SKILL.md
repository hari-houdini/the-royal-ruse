---
name: add-powerup
description: Add a new power-up to the game. Use when designing a new ability for a specific role at a specific tier. Triggers include "new power-up", "add a Tier X power-up", "new ability for X role", "add a power-up called X". This skill touches the power-up catalogue, the shop presentation logic, and the Edge Function resolver. Phase 2 and beyond only.
---

# Skill: Add a New Power-Up

Power-ups are role-locked, single-use, and resolved server-side. Each role has 9 power-ups (3 per tier). The shop presents 3 random options from the active role's pool each round.

**Phase gate:** Power-up effect logic is a Phase 2 feature. In Phase 1, the shop UI may be stubbed. Do not implement effect resolution until Phase 2 begins.

## Before You Start

1. Read `docs/tdd/THE_ROYAL_RUSE_TDD.md` Section 5.4 for the full power-up catalogue and pricing.
2. Confirm the role already exists — use `add-role` skill first if not.
3. Confirm the tier and pricing:
   - Tier I = 75 pts
   - Tier II = 350 pts
   - Tier III = 850 pts
4. Identify the effect category:
   - **Score Manipulation** — changes points at scoring time
   - **Information** — reveals hidden data to the buyer
   - **Accusation Interference** — changes who Police can or cannot accuse
   - **Alliance Interference** — disrupts or exposes alliances
   - **Objective Sabotage** — prevents another player completing their objective
5. Confirm the power-up key — unique, `snake_case`, format: `role_name_effect_name` (e.g., `king_royal_shield`, `thief_shadow_step`)

## Files to Touch

1. `src/core/domain/powerup_catalogue.gd` — add the power-up definition
2. `supabase/functions/resolve-scoring/index.ts` — add effect resolution logic
3. `test/unit/domain/test_powerup_catalogue.gd` — add catalogue tests
4. (Phase 2) Scene: power-up shop UI — no changes in Phase 1

---

## Step 1 — Add to Power-Up Catalogue

```gdscript
# src/core/domain/powerup_catalogue.gd
class_name PowerupCatalogue
extends RefCounted

## Static catalogue of all power-up definitions.
## Power-ups are role-locked and presented randomly (3 options per round).

class PowerupDefinition:
    extends RefCounted
    var key: String           # Unique snake_case identifier
    var display_name: String  # Human-readable name shown in shop
    var description: String   # One sentence — what it does
    var role_id: String       # Role that can purchase this
    var tier: int             # 1, 2, or 3
    var cost: int             # 75 (T1), 350 (T2), 850 (T3)
    var effect_category: String  # Score|Information|Accusation|Alliance|Objective

# ── Tier costs (never change these) ──────────────────────────────────────────
const TIER_COSTS: Dictionary = { 1: 75, 2: 350, 3: 850 }

## Returns all power-up definitions. Add new entries here.
static func all_powerups() -> Array[PowerupDefinition]:
    var catalogue: Array[PowerupDefinition] = []

    # ── Your New Power-Up ─────────────────────────────────────────────────────
    var your_powerup := PowerupDefinition.new()
    your_powerup.key = "role_name_effect_name"      # e.g. "thief_smoke_screen"
    your_powerup.display_name = "Effect Name"        # e.g. "Smoke Screen"
    your_powerup.description = "One sentence describing the exact mechanical effect."
    your_powerup.role_id = "your_role_id"            # e.g. "thief"
    your_powerup.tier = 2                            # 1, 2, or 3
    your_powerup.cost = TIER_COSTS[your_powerup.tier]
    your_powerup.effect_category = "Accusation Interference"
    catalogue.append(your_powerup)

    return catalogue

## Returns all power-ups available to a specific role, grouped by tier.
static func for_role(role_id: String) -> Dictionary:
    var result: Dictionary = { 1: [], 2: [], 3: [] }
    for pwup: PowerupDefinition in all_powerups():
        if pwup.role_id == role_id:
            result[pwup.tier].append(pwup)
    return result

## Returns 3 random power-up options for a role's shop presentation.
## Picks one from each tier where available, or fills from random if a tier is empty.
static func random_shop_options(role_id: String) -> Array[PowerupDefinition]:
    var by_tier: Dictionary = for_role(role_id)
    var options: Array[PowerupDefinition] = []
    for tier: int in [1, 2, 3]:
        var pool: Array = by_tier[tier]
        if not pool.is_empty():
            options.append(pool[randi() % pool.size()])
    return options

## Returns true if a player can afford the given power-up.
static func can_afford(powerup_key: String, player_points: int) -> Result:
    for pwup: PowerupDefinition in all_powerups():
        if pwup.key == powerup_key:
            if player_points >= pwup.cost:
                return Result.ok()
            return Result.fail(
                "Insufficient points: need %d, have %d" % [pwup.cost, player_points]
            )
    return Result.fail("Unknown power-up key: %s" % powerup_key)
```

## Step 2 — Add Effect Resolution to Edge Function

```typescript
// In supabase/functions/resolve-scoring/index.ts
// In the power-up effect resolution section:

async function applyPowerupEffects(
  purchases: PowerupPurchase[],
  deltas: Record<string, PlayerDelta>,
  ctx: RoundContext,
  supabase: SupabaseClient
): Promise<void> {
  for (const purchase of purchases) {
    if (!purchase.activated) continue
    switch (purchase.powerup_key) {
      // ... existing cases ...

      case "role_name_effect_name": {
        // Effect: [describe exactly what happens]
        // Example — Score Manipulation:
        const buyer = deltas[purchase.player_id]
        if (buyer) {
          buyer.powerup_delta = (buyer.powerup_delta ?? 0) + 50
        }
        break
      }

      // Information effects — send private Realtime event to buyer:
      case "your_information_powerup": {
        const targetRole = ctx.role_assignments.find(
          (a: any) => a.player_id === ctx.some_target_id
        )
        await supabase.channel(`player:${purchase.player_id}`)
          .send({
            type: "broadcast",
            event: "powerup_info",
            payload: { powerup_key: purchase.powerup_key, revealed: targetRole }
          })
        break
      }
    }
  }
}
```

## Step 3 — Write GUT Tests

```gdscript
# test/unit/domain/test_powerup_catalogue.gd
extends GutTest

func test_new_powerup_exists_in_catalogue() -> void:
    var all: Array[PowerupCatalogue.PowerupDefinition] = PowerupCatalogue.all_powerups()
    var keys: Array = all.map(func(p): return p.key)
    assert_has(keys, "role_name_effect_name")

func test_new_powerup_has_correct_cost() -> void:
    for pwup in PowerupCatalogue.all_powerups():
        if pwup.key == "role_name_effect_name":
            assert_eq(pwup.cost, 350, "Tier II should cost 350")
            return
    fail("Power-up not found in catalogue")

func test_new_powerup_is_role_locked_to_correct_role() -> void:
    var role_options: Dictionary = PowerupCatalogue.for_role("your_role_id")
    var tier_keys: Array = role_options[2].map(func(p): return p.key)
    assert_has(tier_keys, "role_name_effect_name")

func test_new_powerup_not_available_to_other_roles() -> void:
    var other_role_options: Dictionary = PowerupCatalogue.for_role("king")
    var all_keys: Array = []
    for tier in [1, 2, 3]:
        all_keys.append_array(other_role_options[tier].map(func(p): return p.key))
    assert_does_not_have(all_keys, "role_name_effect_name")

func test_can_afford_returns_ok_when_points_sufficient() -> void:
    var result: Result = PowerupCatalogue.can_afford("role_name_effect_name", 400)
    assert_true(result.success)

func test_can_afford_returns_fail_when_points_insufficient() -> void:
    var result: Result = PowerupCatalogue.can_afford("role_name_effect_name", 349)
    assert_false(result.success)
    assert_string_contains(result.error_message, "Insufficient points")
```

## Checklist

- [ ] Power-up key is unique across the entire catalogue (`snake_case`, `role_effect_format`)
- [ ] `display_name` is title-case, 2–3 words maximum
- [ ] `description` is one sentence describing the mechanical effect precisely
- [ ] `role_id` matches an existing role's identifier string exactly
- [ ] `tier` is 1, 2, or 3 — `cost` uses `TIER_COSTS[tier]` (never hardcode the cost)
- [ ] `effect_category` is one of the five defined categories
- [ ] Each role has at most 3 power-ups per tier (9 total per role maximum)
- [ ] Edge Function resolver has a `case` for this power-up key
- [ ] Information-type effects send a private Realtime event to the buyer only
- [ ] GUT tests: key exists, correct cost, role-locked, can_afford pass/fail
- [ ] All GUT tests passing headless

## Power-Up Count per Role (Current State — Update When Adding)

| Role           | T1 | T2 | T3 | Total |
|----------------|----|----|-----|-------|
| King           | 3  | 3  | 3  | 9     |
| Queen          | 3  | 3  | 3  | 9     |
| Police         | 3  | 3  | 3  | 9     |
| Thief          | 3  | 3  | 3  | 9     |
| Prime Minister | 3  | 3  | 3  | 9     |
| General        | 3  | 3  | 3  | 9     |
| Spy            | 3  | 3  | 3  | 9     |
| Merchant       | 3  | 3  | 3  | 9     |
| Jester         | 3  | 3  | 3  | 9     |
| Priest         | 3  | 3  | 3  | 9     |
