---
name: update-export-config
description: Update Godot export presets and the GitHub Actions export pipeline. Use when adding a new target platform, changing export settings, updating Godot version, fixing export template issues, or modifying the CI/CD deployment targets. Triggers include "new platform", "update build", "export settings", "change export target", "update Godot version in CI".
---

# Skill: Update Export Configuration

Godot export configuration spans two files: `export_presets.cfg` (Godot's export settings) and `.github/workflows/export.yml` (CI/CD pipeline). Both must be updated together — an export preset with no CI job, or a CI job pointing to a non-existent preset, will silently fail.

## Before You Start

1. Read `docs/prd/PRD-1f-vertical-slice-deployment.md` for the full deployment rationale.
2. Identify what is changing:
   - New platform (Web, Android, iOS, macOS, Windows, Linux)
   - Changed export path or preset name
   - Updated Godot version in CI
   - New deployment target (Netlify, TestFlight, etc.)
3. For iOS changes: requires macOS runner in CI. macOS minutes cost 10× more than Linux. Plan accordingly.
4. For Android changes: requires keystore configuration in GitHub Secrets.

## Files to Touch

1. `export_presets.cfg` — Godot export preset definitions
2. `.github/workflows/export.yml` — CI/CD export jobs
3. `netlify.toml` — if Web export headers change
4. `ExportOptions.plist` — if iOS signing configuration changes

---

## export_presets.cfg Reference

This file is managed by Godot's editor (Project → Export) but can also be edited manually. Each preset section follows this structure:

```ini
[preset.N]
name="Platform Name"           # Must match the name used in CI: --export-release "Platform Name"
platform="Web"                 # Godot platform string: Web|Android|iOS|macOS|Windows Desktop|Linux
runnable=true
export_filter="all_resources"
export_path="build/platform/output_filename.ext"
script_export_mode=1

[preset.N.options]
# Platform-specific options set in Godot editor — do not manually edit these
# unless you know exactly what you're changing
```

### Web Preset (Reference)

```ini
[preset.0]
name="Web"
platform="Web"
runnable=true
export_filter="all_resources"
export_path="build/web/index.html"
script_export_mode=1

[preset.0.options]
custom_template/debug=""
custom_template/release=""
variant/extensions_support=false
vram_texture_compression/for_desktop=true
vram_texture_compression/for_mobile=false
html/export_icon=true
html/custom_html_shell=""
html/head_include=""
html/canvas_resize_policy=2
html/focus_canvas_on_start=true
html/experimental_virtual_keyboard=false
progressive_web_app/enabled=false
```

### Android Preset (Reference)

```ini
[preset.1]
name="Android"
platform="Android"
runnable=true
export_filter="all_resources"
export_path="build/android/the_royal_ruse.apk"
script_export_mode=1

[preset.1.options]
custom_template/debug=""
custom_template/release=""
gradle_build/use_gradle_build=false
gradle_build/gradle_build_directory=""
gradle_build/export_format=0
architectures/armeabi-v7a=false
architectures/arm64-v8a=true
architectures/x86=false
architectures/x86_64=false
keystore/debug="res://debug.keystore"
keystore/debug_user="androiddebugkey"
keystore/debug_password="android"
keystore/release=""
keystore/release_user=""
keystore/release_password=""
```

### iOS Preset (Reference)

```ini
[preset.2]
name="iOS"
platform="iOS"
runnable=false
export_filter="all_resources"
export_path="build/ios/the_royal_ruse.xcodeproj"
script_export_mode=1

[preset.2.options]
custom_template/debug=""
custom_template/release=""
application/bundle_identifier="com.yourname.royalruse"
application/short_version="1.0"
application/version="1"
application/signature=""
application/export_method_debug=0
application/export_method_release=1
```

---

## GitHub Actions Export Workflow Reference

```yaml
# .github/workflows/export.yml

name: Build & Deploy

on:
  push:
    branches: [main]

jobs:
  # ── Step 1: Tests must pass before any export ──────────────────────────────
  run-gut-tests:
    name: Run GUT Test Suite
    runs-on: ubuntu-latest
    container:
      image: barichello/godot-ci:4.3    # Update version here when upgrading Godot
    steps:
      - uses: actions/checkout@v4
      - name: Run GUT headless
        run: |
          godot --headless --path . \
            -s addons/gut/gut_cmdln.gd \
            -gdir=res://test/ \
            -ginclude_subdirs \
            -gexit

  # ── Web Export + Netlify Deploy ────────────────────────────────────────────
  export-web:
    name: Export Web (HTML5)
    needs: run-gut-tests         # Never deploy without passing tests
    runs-on: ubuntu-latest
    container:
      image: barichello/godot-ci:4.3
    steps:
      - uses: actions/checkout@v4
      - name: Create export directory
        run: mkdir -p build/web
      - name: Export Web
        run: godot --headless --export-release "Web" build/web/index.html
      - name: Write Netlify security headers
        # SharedArrayBuffer headers — required for Godot WASM threading
        run: |
          cat > build/web/_headers << 'EOF'
          /*
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

  # ── Android Export ─────────────────────────────────────────────────────────
  export-android:
    name: Export Android (APK)
    needs: run-gut-tests
    runs-on: ubuntu-latest
    container:
      image: barichello/godot-ci:4.3
    steps:
      - uses: actions/checkout@v4
      - name: Setup Android keystore
        run: echo "${{ secrets.ANDROID_KEYSTORE_BASE64 }}" | base64 -d > /root/release.keystore
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

  # ── iOS Export → TestFlight ────────────────────────────────────────────────
  export-ios:
    name: Export iOS (IPA → TestFlight)
    needs: run-gut-tests
    runs-on: macos-latest          # macOS REQUIRED for iOS — costs 10x Linux minutes
    steps:
      - uses: actions/checkout@v4
      - name: Install Godot on macOS
        run: |
          wget -q https://github.com/godotengine/godot/releases/download/4.3-stable/Godot_v4.3-stable_macos.universal.zip
          unzip -q Godot_v4.3-stable_macos.universal.zip
          mv "Godot.app" /Applications/Godot.app
          sudo ln -s /Applications/Godot.app/Contents/MacOS/Godot /usr/local/bin/godot
      - name: Export iOS Xcode project
        run: |
          mkdir -p build/ios
          godot --headless --export-release "iOS" build/ios/the_royal_ruse.xcodeproj
      - name: Archive and upload to TestFlight
        run: |
          xcodebuild -project build/ios/the_royal_ruse.xcodeproj \
            -scheme the_royal_ruse \
            -archivePath build/ios/the_royal_ruse.xcarchive \
            -destination "generic/platform=iOS" \
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

## When Updating the Godot Version in CI

Update the Docker image tag in every job that uses it:

```yaml
# Change this in ALL jobs that use the container:
image: barichello/godot-ci:4.3    # ← Update this version string

# Also update the manual Godot download URL in the iOS job:
wget -q https://github.com/godotengine/godot/releases/download/4.3-stable/Godot_v4.3-stable_macos.universal.zip
```

Check available tags at: https://hub.docker.com/r/barichello/godot-ci/tags

## netlify.toml Reference

```toml
# netlify.toml — repo root
[build]
  publish = "build/web"

# SharedArrayBuffer headers — required for Godot WASM threading
# Without these, Godot falls back to single-threaded or crashes on some browsers
[[headers]]
  for = "/*"
    [headers.values]
    Cross-Origin-Opener-Policy = "same-origin"
    Cross-Origin-Embedder-Policy = "require-corp"
```

## GitHub Secrets Required

| Secret | Used By | Description |
|--------|---------|-------------|
| `NETLIFY_AUTH_TOKEN` | Web export | Netlify CLI authentication token |
| `NETLIFY_SITE_ID` | Web export | Target Netlify site ID |
| `ANDROID_KEYSTORE_BASE64` | Android export | Release keystore, base64-encoded |
| `ANDROID_KEYSTORE_USER` | Android export | Keystore alias |
| `ANDROID_KEYSTORE_PASSWORD` | Android export | Keystore password |
| `APPLE_ID` | iOS export | Apple ID email |
| `APPLE_APP_SPECIFIC_PASSWORD` | iOS export | App-specific password from Apple ID settings |
| `SUPABASE_URL` | Build env | Supabase project URL |
| `SUPABASE_ANON_KEY` | Build env | Supabase anonymous key |

## Checklist

- [ ] `export_presets.cfg` updated in Godot editor (not just manually) — Godot regenerates the file on next export
- [ ] Preset name in `export_presets.cfg` matches exactly the string in `--export-release "Name"` in CI
- [ ] Export output path in preset matches the path used in CI upload/deploy step
- [ ] `needs: run-gut-tests` present on all export jobs — never export without passing tests
- [ ] Godot version consistent across all CI jobs (`barichello/godot-ci:X.X`)
- [ ] `netlify.toml` has both COOP and COEP headers for Web export
- [ ] New GitHub Secrets documented in this file and added to the repo Settings → Secrets
- [ ] First manual run of the CI export job verified before merging
- [ ] iOS export only runs on `macos-latest` runner (never ubuntu for iOS)
