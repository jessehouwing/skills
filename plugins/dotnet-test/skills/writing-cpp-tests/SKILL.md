---
name: writing-cpp-tests
description: >
  Write new C++ unit tests and implement concrete fixes in existing C++ test code using
  GoogleTest (gtest), GoogleMock (gmock), Boost.Test, and Microsoft Native Test Framework
  (CppUnitTest) with modern best practices.
  USE FOR: write C++ unit tests, write gtest tests, create test fixture, write gmock mock,
  fix gtest assertions, EXPECT_EQ vs ASSERT_EQ, parameterized tests, TEST_P, MOCK_METHOD,
  mock class, matchers, actions, expectations, InSequence, NiceMock, StrictMock,
  write CppUnitTest tests, TEST_CLASS, TEST_METHOD, death tests, typed tests,
  write Boost.Test tests, BOOST_AUTO_TEST_CASE, BOOST_CHECK, BOOST_FIXTURE_TEST_CASE,
  BOOST_DATA_TEST_CASE, BOOST_TEST_MODULE,
  something seems off with my C++ tests, review C++ tests and fix issues.
  DO NOT USE FOR: broad test quality audits (use cpp-test-anti-patterns),
  assertion diversity analysis (use cpp-assertion-quality),
  running or building tests (use run-cpp-tests),
  MSTest/.NET tests (use writing-mstest-tests).
license: MIT
---

# Writing C++ Tests

Help users write effective C++ unit tests with GoogleTest/GoogleMock, Boost.Test, or Microsoft Native Test Framework using current APIs and best practices.

## When to Use

- User wants to write new C++ unit tests (gtest, gmock, Boost.Test, or MS Native)
- User wants to improve or modernize existing C++ tests
- User asks about gtest assertions, fixtures, parameterized tests, or mocking patterns
- User needs help writing a mock class with `MOCK_METHOD`
- User asks about matchers, actions, or expectation ordering in gmock
- User needs help fixing a specific gtest/gmock compilation or runtime error
- User wants to write Microsoft Native Test Framework tests (CppUnitTest.h)
- User wants to write Boost.Test tests (BOOST_AUTO_TEST_CASE, fixtures, data-driven)

## When Not to Use

- User needs a test quality audit or anti-pattern detection (use `cpp-test-anti-patterns`)
- User needs assertion diversity analysis (use `cpp-assertion-quality`)
- User needs to run, build, or execute tests (use `run-cpp-tests`)
- User needs coverage analysis (use `coverage-analysis`)
- User is writing .NET tests (use `writing-mstest-tests`)

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Code under test | No | The C++ production code (`.h`/`.cpp`) to be tested |
| Existing test code | No | Current tests to fix, update, or extend |
| Test scenario description | No | What behavior the user wants to test |

## Response Guidelines

- **Specific API or pattern questions** (assertions, matchers, fixtures): Jump directly to the relevant section. Do not follow the full workflow.
- **Write new tests from scratch**: Follow the full workflow.
- **Review and fix existing tests**: Fix only the issues present. Do not add unrelated improvements.

## Workflow

### Step 1: Investigate the Repo First

Before writing any test, read the existing test files and build configuration:

1. **Existing tests** — find `*_test.cpp`, `*_test.cc`, `*Test.cpp` and copy their style
2. **`CMakeLists.txt`** — look for `FetchContent_Declare(googletest ...)`, `find_package(GTest)`, or `add_subdirectory(googletest)`
3. **`*.vcxproj` / `*.sln`** — check for GoogleTest NuGet packages or MS Native Test Framework references
4. **Package managers** — `vcpkg.json`, `conanfile.txt` for gtest dependency

Use whatever test style and layout the repo already uses. Do not introduce a different framework.

### Step 2: Write test classes following conventions

#### GoogleTest conventions

- One test file per source file: `calculator.cpp` → `calculator_test.cpp`
- Name test suites after the class/component: `CalculatorTest`, `UserStoreTest`
- Name tests as `SuiteName.MethodOrBehavior_Condition_ExpectedResult`
- Use `TEST()` for simple tests, `TEST_F()` for fixtures, `TEST_P()` for parameterized

