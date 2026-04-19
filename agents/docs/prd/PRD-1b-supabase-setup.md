# PRD-1b — Supabase Setup

| Field        | Value                                           |
|--------------|-------------------------------------------------|
| Version      | 1.0                                             |
| Date         | April 2026                                      |
| Status       | Approved                                        |
| Sub-Phase    | 1b of 6                                         |
| Duration     | Weeks 1–2 (parallel with PRD-1a)               |
| Depends On   | Nothing — runs in parallel with PRD-1a          |
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

The game requires a backend that handles four distinct concerns: user authentication (Google Sign-In and guest sessions), a relational database for sessions, rounds, roles, scores, alliances, and power-ups, real-time communication for game state broadcasting between players, and server-side scoring logic that the client cannot tamper with. This backend must cost zero at 500 monthly active users and require no server management by a solo developer.

---

## 2. Solution

Provision a new **Supabase project** and configure it to serve all four backend concerns:

1. **Supabase Auth** — Google OAuth + anonymous (guest) sign-in
2. **Supabase PostgreSQL** — all 8 core tables with Row-Level Security policies
3. **Supabase Realtime** — WebSocket channels for game state sync and chat
4. **Supabase Edge Functions** — server-side scoring authority (Deno/TypeScript, zero server management)

Implement all four **Supabase Adapter** classes in GDScript so they satisfy the Port interfaces defined in PRD-1a.

---

## 3. Design Choices & Justification

### 3.1 Supabase vs Backend Alternatives

| Criterion                    | **Supabase**        | **Firebase**          | **PocketBase**         | **Appwrite**           | **Self-hosted Nakama** |
|------------------------------|---------------------|-----------------------|------------------------|------------------------|------------------------|
| Free tier MAU limit          | ✅ 50,000 MAU       | ✅ Spark plan (limited)| ✅ Self-host only       | ✅ Self-host only       | ✅ Self-host only       |
| Database type                | ✅ PostgreSQL (relational) | ❌ Firestore (document) | ✅ SQLite             | ✅ MariaDB              | ✅ CockroachDB          |
| Real-time                    | ✅ Native WebSocket  | ✅ Native             | ⚠️ Server-sent events  | ✅ WebSocket            | ✅ Built for games      |
| Server-side logic            | ✅ Edge Functions (Deno) | ✅ Cloud Functions  | ⚠️ JavaScript hooks    | ✅ Functions            | ✅ Lua server scripts   |
| Google Auth                  | ✅ Built-in OAuth    | ✅ Built-in           | ⚠️ Manual OAuth setup  | ✅ Built-in             | ❌ Requires custom     |
| Zero server management       | ✅ Fully managed    | ✅ Fully managed      | ❌ Needs a VPS         | ❌ Needs a VPS          | ❌ Needs a VPS          |
| Open source                  | ✅ Apache 2.0       | ❌ Proprietary        | ✅ MIT                 | ✅ BSD                  | ✅ Apache 2.0           |
| SQL query expressiveness     | ✅ Full PostgreSQL   | ❌ Limited query API  | ⚠️ SQLite limits       | ⚠️ Subset of SQL        | ⚠️ Limited              |

**Decision: Supabase.** The relational schema (rounds reference sessions, round_roles reference rounds and players, alliances reference rounds) is a natural SQL model. Firestore's document model would require denormalisation that complicates scoring queries. PocketBase and Appwrite require a VPS — they're not truly zero-cost. Nakama is excellent for game backends but overkill at this scale and requires self-hosting. Supabase gives a managed PostgreSQL with real-time and auth under one free tier.

### 3.2 Edge Functions vs Client-Side Scoring

| Approach                | Security       | Complexity     | Verdict                    |
|-------------------------|----------------|----------------|----------------------------|
| **Edge Functions**      | ✅ Server-auth  | ⚠️ Deno/TS    | ✅ Chosen — required for anti-cheat |
| Client-side scoring     | ❌ Trivially cheateable | ✅ Simple | ❌ Unacceptable for competitive game |
| Supabase database triggers | ⚠️ PL/pgSQL | ⚠️ Complex to debug | ⚠️ Viable fallback if Edge Functions unavailable |

