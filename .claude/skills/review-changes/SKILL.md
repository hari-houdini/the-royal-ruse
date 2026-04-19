---
name: review-changes
description: Review staged and unstaged git changes against the project's coding standards, architecture rules, and TDD discipline. Scrutinises all file types (GDScript, TypeScript, SQL, YAML, Markdown). Flags violations, warnings, and confirmations with severity levels. Triggers include "review my changes", "check my code", "review this", "code review", "did I do this right", "check before I commit".
---

# Skill: Review Changes

This skill inspects `git diff` (unstaged) and `git diff --cached` (staged) and reviews every changed file against the standards defined in `CLAUDE.md`, `AGENTS.md`, and the project's PRDs and TDD document. It never looks at committed history — only the current working tree delta.

## Severity Levels

| Icon | Level | Meaning |
|------|-------|---------|
| 🔴 | **Wrong** | Violates a hard rule. Must be fixed before committing. |
| 🟡 | **Could be better** | Suboptimal but not broken. Should be addressed. |
| 🟢 | **Correct** | Explicitly confirms something done right. |
| ⚠️ | **Warning** | Process concern (e.g., test written after implementation). Not a code bug. |
| 🔗 | **Interaction consequence** | A file this change interacts with has a downstream concern. |

---

## Step 1 — Collect the Diffs

```bash
# Staged changes (what will be committed)
git diff --cached --unified=5

# Unstaged changes (what is modified but not staged)
git diff --unified=5

# Full list of all changed files (staged + unstaged, deduplicated)
git diff --cached --name-only
git diff --name-only
```

Collect both diffs. Process them separately. Never merge them into one stream — findings must be attributed to the correct bucket.

---

## Step 2 — Identify Changed Files and Their Types

For each changed file, determine its type and which review ruleset applies:

| File Pattern | Type | Ruleset |
|---|---|---|
| `src/core/domain/**/*.gd` | Domain Entity / Value Object / Domain Service | GDScript standards + Domain Layer rules |
| `src/core/use_cases/**/*.gd` | Use Case | GDScript standards + Use Case rules |
| `src/core/ports/**/*.gd` | Port Interface | GDScript standards + Port rules |
| `src/adapters/**/*.gd` | Adapter | GDScript standards + Adapter rules |
| `src/autoloads/**/*.gd` | Autoload | GDScript standards + Autoload rules |
| `src/scenes/**/*.gd` | Scene Script | GDScript standards + Scene rules |
| `test/unit/**/*.gd` | GUT Test | GUT test standards |
| `test/unit/mocks/**/*.gd` | Mock | Mock standards |
| `supabase/functions/**/*.ts` | Edge Function | TypeScript + Edge Function rules |
| `supabase/migrations/**/*.sql` | Migration | SQL standards |
| `.github/workflows/**/*.yml` | CI Workflow | YAML standards |
| `docs/**/*.md` | Documentation | Markdown standards |
| `export_presets.cfg` | Export Config | Export config rules |
| `netlify.toml` | Netlify Config | Web export rules |

---

## Step 3 — Run the Review

For each changed file (staged and unstaged separately), apply the checks below. Report findings in this format:

```
[STAGED / UNSTAGED] — path/to/file.gd

🔴 Wrong — Line 42: `var player_id = ""` — missing type annotation.
   Rule: AGENTS.md → GDScript Standards → "Always use explicit types — no untyped vars"
   Fix: Change to `var player_id: String = ""`
   Skill: No specific skill — apply inline.

🟡 Could be better — Line 87: Public function `execute()` has no `##` doc comment.
   Rule: CLAUDE.md → ALWAYS → "Add a ## doc comment to every public function"
   Fix: Add a doc comment above the function describing what it does and its parameters.
   Skill: No specific skill — apply inline.

🟢 Correct — Result pattern used consistently. All port calls are awaited and checked.
```

---

## Review Checklists by File Type

### GDScript — All Files

```
TYPE ANNOTATIONS
[ ] Every var declaration has an explicit type: var x: int = 0
[ ] Every function parameter is typed: func execute(id: String) -> Result:
[ ] Every function has a return type annotation
[ ] No bare `var x = value` anywhere in the diff

