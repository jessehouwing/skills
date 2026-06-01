---
name: cpp-assertion-quality
description: >
  Analyzes the variety and depth of assertions across C++ test suites
  (GoogleTest, GoogleMock, Boost.Test, Microsoft Native Test Framework). Use when the
  user asks to evaluate assertion quality in C++ tests, find shallow testing,
  identify assertion-free tests, measure assertion diversity, or audit whether
  tests verify different facets of correctness. Produces metrics and actionable
  recommendations.
  DO NOT USE FOR: writing new tests (use the code-testing pipeline agents),
  running tests (use run-cpp-tests), .NET assertion analysis (use
  assertion-quality), or other anti-patterns like flakiness (use
  cpp-test-anti-patterns).
license: MIT
---

# C++ Assertion Diversity Analysis

Analyze C++ test code to measure how varied and meaningful the assertions are. Reveals whether tests verify different facets of correctness — not just equality checks but also exceptions, death tests, string matching, container contents, floating-point tolerance, and mock interactions.

## When to Use

- User asks to evaluate assertion quality or depth in C++ tests
- User asks "are my C++ tests actually testing anything meaningful?"
- User wants to know if assertions are too shallow or repetitive
- User asks for assertion diversity metrics on GoogleTest code

## When Not to Use

- User wants to write new tests (use the code-testing pipeline agents)
- User wants to detect anti-patterns beyond assertions (use `cpp-test-anti-patterns`)
- User wants .NET assertion analysis (use `assertion-quality`)
- User wants to run tests (use `run-cpp-tests`)

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Test code | Yes | One or more C++ test files to analyze |
| Production code | No | The code under test, to evaluate assertion coverage |

## Workflow

### Step 1: Gather the test code

Read all test files. Identify files by `*_test.cpp`, `*_test.cc`, `*Test.cpp`, or framework includes.

### Step 2: Classify every assertion

For each test method, identify all assertions and classify them:

#### GoogleTest assertion categories

| Category | Examples | What it verifies |
|----------|---------|-----------------|
| **Equality** | `EXPECT_EQ`, `ASSERT_EQ`, `EXPECT_NE`, `ASSERT_NE` | Value equality/inequality |
| **Boolean** | `EXPECT_TRUE`, `EXPECT_FALSE`, `ASSERT_TRUE`, `ASSERT_FALSE` | Condition holds |
| **Comparison** | `EXPECT_LT`, `EXPECT_LE`, `EXPECT_GT`, `EXPECT_GE` | Ordering and magnitude |
| **String** | `EXPECT_STREQ`, `EXPECT_STRNE`, `EXPECT_STRCASEEQ`, `EXPECT_STRCASENE` | C-string content |
| **Floating-point** | `EXPECT_FLOAT_EQ`, `EXPECT_DOUBLE_EQ`, `EXPECT_NEAR` | Approximate equality |
| **Exception** | `EXPECT_THROW`, `EXPECT_ANY_THROW`, `EXPECT_NO_THROW` | Error handling behavior |
| **Death** | `EXPECT_DEATH`, `ASSERT_DEATH`, `EXPECT_EXIT` | Process termination behavior |
| **Predicate** | `EXPECT_PRED1`, `EXPECT_PRED2`, `EXPECT_PRED_FORMAT1` | Custom predicate verification |
| **Type** | `EXPECT_THAT` with `WhenDynamicCastTo<T>` | Runtime type checking |
| **Matcher-based** | `EXPECT_THAT` with matchers (`HasSubstr`, `ContainsRegex`, `ElementsAre`, etc.) | Rich structural/content checks |
| **Mock verification** | `EXPECT_CALL` (implicitly verified at mock destruction) | Interaction verification |
| **Negative** | `EXPECT_NE`, `EXPECT_NO_THROW`, `EXPECT_FALSE`, negative matchers | What should NOT happen |

#### Microsoft Native Test Framework categories

| Category | Examples | What it verifies |
|----------|---------|-----------------|
| **Equality** | `Assert::AreEqual`, `Assert::AreNotEqual` | Value equality |
| **Boolean** | `Assert::IsTrue`, `Assert::IsFalse` | Condition |
| **Null** | `Assert::IsNull`, `Assert::IsNotNull` | Pointer validity |
| **Identity** | `Assert::AreSame`, `Assert::AreNotSame` | Reference identity |
| **Exception** | `Assert::ExpectException<T>` | Error handling |
| **Tolerance** | `Assert::AreEqual(expected, actual, tolerance)` | Float/double approximation |
| **Unconditional** | `Assert::Fail` | Force failure |
#### Boost.Test assertion categories

