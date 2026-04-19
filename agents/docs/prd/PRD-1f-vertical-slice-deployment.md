# PRD-1f — Vertical Slice & Deployment

| Field        | Value                                                           |
|--------------|-----------------------------------------------------------------|
| Version      | 1.0                                                             |
| Date         | April 2026                                                      |
| Status       | Approved                                                        |
| Sub-Phase    | 1f of 6                                                         |
| Duration     | Weeks 7–8                                                       |
| Depends On   | PRD-1e (Alliance, Objectives & Chat) — all prior sub-phases     |
| Unlocks      | Phase 2 (Power-ups, Art, Audio)                                 |
| Parent PRD   | [PRD-000 — Main](./PRD-000-main.md)                             |

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

All game systems exist as tested, isolated modules but they have never run together as a complete game loop. There is no integrated scene flow, no deployment pipeline, and no accessible build on any target platform. This sub-phase integrates everything, deploys to Web (Netlify), Android (APK sideload), and iOS (TestFlight), and defines the acceptance criteria that declare the vertical slice complete.

---

## 2. Solution

1. **Scene integration** — Wire all scenes (Lobby → Game → Vote → Score) into a coherent flow driven by `GameBus` signals. Each scene transition is triggered by a domain event, not by hardcoded scene paths.
2. **Placeholder UI** — Functional UI with no art assets: text labels, colour-coded panels, basic buttons. Sufficient to play a complete 3-round session.
3. **CI/CD export pipeline** — GitHub Actions workflow that builds Web, Android, and iOS exports on every push to `main`, triggered only if all GUT tests pass.
4. **Deployment targets:**
   - Web → Netlify (automatic on every passing CI run)
   - Android → APK artefact attached to GitHub release
   - iOS → Xcode archive uploaded to TestFlight via `altool`

---

## 3. Design Choices & Justification

### 3.1 Web Hosting: Netlify vs Alternatives

| Platform          | Free Tier                          | Auto-deploy from GitHub | Custom Domain | Verdict             |
|-------------------|------------------------------------|-------------------------|---------------|---------------------|
| **Netlify**       | 100 GB bandwidth, 300 build min/mo | ✅ Native                | ✅ Free        | ✅ Chosen           |
| GitHub Pages      | Unlimited (public repos)           | ✅ Via Actions           | ✅ Free        | ⚠️ No HTTPS redirect, limited headers control |
| Cloudflare Pages  | Unlimited requests, 500 builds/mo  | ✅ Native                | ✅ Free        | ✅ Strong alternative — choose if Netlify limits are hit |
| Vercel            | 100 GB, 6000 build min/mo          | ✅ Native                | ✅ Free        | ⚠️ Optimised for frontend frameworks, not static WASM |

**Decision: Netlify.** Godot's HTML5 export produces a static bundle (WASM + JS + HTML). Netlify deploys it with one `netlify.toml` config file, handles HTTPS automatically, and integrates with GitHub Actions via the `netlify-cli`. The `SharedArrayBuffer` headers required for Godot's WASM threads are configurable in `netlify.toml` — not all static hosts support custom headers without paid plans.

### 3.2 Android Distribution: APK vs Play Store

Play Store submission requires a developer account ($25 one-time fee) and a review process (1–7 days). For the vertical slice, a direct APK sideload is sufficient — testers enable "Install from unknown sources" on their Android device.

**Decision: APK sideload for vertical slice. Play Store in Phase 3.**

### 3.3 iOS Distribution: TestFlight vs Ad Hoc vs App Store

| Method        | Requires Apple Dev Account ($99/yr) | Max Testers | Verdict                  |
|---------------|--------------------------------------|-------------|--------------------------|
| **TestFlight**| ✅ Yes                               | 10,000      | ✅ Chosen — best UX for beta |
| Ad Hoc        | ✅ Yes                               | 100 devices | ⚠️ Requires UDID registration per device |
| App Store     | ✅ Yes + review                      | Unlimited   | ❌ Not for vertical slice  |

**Decision: TestFlight.** An Apple Developer account ($99/year) is the only unavoidable cost in the entire stack — iOS development requires it with no free alternative. It is categorised as a "development tool cost," not an infrastructure cost.