NAMING
[ ] Files are snake_case.gd
[ ] class_name is PascalCase
[ ] Variables and functions are snake_case
[ ] Constants are SCREAMING_SNAKE_CASE
[ ] Signals are past_tense_snake_case

DOC COMMENTS
[ ] Every public function (no _ prefix) has a ## doc comment above it
[ ] [param name] used to document each parameter
[ ] class_name line is preceded by a ## class-level doc comment

FILE SIZE
[ ] File does not exceed 200 lines (count the diff context lines + existing)
[ ] If it does, flag it for extraction

RESULT PATTERN
[ ] All Use Case execute() functions return Result
[ ] All port interface methods return Result
[ ] await used on every async port call
[ ] Result.success is checked before Result.data is accessed

SIGNALS
[ ] All signal declarations include typed parameters
[ ] No untyped signal: signal my_signal(data) — must be signal my_signal(data: Dictionary)
```

### GDScript — Domain Layer (`src/core/domain/`)

```
LAYER ISOLATION
[ ] No imports from src/adapters/
[ ] No imports from src/scenes/
[ ] No Godot scene node dependencies (no Node, Control, etc. as base class)
[ ] Extends RefCounted only (or pure GDScript class)
[ ] No HTTP calls, no WebSocket calls, no file I/O
[ ] No direct Supabase references

ENTITIES
[ ] Has an identity field (e.g., entity_id: String)
[ ] Constructor validates invariants with assert()
[ ] Mutation methods return Result on failure paths

VALUE OBJECTS
[ ] No mutation methods after _init()
[ ] Has an equals() method if comparison is needed
[ ] All properties set in _init() and never modified
```

### GDScript — Use Cases (`src/core/use_cases/`)

```
DEPENDENCY INJECTION
[ ] All dependencies declared as typed port interface vars (INetworkPort, not SupabaseNetworkAdapter)
[ ] All dependencies injected via _init() — never fetched from ServiceLocator inside a method
[ ] No adapter class names appear anywhere in the use case file

VALIDATION FIRST
[ ] execute() validates all inputs at the TOP before any port calls
[ ] Each validation failure returns Result.fail() with a descriptive message

SERVER AUTHORITY
[ ] No score arithmetic in the use case
[ ] No role assignment logic in the use case
[ ] No alliance evaluation in the use case
[ ] These belong in Edge Functions and domain services — flag if present
```

### GDScript — Port Interfaces (`src/core/ports/`)

```
INTERFACE GUARDS
[ ] Every method body contains push_error("PortName.method_name() not implemented by %s" % get_class())
[ ] Every method returns Result.fail("not implemented") after push_error()
[ ] No concrete logic in any method
[ ] Signals declared with typed parameters
```

### GDScript — Adapters (`src/adapters/`)

```
PORT COMPLIANCE
[ ] Extends the correct IPort class
[ ] Implements every method declared on the port interface (check for missing overrides)
[ ] Does not emit GameBus signals directly — emits own port signals instead
[ ] No game logic (no scoring, no role assignment)
```

### GDScript — Scenes (`src/scenes/`)

```
SCENE RULES
[ ] Subscribes to GameBus signals in _ready() — not in _init()
[ ] Disconnects all GameBus signals in _exit_tree()
[ ] Uses ServiceLocator.network() etc. — never imports adapters
[ ] All Use Case calls are awaited
[ ] Result.success checked before proceeding
[ ] No hardcoded scene file paths — uses SceneManager.go_to("key") only
[ ] New scenes are registered in SceneManager.SCENE_PATHS (check if it exists there)
```

### GDScript — GUT Tests (`test/unit/`)

```
TDD DISCIPLINE
[ ] Test file name starts with test_
[ ] Every test function starts with test_
[ ] Mocks used for all port dependencies — no real adapters
[ ] No real Supabase calls, no HTTP calls, no file I/O