| Category | Examples | What it verifies |
|----------|---------|------------------|
| **Equality** | `BOOST_CHECK_EQUAL`, `BOOST_CHECK_NE`, `BOOST_REQUIRE_EQUAL` | Value equality/inequality |
| **Boolean** | `BOOST_CHECK(expr)`, `BOOST_REQUIRE(expr)` | Condition holds |
| **Comparison** | `BOOST_CHECK_LT`, `BOOST_CHECK_LE`, `BOOST_CHECK_GT`, `BOOST_CHECK_GE` | Ordering |
| **Floating-point** | `BOOST_CHECK_CLOSE(a, b, pct)`, `BOOST_CHECK_SMALL(a, tol)` | Approximate equality |
| **Exception** | `BOOST_CHECK_THROW(expr, T)`, `BOOST_CHECK_NO_THROW(expr)` | Error handling |
| **Message** | `BOOST_CHECK_MESSAGE(expr, msg)` | Condition with diagnostic |
| **Universal** | `BOOST_TEST(expr)` | Modern all-in-one (1.59+) |
| **Unconditional** | `BOOST_FAIL(msg)` | Force failure |
| **Warning** | `BOOST_WARN(expr)`, `BOOST_WARN_EQUAL(a, b)` | Non-failing diagnostic |

**Boost.Test severity levels:**
- `BOOST_WARN_*` — warning only (test still passes)
- `BOOST_CHECK_*` — non-fatal (test continues, reports failure)
- `BOOST_REQUIRE_*` — fatal (test aborts on failure)
### Step 3: Compute metrics

#### Per-test metrics
- **Assertion count**: Number of `EXPECT_*`/`ASSERT_*` macros (count `EXPECT_CALL` as one assertion per call)
- **Assertion categories**: Which categories each test uses
- **Fatal vs non-fatal ratio**: `ASSERT_*` vs `EXPECT_*` usage

#### Suite-wide metrics
- **Average assertions per test**: Total / test count
- **Assertion type spread**: Distinct categories used (out of 12 for GoogleTest)
- **Tests with zero assertions**: Count and % (excluding tests with only `EXPECT_CALL` — those verify via mock destruction)
- **Tests with only trivial assertions**: Only `EXPECT_TRUE(true)` or `ASSERT_NE(nullptr, ptr)` with no value check
- **Tests using matchers**: Count and % (matchers indicate richer verification)
- **Tests with death tests**: Count and %
- **Tests with exception assertions**: Count and %
- **Tests with mock verification**: Count and % (tests that use `EXPECT_CALL`)
- **Single-category tests**: Count and % using only one assertion category

### Step 4: Apply calibration rules

- **Mock-only tests are valid.** A test with `EXPECT_CALL` and an action but no `EXPECT_*`/`ASSERT_*` is still asserting — mock expectations are verified at destruction.
- **Death tests are inherently low-count.** `EXPECT_DEATH(stmt, regex)` may be the only assertion. Don't penalize.
- **Boolean assertions checking meaningful expressions are not trivial.** `EXPECT_TRUE(result.IsValid())` checks behavior. `EXPECT_TRUE(true)` is trivial.
- **Guard ASSERTs before dereference are good practice.** `ASSERT_NE(nullptr, ptr)` before `EXPECT_EQ(42, ptr->value)` — don't count the null check as the "only" assertion.
- **Matchers indicate sophisticated testing.** `EXPECT_THAT(vec, ElementsAre(1, 2, 3))` is richer than `EXPECT_EQ(3, vec.size())`.
- **If diversity is good, say so.**

### Step 5: Report findings

1. **Summary Dashboard** — Key metrics table with assessment
2. **Category Breakdown** — Usage count per category, representative examples
3. **Gap Analysis** — Missing assertion types relative to what the code under test does (e.g., code returns containers but no `ElementsAre`/`UnorderedElementsAre` matchers used)
4. **Recommendations** — Which tests would benefit from matchers, death tests, exception checks, etc.
5. **Assertion-free tests** — List with method names and what they appear to test

## Validation

- [ ] Every assertion classified into at least one category
- [ ] `EXPECT_CALL` counted as mock verification, not ignored
- [ ] Death tests not penalized for low assertion count
- [ ] Guard `ASSERT_NE(nullptr, ...)` not flagged as the sole meaningful assertion
- [ ] Recommendations are concrete (name specific tests, suggest specific matchers)

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Ignoring `EXPECT_CALL` as assertions | Mock expectations ARE assertions — they fail at mock destruction |
| Penalizing death tests | These inherently have one assertion per test |
| Flagging guard null checks as trivial | Only trivial if it's the ONLY assertion and there's no dereference after |
| Missing matcher-based assertions | `EXPECT_THAT` with matchers is a distinct (richer) category from plain `EXPECT_EQ` |
| Not recognizing `SCOPED_TRACE` | It's not an assertion — it adds context. Don't count it. |
| Confusing `ON_CALL` with `EXPECT_CALL` | `ON_CALL` sets default behavior but doesn't assert. Only `EXPECT_CALL` verifies. |
