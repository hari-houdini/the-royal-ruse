---
name: write-gut-test
description: Write a GUT (Godot Unit Test) test file for a domain class, use case, or value object. Use when implementing TDD (write test first), adding regression coverage, or verifying a bug fix. Triggers include "write a test", "TDD for", "add tests for", "test this module", "write a GUT test".
---

# Skill: Write a GUT Test

GUT (Godot Unit Test) is the test framework for this project. All domain logic follows TDD: write the failing test first (Red), then implement (Green), then refactor (Refactor/Blue).

**What to test:** External behaviour only — inputs and outputs. Never test internal implementation details (private variable names, internal call order, etc.).

**What NOT to test:** Supabase adapters against a real connection, scene nodes, or anything requiring the Godot scene tree.

## Before You Start

1. Identify the module under test — is it a Domain Entity, Value Object, Domain Service, or Use Case?
2. If testing a Use Case, you need mock ports — use the `write-mock` skill first.
3. Confirm the test file location: `test/unit/domain/` for entities/services, `test/unit/use_cases/` for use cases.

## Test File Structure

```gdscript
# test/unit/[domain|use_cases]/test_module_name.gd
extends GutTest

## GUT test suite for [ModuleName].
## Tests are ordered: happy path first, then validation failures, then edge cases.

# ── Test State ────────────────────────────────────────────────────────────────
# Declare all variables used across tests. Reset in before_each().
var _subject: ModuleName
var _mock_dependency: MockDependencyPort

# ── Setup / Teardown ──────────────────────────────────────────────────────────

func before_each() -> void:
    ## Runs before every test. Create fresh instances — never share state between tests.
    _mock_dependency = MockDependencyPort.new()
    _subject = ModuleName.new(_mock_dependency)

func after_each() -> void:
    ## Runs after every test. Clean up if needed.
    _subject = null

# ── Happy Path Tests ──────────────────────────────────────────────────────────

func test_succeeds_with_valid_input() -> void:
    # Arrange
    var input := "valid_value"

    # Act
    var result: Result = _subject.execute(input)

    # Assert — one logical assertion per test
    assert_true(result.success, "Expected success for valid input '%s'" % input)

func test_returns_correct_data_on_success() -> void:
    var result: Result = _subject.execute("valid_value")
    assert_eq(result.data, "expected_output", "Result data should equal expected_output")

# ── Validation Failure Tests ──────────────────────────────────────────────────

func test_fails_when_input_is_empty() -> void:
    var result: Result = _subject.execute("")
    assert_false(result.success, "Expected failure for empty input")

func test_error_message_mentions_field_name_on_empty_input() -> void:
    var result: Result = _subject.execute("")
    assert_string_contains(result.error_message, "field_name",
        "Error message should identify the failing field")

func test_fails_when_value_exceeds_maximum() -> void:
    var result: Result = _subject.execute("x".repeat(201))
    assert_false(result.success)

# ── Edge Case Tests ───────────────────────────────────────────────────────────

func test_handles_boundary_value_at_maximum() -> void:
    # Test exactly at the boundary — not over, not under
    var result: Result = _subject.execute("x".repeat(200))
    assert_true(result.success, "Should succeed at the exact maximum boundary")

func test_handles_single_item_collection() -> void:
    # Collections with 1 item behave differently from 0 or N
    var result: Result = _subject.execute_on_collection(["single_item"])
    assert_true(result.success)
    assert_eq(result.data.size(), 1)

# ── Mock Interaction Tests ────────────────────────────────────────────────────

func test_calls_dependency_once_on_success() -> void:
    _subject.execute("valid_value")
    assert_eq(_mock_dependency.get_call_count(), 1,
        "Should call the dependency exactly once for a single execution")

func test_does_not_call_dependency_on_validation_failure() -> void:
    _subject.execute("")  # Invalid — should fail before calling dependency
    assert_eq(_mock_dependency.get_call_count(), 0,
        "Should not reach the dependency if input validation fails")
```

## Common Assertion Reference

```gdscript
# Equality
assert_eq(actual, expected, "message")
assert_ne(actual, expected, "message")

# Boolean
assert_true(condition, "message")
assert_false(condition, "message")

# Null
assert_null(value, "message")
assert_not_null(value, "message")

# Strings
assert_string_contains(string, substring, "message")
assert_string_starts_with(string, prefix, "message")

# Collections
assert_has(array, item, "message")
assert_does_not_have(array, item, "message")
assert_eq(array.size(), expected_size, "message")

# Type checking
assert_is(object, ExpectedClass, "message")
```

## Async Test Pattern (for Use Cases that await port calls)

```gdscript
# When the subject uses `await` internally, GUT handles it automatically
# as long as you mark the test with `await` on the subject call:

func test_async_use_case_succeeds() -> void:
    # Act — await the use case (GUT handles async tests transparently)
    var result: Result = await _subject.execute("valid_value")
    # Assert
    assert_true(result.success)
```

## TDD Cycle Checklist

- [ ] Test file created **before** the implementation file
- [ ] `godot --headless` run confirms test FAILS (Red) — never skip this
- [ ] Minimum implementation written to make test pass (Green)
- [ ] `godot --headless` run confirms test PASSES (Green)
- [ ] Code refactored for clarity — tests still pass (Refactor)
- [ ] Test covers: happy path, each validation failure, each edge case
- [ ] Mock interactions verified (call count, last argument)
- [ ] No real Supabase, HTTP, or file I/O in any test
- [ ] Test file name matches: `test_module_name.gd` (prefix always `test_`)
- [ ] All test function names start with `test_`
- [ ] Each test function tests exactly ONE behaviour

## Running Tests

```bash
# Run all tests headless
godot --headless --path . -s addons/gut/gut_cmdln.gd -gdir=res://test/ -ginclude_subdirs -gexit

# Run a specific test file
godot --headless --path . -s addons/gut/gut_cmdln.gd -gtest=res://test/unit/domain/test_module_name.gd -gexit
```