TEST QUALITY
[ ] Each test function tests exactly ONE behaviour
[ ] Meaningful failure message in every assertion (third argument to assert_eq etc.)
[ ] Happy path covered
[ ] Each validation failure case covered
[ ] At least one edge case covered (boundary value, empty collection, etc.)
[ ] Mock interaction counts verified where applicable

ASYNC
[ ] await used on use case calls that involve network ports
```

### GDScript — Mocks (`test/unit/mocks/`)

```
MOCK COMPLETENESS
[ ] Extends the correct IPort class
[ ] Implements EVERY method from the port interface
[ ] Has configure_failure() test control method
[ ] Records interactions (_call_count, _call_log)
[ ] Has assertion helper methods (get_call_count, was_called_with, get_last_argument)
[ ] Emits port signals where the real adapter would emit them
[ ] No real network, HTTP, or Supabase calls anywhere
```

### TypeScript — Edge Functions (`supabase/functions/`)

```
IDEMPOTENCY
[ ] Checks for prior execution (completed_at IS NOT NULL or equivalent) BEFORE any writes
[ ] Returns 409 Conflict if already processed

CREDENTIALS
[ ] Uses SUPABASE_SERVICE_ROLE_KEY (not anon key) for the admin client
[ ] No hardcoded URLs or keys — all via Deno.env.get()

INPUT VALIDATION
[ ] Required fields validated at the top of serve()
[ ] Returns 400 for missing required fields

ERROR HANDLING
[ ] All Supabase query errors caught and returned as JSON { error: message } with correct HTTP status
[ ] No unhandled promise rejections

TYPES
[ ] No `any` on domain data — interfaces defined for all input/output shapes
[ ] Return types explicit on all helper functions

DOMAIN LOGIC
[ ] Pure computation functions extracted — no Supabase calls inside computation functions
[ ] successResponse() and errorResponse() helpers used consistently

FILE SIZE
[ ] Under 150 lines — flag for extraction if exceeded
```

### SQL — Migrations (`supabase/migrations/`)

```
FILE NAMING
[ ] Filename starts with a timestamp: YYYYMMDDHHMMSS_
[ ] Timestamp is realistic (not future-dated, not a placeholder)

TABLE STANDARDS
[ ] Primary key is UUID with gen_random_uuid()
[ ] All timestamps are TIMESTAMPTZ (not TIMESTAMP)
[ ] Status/enum fields use TEXT + CHECK constraint (not PostgreSQL ENUM type)
[ ] All NOT NULL columns have a DEFAULT where applicable
[ ] Foreign keys use ON DELETE CASCADE where child is meaningless without parent

RLS — MANDATORY
[ ] ALTER TABLE ... ENABLE ROW LEVEL SECURITY present immediately after CREATE TABLE
[ ] At least one SELECT policy defined for every new table
[ ] Policy names follow convention: "tablename_action_subject"
[ ] No client INSERT/UPDATE/DELETE on score-related tables (rounds, round_roles, scores, alliances)

INDEXES
[ ] Index added for every foreign key column
[ ] Index added for every column used in a WHERE clause in existing queries
```

### YAML — GitHub Actions (`.github/workflows/`)

```
VERSION PINNING
[ ] All `uses:` actions pinned to a specific version (e.g., @v4, not @main or @latest)

TEST DEPENDENCY
[ ] All export/deploy jobs have `needs: run-gut-tests`
[ ] No deploy job can run if tests fail

SECRETS
[ ] All credentials via ${{ secrets.SECRET_NAME }} — no hardcoded values
[ ] No secrets logged with echo or similar

GODOT VERSION CONSISTENCY
[ ] `barichello/godot-ci` image tag is the same across all jobs
[ ] iOS job uses `runs-on: macos-latest` (not ubuntu)
```

### Markdown — Documentation (`docs/`)

```
STRUCTURE
[ ] H1 used only once (document title)
[ ] Heading levels not skipped (no H1 → H3)
[ ] All code blocks have a language specifier