### 3.4 Godot CI Export: godot-ci vs Manual Setup

| Approach             | Verdict                                                        |
|----------------------|----------------------------------------------------------------|
| **barichello/godot-ci** | ✅ Chosen. Pre-built Docker image with Godot 4 + all export templates + Android SDK + rcedit for Windows. Maintained and widely used. |
| Manual Godot install  | ❌ ~15 minutes of setup steps per CI run, fragile to Godot version changes |
| Self-hosted runner    | ⚠️ Eliminates the Docker pull time but requires a persistent machine |

### 3.5 Scene Navigation Pattern: SceneManager vs Direct scene_change

| Approach                  | Verdict                                                     |
|---------------------------|-------------------------------------------------------------|
| **GameBus-driven transitions (chosen)** | ✅ Scenes subscribe to domain events and request transitions via a SceneManager. No scene knows the path of the next scene |
| Direct `get_tree().change_scene_to_file()` | ❌ Tight coupling. Every scene must know the file path of every other scene |
| State machine node         | ⚠️ Adds a node to the scene tree for what can be handled in GDScript |

---

## 4. User Stories

1. As a player, I want to open the game and choose to sign in with Google or continue as a guest, so that I can start playing immediately.
2. As an admin (guest or signed-in), I want to create a session, configure rounds, and receive a session code on the next screen, so that I can share it with friends.
3. As a player, I want to enter a session code and see the lobby screen with live player names appearing as they join, so that I can confirm my friends are in.
4. As an admin, I want a "Start Game" button in the lobby, so that I can begin the session when I'm ready.
5. As all players, I want to see my role card privately on screen after the round starts, so that I know my role for this round.
6. As all players, I want to see the Police player's name displayed prominently and publicly after role reveal, so that the discussion can begin.
7. As all players, I want a visible discussion timer counting down, so that I know how long I have to chat and deliberate.
8. As all players, I want the chat input to become disabled when the Police vote countdown starts, so that the vote phase is clearly distinct from discussion.
9. As Police, I want to see all player role cards with names during the vote countdown, so that I can make my accusation.
10. As all players, I want to see the Police's cursor/selection moving in real time during the countdown, so that the tension of the final seconds is visible.
11. As all players, I want to see a clear score breakdown after each round, showing base points, bonuses, steals, alliance reveals, and cumulative totals, so that I understand what happened.
12. As all players, I want the game to automatically advance to the next round after the score screen, so that the session flows without manual intervention.
13. As all players, I want to see the final standings after the last round, so that the winner is declared clearly.
14. As a signed-in player, I want my final score saved to Supabase at game end, so that my history is preserved.
15. As a developer, I want the game to be accessible via a Netlify URL, an Android APK, and TestFlight, so that the vertical slice can be played on all three target platforms.
16. As a developer, I want every push to main to trigger a full CI run (tests + export), so that the deployed build is always the latest passing version.

---

## 5. Implementation Decisions

### 5.1 Module: SceneManager Autoload

```gdscript
# res://src/autoloads/scene_manager.gd
class_name SceneManager
extends Node

## The single source of all scene transitions in the game.
## No scene file path is referenced anywhere except in this file.
## Transitions are triggered by calling SceneManager.go_to_*() methods.

const SCENE_PATHS: Dictionary = {
    "main_menu":   "res://src/scenes/main_menu/main_menu_scene.tscn",
    "lobby":       "res://src/scenes/lobby/lobby_scene.tscn",
    "role_reveal": "res://src/scenes/role_reveal/role_reveal_scene.tscn",
    "discussion":  "res://src/scenes/discussion/discussion_scene.tscn",
    "vote":        "res://src/scenes/vote/vote_scene.tscn",
    "score":       "res://src/scenes/score/score_scene.tscn",
    "game_end":    "res://src/scenes/game_end/game_end_scene.tscn",
}

func _ready() -> void:
    # Subscribe to GameBus domain events and trigger corresponding transitions
    GameBus.session_status_changed.connect(_on_session_status_changed)

func _on_session_status_changed(new_status: GameSession.Status) -> void:
    match new_status:
        GameSession.Status.LOBBY:
            go_to("lobby")
        GameSession.Status.ROLE_REVEAL:
            go_to("role_reveal")
        GameSession.Status.DISCUSSION:
            go_to("discussion")
        GameSession.Status.VOTE:
            go_to("vote")
        GameSession.Status.SCORING:
            pass  # Score scene opens after Edge Function responds via GameBus.scoring_resolved
        GameSession.Status.ROUND_END:
            go_to("score")
        GameSession.Status.GAME_END:
            go_to("game_end")

func go_to(scene_key: String) -> void:
    if not SCENE_PATHS.has(scene_key):
        push_error("SceneManager: unknown scene key '%s'" % scene_key)
        return
    get_tree().change_scene_to_file(SCENE_PATHS[scene_key])
```