**Decision: Edge Functions.** The client sends intents; the server resolves scores. This is non-negotiable for game integrity. Edge Functions are Deno-based TypeScript deployed to Supabase's edge network — zero server management, cold-start under 50ms.

### 3.3 Row-Level Security vs Application-Layer Security

In Supabase, Row-Level Security (RLS) is enforced at the database level — not in application code. Even if a bug in a Godot adapter sends a malformed query, PostgreSQL rejects it before it executes.

**Why this matters:** Without RLS, a player could query `SELECT * FROM round_roles` and see everyone's roles before the round ends — trivially breaking the game. With RLS, the database itself enforces: "you can only read your own role, and only after the round is complete."

---

## 4. User Stories

1. As a player, I want to sign in with my Google account, so that my scores are saved across sessions.
2. As a player, I want to play as a guest without signing in, so that I can try the game with no friction.
3. As a signed-in player, I want my cumulative scores to be stored in the cloud, so that I can track my performance over time.
4. As a player, I want game state updates to arrive in real time without polling, so that vote countdowns and score reveals feel instant across all devices.
5. As a player, I want my role to be revealed only to me (not broadcast to all players), so that the game remains fair.
6. As a player, I want to know the police's identity is visible to everyone, while all other roles remain secret.
7. As a system, I want all score calculations to run server-side, so that no client can manipulate their own score.
8. As a system, I want alliance assignments to be stored server-side and never exposed to clients until scoring, so that blind alliances remain genuinely blind.
9. As a system, I want power-up affordability to be validated server-side before a purchase is recorded, so that players cannot spend points they don't have.
10. As a developer, I want Row-Level Security on every table, so that players cannot query data they are not authorised to see, regardless of bugs in the adapter code.
11. As a developer, I want a working GDScript Supabase adapter that the rest of the codebase can call through Port interfaces, so that no other file imports Supabase-specific code.
12. As a developer, I want an anonymous (guest) auth adapter that creates a local ephemeral identity, so that guest players can join sessions without a Supabase account.
13. As a system, I want guest session data to be automatically purged after 30 days, so that the database does not accumulate abandoned guest records.

---

## 5. Implementation Decisions

### 5.1 Database Schema

All tables use `uuid` primary keys generated server-side. Timestamps use `timestamptz` (timezone-aware). All foreign keys have `ON DELETE CASCADE` where appropriate.