PRDs and SKILLs
[ ] Metadata table present at top (Version, Date, Status, etc.)
[ ] Version number updated if content changed
[ ] All cross-references use relative paths (../prd/PRD-1a-godot-scaffold.md)
```

---

## Step 4 — TDD Compliance Check

After reviewing all individual files, perform this cross-file check:

```
For every changed file in src/core/domain/ or src/core/use_cases/:
  → Is there a corresponding changed file in test/unit/?

  If NO changed test file exists:
    🔴 Wrong — No test changes found for modified domain file: path/to/domain_file.gd
       Rule: TDD requires tests to be written or updated when domain logic changes.
       Fix: Write or update the corresponding GUT test before committing.
       Skill: write-gut-test

For every changed test file in test/unit/:
  → Does git log --follow show the implementation file was modified MORE RECENTLY than the test?
    (Check: if the implementation and test are both new in this diff, warn if implementation
     appears before the test in the diff order — heuristic only)

  If YES (implementation modified after test in same diff):
    ⚠️ Warning — Implementation file appears modified after test file in this diff.
       TDD expects the test to be written first (Red), then the implementation (Green).
       This may indicate the cycle was reversed.
       Rule: CLAUDE.md → ALWAYS → "Write the GUT test before the implementation (TDD — Red-Green-Refactor)"
```

---

## Step 5 — Interaction Consequences Check

For each changed file, check whether it interacts with other files that may now be inconsistent:

```
CHANGED: src/core/ports/i_some_port.gd (method signature changed)
  🔗 Interaction consequence: All adapters implementing this port must be updated.
     Check: src/adapters/supabase/supabase_some_adapter.gd
     Check: src/adapters/local_wifi/local_wifi_some_adapter.gd (Phase 2)
     Check: test/unit/mocks/mock_some_port.gd
     Consequence: If adapters are not updated, they will silently fail at runtime
     (push_error guard will fire but GDScript will not throw an exception).

CHANGED: src/core/domain/entities/role.gd (new RoleId enum value added)
  🔗 Interaction consequence: ObjectiveEvaluator and assign-roles Edge Function
     must both have a corresponding case for the new role_id.
     Check: src/core/domain/objective_evaluator.gd — does it have a match case?
     Check: supabase/functions/assign-roles/index.ts — is it in ROLE_DEFINITIONS?
     Consequence: New role will be assigned but objective will always return false
     and bonus will always be 0. Silent bug.

CHANGED: src/autoloads/game_bus.gd (new signal added)
  🔗 Interaction consequence: If no scene subscribes to this signal yet, verify
     that the emitter exists. An emitted signal with no subscribers is not an error,
     but an unemitted signal with a subscriber will never fire.

CHANGED: src/autoloads/scene_manager.gd (new scene key added)
  🔗 Interaction consequence: Verify the .tscn file exists at the path registered.
     A missing .tscn causes a runtime crash with no compile-time warning in Godot.

