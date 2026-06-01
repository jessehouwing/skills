# C++ Extension

Language-specific guidance for C++ test generation using GoogleTest (gtest) and GoogleMock (gmock) with CMake/CTest or Visual Studio (vcxproj/MSBuild).

## Rule #1: Investigate the Repo First

Before writing any test or running any command, read:

1. **Existing tests** — find `*_test.cpp`, `*_test.cc`, `*Test.cpp` files and copy their style (fixture usage, mock patterns, assertion style)
2. **`CMakeLists.txt`** — look for `FetchContent_Declare(googletest ...)`, `find_package(GTest)`, or `add_subdirectory(googletest)` to understand how gtest is integrated
3. **`*.vcxproj` / `*.sln`** — Visual Studio projects; check for GoogleTest NuGet packages or Microsoft Native Test Framework references
4. **Build/CI scripts** — `Makefile`, `CMakePresets.json`, `.github/workflows/*.yml`, `ci/` directory
5. **`conanfile.txt` / `vcpkg.json`** — package manager integration for gtest

Use whatever test style and layout the repo already uses. Do not introduce a different test framework if the repo already uses GoogleTest.

## Toolchain Detection

| Indicator | Meaning |
|-----------|---------|
| `CMakeLists.txt` with `FetchContent_Declare(googletest ...)` | GoogleTest fetched at configure time (CMake build) |
| `find_package(GTest REQUIRED)` | GoogleTest installed system-wide or via package manager (CMake) |
| `*.vcxproj` with `Microsoft.VisualStudio.TestTools.CppUnitTestFramework` | Microsoft Native Test Framework (MSBuild) |
| `*.vcxproj` with GoogleTest NuGet package or `packages.config` referencing `googletest` | GoogleTest via Visual Studio (MSBuild) |
| `*.sln` containing a test project | Visual Studio solution — use `msbuild` or `dotnet build` |
| `vcpkg.json` with `gtest` | vcpkg dependency management |
| `vcpkg.json` with `boost-test` | Boost.Test via vcpkg |
| `#include <boost/test/unit_test.hpp>` or `BOOST_TEST_MODULE` | Boost.Test framework |
| `conanfile.txt` / `conanfile.py` with `gtest` | Conan dependency management |
| `CMakePresets.json` | Build presets — use them for configure/build commands |
| `BUILD.bazel` with `@googletest` | Bazel build system (use `bazel test` instead of CTest) |

## Build Commands (CMake)

| Scope | Command |
|-------|---------|
| Configure (first time) | `cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug` |
| Build all | `cmake --build build -j` |
| Build specific target | `cmake --build build --target my_tests -j` |
| Reconfigure after CMakeLists change | `cmake --build build -j` (auto-reconfigures) |
| Clean build | `cmake --build build --clean-first` |

If the project uses `CMakePresets.json`, prefer preset-based commands:

```bash
cmake --preset debug
cmake --build --preset debug
```

## Build Commands (Visual Studio / MSBuild)

For `.vcxproj` / `.sln` projects (Windows, Visual Studio):

| Scope | Command |
|-------|---------|
| Build solution | `msbuild MySolution.sln /p:Configuration=Debug /p:Platform=x64` |
| Build specific project | `msbuild Tests/MyTests.vcxproj /p:Configuration=Debug /p:Platform=x64` |
| Build via dotnet | `dotnet build MySolution.sln -c Debug` |
| Rebuild (clean + build) | `msbuild MySolution.sln /t:Rebuild /p:Configuration=Debug` |
| Restore NuGet packages | `msbuild /t:Restore` or `nuget restore MySolution.sln` |

**Notes:**
- Ensure the Platform (x64/x86/ARM64) matches the test project configuration
- If GoogleTest is via NuGet, packages restore automatically on build
- Use Developer Command Prompt or `vcvarsall.bat` to set up the environment for `msbuild`

## Test Commands (CMake / CTest)

| Scope | Command |
|-------|---------|
| All tests | `ctest --test-dir build --output-on-failure` |
| Single test suite | `ctest --test-dir build -R "CalculatorTest" --output-on-failure` |
| Single test | `ctest --test-dir build -R "CalculatorTest.AddsTwoNumbers" --output-on-failure` |
| Verbose output | `ctest --test-dir build -V` |
| Direct gtest binary | `./build/my_tests --gtest_filter=SuiteName.TestName` |
| Multiple filters | `./build/my_tests --gtest_filter=Suite1.*:Suite2.Specific` |
| Exclude pattern | `./build/my_tests --gtest_filter=-Suite.Slow*` |
| List all tests | `./build/my_tests --gtest_list_tests` |
| Repeat to catch flakes | `./build/my_tests --gtest_repeat=10 --gtest_break_on_failure` |

