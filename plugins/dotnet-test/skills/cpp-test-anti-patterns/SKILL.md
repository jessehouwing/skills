---
name: cpp-test-anti-patterns
description: >
  Audits existing C++ test code (GoogleTest, GoogleMock, Boost.Test, Microsoft
  Native Test Framework) for anti-patterns and quality issues — produces a
  severity-ranked report (Critical / High / Medium / Low) with concrete
  code-level fixes. INVOKE THIS SKILL when the user asks to audit, review,
  rank, or find problems in existing C++ tests — including prompts about:
  "audit my C++ tests", "review gtest quality", "review boost test quality",
  "are these tests good", leaked mocks, uninteresting mock calls, no
  assertions, sleep in tests, shared global state, over-mocking with gmock,
  test ordering dependencies, EXPECT vs ASSERT misuse, BOOST_CHECK vs
  BOOST_REQUIRE misuse.
  DO NOT USE FOR: writing new tests (use the code-testing pipeline agents);
  running tests (use run-cpp-tests); .NET tests (use test-anti-patterns).
license: MIT
---

# C++ Test Anti-Pattern Detection

Quick, pragmatic analysis of C++ test code for anti-patterns and quality issues that undermine test reliability, maintainability, and diagnostic value.

## When to Use

- User asks to review C++ test quality or find test smells
- User wants to know why GoogleTest/GoogleMock or Boost.Test tests are flaky or unreliable
- User asks "are my C++ tests good?" or "what's wrong with my tests?"
- User requests a test audit or test code review for C++ code
- User wants to improve existing C++ test code

## When Not to Use

- User wants to write new tests (use the code-testing pipeline agents)
- User wants to run tests (use `run-cpp-tests`)
- User wants to audit .NET tests (use `test-anti-patterns`)
- User wants to measure code coverage (out of scope)

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Test code | Yes | One or more C++ test files to analyze |
| Production code | No | The code under test, for context |
| Specific concern | No | A focused area like "flakiness" or "mocking" |

## Workflow

### Step 1: Gather the test code

Read the test files. Scan for files matching `*_test.cpp`, `*_test.cc`, `*Test.cpp`, or containing `#include <gtest/gtest.h>`, `#include "CppUnitTest.h"`, or `#include <boost/test/unit_test.hpp>`.

### Step 2: Scan for anti-patterns

#### Critical — Tests that give false confidence

| Anti-Pattern | What to Look For |
|---|---|
| **No assertions** | `TEST` or `TEST_F` methods that call production code but have no `EXPECT_*` or `ASSERT_*` macros. Passing without verifying anything. |
| **Always-true assertions** | `EXPECT_TRUE(true)`, `ASSERT_EQ(x, x)`, or conditions that can never fail. |
| **Swallowed exceptions** | `try { ... } catch (...) { }` without failing the test. Use `EXPECT_THROW` instead. |
| **ASSERT in wrong place** | `ASSERT_*` in a helper function called from multiple tests — if it fails, subsequent tests in the same binary may crash due to `ASSERT_*` using `return`. Use `EXPECT_*` in helpers or use `ASSERT_*` only in `TEST`/`TEST_F` bodies. |
| **Dead EXPECT after ASSERT** | `ASSERT_NE(ptr, nullptr)` followed by `EXPECT_EQ(ptr->value, 42)` is fine, but `ASSERT_*` in loops or conditionals that skip remaining checks masks failures. |
| **Mock with no expectations** | Creating a `StrictMock<>` or `NiceMock<>` but never setting `EXPECT_CALL`. The mock does nothing useful. |
| **Coverage touching** | Test that calls every method but asserts nothing meaningful — `auto result = sut.Foo(); (void)result;` or only `ASSERT_NE(nullptr, result)` systematically. |

#### High — Tests likely to cause pain

| Anti-Pattern | What to Look For |
|---|---|
| **Leaked mock objects** | Warning: "Mock object leaked" at test end. Mock must be destroyed before test completes (use local scope or `std::unique_ptr`). |
| **Uninteresting mock calls** | Many "Uninteresting mock function call" warnings. Use `NiceMock<>` if calls are expected but unimportant, or add `EXPECT_CALL` for each. |
| **Sleep for synchronization** | `sleep()`, `usleep()`, `std::this_thread::sleep_for()` in tests. Use condition variables, futures, or latches instead. |
| **Global/static mutable state** | Tests sharing global variables, singletons, or static class members without resetting between tests. Causes ordering dependencies. |
| **Test ordering dependency** | Tests that pass in suite but fail when run individually (`--gtest_filter=TestName`) or with `--gtest_shuffle`. |
| **Over-mocking** | More `EXPECT_CALL`/`ON_CALL` lines than actual test logic. Verifying exact call sequences with `InSequence` when order doesn't matter. |
| **Strict mock overuse** | Using `StrictMock<>` everywhere — any unexpected call fails the test even if irrelevant. Prefer `NiceMock<>` unless strict verification is needed. |
| **EXPECT_CALL after action** | `EXPECT_CALL` set *after* the code under test runs. Expectations must be set *before* the action. |

#### Medium — Maintainability and clarity issues