```cpp
#include <gtest/gtest.h>
#include "calculator.h"

TEST(CalculatorTest, Add_TwoPositiveNumbers_ReturnsSum) {
    Calculator calc;
    EXPECT_EQ(calc.Add(2, 3), 5);
}

TEST(CalculatorTest, Divide_ByZero_Throws) {
    Calculator calc;
    EXPECT_THROW(calc.Divide(10, 0), std::invalid_argument);
}
```

#### Microsoft Native Test Framework conventions

- Use `TEST_CLASS` for grouping and `TEST_METHOD` for individual tests
- Wrap in a namespace
- Use `Assert::` methods for all assertions

```cpp
#include "pch.h"
#include "CppUnitTest.h"
#include "../MyApp/calculator.h"

using namespace Microsoft::VisualStudio::CppUnitTestFramework;

namespace CalculatorTests
{
    TEST_CLASS(CalculatorTest)
    {
    public:
        TEST_METHOD(Add_TwoPositiveNumbers_ReturnsSum)
        {
            Calculator calc;
            Assert::AreEqual(5, calc.Add(2, 3));
        }

        TEST_METHOD(Divide_ByZero_Throws)
        {
            Calculator calc;
            Assert::ExpectException<std::invalid_argument>([&calc]() {
                calc.Divide(10, 0);
            });
        }
    };
}
```

### Step 3: Use test fixtures for shared setup

Use fixtures when multiple tests share initialization or teardown:

```cpp
#include <gtest/gtest.h>
#include <memory>
#include "user_store.h"

class UserStoreTest : public ::testing::Test {
protected:
    void SetUp() override {
        store = std::make_unique<UserStore>(":memory:");
        store->Seed({{"alice"}, {"bob"}});
    }

    void TearDown() override {
        // Optional cleanup (prefer RAII instead)
    }

    std::unique_ptr<UserStore> store;
};

TEST_F(UserStoreTest, Find_ExistingUser_ReturnsUser) {
    auto user = store->Find("alice");
    ASSERT_TRUE(user.has_value());
    EXPECT_EQ(user->name, "alice");
}

TEST_F(UserStoreTest, Find_MissingUser_ReturnsNullopt) {
    auto user = store->Find("unknown");
    EXPECT_FALSE(user.has_value());
}
```

**Key rules:**
- Fixture class inherits from `::testing::Test`
- Use `protected` for member variables (accessed by `TEST_F` macros)
- Override `SetUp()` / `TearDown()` — note the capitalization (not `Setup`)
- Use `RAII` (smart pointers, unique handles) over explicit `TearDown()` when possible

#### MS Native Test Framework lifecycle

```cpp
TEST_CLASS(MyTest)
{
public:
    TEST_CLASS_INITIALIZE(ClassSetup)   // Once before all tests in class
    {
        // shared expensive setup
    }

    TEST_CLASS_CLEANUP(ClassTeardown)   // Once after all tests in class
    {
    }

    TEST_METHOD_INITIALIZE(TestSetup)   // Before each test method
    {
    }

    TEST_METHOD_CLEANUP(TestTeardown)   // After each test method
    {
    }
};
```

### Step 4: Use parameterized tests for multiple inputs

```cpp
#include <gtest/gtest.h>
#include "calculator.h"

struct AddTestCase {
    int a;
    int b;
    int expected;
};

class AddTest : public ::testing::TestWithParam<AddTestCase> {};

TEST_P(AddTest, ReturnsCorrectSum) {
    auto [a, b, expected] = GetParam();
    EXPECT_EQ(Calculator{}.Add(a, b), expected);
}

INSTANTIATE_TEST_SUITE_P(Calculator, AddTest, ::testing::Values(
    AddTestCase{2, 3, 5},
    AddTestCase{-1, 1, 0},
    AddTestCase{0, 0, 0},
    AddTestCase{INT_MAX, 0, INT_MAX}
));
```

**Value generators:**