### 5.2 Placeholder Scene Layout Contract

Each scene is a `Control` node with:
- A dark background panel (`ColorRect`)
- A title label (`Label`) showing the scene name
- Relevant UI elements built from Godot's built-in Control nodes (no custom art)
- All interactive elements use Godot's default theme (grey buttons, white text)

This is explicitly temporary. Phase 2 replaces all UI with the illustrated Indian royalty theme.

**Lobby Scene:**
```
[VBoxContainer]
  [Label]          "THE ROYAL RUSE"
  [Label]          "Session Code: {code}"
  [VBoxContainer]  Player list (one Label per player, updates via Realtime)
  [Label]          "Waiting for players... ({n}/10)"
  [ProgressBar]    Lobby timer (60 seconds)
  [Button]         "Start Game" (admin only, enabled when ≥4 players)
```

**Vote Scene:**
```
[VBoxContainer]
  [Label]          "VOTE — {countdown}s remaining"
  [HFlowContainer] Player cards (one per player):
    [PanelContainer]
      [VBoxContainer]
        [Label]    Role name
        [Label]    Player name
        [Button]   "Accuse" (Police only, tappable)
```

**Score Scene:**
```
[VBoxContainer]
  [Label]          "Round {n} Results"
  [VBoxContainer]  Per-player rows:
    [HBoxContainer]
      [Label]      Player name + role
      [Label]      Base: {n}
      [Label]      Steal: {n}
      [Label]      Bonus: {n}
      [Label]      Total: {n}
  [Label]          Alliance reveal (if any): "{p1} and {p2} were allies — bonus {'doubled!' | 'not triggered'}"
  [Label]          "Cumulative Scores:"
  [VBoxContainer]  Sorted cumulative scores
  [Button]         "Next Round" (admin only) — or auto-advance after 10s
```

### 5.3 GitHub Actions — Export Pipeline