## Test Commands (Visual Studio / vstest)

For `.vcxproj`-based test projects, use `vstest.console.exe` or `dotnet test`:

| Scope | Command |
|-------|---------|
| All tests in project | `vstest.console.exe x64\Debug\MyTests.dll` |
| All tests via dotnet | `dotnet test MySolution.sln -c Debug` |
| Filter by test name | `vstest.console.exe MyTests.dll /TestCaseFilter:"Name~CalculatorTest"` |
| Filter via dotnet | `dotnet test --filter "Name~CalculatorTest"` |
| List discovered tests | `vstest.console.exe MyTests.dll /ListTests` |
| Run with logger | `vstest.console.exe MyTests.dll /Logger:trx` |

**Notes:**
- `vstest.console.exe` is in the Visual Studio installation (e.g., `C:\Program Files\Microsoft Visual Studio\2022\Enterprise\Common7\IDE\Extensions\TestPlatform\`)
- GoogleTest and Microsoft Native Test Framework tests are both discoverable via VS Test Explorer and `vstest.console.exe`
- Test DLLs are output to `$(Configuration)\$(Platform)\` or `x64\Debug\` by default
- The Google Test Adapter is included with the "Desktop development with C++" workload in VS 2017+

## Lint / Format Command

| Tool | Command |
|------|---------|
| clang-format | `clang-format -i path/to/file.cpp` |
| cmake-format | `cmake-format -i CMakeLists.txt` |
| clang-tidy | `clang-tidy path/to/file.cpp -- -I include/` |
| Whole project (if `.clang-format` exists) | `find tests/ -name '*.cpp' \| xargs clang-format -i` |

## Project Layout (CMake)

Typical directory structures for C++ projects with gtest:

```
project/
├── CMakeLists.txt         # Root CMake — includes tests/
├── src/
│   ├── calculator.cpp
│   └── calculator.h
├── include/               # Public headers (optional)
├── tests/
│   ├── CMakeLists.txt     # Test target definitions
│   ├── calculator_test.cpp
│   └── service_test.cpp
└── build/                 # Out-of-source build (gitignored)
```

Alternative layouts:
- `test/` instead of `tests/`
- `*_test.cc` instead of `*_test.cpp`
- Test files colocated with sources (less common in C++)

## Project Layout (Visual Studio / MSBuild)

Typical directory structure for `.vcxproj`-based projects:

```
MySolution/
├── MySolution.sln
├── MyApp/
│   ├── MyApp.vcxproj
│   ├── calculator.cpp
│   └── calculator.h
├── MyApp.Tests/
│   ├── MyApp.Tests.vcxproj     # Test project — references MyApp
│   ├── calculator_test.cpp
│   └── pch.h                   # Precompiled header (optional)
└── packages/                    # NuGet packages (GoogleTest, etc.)
```

**Key setup for vcxproj test projects:**
- Add a project reference to the code under test (right-click References → Add Reference)
- Or link to `.obj` / `.lib` files from the production project
- Add the source project's header directory to **Additional Include Directories** (`Project > Properties > C/C++ > General`)
- For GoogleTest: install via NuGet (`Microsoft.googletest.v140.windesktop.msvcstl.static.rt-dyn`) or use the VS project template ("Google Test" under Add > New Project)
- For Microsoft Native Test Framework: use the "Native Unit Test Project" template

## CMake Test Registration

When adding a new test file, register it in the test `CMakeLists.txt`:

```cmake
# tests/CMakeLists.txt
add_executable(my_tests
  calculator_test.cpp
  service_test.cpp      # ← add new test source here
)
target_link_libraries(my_tests
  GTest::gtest
  GTest::gmock
  GTest::gtest_main     # Provides main() — no need to write your own
  my_library            # Link against the library under test
)

include(GoogleTest)
gtest_discover_tests(my_tests)
```

If the project uses a single test binary, simply add the new `.cpp` file to the existing `add_executable` source list. If the project uses multiple test binaries, follow the existing pattern.

## CMake FetchContent Setup (New Projects)

For projects that don't yet have GoogleTest:

```cmake
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.17.0.zip
)
# Prevent installing GoogleTest with the main project
set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)
```

## Test Patterns

For detailed test-writing guidance including fixtures, parameterized tests, mocking, matchers, actions, and expectation ordering, see the **`writing-cpp-tests`** skill. Below is a minimal quick-reference.

### Quick Reference: Basic Test

```cpp
#include <gtest/gtest.h>
#include "calculator.h"