| Generator | Usage |
|-----------|-------|
| `::testing::Values(v1, v2, ...)` | Explicit list of values |
| `::testing::ValuesIn(container)` | Values from a container/array |
| `::testing::Range(begin, end, step)` | Numeric range |
| `::testing::Bool()` | `true` and `false` |
| `::testing::Combine(g1, g2, ...)` | Cartesian product of generators |

### Step 5: Use appropriate assertions

#### EXPECT vs ASSERT

- Use `EXPECT_*` for checks where the test can continue after failure (reports all failures)
- Use `ASSERT_*` for preconditions where continuing is meaningless (test aborts on failure)

```cpp
TEST_F(UserStoreTest, GetUser_ReturnsCorrectFields) {
    auto user = store->Get(1);
    ASSERT_TRUE(user.has_value());        // Abort if null — can't check fields
    EXPECT_EQ(user->name, "alice");       // Continue checking other fields
    EXPECT_EQ(user->email, "alice@x.com");
    EXPECT_GT(user->id, 0);
}
```

#### Assertion reference

| Category | Fatal | Non-fatal | Description |
|----------|-------|-----------|-------------|
| Equality | `ASSERT_EQ(a, b)` | `EXPECT_EQ(a, b)` | `a == b` |
| Inequality | `ASSERT_NE(a, b)` | `EXPECT_NE(a, b)` | `a != b` |
| Less than | `ASSERT_LT(a, b)` | `EXPECT_LT(a, b)` | `a < b` |
| Less/equal | `ASSERT_LE(a, b)` | `EXPECT_LE(a, b)` | `a <= b` |
| Greater than | `ASSERT_GT(a, b)` | `EXPECT_GT(a, b)` | `a > b` |
| Greater/equal | `ASSERT_GE(a, b)` | `EXPECT_GE(a, b)` | `a >= b` |
| Boolean true | `ASSERT_TRUE(cond)` | `EXPECT_TRUE(cond)` | Condition is true |
| Boolean false | `ASSERT_FALSE(cond)` | `EXPECT_FALSE(cond)` | Condition is false |
| C-string equal | `ASSERT_STREQ(a, b)` | `EXPECT_STREQ(a, b)` | C-string equality |
| C-string not equal | `ASSERT_STRNE(a, b)` | `EXPECT_STRNE(a, b)` | C-string inequality |
| Float near | `ASSERT_NEAR(a, b, tol)` | `EXPECT_NEAR(a, b, tol)` | `|a - b| <= tol` |
| Float equal | `ASSERT_FLOAT_EQ(a, b)` | `EXPECT_FLOAT_EQ(a, b)` | 4 ULP tolerance |
| Double equal | `ASSERT_DOUBLE_EQ(a, b)` | `EXPECT_DOUBLE_EQ(a, b)` | 4 ULP tolerance |
| Throws type | `ASSERT_THROW(stmt, T)` | `EXPECT_THROW(stmt, T)` | Throws exception of type T |
| No throw | `ASSERT_NO_THROW(stmt)` | `EXPECT_NO_THROW(stmt)` | No exception thrown |
| Any throw | `ASSERT_ANY_THROW(stmt)` | `EXPECT_ANY_THROW(stmt)` | Any exception thrown |
| Death test | `ASSERT_DEATH(stmt, re)` | `EXPECT_DEATH(stmt, re)` | Process dies with matching stderr |
| Matcher | `ASSERT_THAT(val, m)` | `EXPECT_THAT(val, m)` | gmock matcher `m` |

#### MS Native Test Framework assertions

| Assertion | Description |
|-----------|-------------|
| `Assert::AreEqual(expected, actual)` | Equality (int, string, wchar_t*, etc.) |
| `Assert::AreEqual(exp, act, tolerance)` | Floating-point with tolerance |
| `Assert::AreNotEqual(a, b)` | Inequality |
| `Assert::AreSame(a, b)` | Reference identity |
| `Assert::IsTrue(condition)` | Boolean true |
| `Assert::IsFalse(condition)` | Boolean false |
| `Assert::IsNull(ptr)` | Pointer is null |
| `Assert::IsNotNull(ptr)` | Pointer is not null |
| `Assert::ExpectException<T>(func)` | Expects exception of type T |
| `Assert::Fail(message)` | Unconditional failure |

