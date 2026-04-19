# CLAUDE.md — The Royal Ruse

<project_context>

## What This Project Is

The Royal Ruse is a cross-platform social deduction roguelike — a reimagining of the Indian childhood game *Raja, Rani, Police, Thief* — built for iOS, Android, and Web. It is a real-time multiplayer game for 4–10 players with role-based objectives, a blind alliance system, and a tiered power-up economy.

**Engine:** Godot 4.3 (GDScript)
**Backend:** Supabase (Auth · PostgreSQL · Realtime · Edge Functions)
**Platforms:** iOS · Android · Web (HTML5/WASM)
**Developer:** Solo
**Budget:** Zero — all tools are open-source or free-tier

## Canonical Documents

Before making any non-trivial change, read the relevant document first.

| Document | Path | Purpose |
|----------|------|---------|
| Technical Design Document | `docs/tdd/THE_ROYAL_RUSE_TDD.md` | Architecture, tech stack decisions, data model, security, roadmap |
| Phase 1 Master PRD | `docs/prd/PRD-000-main.md` | Phase 1 overview, principles, dependency map |
| PRD-1a — Godot Scaffold | `docs/prd/PRD-1a-godot-scaffold.md` | Project structure, port interfaces, autoloads, GUT setup |
| PRD-1b — Supabase Setup | `docs/prd/PRD-1b-supabase-setup.md` | Schema, RLS, Edge Functions, adapter implementations |
| PRD-1c — Session/Lobby/Roles | `docs/prd/PRD-1c-session-lobby-roles.md` | Session flow, role assignment, name generation |
| PRD-1d — Vote & Scoring | `docs/prd/PRD-1d-vote-scoring.md` | Police vote, scoring engine, Wanted Brand |
| PRD-1e — Alliance/Objectives/Chat | `docs/prd/PRD-1e-alliance-objectives-chat.md` | Alliance evaluator, objective evaluator, chat service |
| PRD-1f — Deployment | `docs/prd/PRD-1f-vertical-slice-deployment.md` | CI/CD, export pipeline, acceptance checklist |

</project_context>

---

<architecture>

## Architecture — Non-Negotiable Rules

This project uses **Hexagonal Architecture (Ports and Adapters)**. Every architectural decision flows from this.

### Layer Map

```
src/core/domain/        ← Domain Layer: pure GDScript, zero external deps
src/core/ports/         ← Port interfaces: abstract contracts only
src/core/use_cases/     ← Use Cases: orchestrate domain via ports
src/adapters/           ← Adapter Layer: concrete implementations of ports
src/autoloads/          ← Global singletons: ServiceLocator, GameBus, SceneManager, Config
src/scenes/             ← Scene Layer: Godot scenes and UI scripts
test/                   ← GUT test suite
```

### Import Direction (enforced — never violate)

```
Scene Layer  →  Use Cases  →  Port Interfaces  ←  Adapters
                                     ↑
                            Domain Layer (no imports from above)
```

- Domain Layer imports: nothing external. `RefCounted` base only.
- Use Cases import: Port interfaces and Domain entities only.
- Adapters import: Port interfaces they implement + external SDKs.
- Scenes import: Use Cases and `GameBus` signals only. Never import adapters.

### Key Autoloads

| Autoload        | Purpose                                                   | Access Pattern              |
|-----------------|-----------------------------------------------------------|-----------------------------|
| `ServiceLocator`| Wires adapters to ports at startup                        | `ServiceLocator.network()`  |
| `GameBus`       | Global signal hub for domain events                       | `GameBus.scoring_resolved.connect(...)` |
| `SceneManager`  | All scene transitions — no hardcoded paths anywhere else  | `SceneManager.go_to("vote")`|
| `Config`        | Environment variable access                               | `Config.SUPABASE_URL`       |

### Server Authority

The client is untrusted. All score calculations happen in Supabase Edge Functions. The client sends **intents** and receives **resolved state**. Never calculate scores client-side.

### Result Pattern

Every Use Case and Port method returns a `Result` value object. Never use raw returns or exceptions.

```gdscript
# Correct
func execute(...) -> Result:
    return Result.ok(data)    # success
    return Result.fail("message")  # failure

# Wrong — never do this
func execute(...) -> String:  # raw return
func execute(...) -> void:    # silently fails
```

</architecture>

---

<rules>

## Hard Rules — Read Before Every Change

### NEVER