```sql
-- Table: users
-- Stores persistent player identity. Guests have null google_id.
CREATE TABLE users (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  google_id     TEXT UNIQUE,                    -- null for guests
  display_name  TEXT NOT NULL,
  is_guest      BOOLEAN NOT NULL DEFAULT false,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  last_seen_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Table: sessions
CREATE TABLE sessions (
  id                       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_code             TEXT UNIQUE NOT NULL,          -- 6-char human-readable
  admin_player_id          UUID REFERENCES users(id),
  status                   TEXT NOT NULL DEFAULT 'lobby', -- lobby|round_start|role_reveal|discussion|vote|scoring|round_end|game_end
  max_rounds               INT NOT NULL DEFAULT 5 CHECK (max_rounds <= 20),
  current_round            INT NOT NULL DEFAULT 0,
  discussion_duration_secs INT NOT NULL DEFAULT 120,
  created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
  expires_at               TIMESTAMPTZ NOT NULL DEFAULT now() + INTERVAL '2 hours'
);

-- Table: session_players
CREATE TABLE session_players (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id  UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  user_id     UUID NOT NULL REFERENCES users(id),
  display_name TEXT NOT NULL,
  is_guest    BOOLEAN NOT NULL DEFAULT false,
  joined_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
  is_active   BOOLEAN NOT NULL DEFAULT true
);

-- Table: rounds
CREATE TABLE rounds (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id          UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  round_number        INT NOT NULL,
  police_player_id    UUID REFERENCES users(id),
  thief_player_id     UUID REFERENCES users(id),
  accomplice_player_id UUID REFERENCES users(id),       -- null until Thief nominates
  accused_player_id   UUID REFERENCES users(id),        -- null until Police votes
  outcome             TEXT,                              -- 'caught'|'escaped'|'timeout'
  created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at        TIMESTAMPTZ
);

-- Table: round_roles
CREATE TABLE round_roles (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  round_id            UUID NOT NULL REFERENCES rounds(id) ON DELETE CASCADE,
  player_id           UUID NOT NULL REFERENCES users(id),
  role_id             TEXT NOT NULL,                     -- 'king'|'queen'|etc.
  base_points         INT NOT NULL,
  objective_completed BOOLEAN NOT NULL DEFAULT false,
  objective_bonus     INT NOT NULL DEFAULT 0,
  final_points        INT NOT NULL DEFAULT 0             -- set by Edge Function at scoring
);

-- Table: alliances
CREATE TABLE alliances (
  id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  round_id                UUID NOT NULL REFERENCES rounds(id) ON DELETE CASCADE,
  player1_id              UUID NOT NULL REFERENCES users(id),
  player2_id              UUID NOT NULL REFERENCES users(id),
  alliance_type           TEXT NOT NULL,                 -- e.g. 'thief_jester'
  complementary_achieved  BOOLEAN NOT NULL DEFAULT false,
  multiplier_applied      BOOLEAN NOT NULL DEFAULT false
);

-- Table: powerup_purchases
CREATE TABLE powerup_purchases (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  round_id     UUID NOT NULL REFERENCES rounds(id) ON DELETE CASCADE,
  player_id    UUID NOT NULL REFERENCES users(id),
  powerup_key  TEXT NOT NULL,
  tier         INT NOT NULL CHECK (tier IN (1, 2, 3)),
  points_spent INT NOT NULL,
  activated    BOOLEAN NOT NULL DEFAULT false,
  purchased_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Table: scores
CREATE TABLE scores (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  session_id        UUID NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  player_id         UUID NOT NULL REFERENCES users(id),
  cumulative_points INT NOT NULL DEFAULT 0,
  final_rank        INT,                                 -- null until game_end
  recorded_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5.2 Row-Level Security Policies

```sql
-- Enable RLS on all tables
ALTER TABLE users             ENABLE ROW LEVEL SECURITY;
ALTER TABLE sessions          ENABLE ROW LEVEL SECURITY;
ALTER TABLE session_players   ENABLE ROW LEVEL SECURITY;
ALTER TABLE rounds            ENABLE ROW LEVEL SECURITY;
ALTER TABLE round_roles       ENABLE ROW LEVEL SECURITY;
ALTER TABLE alliances         ENABLE ROW LEVEL SECURITY;
ALTER TABLE powerup_purchases ENABLE ROW LEVEL SECURITY;
ALTER TABLE scores            ENABLE ROW LEVEL SECURITY;

-- users: each player can only read/update their own row
CREATE POLICY "users_own_row" ON users
  FOR ALL USING (auth.uid() = id);

-- sessions: readable by any participant of that session
CREATE POLICY "sessions_participant_read" ON sessions
  FOR SELECT USING (
    id IN (SELECT session_id FROM session_players WHERE user_id = auth.uid())
  );

-- session_players: readable by any participant
CREATE POLICY "session_players_participant_read" ON session_players
  FOR SELECT USING (
    session_id IN (SELECT session_id FROM session_players WHERE user_id = auth.uid())
  );

-- round_roles: player can only read their OWN role, and only when round is complete
-- Police role is broadcast separately via private channel — not from this table directly
CREATE POLICY "round_roles_own_role_only" ON round_roles
  FOR SELECT USING (
    player_id = auth.uid() OR
    round_id IN (SELECT id FROM rounds WHERE completed_at IS NOT NULL)
  );