### Step 6: Write mocks with GoogleMock

#### Define a mock class

Given an interface:

```cpp
class Database {
public:
    virtual ~Database() = default;
    virtual bool Connect(const std::string& url) = 0;
    virtual std::optional<Record> Get(int id) = 0;
    virtual void Put(int id, const Record& record) = 0;
};
```

The mock:

```cpp
#include <gmock/gmock.h>

class MockDatabase : public Database {
public:
    MOCK_METHOD(bool, Connect, (const std::string& url), (override));
    MOCK_METHOD(std::optional<Record>, Get, (int id), (override));
    MOCK_METHOD(void, Put, (int id, const Record& record), (override));
};
```

**`MOCK_METHOD` rules:**
- 4 parameters: `(ReturnType, MethodName, (Args), (Qualifiers))`
- Use `(override)` for virtual methods, `(const, override)` for const methods
- The base class destructor **must** be virtual
- For return types with commas (templates), wrap in extra parentheses: `MOCK_METHOD((std::pair<int,int>), Func, ())`

#### Set expectations

```cpp
TEST(ServiceTest, Publish_Connected_SendsNotification) {
    MockNotifier notifier;
    Service service(notifier);

    EXPECT_CALL(notifier, Send("hello")).Times(1);
    service.Publish("hello");
}

TEST(ServiceTest, Publish_Disconnected_SkipsSend) {
    MockNotifier notifier;
    Service service(notifier);

    using ::testing::Return;
    ON_CALL(notifier, IsConnected()).WillByDefault(Return(false));
    EXPECT_CALL(notifier, Send(::testing::_)).Times(0);
    service.Publish("hello");
}
```

#### NiceMock vs StrictMock

| Wrapper | Behavior on uninteresting calls |
|---------|-------------------------------|
| (default) | Warning printed |
| `NiceMock<MockFoo>` | Silently ignored |
| `StrictMock<MockFoo>` | Test failure |

Use `NiceMock<>` when you only care about specific calls. Use `StrictMock<>` when the test must verify that *no* unexpected calls happen.

#### Common matchers

| Matcher | Description |
|---------|-------------|
| `::testing::_` | Matches anything |
| `Eq(val)` | Exact equality |
| `Ne(val)` | Not equal |
| `Gt(val)` / `Lt(val)` / `Ge(val)` / `Le(val)` | Comparison |
| `HasSubstr("x")` | String contains |
| `StartsWith("x")` / `EndsWith("x")` | String prefix/suffix |
| `MatchesRegex(re)` | Regex match |
| `IsEmpty()` | Empty container or string |
| `SizeIs(n)` | Container size |
| `Contains(x)` | Container has element |
| `Each(m)` | Every element matches m |
| `ElementsAre(...)` | Exact container contents in order |
| `UnorderedElementsAre(...)` | Same elements, any order |
| `Field(&T::member, m)` | Struct field matches m |
| `Property(&T::getter, m)` | Property matches m |
| `Pointee(m)` | Dereferenced pointer matches m |
| `IsNull()` / `NotNull()` | Pointer checks |
| `AllOf(m1, m2)` | All matchers pass |
| `AnyOf(m1, m2)` | At least one passes |
| `Not(m)` | Negation |

#### Common actions

| Action | Description |
|--------|-------------|
| `Return(val)` | Return a value |
| `ReturnRef(var)` | Return a reference |
| `ReturnArg<N>()` | Return the Nth argument |
| `Throw(exception)` | Throw an exception |
| `DoAll(a1, a2, ...)` | Perform multiple actions |
| `Invoke(func)` | Call a function/lambda |
| `InvokeWithoutArgs(func)` | Call a zero-arg function |
| `SaveArg<N>(ptr)` | Save the Nth argument to pointer |
| `SetArgReferee<N>(val)` | Set a reference argument |
| `DoDefault()` | Use the ON_CALL default |