| Anti-Pattern | What to Look For |
|---|---|
| **Poor naming** | Test names like `Test1`, `BasicTest`, or names that don't describe scenario and expected outcome. Good: `Add_NegativeNumbers_ReturnsZero`. |
| **Magic values** | Unexplained numbers in assertions: `EXPECT_EQ(42, result)` — what does 42 mean? |
| **Giant tests** | Test methods exceeding ~40 lines or testing multiple unrelated behaviors. |
| **Duplicate tests** | 3+ test methods with near-identical bodies differing only in input values. Should use `TEST_P` (parameterized tests). |
| **Missing fixture reuse** | Multiple tests repeating identical setup instead of using `TEST_F` with `SetUp()`. |
| **EXPECT vs ASSERT misuse** | Using `ASSERT_*` everywhere (aborts on first failure, losing subsequent diagnostic info) or using `EXPECT_*` before dereferencing a pointer (should `ASSERT_NE(nullptr, ptr)` first). |
| **Assertion messages absent** | No `<< "context"` on assertions that would be hard to diagnose from the default output. |

#### Low — Style and hygiene

| Anti-Pattern | What to Look For |
|---|---|
| **std::cout debugging** | Leftover `std::cout << ...` or `printf` statements. Use `SCOPED_TRACE()` or `RecordProperty()` instead. |
| **Unused fixture members** | `SetUp()` initializes members never used by some tests. |
| **RAII violations** | Raw `new` without `delete` or smart pointers in test setup. |
| **Inconsistent naming** | Mix of `TestSuiteName_TestName` styles in the same file. |
| **Disabled tests without reason** | `DISABLED_TestName` without a comment explaining why or a tracking issue. |

#### Boost.Test-Specific Anti-Patterns

| Severity | Anti-Pattern | What to Look For |
|---|---|---|
| **Critical** | **Missing BOOST_TEST_MODULE** | No `#define BOOST_TEST_MODULE` before the include → linker error `LNK2005 _main already defined` or test binary that doesn't discover tests. |
| **Critical** | **BOOST_CHECK where BOOST_REQUIRE needed** | `BOOST_CHECK(ptr != nullptr)` followed by `ptr->method()` — if check fails, execution continues and crashes. Use `BOOST_REQUIRE` for precondition guards. |
| **High** | **BOOST_CHECK_EQUAL on floating-point** | `BOOST_CHECK_EQUAL(0.1 + 0.2, 0.3)` — will fail due to floating-point. Use `BOOST_CHECK_CLOSE` or `BOOST_CHECK_SMALL`. |
| **High** | **Manual test registration** | Defining `test_suite* init_unit_test_suite(...)` when `BOOST_AUTO_TEST_CASE` / `BOOST_AUTO_TEST_SUITE` handles this automatically. |
| **Medium** | **BOOST_WARN overuse** | `BOOST_WARN_*` never fails a test — it only logs. If you expect something to hold, use `BOOST_CHECK_*`. |
| **Medium** | **Missing fixture cleanup** | `BOOST_FIXTURE_TEST_CASE` with a fixture that allocates but doesn't clean up in destructor — resources leak across tests. |
| **Low** | **Static linking main.cpp in test project** | Including a `main()` alongside Boost.Test header module — causes `LNK2005`. Boost defines its own `main` when using the header-only variant with `BOOST_TEST_MODULE`. |

### Step 3: Calibrate severity

- **Critical/High**: Issues that cause false passes or flakiness.
- **Medium**: Active maintainability harm — duplicated tests, meaningless names.
- **Low**: Style issues, cosmetic naming mismatches.
- **Not an issue**: Separate tests for distinct edge cases. `NiceMock<>` used intentionally. `ASSERT_*` used correctly as a precondition guard.

If the tests are well-written, say so up front.

### Step 4: Report findings

1. **Summary** — Total issues by severity. Lead with quality assessment.
2. **Critical and High findings** — Each with: anti-pattern name, location (file, test name), why it's a problem, concrete fix (before/after code).
3. **Medium and Low** — Summarized in a table unless user wants detail.
4. **Positive observations** — Good patterns: proper fixture use, `NiceMock` vs `StrictMock` choices, `SCOPED_TRACE`, parameterized tests, clear naming.

### Step 5: Prioritize

1. **Critical** — Fix immediately (false confidence)
2. **High** — Fix soon (flakiness, leaked mocks)
3. **Medium/Low** — Fix opportunistically

## Validation

- [ ] Every finding includes a specific location (file, test name)
- [ ] Every Critical/High finding includes a concrete fix
- [ ] Report covers assertions, mocking, isolation, naming, structure
- [ ] Positive observations included
- [ ] GoogleTest and GoogleMock specifics are correct (macro names, behaviors)

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Flagging `NiceMock` usage as an anti-pattern | `NiceMock` is appropriate when unrelated calls should be silently ignored |
| Flagging `ASSERT_*` as always wrong | `ASSERT_*` is correct for precondition guards (null check before dereference) |
| Reporting `DISABLED_` tests with tracking comments as high | These are Low — they're consciously deferred |
| Missing GoogleMock-specific patterns | Check for `EXPECT_CALL` ordering, `ON_CALL` vs `EXPECT_CALL` confusion, `WillOnce`/`WillRepeatedly` misuse |
| Flagging parameterized test verbosity | `TEST_P` with `INSTANTIATE_TEST_SUITE_P` is intentionally verbose — it's the correct pattern |
| Not checking for `testing::InitGoogleTest` | Missing initialization causes all `--gtest_*` flags to be silently ignored |
| Flagging `BOOST_WARN` as always wrong | `BOOST_WARN` is appropriate for non-critical optional checks that should not fail the test |
| Not recognizing Boost.Test severity model | Boost uses three levels (WARN/CHECK/REQUIRE) — each is valid in the right context |