- `NEVER` calculate or modify scores client-side. Scores flow from Edge Functions only.
- `NEVER` import an adapter directly into a scene or use case. Always use `ServiceLocator`.
- `NEVER` hardcode a scene file path. All transitions go through `SceneManager`.
- `NEVER` emit `GameBus` signals from an adapter. Adapters emit their own port signals; use cases or autoloads translate to GameBus events.
- `NEVER` store Supabase credentials in committed code. They live in `res://.env` (git-ignored) and GitHub Secrets.
- `NEVER` add a `var` without a type annotation in GDScript.
- `NEVER` use `\n` inside a string to create paragraphs in GDScript — use separate statements.
- `NEVER` use unicode bullet characters manually — use Godot's `RichTextLabel` BBCode or Label arrays.
- `NEVER` write a scoring Edge Function that can execute twice for the same `round_id`. Always add an idempotency check first.
- `NEVER` skip RLS when creating a new table. Every `CREATE TABLE` is immediately followed by `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` and the appropriate policies.
- `NEVER` trust client-supplied values for score deltas, role assignments, or alliance data.
- `NEVER` add power-up effect logic without a corresponding Edge Function resolver entry.
- `NEVER` write a test that calls a real Supabase endpoint. Use mock port implementations.
- `NEVER` exceed 200 lines in a single GDScript file. Split at logical boundaries.
- `NEVER` use `await` without checking the returned `Result` for success.

### ALWAYS

- `ALWAYS` read the relevant PRD before implementing a feature. The interface signatures are specified — match them exactly.
- `ALWAYS` write the GUT test before the implementation (TDD — Red-Green-Refactor).
- `ALWAYS` inject dependencies via the constructor (`_init`), never by reaching into `ServiceLocator` from within domain classes.
- `ALWAYS` use `push_error()` in port interface base methods so unimplemented adapters fail loudly.
- `ALWAYS` add a doc comment (`##`) to every public function.
- `ALWAYS` register new `GameBus` signals in `game_bus.gd` before emitting them.
- `ALWAYS` add new scenes to `SceneManager.SCENE_PATHS` before navigating to them.
- `ALWAYS` use `snake_case` for files, variables, and functions; `PascalCase` for class names; `SCREAMING_SNAKE_CASE` for constants.
- `ALWAYS` use `WidthType.DXA` (never percentage) when touching any docx generation scripts.
- `ALWAYS` validate all inputs at the top of a Use Case's `execute()` before any port calls.
- `ALWAYS` run `godot --headless` GUT tests locally before pushing.
- `ALWAYS` check if a `round_id` is already scored before writing any scoring logic.

### ASK BEFORE DOING

When given a task that involves any of the following, stop and ask the user which direction to take before proceeding:

- Adding a new phase of development not covered by an existing PRD
- Changing a port interface signature (affects all adapters and all tests)
- Adding a new `GameBus` signal that touches the scoring or state machine flow
- Introducing a new external dependency (npm package, GDScript addon, Deno module)
- Changing the database schema in a way that requires a migration

</rules>

---

<code_style>

## Coding Standards

### GDScript 4

```gdscript
# ── Naming ────────────────────────────────────────────────────────────────────
# Files:       snake_case.gd
# Classes:     PascalCase (class_name PlayerScore)
# Variables:   snake_case
# Functions:   snake_case
# Constants:   SCREAMING_SNAKE_CASE
# Signals:     past_tense_snake_case (scoring_resolved, round_ended)
# Enums:       PascalCase enum name, SCREAMING_SNAKE_CASE values

# ── Types ─────────────────────────────────────────────────────────────────────
# ALWAYS use static types — no untyped vars, no untyped returns
var player_id: String = ""          # ✅
var player_id = ""                  # ❌
func get_score() -> int: ...        # ✅
func get_score(): ...               # ❌

# ── Doc comments ──────────────────────────────────────────────────────────────
## Every public function has a ## doc comment above it.
## Use [param name] to document parameters.
## [param session_id] The UUID of the current session.
func execute(session_id: String) -> Result:

# ── File structure order ──────────────────────────────────────────────────────
# 1. class_name and extends
# 2. ## class-level doc comment
# 3. Signals
# 4. Constants
# 5. @export vars
# 6. Private vars
# 7. _init() / _ready()
# 8. Public functions
# 9. Private functions (prefixed _)

# ── Line limits ───────────────────────────────────────────────────────────────
# Maximum 200 lines per file. Split at logical domain boundaries if exceeded.
# Maximum 80 characters per line where practical.

# ── Error handling ────────────────────────────────────────────────────────────
# Always return Result. Always check result.success before using result.data.
var result: Result = await use_case.execute(payload)
if not result.success:
    push_error("Failed: %s" % result.error_message)
    return

# ── Signals ───────────────────────────────────────────────────────────────────
# Declare typed signals:
signal scoring_resolved(round_scores: Array[Dictionary])   # ✅
signal scoring_resolved(data)                              # ❌

# ── await ─────────────────────────────────────────────────────────────────────
# All port methods that involve network I/O must be awaited.
# Mark the calling function as async (no keyword needed — GDScript infers it).
```