#### Expectation ordering

```cpp
using ::testing::InSequence;

TEST(ServiceTest, ConnectsThenQueries) {
    InSequence seq;  // All EXPECT_CALLs below must happen in order
    MockDatabase db;

    EXPECT_CALL(db, Connect("localhost"));
    EXPECT_CALL(db, Get(42));

    Service svc(db);
    svc.FetchRecord(42);
}
```

For partial ordering (some calls ordered, others independent):

```cpp
using ::testing::Sequence;

TEST(ServiceTest, PartialOrder) {
    Sequence s1, s2;
    MockDatabase db;

    EXPECT_CALL(db, Connect("host")).InSequence(s1, s2);  // Must be first
    EXPECT_CALL(db, Get(1)).InSequence(s1);               // After Connect
    EXPECT_CALL(db, Get(2)).InSequence(s2);               // After Connect (independent of Get(1))
}
```

### Step 7: Test edge cases and error paths

#### Exception testing

```cpp
TEST(ParserTest, Parse_EmptyInput_ThrowsInvalidArgument) {
    Parser p;
    EXPECT_THROW(p.Parse(""), std::invalid_argument);
}

TEST(ParserTest, Parse_EmptyInput_ExceptionHasMessage) {
    Parser p;
    try {
        p.Parse("");
        FAIL() << "Expected std::invalid_argument";
    } catch (const std::invalid_argument& e) {
        EXPECT_THAT(e.what(), ::testing::HasSubstr("empty"));
    }
}
```

#### Death tests (process termination)

```cpp
TEST(ContractTest, NullPointer_Aborts) {
    EXPECT_DEATH(ProcessData(nullptr), ".*null.*");
}
```

Death tests fork a subprocess — name the suite with `DeathTest` suffix for gtest to handle correctly:

```cpp
using ContractDeathTest = ContractTest;  // Alias for death test suite naming

TEST_F(ContractDeathTest, NullPointer_Aborts) {
    EXPECT_DEATH(ProcessData(nullptr), "");
}
```

#### Testing private/internal code

Prefer testing through the public API. If necessary, use a preprocessor-guarded `friend`:

```cpp
class MyClass {
#ifdef UNIT_TESTING
    friend class MyClassTest;
#endif
    int InternalMethod();
};
```

Define `UNIT_TESTING` only in the test build:

```cmake
target_compile_definitions(my_tests PRIVATE UNIT_TESTING)
```

### Step 8: Apply best practices

#### DO

- Keep tests deterministic and isolated — no shared mutable state between tests
- Prefer dependency injection (constructor injection) over globals
- Use `ASSERT_*` for preconditions, `EXPECT_*` for multiple checks per test
- Link `GTest::gtest_main` — avoid writing your own `main()` unless custom initialization is needed
- Use `NiceMock<>` when uninteresting calls are expected
- Base class destructors must be `virtual` for mocks to work
- Use structured bindings (`auto [a, b, c] = GetParam()`) in parameterized tests
- Prefer RAII and smart pointers over manual `TearDown()` cleanup

#### DON'T

- Don't depend on test execution order — each test must be independent
- Don't use `sleep` or `std::this_thread::sleep_for` for synchronization — use condition variables, latches, or futures
- Don't over-mock simple value objects — only mock interfaces/abstractions
- Don't test private methods directly — test through the public API
- Don't use `ASSERT_*` after the point where cleanup is needed — prefer RAII/fixtures
- Don't leak mock objects — ensure mocks are destroyed before test ends (use local scope)
- Don't write your own `main()` unless you need custom global setup (e.g., `::testing::AddGlobalTestEnvironment`)

## Boost.Test (via VS Test Adapter)

Boost.Test is supported in Visual Studio via the Boost.Test Adapter (included with "Desktop development with C++" workload). Tests are discoverable in Test Explorer and runnable via `vstest.console.exe`.

### Installation

```bash
# Dynamic library
vcpkg install boost-test

# Static library
vcpkg install boost-test:x86-windows-static

# Integrate with VS
vcpkg integrate install
```