CHANGED: supabase/migrations/*.sql (new table added)
  🔗 Interaction consequence: If no RLS policy is defined for this table,
     all client queries will be silently denied (RLS default = deny all).
     Verify ALTER TABLE ... ENABLE ROW LEVEL SECURITY and at least one SELECT policy exist.
```

---

## Step 6 — Format and Deliver the Report

Structure the output exactly as follows:

```
═══════════════════════════════════════════════════════
 THE ROYAL RUSE — Code Review
 Staged:   N files changed
 Unstaged: N files changed
═══════════════════════════════════════════════════════

── STAGED CHANGES ──────────────────────────────────────

src/core/use_cases/submit_vote.gd
  🟢 Correct — Type annotations present on all vars and function signatures.
  🟢 Correct — Result pattern used. All port calls are awaited and checked.
  🔴 Wrong — Line 23: `var network = ServiceLocator.network()` — missing type annotation.
     Rule: AGENTS.md → GDScript Standards → "Always use explicit types"
     Fix: `var network: INetworkPort = ServiceLocator.network()`
     Skill: No specific skill — apply inline.
  🟡 Could be better — Line 45: Public function `_validate_input()` has no ## doc comment.
     Rule: CLAUDE.md → ALWAYS → "Add a ## doc comment to every public function"
     Fix: Add doc comment above the function.
     Skill: No specific skill — apply inline.

── UNSTAGED CHANGES ────────────────────────────────────

test/unit/use_cases/test_submit_vote.gd
  🟢 Correct — Mock dependencies injected via constructor.
  🟢 Correct — Both happy path and validation failure covered.
  🟡 Could be better — No edge case test for Police accusing themselves.
     Rule: write-gut-test skill → "At least one edge case covered"
     Fix: Add test_fails_when_police_accuses_themselves()
     Skill: write-gut-test

── TDD COMPLIANCE ──────────────────────────────────────

  🟢 Correct — test_submit_vote.gd modified alongside submit_vote.gd.

── INTERACTION CONSEQUENCES ────────────────────────────

  (None detected)

── SUMMARY ─────────────────────────────────────────────

  🔴 Wrong:            1   ← Must fix before committing
  🟡 Could be better:  2   ← Should address
  🟢 Correct:          5
  ⚠️  Warnings:         0
  🔗 Consequences:      0

  Ready to commit? NO — 1 blocking issue found.
```

---

## Step 7 — Offer to Apply Fixes

After delivering the report, always ask:

```
I found [N] issues. Would you like me to:

A) Apply all 🔴 Wrong fixes automatically (I'll edit the files directly)
B) Apply all 🔴 Wrong and 🟡 Could be better fixes automatically
C) Walk through each fix one at a time so you apply them yourself
D) Leave it — I'll fix them manually

Reply with A, B, C, or D.
```

If the user chooses A or B: apply the changes directly to the working files using file editing tools. After applying, re-run the relevant diff sections to confirm the fixes are clean.

If the user chooses C: present each fix one at a time with the exact change needed, wait for confirmation, then move to the next.

If the user chooses D: close the review. Do not re-run unless asked.

---

## Git Hook Integration

### Installing the pre-commit hook

Create `.git/hooks/pre-commit` in the repo:

```bash
#!/bin/sh
# The Royal Ruse — pre-commit code review hook
# Runs the review-changes skill via Claude Code before every commit.
# To bypass: git commit --no-verify

echo "Running code review via Claude Code..."
echo "To skip this hook: git commit --no-verify"
echo ""

# Check if Claude Code is available
if ! command -v claude &> /dev/null; then
  echo "Claude Code not found — skipping automated review."
  echo "Run 'review my changes' manually in Claude Code before committing."
  exit 0
fi

# Run the review skill
claude "review my changes" --skill review-changes

# Exit code 0 = allow commit, 1 = block commit
# The hook does NOT automatically block — Claude Code delivers the report
# and the developer decides whether to proceed.
# To make blocking automatic, change the line above to:
# claude "review my changes" --skill review-changes --exit-on-error
exit 0
```

```bash
# Make it executable
chmod +x .git/hooks/pre-commit
```

### Bypassing the hook

```bash
# One-time bypass (emergency commit, WIP, docs-only change)
git commit --no-verify -m "your message"

# Temporary disable
chmod -x .git/hooks/pre-commit

# Re-enable
chmod +x .git/hooks/pre-commit
```

### Sharing the hook with the team

Git hooks are not committed with the repo by default. To share:

```bash
# Store the hook in a tracked directory
mkdir -p .githooks
cp .git/hooks/pre-commit .githooks/pre-commit
git add .githooks/pre-commit

# Add setup instruction to README:
# git config core.hooksPath .githooks
```

---

## Triggering the Review Manually

In Claude Code terminal, any of these trigger this skill:

```
review my changes
check my code
review this
code review
did I do this right
check before I commit
```

The skill reads the live git diff at the moment it is invoked. Run it as many times as needed during a working session — each run reflects the current state of the working tree.