TEST(CalculatorTest, AddsTwoPositiveNumbers) {
    Calculator calc;
    EXPECT_EQ(calc.Add(2, 3), 5);
}
```

### Quick Reference: Fixture

```cpp
class UserStoreTest : public ::testing::Test {
protected:
    void SetUp() override { store = std::make_unique<UserStore>(":memory:"); }
    std::unique_ptr<UserStore> store;
};

TEST_F(UserStoreTest, FindsExistingUser) {
    EXPECT_TRUE(store->Find("alice").has_value());
}
```

### Quick Reference: Mock

```cpp
class MockNotifier : public Notifier {
public:
    MOCK_METHOD(void, Send, (const std::string& message), (override));
};

TEST(ServiceTest, PublishSendsNotification) {
    MockNotifier notifier;
    Service service(notifier);
    EXPECT_CALL(notifier, Send("hello")).Times(1);
    service.Publish("hello");
}
```

## Microsoft Native Test Framework (vcxproj)

For detailed MS Native Test Framework patterns (assertions, lifecycle, traits, DLL testing strategies), see the **`writing-cpp-tests`** skill.

### Quick Setup

1. Add a **Native Unit Test Project** to the solution (Add → New Project → "Native Unit Test Project")
2. Add a **project reference** to the code under test
3. Add the source project's header directory to **Additional Include Directories**

### Quick Reference: Basic Test

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
        TEST_METHOD(AddsTwoPositiveNumbers)
        {
            Calculator calc;
            Assert::AreEqual(5, calc.Add(2, 3));
        }
    };
}
```

### Google Test in Visual Studio (vcxproj)

Google Test is integrated into VS as part of the "Desktop development with C++" workload.

**Setup via project template:**
1. Right-click solution → Add → New Project → "Google Test Project"
2. Choose the project to test and select static or dynamic linking to gtest binaries

**Setup via NuGet:**
1. Create a new empty C++ project
2. Manage NuGet Packages → install a GoogleTest package (e.g., `Microsoft.googletest.v140.windesktop.msvcstl.static.rt-dyn`)
3. Add project reference to the code under test

### When to Use Which Framework (vcxproj)

| Framework | Use when |
|-----------|----------|
| Microsoft Native Test Framework | Simple VS-only projects, no mocking needed, team prefers MS tooling |
| GoogleTest (via NuGet/vcpkg) | Need gmock for mocking, cross-platform code, team familiar with gtest |
| GoogleTest (via CMake) | CMake-based projects opened in VS via "Open Folder" or CMake integration |

**Prefer GoogleTest** when mocking is needed (gmock has no equivalent in the MS framework) or when the project must build on non-Windows platforms.

## Assertion Quick Reference (GoogleTest)

| Category | Fatal | Non-fatal |
|----------|-------|-----------|
| Equality | `ASSERT_EQ(a, b)` | `EXPECT_EQ(a, b)` |
| Inequality | `ASSERT_NE(a, b)` | `EXPECT_NE(a, b)` |
| Comparison | `ASSERT_LT/LE/GT/GE` | `EXPECT_LT/LE/GT/GE` |
| Boolean | `ASSERT_TRUE/FALSE` | `EXPECT_TRUE/FALSE` |
| String | `ASSERT_STREQ(a, b)` | `EXPECT_STREQ(a, b)` |
| Float | `ASSERT_NEAR(a, b, tol)` | `EXPECT_NEAR(a, b, tol)` |
| Exception | `ASSERT_THROW(stmt, T)` | `EXPECT_THROW(stmt, T)` |
| Matcher | `ASSERT_THAT(val, m)` | `EXPECT_THAT(val, m)` |

- `ASSERT_*` aborts on failure (use for preconditions)
- `EXPECT_*` continues (use for multiple checks per test)

For full assertion tables, matchers, and actions reference, see the **`writing-cpp-tests`** skill.

## Testing Internals

Use a preprocessor-guarded `friend` as a last resort (prefer testing through the public API):

```cpp
class MyClass {
#ifdef UNIT_TESTING
    friend class MyClassTest;
#endif
};
```

```cmake
target_compile_definitions(my_tests PRIVATE UNIT_TESTING)
```

## Coverage

### GCC + gcov + lcov

```cmake
option(ENABLE_COVERAGE "Enable coverage flags" OFF)

if(ENABLE_COVERAGE)
  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    target_compile_options(my_tests PRIVATE --coverage)
    target_link_options(my_tests PRIVATE --coverage)
  elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    target_compile_options(my_tests PRIVATE -fprofile-instr-generate -fcoverage-mapping)
    target_link_options(my_tests PRIVATE -fprofile-instr-generate)
  endif()
endif()
```

```bash
cmake -S . -B build-cov -DENABLE_COVERAGE=ON
cmake --build build-cov -j
ctest --test-dir build-cov
lcov --capture --directory build-cov --output-file coverage.info
lcov --remove coverage.info '/usr/*' '*/googletest/*' --output-file coverage.info
genhtml coverage.info --output-directory coverage
```