### Basic test structure

```cpp
#define BOOST_TEST_MODULE MyTestModule
#include <boost/test/included/unit_test.hpp>  // single-header variant
#include "calculator.h"

BOOST_AUTO_TEST_SUITE(CalculatorTests)

BOOST_AUTO_TEST_CASE(Add_TwoPositiveNumbers_ReturnsSum) {
    Calculator calc;
    BOOST_CHECK_EQUAL(calc.Add(2, 3), 5);
}

BOOST_AUTO_TEST_CASE(Divide_ByZero_Throws) {
    Calculator calc;
    BOOST_CHECK_THROW(calc.Divide(10, 0), std::invalid_argument);
}

BOOST_AUTO_TEST_SUITE_END()
```

**Include options:**
- `#include <boost/test/included/unit_test.hpp>` — single-header (simplest, no separate compilation)
- `#include <boost/test/unit_test.hpp>` — standalone library (link against `boost_unit_test_framework`)

### Fixtures

```cpp
struct DatabaseFixture {
    DatabaseFixture() : db(":memory:") { db.Seed(); }   // setup
    ~DatabaseFixture() { db.Reset(); }                   // teardown
    Database db;
};

BOOST_FIXTURE_TEST_CASE(Find_ExistingUser_ReturnsUser, DatabaseFixture) {
    auto user = db.Find("alice");
    BOOST_REQUIRE(user.has_value());
    BOOST_CHECK_EQUAL(user->name, "alice");
}
```

### Data-driven tests

```cpp
#include <boost/test/data/test_case.hpp>
#include <boost/test/data/monomorphic.hpp>

namespace bdata = boost::unit_test::data;

BOOST_DATA_TEST_CASE(Add_MultipleInputs, 
    bdata::make({1, -1, 0}) ^ bdata::make({2, 1, 0}) ^ bdata::make({3, 0, 0}),
    a, b, expected) {
    BOOST_CHECK_EQUAL(Calculator{}.Add(a, b), expected);
}
```

### Assertion reference

| Level | Check (non-fatal) | Require (fatal) | Description |
|-------|-------------------|-----------------|-------------|
| Boolean | `BOOST_CHECK(expr)` | `BOOST_REQUIRE(expr)` | Expression is true |
| Equality | `BOOST_CHECK_EQUAL(a, b)` | `BOOST_REQUIRE_EQUAL(a, b)` | `a == b` |
| Inequality | `BOOST_CHECK_NE(a, b)` | `BOOST_REQUIRE_NE(a, b)` | `a != b` |
| Less than | `BOOST_CHECK_LT(a, b)` | `BOOST_REQUIRE_LT(a, b)` | `a < b` |
| Less/equal | `BOOST_CHECK_LE(a, b)` | `BOOST_REQUIRE_LE(a, b)` | `a <= b` |
| Greater than | `BOOST_CHECK_GT(a, b)` | `BOOST_REQUIRE_GT(a, b)` | `a > b` |
| Greater/equal | `BOOST_CHECK_GE(a, b)` | `BOOST_REQUIRE_GE(a, b)` | `a >= b` |
| Float close | `BOOST_CHECK_CLOSE(a, b, pct)` | `BOOST_REQUIRE_CLOSE(a, b, pct)` | Within `pct`% |
| Near zero | `BOOST_CHECK_SMALL(a, tol)` | `BOOST_REQUIRE_SMALL(a, tol)` | `|a| < tol` |
| Exception | `BOOST_CHECK_THROW(expr, T)` | `BOOST_REQUIRE_THROW(expr, T)` | Throws type T |
| No throw | `BOOST_CHECK_NO_THROW(expr)` | `BOOST_REQUIRE_NO_THROW(expr)` | No exception |
| Message | `BOOST_CHECK_MESSAGE(expr, msg)` | `BOOST_REQUIRE_MESSAGE(expr, msg)` | With custom message |
| Universal | `BOOST_TEST(expr)` | — | Modern universal assertion (1.59+) |