-- alliances: NEVER readable by clients before round completion
CREATE POLICY "alliances_post_round_only" ON alliances
  FOR SELECT USING (
    round_id IN (SELECT id FROM rounds WHERE completed_at IS NOT NULL)
  );

-- scores: all participants can read all scores (leaderboard)
CREATE POLICY "scores_session_participants_read" ON scores
  FOR SELECT USING (
    session_id IN (SELECT session_id FROM session_players WHERE user_id = auth.uid())
  );

-- powerup_purchases: players can only see their own purchases
CREATE POLICY "powerup_own_only" ON powerup_purchases
  FOR SELECT USING (player_id = auth.uid());
```

### 5.3 Edge Function Scaffold

Edge Functions are TypeScript files deployed to Supabase's Deno edge runtime. They run server-side, have admin access to the database (bypassing RLS), and are invoked via HTTP from the Godot client.

```typescript
// supabase/functions/resolve-scoring/index.ts
// Called by the client after Police submits their vote.
// Resolves the full scoring pipeline for one round.

import { serve } from "https://deno.land/std@0.177.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"

serve(async (req: Request) => {
  const { round_id, session_id } = await req.json()

  // Validate required parameters
  if (!round_id || !session_id) {
    return new Response(JSON.stringify({ error: "Missing round_id or session_id" }), { status: 400 })
  }

  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!   // Admin key — bypasses RLS
  )

  // Full scoring logic implemented in PRD-1d
  // At this stage: scaffold only — returns a stub response

  return new Response(JSON.stringify({ status: "scoring_complete", scores: [] }), {
    headers: { "Content-Type": "application/json" },
    status: 200,
  })
})
```

```typescript
// supabase/functions/assign-roles/index.ts
// Called by the admin client when starting a round.
// Assigns roles, creates alliances, generates powerup pools server-side.

import { serve } from "https://deno.land/std@0.177.0/http/server.ts"

serve(async (req: Request) => {
  // Full implementation in PRD-1c
  return new Response(JSON.stringify({ status: "roles_assigned", assignments: [] }), { status: 200 })
})
```

### 5.4 Module: SupabaseNetworkAdapter

**What it does:** Implements `INetworkPort` using Supabase Realtime channels. When the game broadcasts `vote_cast`, this adapter publishes it to a Supabase channel. When a Supabase channel receives an event, it emits `game_event_received` on the port interface.

```gdscript
# res://src/adapters/supabase/supabase_network_adapter.gd
class_name SupabaseNetworkAdapter
extends INetworkPort

const SUPABASE_URL: String = "https://YOUR_PROJECT.supabase.co"
const SUPABASE_ANON_KEY: String = "YOUR_ANON_KEY"

var _http_client: HTTPClient
var _current_channel: String
var _websocket: WebSocketPeer

## Connects to the Supabase Realtime channel for the given session.
func connect_to_session(session_code: String) -> Result:
    _current_channel = "session:%s" % session_code
    # WebSocket connection to Supabase Realtime endpoint
    # Full implementation: establish WSS handshake, subscribe to channel
    return Result.ok()

func disconnect_from_session() -> void:
    if _websocket:
        _websocket.close()

## Broadcasts to the session channel via Supabase Realtime.
func broadcast_event(event_name: String, payload: Dictionary) -> Result:
    if not is_connected():
        return Result.fail("Not connected to a session channel")
    var message := {
        "type": "broadcast",
        "event": event_name,
        "payload": payload
    }
    var json: String = JSON.stringify(message)
    var err: Error = _websocket.send_text(json)
    return Result.ok() if err == OK else Result.fail("WebSocket send failed: %d" % err)

func send_private_event(player_id: String, event_name: String, payload: Dictionary) -> Result:
    # Sends to player-specific channel: "player:{player_id}"
    return Result.ok()