### Clang + llvm-cov

```bash
cmake -S . -B build-llvm -DENABLE_COVERAGE=ON -DCMAKE_CXX_COMPILER=clang++
cmake --build build-llvm -j
LLVM_PROFILE_FILE="build-llvm/default.profraw" ctest --test-dir build-llvm
llvm-profdata merge -sparse build-llvm/default.profraw -o build-llvm/default.profdata
llvm-cov report build-llvm/my_tests -instr-profile=build-llvm/default.profdata
```

## Sanitizers

```cmake
option(ENABLE_ASAN "Enable AddressSanitizer" OFF)
option(ENABLE_UBSAN "Enable UndefinedBehaviorSanitizer" OFF)
option(ENABLE_TSAN "Enable ThreadSanitizer" OFF)

if(ENABLE_ASAN)
  add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
  add_link_options(-fsanitize=address)
endif()
if(ENABLE_UBSAN)
  add_compile_options(-fsanitize=undefined -fno-omit-frame-pointer)
  add_link_options(-fsanitize=undefined)
endif()
if(ENABLE_TSAN)
  add_compile_options(-fsanitize=thread)
  add_link_options(-fsanitize=thread)
endif()
```

- ASan and TSan cannot be combined in the same binary
- Run sanitizer builds in CI for early detection of memory/race issues

## Common Errors

### GoogleTest / CMake errors

| Error | Fix |
|-------|-----|
| `undefined reference to 'testing::...'` | Link `GTest::gtest` and `GTest::gtest_main` to the test target |
| `undefined reference to 'MyFunc'` | Link the library under test to the test target |
| `fatal error: gtest/gtest.h: No such file` | Ensure `FetchContent_MakeAvailable(googletest)` or `find_package(GTest)` is called |
| `error: 'TEST_F' ... not a class` | The fixture class must inherit from `::testing::Test` |
| `error: use of deleted function 'MockFoo()'` | Add a default constructor or use `NiceMock<MockFoo>` |
| `mock object leaked` | Ensure mock object is destroyed before test ends (use local scope or pointers) |
| `Uninteresting mock function call` | Use `NiceMock<>` or add an `EXPECT_CALL`/`ON_CALL` for the method |
| `MOCK_METHOD` parse error with templates | Wrap types containing commas in parentheses: `MOCK_METHOD((std::pair<int,int>), Func, ())` |
| `Actual function call count doesn't match` | Verify the expected call count with `.Times(n)` or adjust logic |
| `gtest_discover_tests` fails at configure | Ensure the test binary can be built before CTest discovery runs — use `DISCOVERY_MODE PRE_TEST` |

### Visual Studio / MSBuild errors

| Error | Fix |
|-------|-----|
| `fatal error C1083: Cannot open include file: 'CppUnitTest.h'` | Add `$(VCInstallDir)Auxiliary\VS\UnitTest\include` to VC++ Include Directories |
| `LNK2019: unresolved external symbol` referencing test framework | Add `$(VCInstallDir)Auxiliary\VS\UnitTest\lib` to VC++ Library Directories |
| `LNK2019: unresolved external symbol` for code under test | Add project reference, or add `.obj`/`.lib` to Linker → Additional Dependencies |
| `LNK1120: N unresolved externals` | Downstream of LNK2019 — fix individual unresolved symbols first |
| Tests not appearing in Test Explorer | Build the test project; ensure test adapter (Google Test Adapter or MS native) is installed and enabled |
| `error C2338: Test class must be in a namespace` | Wrap `TEST_CLASS` in a `namespace { }` block |
| `cannot open file 'CppUnitTest.lib'` | Ensure Platform (x64/x86) matches in both the test project configuration and the UnitTest lib path |
| DLL functions not accessible from test project | Add `__declspec(dllexport)` to the DLL function declarations, or link `.obj` files directly |

## Flaky Test Guardrails

- Never use `sleep` for synchronization — use condition variables, latches, or futures
- Make temp directories unique per test; clean in `TearDown()`
- Use `--gtest_repeat=N --gtest_break_on_failure` to detect intermittent failures

## Best Practices

For comprehensive DO/DON'T guidelines, see the **`writing-cpp-tests`** skill.

Key points:
- Keep tests deterministic and isolated
- Prefer dependency injection over globals
- Use `ASSERT_*` for preconditions, `EXPECT_*` for multiple checks
- Link `GTest::gtest_main` — avoid writing your own `main()`
- Use `NiceMock<>` when uninteresting calls are expected
- Base class destructors must be `virtual` for mocks