- `BOOST_CHECK_*` — non-fatal (test continues on failure)
- `BOOST_REQUIRE_*` — fatal (test aborts on failure)
- `BOOST_WARN_*` — warning only (test still passes)

### Visual Studio project setup

**Option A: Separate test project** (recommended)
1. Add a Console App project to the solution
2. Install Boost.Test via vcpkg or NuGet
3. Delete the default `main()` (Boost.Test defines its own)
4. Add `#define BOOST_TEST_MODULE` before the include
5. Add project reference to the code under test

**Option B: Tests inside the project**
1. Add a Boost.Test item (Add → New Item → Visual C++ → Test → Boost.Test)
2. Create a separate build configuration (e.g., "Debug UnitTests")
3. Exclude test files from Debug/Release configs, exclude `main.cpp` from test config

**Static library linking:** For static Boost.Test, set in vcxproj:
- Vcpkg triplet: `x86-windows-static` (in `<VcpkgTriplet>` property)
- Runtime Library: `/MTd` (debug) or `/MT` (release)

### When to use Boost.Test vs alternatives

| Use Boost.Test when | Use GoogleTest when | Use MS Native when |
|--------------------|--------------------|-------------------|
| Project already uses Boost heavily | Need gmock for mocking | Simple VS-only project, no mocking |
| Cross-platform without GoogleTest dependency | Team familiar with gtest | Team prefers MS tooling |
| Need data-driven via `boost::unit_test::data` | Need rich matchers (gmock) | Quick setup, no external deps |

## Google C++ Style Guide — Test & Mock Conventions

The [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html) contains several rules that directly affect how tests and mocks are written. Apply these when working in codebases that follow Google style.

### Test file naming

- Use `_test.cc` suffix: `foo_bar_test.cc` (not `_unittest.cc` or `_regtest.cc` — those are deprecated)
- Filenames are all lowercase with underscores

### Include order in test files

In `dir/foo_test.cc` testing `dir2/foo.h`, order includes as:

1. `dir2/foo.h` (the header under test — **always first**)
2. C system headers (e.g., `<unistd.h>`)
3. C++ standard library headers (e.g., `<string>`, `<vector>`)
4. Other libraries' headers (e.g., `<gmock/gmock.h>`, `<gtest/gtest.h>`)
5. Your project's headers

Separate each group with a blank line. Placing the header under test first ensures that missing includes in that header are caught immediately.

### Access control in test fixtures

- Class data members must be `private` in production code
- **Exception for tests:** data members of a test fixture class defined in a `.cc` file may be `protected` when using GoogleTest (so `TEST_F` can access them)
- If the fixture is defined in a `.h` file, keep data members `private`

```cpp
// In foo_test.cc — protected is allowed here
class FooTest : public ::testing::Test {
protected:
    Foo foo_;                    // OK — test fixture in .cc file
    std::string test_data_;
};
```

### Using `friend` for testing

- A unit test class may be declared as a `friend` of the class it tests
- Friends should be defined in the same file so the reader can find all private-member access
- Prefer testing through the public API; use `friend` only when necessary

```cpp
class Foo {
    friend class FooTest;  // Grant test access to internals
    FRIEND_TEST(FooTest, InternalStateIsConsistent);  // GoogleTest macro
    int internal_state_;
};
```

### Virtual destructors and `override` (critical for mocks)

- Base class destructors **must** be `virtual` for mocks to work correctly (prevents undefined behavior when deleting through base pointer)
- Always annotate overrides with `override` (never re-state `virtual` on the override)
- The compiler will error if a method marked `override` doesn't actually override a base virtual — this catches mock method signature mismatches

```cpp
class Database {
public:
    virtual ~Database() = default;                        // MUST be virtual
    virtual bool Connect(const std::string& url) = 0;
};

class MockDatabase : public Database {
public:
    MOCK_METHOD(bool, Connect, (const std::string& url), (override));  // use override
};
```

### RTTI in tests

- `dynamic_cast` and `typeid` may be used freely in unit tests
- Useful for verifying that a factory returns the expected dynamic type
- Useful for managing relationships between objects and their mocks