func is_connected() -> bool:
    return _websocket != null and _websocket.get_ready_state() == WebSocketPeer.STATE_OPEN
```

### 5.5 Module: SupabaseAuthAdapter

```gdscript
# res://src/adapters/supabase/supabase_auth_adapter.gd
class_name SupabaseAuthAdapter
extends IAuthPort

const SUPABASE_URL: String = "https://YOUR_PROJECT.supabase.co"
const SUPABASE_ANON_KEY: String = "YOUR_ANON_KEY"
const GOOGLE_CLIENT_ID: String = "YOUR_GOOGLE_CLIENT_ID"

var _current_player_id: String
var _is_authenticated: bool

## Opens a system browser to Google's OAuth URL, waits for the redirect,
## and exchanges the token with Supabase Auth.
func sign_in_with_google() -> Result:
    # Implementation uses Godot's OS.shell_open() to open the OAuth URL
    # and a local HTTP listener to catch the redirect callback.
    # Full implementation requires the OAuth PKCE flow.
    return Result.fail("not yet implemented — PRD-1b")

func sign_in_as_guest(display_name: String) -> Result:
    # Calls Supabase Auth anonymous sign-in endpoint
    # POST /auth/v1/signup with no email — creates an anonymous user
    var http := HTTPRequest.new()
    # ... full implementation in development
    _current_player_id = "guest_%s" % Time.get_unix_time_from_system()
    _is_authenticated = false
    signed_in.emit(_current_player_id, display_name, true)
    return Result.ok({"player_id": _current_player_id, "display_name": display_name})

func sign_out() -> Result:
    _current_player_id = ""
    _is_authenticated = false
    signed_out.emit()
    return Result.ok()

func get_current_player_id() -> String:
    return _current_player_id

func is_signed_in() -> bool:
    return _is_authenticated
```

### 5.6 Module: SupabaseStorageAdapter

```gdscript
# res://src/adapters/supabase/supabase_storage_adapter.gd
class_name SupabaseStorageAdapter
extends IStoragePort

const SUPABASE_URL: String = "https://YOUR_PROJECT.supabase.co"
const SUPABASE_ANON_KEY: String = "YOUR_ANON_KEY"

## Inserts or updates a score row in the scores table via Supabase REST API.
func save_session_score(session_id: String, player_id: String, cumulative_points: int, final_rank: int) -> Result:
    var body := {
        "session_id": session_id,
        "player_id": player_id,
        "cumulative_points": cumulative_points,
        "final_rank": final_rank
    }
    # POST to /rest/v1/scores with Authorization: Bearer {jwt}
    return Result.ok()

func fetch_session_scores(session_id: String) -> Result:
    # GET /rest/v1/scores?session_id=eq.{session_id}&order=cumulative_points.desc
    return Result.ok([])

func stage_offline_round(round_data: Dictionary) -> Result:
    # Write to Godot ConfigFile at user://offline_rounds.cfg
    var config := ConfigFile.new()
    var key: String = "round_%d" % Time.get_unix_time_from_system()
    config.set_value("staged", key, round_data)
    var err: Error = config.save("user://offline_rounds.cfg")
    return Result.ok() if err == OK else Result.fail("ConfigFile save failed: %d" % err)

func sync_offline_data() -> Result:
    # Read staged data and POST each to Supabase
    return Result.ok(0)
```

### 5.7 Module: SupabaseChatAdapter

```gdscript
# res://src/adapters/supabase/supabase_chat_adapter.gd
class_name SupabaseChatAdapter
extends IChatPort

var _locked: bool = false
var _session_id: String
var _chat_websocket: WebSocketPeer

func join_chat_channel(session_id: String) -> Result:
    _session_id = session_id
    # Connect to Supabase Realtime channel "chat:{session_id}"
    return Result.ok()

func send_message(session_id: String, message: String) -> Result:
    if _locked:
        return Result.fail("Chat is locked during vote phase")
    if message.length() > 200:
        return Result.fail("Message exceeds 200 character limit")
    # Broadcast to "chat:{session_id}" Realtime channel
    return Result.ok()