### Deno / TypeScript (Edge Functions)

```typescript
// ── File naming ───────────────────────────────────────────────────────────────
// Edge Functions: supabase/functions/kebab-case-name/index.ts

// ── Types ─────────────────────────────────────────────────────────────────────
// Always use explicit TypeScript types. No `any` except when interfacing with
// unknown Supabase JSON shapes — and even then, define an interface.
interface RoundResult {
  round_id: string
  outcome: "caught" | "escaped" | "timeout"
  // ...
}

// ── Idempotency ───────────────────────────────────────────────────────────────
// Every state-mutating Edge Function checks for prior execution first.
const { data: existing } = await supabase
  .from("rounds").select("completed_at").eq("id", round_id).single()
if (existing?.completed_at) {
  return new Response(JSON.stringify({ error: "Already processed" }), { status: 409 })
}

// ── Error responses ───────────────────────────────────────────────────────────
// All errors return JSON with an "error" key and an appropriate HTTP status.
return new Response(JSON.stringify({ error: "Missing round_id" }), { status: 400 })

// ── Environment variables ─────────────────────────────────────────────────────
// Always use Deno.env.get(). Never hardcode credentials.
const supabase = createClient(
  Deno.env.get("SUPABASE_URL")!,
  Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!   // Service role — admin access
)

// ── Validation ────────────────────────────────────────────────────────────────
// Validate all required inputs at the top of the serve() handler.
// Return 400 immediately on missing required fields.

// ── Line limits ───────────────────────────────────────────────────────────────
// Maximum 150 lines per Edge Function file. Extract helpers to shared modules.
```

### SQL / PostgreSQL (Migrations & RLS)

```sql
-- ── File naming ──────────────────────────────────────────────────────────────
-- supabase/migrations/YYYYMMDDHHMMSS_descriptive_name.sql

-- ── Table conventions ─────────────────────────────────────────────────────────
-- Primary keys: UUID, generated server-side with gen_random_uuid()
-- Timestamps:   TIMESTAMPTZ (always timezone-aware, never TIMESTAMP)
-- Foreign keys: ON DELETE CASCADE where child records are meaningless without parent
-- Booleans:     NOT NULL with a DEFAULT
-- Status enums: TEXT with CHECK constraints (not PostgreSQL ENUM type — harder to migrate)

CREATE TABLE example (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  status     TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'done')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ── RLS is mandatory ──────────────────────────────────────────────────────────
-- Every CREATE TABLE is immediately followed by:
ALTER TABLE example ENABLE ROW LEVEL SECURITY;
-- Then at minimum one SELECT policy. Never leave a table without policies.

-- ── Policy naming ─────────────────────────────────────────────────────────────
-- Format: "table_action_subject" — e.g. "scores_select_session_participants"
CREATE POLICY "example_select_own" ON example
  FOR SELECT USING (player_id = auth.uid());

-- ── Never use PostgreSQL ENUM ─────────────────────────────────────────────────
-- Use TEXT + CHECK constraint. ENUMs require a migration to add values; TEXT does not.
```

### YAML (GitHub Actions)

```yaml
# ── File naming ───────────────────────────────────────────────────────────────
# .github/workflows/kebab-case-name.yml

# ── Job naming ────────────────────────────────────────────────────────────────
# Descriptive human-readable names in sentence case.
jobs:
  run-gut-tests:
    name: Run GUT Test Suite

# ── Dependency chaining ───────────────────────────────────────────────────────
# Export jobs always depend on the test job via `needs: run-gut-tests`.
# Never deploy a build that hasn't passed tests.

# ── Secrets ───────────────────────────────────────────────────────────────────
# All credentials via ${{ secrets.SECRET_NAME }}. Never hardcode.

# ── Pinned versions ───────────────────────────────────────────────────────────
# Pin action versions to avoid supply-chain attacks.
uses: actions/checkout@v4   # ✅ pinned
uses: actions/checkout@main # ❌ unpinned
```

### Markdown (PRDs, Skills, Docs)