```cpp
TEST(FactoryTest, CreateWidget_ReturnsConcreteWidget) {
    auto widget = factory.Create("fancy");
    ASSERT_NE(dynamic_cast<FancyWidget*>(widget.get()), nullptr);
}
```

### Naming conventions for test code

| Entity | Convention | Example |
|--------|-----------|---------|
| Test fixture class | PascalCase (type name) | `FooBarTest` |
| Mock class | PascalCase with `Mock` prefix | `MockDatabase` |
| Test suite / fixture | PascalCase | `CalculatorTest` |
| Test name (in `TEST`/`TEST_F`) | PascalCase | `Add_TwoPositives_ReturnsSum` |
| Helper functions in tests | PascalCase | `CreateTestWidget()` |
| Local variables in tests | snake_case | `expected_result` |
| Fixture data members | snake_case with trailing `_` (class) or without (struct) | `mock_db_` |
| Constants | `k` + PascalCase | `kDefaultTimeout` |

### `explicit` constructors in test helpers

- Single-argument constructors must be marked `explicit` (prevents accidental implicit conversions in test setup code)
- Applies to custom test helper types, fake implementations, and builder objects

```cpp
class FakeConnection {
public:
    explicit FakeConnection(int port);  // Prevents FakeConnection c = 8080;
};
```

### Avoid exceptions in production mocks

- Google style forbids C++ exceptions in production code
- Mock actions should use `Return()`, error codes, or `std::optional` — not `Throw()` — unless the codebase explicitly enables exceptions
- In test-only code that never ships, `EXPECT_THROW` / `Throw()` may still be used if the code under test uses exceptions

### Lambda captures in test callbacks

- Prefer explicit captures (`[&mock, &result]`) over default capture (`[&]`) when the lambda escapes the current scope or is passed to async code
- For short, non-escaping lambdas within a single test, default capture by reference is acceptable

```cpp
// OK — short lambda, clearly bounded scope
EXPECT_CALL(mock, Process).WillOnce([&](int x) { return x * 2; });

// Prefer explicit capture when lambda could outlive local variables
auto callback = [&result, &latch](Status s) {
    result = s;
    latch.CountDown();
};
```

## Common Compilation Errors

| Error | Fix |
|-------|-----|
| `undefined reference to 'testing::...'` | Link `GTest::gtest` and `GTest::gtest_main` to the test target |
| `undefined reference to 'MyFunc'` | Link the library under test to the test target |
| `fatal error: gtest/gtest.h: No such file` | Ensure `FetchContent_MakeAvailable(googletest)` or `find_package(GTest)` is called |
| `error: 'TEST_F' ... not a class` | The fixture class must inherit from `::testing::Test` |
| `error: use of deleted function 'MockFoo()'` | Add a default constructor or use `NiceMock<MockFoo>` |
| `mock object leaked` | Ensure mock is destroyed before test ends (use local scope or smart pointers) |
| `Uninteresting mock function call` | Use `NiceMock<>` or add `EXPECT_CALL`/`ON_CALL` |
| `MOCK_METHOD` parse error with templates | Wrap types containing commas in parentheses: `MOCK_METHOD((std::pair<int,int>), Func, ())` |
| `Actual function call count doesn't match` | Verify expected call count with `.Times(n)` or adjust logic |
| `fatal error C1083: 'CppUnitTest.h'` | Add `$(VCInstallDir)Auxiliary\VS\UnitTest\include` to Include Directories |
| `LNK2019: unresolved external symbol` (test framework) | Add `$(VCInstallDir)Auxiliary\VS\UnitTest\lib` to Library Directories |
| `LNK2019: unresolved external symbol` (code under test) | Add project reference, or add `.obj`/`.lib` to Linker → Additional Dependencies |
| `LNK2005: _main already defined` (Boost.Test) | Boost.Test defines `main` — remove or exclude your own `main.cpp` from the test build config |
| `fatal error: boost/test/unit_test.hpp: No such file` | Run `vcpkg install boost-test` and `vcpkg integrate install`, or add Boost include path manually |