func lock_chat() -> void:
    _locked = true

func unlock_chat() -> void:
    _locked = false

func is_locked() -> bool:
    return _locked
```

---

## 6. Testing Decisions

### Adapters Are NOT TDD Targets

Supabase adapters cannot be tested against a real Supabase project in a CI environment without credentials. Instead:

- Each adapter is manually verified against a local Supabase project during development
- All use cases that depend on adapters are tested using **mock port implementations** (fake objects that implement the port interface with controlled, predictable behaviour)

### Mock Port Pattern

```gdscript
# test/unit/mocks/mock_auth_port.gd
class_name MockAuthPort
extends IAuthPort

## A controllable fake implementation of IAuthPort for use in domain tests.
## Call configure_signed_in() before any test that needs an authenticated state.

var _player_id: String = ""
var _is_authenticated: bool = false
var _sign_in_should_fail: bool = false

func configure_signed_in(player_id: String) -> void:
    _player_id = player_id
    _is_authenticated = true

func configure_sign_in_failure() -> void:
    _sign_in_should_fail = true

func sign_in_as_guest(display_name: String) -> Result:
    if _sign_in_should_fail:
        return Result.fail("Simulated auth failure")
    _player_id = "mock_guest_001"
    _is_authenticated = false
    return Result.ok({"player_id": _player_id, "display_name": display_name})

func get_current_player_id() -> String:
    return _player_id

func is_signed_in() -> bool:
    return _is_authenticated
```

### Modules Under Test in PRD-1b

| Module                | Test Type    | Notes                                              |
|-----------------------|--------------|----------------------------------------------------|
| `MockAuthPort`        | Unit         | Verify mock behaves correctly for use in other tests |
| `MockNetworkPort`     | Unit         | Verify signal emission and event queuing           |
| `MockStoragePort`     | Unit         | Verify offline staging and retrieval               |
| `SupabaseChatAdapter` | Manual       | Verified against local Supabase project only       |
| RLS Policies          | Manual/SQL   | Tested via Supabase SQL editor — not GUT           |
| Edge Function scaffold| Manual       | Invoked via curl during development                |

---

## 7. Out of Scope

- Full Edge Function scoring logic (PRD-1d)
- Full Edge Function role assignment logic (PRD-1c)
- Power-up validation in Edge Functions (Phase 2)
- Supabase Storage (no file uploads in Phase 1)
- Email-based auth (Google OAuth only for Phase 1)
- GDPR deletion endpoint (Phase 3)

---

## 8. Further Notes

### Supabase Free Tier Limits

| Resource            | Free Limit       | Our Usage at 500 MAU     |
|---------------------|------------------|--------------------------|
| Database            | 500 MB           | ~50 MB estimated          |
| Bandwidth           | 2 GB/month       | ~200 MB estimated         |
| Edge Function calls | 500,000/month    | ~200,000 estimated        |
| Realtime messages   | 2M/month         | ~500,000 estimated        |
| Auth MAU            | 50,000           | 500 ✅                    |

**Scale trigger:** At ~2,500 MAU, Supabase Realtime message volume approaches the free tier limit. Upgrade to Supabase Pro ($25/month) at that point.

### Environment Variables

Never hardcode Supabase credentials in GDScript files. Store them in a `res://.env` file that is listed in `.gitignore`. A `Config` autoload reads them at startup:

```gdscript
# res://src/autoloads/config.gd
class_name Config
extends Node

var SUPABASE_URL: String
var SUPABASE_ANON_KEY: String

func _ready() -> void:
    var config := ConfigFile.new()
    config.load("res://.env")
    SUPABASE_URL = config.get_value("supabase", "url", "")
    SUPABASE_ANON_KEY = config.get_value("supabase", "anon_key", "")
```

In GitHub Actions, these are set as **Repository Secrets** and injected into the build environment — they are never committed to source control.
