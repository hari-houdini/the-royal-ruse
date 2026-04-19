# AGENTS.md — The Royal Ruse

Universal agent instructions for any AI coding assistant (Cursor, Copilot, Codex, Claude, and others).

---

## What This Project Is

The Royal Ruse is a cross-platform social deduction roguelike — a reimagining of *Raja, Rani, Police, Thief* — for 4–10 players on iOS, Android, and Web.

- **Engine:** Godot 4.3 (GDScript)
- **Backend:** Supabase (Auth · PostgreSQL · Realtime · Edge Functions)
- **Platforms:** iOS · Android · Web (HTML5/WASM)
- **Budget:** Zero — all tools are open-source or free-tier

## Canonical Documents

Read before making any non-trivial change.

| Document | Path |
|----------|------|
| Technical Design Document | `docs/tdd/THE_ROYAL_RUSE_TDD.md` |
| Phase 1 Master PRD | `docs/prd/PRD-000-main.md` |
| Godot Scaffold PRD | `docs/prd/PRD-1a-godot-scaffold.md` |
| Supabase Setup PRD | `docs/prd/PRD-1b-supabase-setup.md` |
| Session/Lobby/Roles PRD | `docs/prd/PRD-1c-session-lobby-roles.md` |
| Vote & Scoring PRD | `docs/prd/PRD-1d-vote-scoring.md` |
| Alliance/Objectives/Chat PRD | `docs/prd/PRD-1e-alliance-objectives-chat.md` |
| Deployment PRD | `docs/prd/PRD-1f-vertical-slice-deployment.md` |

---

## Architecture

This project uses **Hexagonal Architecture (Ports and Adapters)**.

```
src/core/domain/     — pure GDScript, zero external dependencies
src/core/ports/      — abstract port interfaces (no concrete logic)
src/core/use_cases/  — orchestrate domain through port interfaces
src/adapters/        — concrete implementations of ports
src/autoloads/       — ServiceLocator, GameBus, SceneManager, Config
src/scenes/          — Godot scenes and UI
test/                — GUT test suite
```

**Import direction (never violate):**
- Scenes import Use Cases only
- Use Cases import Port interfaces and Domain entities only
- Adapters implement Port interfaces only
- Domain Layer imports nothing external

**Key autoloads:**
- `ServiceLocator` — wires adapters to ports at startup. Always use `ServiceLocator.network()` etc. Never import adapters directly.
- `GameBus` — global signal hub. Scenes subscribe to domain events here.
- `SceneManager` — all scene transitions. `SceneManager.go_to("vote")`. Never hardcode paths.
- `Config` — environment variable access. `Config.SUPABASE_URL`.

**Server authority:** The client is untrusted. All scoring runs in Supabase Edge Functions. The client sends intents; the server returns resolved state.

**Result pattern:** Every Use Case and Port method returns a `Result` value object (`Result.ok(data)` / `Result.fail(message)`). Never return raw values or use exceptions.

---

## Hard Rules

### Never

- Calculate or modify scores client-side
- Import an adapter directly — always use `ServiceLocator`
- Hardcode a scene file path — always use `SceneManager`
- Store Supabase credentials in committed code — use `res://.env` and GitHub Secrets
- Declare a `var` without a type annotation in GDScript
- Skip the idempotency check in a scoring Edge Function
- Create a table without immediately enabling RLS
- Write a test that calls a real Supabase endpoint — use mock port implementations
- Exceed 200 lines in a single GDScript file
- Use `await` without checking the returned `Result` for success

### Always

- Read the relevant PRD before implementing a feature
- Write the GUT test before the implementation (TDD — Red → Green → Refactor)
- Inject dependencies via the constructor (`_init`), not from inside domain classes
- Add `push_error()` guards to port interface base methods
- Add a `##` doc comment to every public function
- Register new GameBus signals in `game_bus.gd` before emitting
- Add new scenes to `SceneManager.SCENE_PATHS` before navigating to them
- Use `snake_case` for files/vars/functions; `PascalCase` for class names; `SCREAMING_SNAKE_CASE` for constants
- Validate all inputs at the top of a Use Case's `execute()` before any port calls

### Ask Before Doing

Stop and ask the user before:
- Adding a new phase of development without an existing PRD
- Changing a port interface signature
- Adding a new GameBus signal touching scoring or state machine flow
- Introducing a new external dependency
- Changing the database schema in a way requiring a migration

---

## Coding Standards

### GDScript 4