```yaml
# .github/workflows/export.yml
name: Build & Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    name: Run GUT Tests
    runs-on: ubuntu-latest
    container:
      image: barichello/godot-ci:4.3
    steps:
      - uses: actions/checkout@v4
      - name: Run GUT
        run: |
          godot --headless --path . \
            -s addons/gut/gut_cmdln.gd \
            -gdir=res://test/ \
            -ginclude_subdirs \
            -gexit

  export-web:
    name: Export Web (HTML5)
    needs: test
    runs-on: ubuntu-latest
    container:
      image: barichello/godot-ci:4.3
    steps:
      - uses: actions/checkout@v4
      - name: Create export directory
        run: mkdir -p build/web
      - name: Export Web
        run: godot --headless --export-release "Web" build/web/index.html
      - name: Write Netlify headers
        run: |
          cat > build/web/_headers << 'EOF'
          /index.html
            Cross-Origin-Opener-Policy: same-origin
            Cross-Origin-Embedder-Policy: require-corp
          EOF
      - name: Deploy to Netlify
        uses: netlify/actions/cli@master
        with:
          args: deploy --dir=build/web --prod
        env:
          NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}

  export-android:
    name: Export Android (APK)
    needs: test
    runs-on: ubuntu-latest
    container:
      image: barichello/godot-ci:4.3
    steps:
      - uses: actions/checkout@v4
      - name: Setup Android keystore
        run: |
          echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d > /root/release.keystore
      - name: Create export directory
        run: mkdir -p build/android
      - name: Export Android APK
        run: godot --headless --export-release "Android" build/android/the_royal_ruse.apk
        env:
          GODOT_ANDROID_KEYSTORE_PATH: /root/release.keystore
          GODOT_ANDROID_KEYSTORE_USER: ${{ secrets.ANDROID_KEYSTORE_USER }}
          GODOT_ANDROID_KEYSTORE_PASSWORD: ${{ secrets.ANDROID_KEYSTORE_PASSWORD }}
      - name: Upload APK as artefact
        uses: actions/upload-artifact@v4
        with:
          name: android-apk
          path: build/android/the_royal_ruse.apk

  export-ios:
    name: Export iOS (IPA → TestFlight)
    needs: test
    runs-on: macos-latest    # Must be macOS — iOS export requires Xcode
    steps:
      - uses: actions/checkout@v4
      - name: Install Godot
        run: |
          wget -q https://github.com/godotengine/godot/releases/download/4.3-stable/Godot_v4.3-stable_macos.universal.zip
          unzip -q Godot_v4.3-stable_macos.universal.zip
          mv "Godot.app" /Applications/Godot.app
          ln -s /Applications/Godot.app/Contents/MacOS/Godot /usr/local/bin/godot
      - name: Export iOS Xcode project
        run: |
          mkdir -p build/ios
          godot --headless --export-release "iOS" build/ios/the_royal_ruse.xcodeproj
      - name: Archive and upload to TestFlight
        run: |
          xcodebuild -project build/ios/the_royal_ruse.xcodeproj \
            -scheme the_royal_ruse \
            -archivePath build/ios/the_royal_ruse.xcarchive \
            archive
          xcodebuild -exportArchive \
            -archivePath build/ios/the_royal_ruse.xcarchive \
            -exportOptionsPlist ExportOptions.plist \
            -exportPath build/ios/ipa
          xcrun altool --upload-app \
            -f build/ios/ipa/the_royal_ruse.ipa \
            -u "${{ secrets.APPLE_ID }}" \
            -p "${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}"
```

### 5.4 Netlify Configuration

```toml
# netlify.toml — placed in the repository root
[build]
  publish = "build/web"

[[headers]]
  for = "/*"
    [headers.values]
    Cross-Origin-Opener-Policy = "same-origin"
    Cross-Origin-Embedder-Policy = "require-corp"
```

**Why these headers?** Godot's HTML5 export uses `SharedArrayBuffer` for multi-threading. Browsers require `Cross-Origin-Opener-Policy: same-origin` and `Cross-Origin-Embedder-Policy: require-corp` to be set before allowing `SharedArrayBuffer`. Without them, Godot falls back to single-threaded mode (slower) or crashes on some browsers. These are security headers — not optional.

### 5.5 Export Preset Configuration (project.godot additions)

Godot export presets are configured in `export_presets.cfg`. Key settings:

```ini
[preset.0]  # Web
name="Web"
platform="Web"
runnable=true
export_filter="all_resources"
export_path="build/web/index.html"

[preset.1]  # Android
name="Android"
platform="Android"
runnable=true
export_filter="all_resources"
export_path="build/android/the_royal_ruse.apk"

[preset.2]  # iOS
name="iOS"
platform="iOS"
runnable=false
export_filter="all_resources"
export_path="build/ios/the_royal_ruse.xcodeproj"
```

### 5.6 GitHub Secrets Required

| Secret Name                     | Purpose                                          |
|---------------------------------|--------------------------------------------------|
| `NETLIFY_AUTH_TOKEN`            | Netlify CLI authentication                       |
| `NETLIFY_SITE_ID`               | Target Netlify site for deployment               |
| `ANDROID_KEYSTORE_BASE64`       | Android release keystore (base64-encoded)        |
| `ANDROID_KEYSTORE_USER`         | Keystore alias                                   |
| `ANDROID_KEYSTORE_PASSWORD`     | Keystore password                                |
| `APPLE_ID`                      | Apple ID for TestFlight upload                   |
| `APPLE_APP_SPECIFIC_PASSWORD`   | App-specific password (not your Apple ID password)|
| `SUPABASE_URL`                  | Supabase project URL (injected at build time)    |
| `SUPABASE_ANON_KEY`             | Supabase anon key                                |

---