```markdown
<!-- ── Headings ──────────────────────────────────────────────────────────── -->
<!-- H1: Document title only (one per document) -->
<!-- H2: Major sections -->
<!-- H3: Sub-sections -->
<!-- Never skip levels (H1 → H3) -->

<!-- ── Tables ────────────────────────────────────────────────────────────── -->
<!-- All comparison tables have a Verdict column with ✅ ⚠️ ❌ markers -->
<!-- ✅ = chosen or preferred  ⚠️ = viable with caveats  ❌ = rejected -->

<!-- ── Code blocks ─────────────────────────────────────────────────────────── -->
<!-- Always specify the language on every code fence -->
```gdscript
# GDScript
```
```typescript
// TypeScript
```

<!-- ── PRD/Skill frontmatter ───────────────────────────────────────────────── -->
<!-- Every PRD and SKILL.md file opens with a metadata table -->
| Field  | Value |
|--------|-------|
| Version| 1.0   |
```

</code_style>

---

<skills>

## Available Skills

Skills are reusable task instructions. When you receive a task that matches a skill trigger, read the relevant `SKILL.md` before writing any code.

| # | Skill | Trigger Phrases | Path |
|---|-------|----------------|------|
| 1 | `add-use-case` | "add a use case", "new feature in game loop", "implement X mechanic" | `.claude/skills/add-use-case/SKILL.md` |
| 2 | `add-port-and-adapter` | "new external dependency", "add support for X service", "add offline support for X" | `.claude/skills/add-port-and-adapter/SKILL.md` |
| 3 | `add-domain-entity` | "new game concept", "add a value object", "new entity" | `.claude/skills/add-domain-entity/SKILL.md` |
| 4 | `add-scene` | "new screen", "new UI scene", "add a scene for X" | `.claude/skills/add-scene/SKILL.md` |
| 5 | `write-gut-test` | "write a test", "TDD for", "add tests for", "test this module" | `.claude/skills/write-gut-test/SKILL.md` |
| 6 | `write-mock` | "mock the X port", "fake adapter for testing", "create a mock" | `.claude/skills/write-mock/SKILL.md` |
| 7 | `add-gamebus-signal` | "new domain event", "add a signal", "broadcast X to all scenes" | `.claude/skills/add-gamebus-signal/SKILL.md` |
| 8 | `write-edge-function` | "server-side logic", "new Edge Function", "scoring logic", "validate on server" | `.claude/skills/write-edge-function/SKILL.md` |
| 9 | `write-migration` | "schema change", "new table", "add column", "database migration" | `.claude/skills/write-migration/SKILL.md` |
| 10 | `add-rls-policy` | "add RLS", "secure this table", "restrict access to", "row-level security" | `.claude/skills/add-rls-policy/SKILL.md` |
| 11 | `debug-realtime` | "Realtime not working", "events not arriving", "WebSocket issue", "channel not connecting" | `.claude/skills/debug-realtime/SKILL.md` |
| 12 | `add-role` | "new role", "add a player role", "new character class" | `.claude/skills/add-role/SKILL.md` |
| 13 | `add-powerup` | "new power-up", "add a Tier X power-up", "new ability" | `.claude/skills/add-powerup/SKILL.md` |
| 14 | `add-alliance-type` | "new alliance", "new alliance pairing", "add alliance between X and Y" | `.claude/skills/add-alliance-type/SKILL.md` |
| 15 | `update-export-config` | "new platform", "update build", "export settings", "change export target" | `.claude/skills/update-export-config/SKILL.md` |
| 16 | `review-changes` | "review my changes", "check my code", "review this", "code review", "did I do this right", "check before I commit" | `.claude/skills/review-changes/SKILL.md` |

</skills>

---

<important>

## Critical Project Facts

- **Mac mini M4 Pro** is the dev machine. iOS export works natively — no need for a separate macOS CI machine for local builds.
- **Godot 4.3** is installed. Do not reference Godot 3 APIs — they are incompatible.
- **GDScript only** for game client code. Do not suggest C# unless the user explicitly asks.
- **Supabase free tier** is the hard constraint. Do not suggest features that require paid Supabase plans for < 500 MAU.
- **Zero art assets exist** currently. Placeholder UI uses Godot's default Control theme. Do not generate image files.
- **No voice chat in Phase 1.** Text chat only via Supabase Realtime.
- **Offline mode is Phase 2.** Do not implement LocalWiFiNetworkAdapter until Phase 2 begins.
- **Power-ups are Phase 2.** The power-up shop UI may be stubbed in Phase 1 but no effect logic exists yet.
- **The TDD document is the single source of truth** for architectural decisions. If a decision seems unclear, read the TDD first.
- **Apple Developer Program ($99/year)** is required for iOS. It is the only unavoidable cost.

</important>