```gdscript
# Types — always explicit
var player_id: String = ""        # ✅
var player_id = ""                # ❌

# Function signatures — always typed
func execute(session_id: String) -> Result:   # ✅
func execute(session_id):                     # ❌

# Doc comments — every public function
## Connects the player to the session channel.
## [param session_code] The 6-character session identifier.
func join(session_code: String) -> Result:

# File structure order:
# 1. class_name + extends
# 2. Signals
# 3. Constants
# 4. Variables
# 5. _init() / _ready()
# 6. Public functions
# 7. Private functions (prefix with _)

# Naming
# Files:      snake_case.gd
# Classes:    PascalCase
# Signals:    past_tense_snake_case
# Constants:  SCREAMING_SNAKE_CASE
```

### Deno / TypeScript (Edge Functions)

```typescript
// Always explicit types — no `any` on domain data
interface RoundResult { round_id: string; outcome: "caught" | "escaped" | "timeout" }

// Idempotency check — mandatory in every state-mutating function
const { data: existing } = await supabase.from("rounds")
  .select("completed_at").eq("id", round_id).single()
if (existing?.completed_at) {
  return new Response(JSON.stringify({ error: "Already processed" }), { status: 409 })
}

// Error responses — always JSON with "error" key + correct HTTP status
return new Response(JSON.stringify({ error: "Missing round_id" }), { status: 400 })

// Credentials — always via Deno.env.get()
Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!  // Never hardcode
```

### SQL / PostgreSQL

```sql
-- PKs: always UUID + gen_random_uuid()
-- Timestamps: always TIMESTAMPTZ
-- Status fields: TEXT + CHECK constraint (never PostgreSQL ENUM)
-- RLS: enabled immediately after every CREATE TABLE

CREATE TABLE example (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  status     TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'active')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
ALTER TABLE example ENABLE ROW LEVEL SECURITY;

-- Policy naming: "table_action_subject"
CREATE POLICY "example_select_own" ON example
  FOR SELECT USING (player_id = auth.uid());
```

### YAML (GitHub Actions)

```yaml
# Pin all action versions
uses: actions/checkout@v4   # ✅
uses: actions/checkout@main # ❌

# Export jobs always depend on the test job
needs: run-gut-tests

# All credentials via secrets
${{ secrets.SECRET_NAME }}
```

---

## Skills Index

When given a task matching these triggers, read the corresponding skill file first.

| Skill | Trigger | Path |
|-------|---------|------|
| `add-use-case` | new feature, game loop mechanic | `docs/skills/add-use-case/SKILL.md` |
| `add-port-and-adapter` | new external service, new transport | `docs/skills/add-port-and-adapter/SKILL.md` |
| `add-domain-entity` | new game concept, new value object | `docs/skills/add-domain-entity/SKILL.md` |
| `add-scene` | new screen, new UI scene | `docs/skills/add-scene/SKILL.md` |
| `write-gut-test` | write a test, TDD for, test this | `docs/skills/write-gut-test/SKILL.md` |
| `write-mock` | mock a port, fake adapter | `docs/skills/write-mock/SKILL.md` |
| `add-gamebus-signal` | new domain event, broadcast to scenes | `docs/skills/add-gamebus-signal/SKILL.md` |
| `write-edge-function` | server-side logic, Edge Function | `docs/skills/write-edge-function/SKILL.md` |
| `write-migration` | schema change, new table, add column | `docs/skills/write-migration/SKILL.md` |
| `add-rls-policy` | secure a table, row-level security | `docs/skills/add-rls-policy/SKILL.md` |
| `debug-realtime` | Realtime not working, WebSocket issue | `docs/skills/debug-realtime/SKILL.md` |
| `add-role` | new player role, new character class | `docs/skills/add-role/SKILL.md` |
| `add-powerup` | new power-up, new ability | `docs/skills/add-powerup/SKILL.md` |
| `add-alliance-type` | new alliance pairing | `docs/skills/add-alliance-type/SKILL.md` |
| `update-export-config` | new platform, export settings | `docs/skills/update-export-config/SKILL.md` |
| `review-changes` | review my changes, check my code, code review, did I do this right | `docs/skills/review-changes/SKILL.md` |

---

## Critical Facts

- Dev machine: Mac mini M4 Pro (iOS export works locally)
- Godot 4.3 installed — do not reference Godot 3 APIs
- GDScript only — do not suggest C# unless explicitly asked
- Supabase free tier is a hard constraint at < 500 MAU
- No art assets exist — placeholder UI only in Phase 1
- No voice chat in Phase 1 — text chat only
- Offline mode is Phase 2 — do not implement LocalWiFiNetworkAdapter yet
- Power-ups are Phase 2 — shop UI stub only in Phase 1
- Apple Developer Program ($99/year) is the only unavoidable cost
