# PRD-000 — The Royal Ruse: Phase 1 Master PRD

| Field       | Value                                      |
|-------------|--------------------------------------------|
| Version     | 1.0                                        |
| Date        | April 2026                                 |
| Status      | Approved                                   |
| Author      | Solo Developer                             |
| Engine      | Godot 4.3 (GDScript)                       |
| Backend     | Supabase (Free Tier)                       |
| Platforms   | iOS · Android · Web (HTML5/WASM)           |

---

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Solution Overview](#2-solution-overview)
3. [Cross-Cutting Design Principles](#3-cross-cutting-design-principles)
4. [Phase 1 Sub-Phase Summary](#4-phase-1-sub-phase-summary)
5. [Success Metrics — Vertical Slice](#5-success-metrics--vertical-slice)
6. [Dependency Map](#6-dependency-map)
7. [Out of Scope for Phase 1](#7-out-of-scope-for-phase-1)
8. [Sub-Phase PRD Index](#8-sub-phase-prd-index)

---

## 1. Problem Statement

No implementation of The Royal Ruse exists. There is no Godot project, no backend schema, no game loop, and no deployable artefact. The team (solo developer) needs a clear, sequenced implementation plan that produces a playable vertical slice within 8 weeks — a 3-round online multiplayer loop demonstrating the complete game state machine — while establishing an architectural foundation that will carry the product through Phase 2 (power-ups, art, audio) and Phase 3 (production launch) without requiring rewrites.

The constraints are non-negotiable: zero software budget, zero infrastructure cost at 500 MAU scale, and a solo developer with zero prior Godot experience who needs to learn while building.

---

## 2. Solution Overview

Phase 1 delivers the vertical slice in six sequenced sub-phases. Each sub-phase is independently releasable as a testable milestone. No sub-phase may begin until its predecessor's acceptance criteria are met.

The foundation is a **Hexagonal Architecture** (Ports and Adapters) implemented in **Godot 4 (GDScript)**, backed by **Supabase** for auth, database, real-time communication, and server-side scoring logic. All game scoring and state transitions are server-authoritative — the client is treated as untrusted and sends only intents.

Test-Driven Development (TDD) is applied to all pure domain logic. GUT (Godot Unit Test framework) is the test runner. UI scenes and network adapters are not TDD targets; they are covered by integration smoke tests.

---

## 3. Cross-Cutting Design Principles

These principles apply to every line of code written across all sub-phases. They are not optional.

### 3.1 Hexagonal Architecture

The codebase is split into three explicit layers that may never leak across boundaries:

```
┌───────────────────────────────────────────────────┐
│  Scene Layer (Godot scenes / UI nodes)            │  ← knows Use Cases only
├───────────────────────────────────────────────────┤
│  Domain Layer (Entities, Use Cases, Value Objects)│  ← knows nothing external
├───────────────────────────────────────────────────┤
│  Adapter Layer (Supabase, WiFi, LocalStorage)     │  ← knows Port interfaces only
└───────────────────────────────────────────────────┘
```

**In plain English:** The game logic (how scoring works, how roles are assigned) has absolutely no idea whether it's talking to Supabase, a local WiFi server, or a mock in a test. It only calls a named interface (a Port). The concrete implementation that actually speaks to Supabase lives in a separate file (an Adapter) and is injected at startup via a ServiceLocator autoload.

This means you can swap Supabase for any other backend later by writing one new Adapter file — zero changes to game logic.

### 3.2 Server Authority

The client never calculates its own score. Every scoring event is resolved by a Supabase Edge Function. The client sends an intent (`vote_cast`, `powerup_purchased`, `accomplice_nominated`) and receives a resolved state update. This prevents cheating and keeps the client stateless between rounds.

### 3.3 TDD on Domain Logic

Every domain use case and entity follows the Red-Green-Refactor cycle:
1. Write a failing GUT test that describes the expected behaviour
2. Write the minimum GDScript to make it pass
3. Refactor for clarity without breaking the test

Adapters are never tested against real external services in the test suite. They are mocked via fake Port implementations.

### 3.4 Zero Cost Discipline

Every tool and hosting choice must satisfy: free at 500 MAU, open-source licence, no credit card required to start. Where a paid tier becomes necessary at scale (e.g. Supabase Pro), this is documented as a known future cost with the exact trigger threshold.

### 3.5 GDScript Coding Standards

All GDScript files follow these conventions (enforced by code review):

```
- snake_case for variables, functions, and file names
- PascalCase for class names only
- ## double-hash doc comments on every public function
- Type annotations on every variable and every function parameter
- No var without a type: var x: int = 0 not var x = 0
- One class per file
- Maximum 200 lines per file — split if exceeded
- Signals declared at the top of the file, before variables
- Constants in SCREAMING_SNAKE_CASE
```

---

## 4. Phase 1 Sub-Phase Summary

| Sub-Phase | Name                              | Duration  | Key Deliverable                                         | PRD Link                                     |
|-----------|-----------------------------------|-----------|---------------------------------------------------------|----------------------------------------------|
| **1a**    | Godot Scaffold & CI               | Week 1–2  | Working Godot project with hexagonal skeleton, GUT passing, GitHub Actions CI green | [PRD-1a](./PRD-1a-godot-scaffold.md)         |
| **1b**    | Supabase Setup                    | Week 1–2  | Schema migrated, Google Auth live, RLS policies active, Edge Function scaffold deployed | [PRD-1b](./PRD-1b-supabase-setup.md)         |
| **1c**    | Session, Lobby & Role Assignment  | Week 3–4  | Players can create/join sessions, lobby timer works, roles assigned correctly by player count | [PRD-1c](./PRD-1c-session-lobby-roles.md)    |
| **1d**    | Vote Mechanic & Scoring Engine    | Week 5–6  | Police countdown, Thief/Accomplice mechanic, false accusation fine, scores resolved server-side | [PRD-1d](./PRD-1d-vote-scoring.md)           |
| **1e**    | Alliance Engine, Objectives & Chat| Week 6–7  | Blind alliances assigned and evaluated, round objectives tracked, in-session text chat functional | [PRD-1e](./PRD-1e-alliance-objectives-chat.md)|
| **1f**    | Vertical Slice & Deployment       | Week 7–8  | Full 3-round loop deployed to Netlify (web), Android APK, iOS TestFlight               | [PRD-1f](./PRD-1f-vertical-slice-deployment.md)|

> Sub-phases 1a and 1b run in parallel. All others are sequential.

---

## 5. Success Metrics — Vertical Slice

The vertical slice is complete when **all** of the following are simultaneously true:

- [ ] 4–10 players can join an online session via a 6-character session code
- [ ] All 10 roles assign correctly and progressively by player count
- [ ] Role reveal → 2-minute discussion (text chat) → 10-second Police vote resolves in sequence without manual intervention
- [ ] Thief steal mechanic, Accomplice bonus/penalty, and false accusation fine all score correctly (server-side)
- [ ] Blind alliances are assigned server-side, evaluated at scoring, and revealed on the score screen
- [ ] Round objectives are tracked and bonuses applied correctly for all 10 roles
- [ ] 3 rounds play sequentially with cumulative scores displayed after each round
- [ ] Signed-in players' final scores are persisted to Supabase
- [ ] The build is accessible on Web (Netlify), Android (APK sideload), and iOS (TestFlight)
- [ ] All GUT domain tests pass in GitHub Actions CI

---

## 6. Dependency Map

```
PRD-1a (Scaffold) ──┐
                    ├──→ PRD-1c (Session/Lobby/Roles) ──→ PRD-1d (Vote/Scoring) ──→ PRD-1e (Alliance/Chat) ──→ PRD-1f (Deploy)
PRD-1b (Supabase) ──┘
```

- PRD-1c cannot start until PRD-1a and PRD-1b are both complete
- PRD-1d requires PRD-1c's session and role assignment to be functional
- PRD-1e requires PRD-1d's scoring pipeline to be in place
- PRD-1f integrates all of the above

---

## 7. Out of Scope for Phase 1

The following are explicitly deferred to Phase 2 or Phase 3:

- Power-up system (all 90 power-ups, shop UI, effect resolution)
- 2.5D art and skeletal animations
- Audio (ambient, SFX, adaptive audio)
- Offline mode (Local WiFi WebSocket)
- Voice chat
- Wanted Brand cross-round status persistence
- App Store / Play Store submissions (TestFlight and APK sideload only)
- GDPR deletion flow UI
- Admin kick functionality UI (backend supports it; UI deferred)
- Model B objectives (chat-axis objectives)

---

## 8. Sub-Phase PRD Index

| Document                                                    | Description                              |
|-------------------------------------------------------------|------------------------------------------|
| [PRD-1a — Godot Scaffold & CI](./PRD-1a-godot-scaffold.md) | Project structure, architecture skeleton, GUT, GitHub Actions |
| [PRD-1b — Supabase Setup](./PRD-1b-supabase-setup.md)      | Schema, Auth, RLS, Edge Functions        |
| [PRD-1c — Session, Lobby & Roles](./PRD-1c-session-lobby-roles.md) | Session flow, lobby timer, role assignment engine |
| [PRD-1d — Vote & Scoring](./PRD-1d-vote-scoring.md)        | Police vote, Thief mechanic, scoring resolution |
| [PRD-1e — Alliance, Objectives & Chat](./PRD-1e-alliance-objectives-chat.md) | Blind alliances, objective tracking, text chat |
| [PRD-1f — Vertical Slice & Deployment](./PRD-1f-vertical-slice-deployment.md) | Integration, CI/CD, web/mobile deployment |