## 6. Testing Decisions

### Vertical Slice Acceptance Test Checklist

The vertical slice is NOT declared complete until a human tester runs through the following checklist on all three platforms (Web, Android, iOS) without developer intervention:

```
SESSION CREATION
[ ] Open the game on Web (Netlify URL)
[ ] Sign in as guest — auto-generated name appears
[ ] Create session with 5 rounds and 2-minute discussion
[ ] Session code appears on screen

LOBBY
[ ] Open the game on a second device (Android)
[ ] Enter the session code — second player appears in first player's lobby
[ ] Open on a third device (iOS via TestFlight)
[ ] Enter the code — third player appears
[ ] Add a fourth device to reach minimum 4 players
[ ] Admin taps "Start Game" — all devices transition to role reveal

ROLE REVEAL
[ ] Each player sees only their own role card
[ ] Police identity is displayed to all players simultaneously
[ ] Role card shows role name and player name

DISCUSSION
[ ] 2-minute countdown is visible on all devices
[ ] Chat messages typed on one device appear on all others in real time
[ ] No perceptible delay (< 500ms) on local network

VOTE
[ ] Chat input is disabled when vote countdown starts
[ ] 10-second countdown is visible on all devices
[ ] Police sees interactive "Accuse" buttons
[ ] Non-Police players see read-only cards
[ ] Police taps a card — selection visible to all in real time
[ ] Police confirms — all devices transition to score screen

SCORING
[ ] Score screen shows base points, steal delta, objective bonus for each player
[ ] Alliance reveal shows correctly (or "No alliance this round")
[ ] Cumulative scores are correct after round 1

3-ROUND LOOP
[ ] Rounds 2 and 3 play through with roles reshuffled each time
[ ] Cumulative scores accumulate correctly across all 3 rounds

GAME END
[ ] Final standings screen displays after round 3
[ ] Signed-in players' scores saved to Supabase (verify in Supabase dashboard)
[ ] Guest players' scores are NOT persisted

CROSS-PLATFORM
[ ] Full session with mixed Web + Android + iOS players completes without error
```

### GUT Tests at This Stage

No new domain logic is introduced in PRD-1f. The GUT test suite from PRD-1a through PRD-1e must all pass in CI before any export runs. The CI pipeline enforces this via `needs: test`.

---

## 7. Out of Scope

- Power-ups (Phase 2)
- 2.5D art, animations, and audio (Phase 2)
- Play Store / App Store submissions (Phase 3)
- GDPR deletion UI (Phase 3)
- Offline Local WiFi mode (Phase 2)
- Performance profiling and optimisation (Phase 3)
- Analytics or crash reporting (Phase 3)

---

## 8. Further Notes

### SharedArrayBuffer and Godot Web Export

Godot's HTML5 export uses WebAssembly threads, which depend on `SharedArrayBuffer`. Chrome and Firefox disabled `SharedArrayBuffer` in 2018 due to Spectre/Meltdown vulnerabilities, then re-enabled it only for cross-origin-isolated contexts — which requires the two security headers listed in Section 5.4. Without them, the game either runs in degraded single-threaded mode or shows a browser error. Always set these headers on your web host.

### Android Debug vs Release Export

During development, use Godot's `--export-debug` flag to produce a debug APK — it includes Godot's built-in profiler and error overlay. For vertical slice distribution, switch to `--export-release` which strips debug symbols and produces a smaller, faster APK. Never distribute debug builds to testers.

### iOS macOS CI Requirement

The `export-ios` job uses `runs-on: macos-latest`. GitHub's macOS runners are slower and consume 10x more CI minutes than Linux runners (1 macOS minute = 10 Linux minutes against the free tier limit). For a public repository with unlimited Linux minutes, only the macOS job counts against limits. Budget approximately 15 minutes per iOS export run.

### Vertical Slice is a Commitment, Not a Preview

The vertical slice is not a rough prototype. It is a complete, correct, deployed implementation of the core game loop. Every system is real: server-authoritative scoring, real-time WebSocket communication, Google Auth, proper RLS on the database. The only thing missing is art and power-ups. This means Phase 2 is purely additive — no rewrites, no architecture changes, only new features built on an already-solid foundation.
