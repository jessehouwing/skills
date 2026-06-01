---
name: run-cpp-tests
description: >
  For C++ test projects: detects the build system (CMake/CTest, MSBuild/vcxproj,
  Bazel) and test framework (GoogleTest, Microsoft Native Test Framework), then
  picks the matching build and test commands. USE FOR: running C++ tests, filtering
  GoogleTest tests with --gtest_filter, running CTest with -R filters, running
  vstest.console.exe for native test DLLs, choosing the right command for the
  detected toolchain, troubleshooting test discovery failures, running a subset
  of tests by suite or name pattern.
  DO NOT USE FOR: writing/generating test code (use the code-testing pipeline
  agents), CI/CD config, debugging failing test logic, or .NET tests (use
  run-tests for dotnet test).
license: MIT
---

# Run C++ Tests

Detect the build system and test framework, build test binaries, and run tests with appropriate commands and filters.

## When to Use

- User wants to run C++ tests in a project
- User needs to run a subset of tests using name filters
- User needs help detecting which build system or test framework is in use
- User wants to understand the correct filter syntax for their C++ test setup

## When Not to Use

- User needs to write or generate test code (use the code-testing pipeline agents)
- User wants to run .NET tests (use `run-tests`)
- User needs CI/CD pipeline configuration
- User needs to debug a failing test

## Inputs

| Input | Required | Description |
|-------|----------|-------------|
| Project path | No | Path to the project root. Defaults to current directory. |
| Filter expression | No | Test name pattern to select specific tests |
| Build configuration | No | Debug or Release. Defaults to Debug. |

## Critical Rules

| Rule | Why |
|------|-----|
| **Always build before running** | C++ tests must be compiled first — there's no equivalent of `dotnet test` that builds and runs in one step for CMake |
| **Do NOT use `--gtest_filter` with CTest** | CTest uses `-R` for regex name filters; `--gtest_filter` is for direct binary execution only |
| **Do NOT mix CTest and vstest** | CMake projects use CTest; vcxproj projects use vstest.console.exe or direct binary |
| **Check CMakePresets.json first** | If presets exist, use `cmake --preset` instead of manual `-S . -B build` |
| **Platform must match** | On Windows vcxproj, ensure x64/x86 matches between build and test commands |

## Workflow

### Step 1: Detect the build system and framework

**Detection files to check** (in order): `CMakeLists.txt` → `CMakePresets.json` → `*.sln` / `*.vcxproj` → `BUILD.bazel` → `Makefile`

| Signal | Build System | Test Runner |
|--------|-------------|-------------|
| `CMakeLists.txt` with `gtest_discover_tests()` or `add_test()` | CMake | CTest |
| `CMakeLists.txt` with `FetchContent_Declare(googletest ...)` | CMake + GoogleTest | CTest or direct binary |
| `*.vcxproj` with GoogleTest NuGet or adapter | MSBuild | vstest.console.exe or direct binary |
| `*.vcxproj` with `CppUnitTestFramework` | MSBuild | vstest.console.exe |
| `BUILD.bazel` with `@googletest` | Bazel | `bazel test` |
| `Makefile` with gtest link flags | Make | Direct binary |

**Framework detection in source files:**

| Include | Framework |
|---------|-----------|
| `#include <gtest/gtest.h>` | GoogleTest |
| `#include <gmock/gmock.h>` | GoogleTest + GoogleMock |
| `#include "CppUnitTest.h"` | Microsoft Native Test Framework |

### Step 2: Build the test binary

#### CMake

```bash
# With presets (preferred)
cmake --preset debug
cmake --build --preset debug

# Without presets
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j

# Build only the test target
cmake --build build --target my_tests -j
```

#### MSBuild (vcxproj)

```bash
msbuild MySolution.sln /p:Configuration=Debug /p:Platform=x64
# Or via dotnet:
dotnet build MySolution.sln -c Debug
```

#### Bazel

```bash
bazel build //path/to:my_test
```

### Step 3: Run tests

#### CTest (CMake projects)

| Scope | Command |
|-------|---------|
| All tests | `ctest --test-dir build --output-on-failure` |
| Filter by name regex | `ctest --test-dir build -R "CalculatorTest" --output-on-failure` |
| Exclude by name regex | `ctest --test-dir build -E "SlowTest" --output-on-failure` |
| Verbose output | `ctest --test-dir build -V` |
| Parallel execution | `ctest --test-dir build -j8 --output-on-failure` |
| List all tests | `ctest --test-dir build -N` |

#### Direct GoogleTest binary

| Scope | Command |
|-------|---------|
| All tests | `./build/my_tests` |
| Filter by pattern | `./build/my_tests --gtest_filter=SuiteName.TestName` |
| Multiple patterns | `./build/my_tests --gtest_filter=Suite1.*:Suite2.Specific` |
| Exclude pattern | `./build/my_tests --gtest_filter=-Suite.Slow*` |
| List all tests | `./build/my_tests --gtest_list_tests` |
| Repeat to find flakes | `./build/my_tests --gtest_repeat=10 --gtest_break_on_failure` |
| Shuffle order | `./build/my_tests --gtest_shuffle` |
| XML output | `./build/my_tests --gtest_output=xml:results.xml` |

#### vstest.console.exe (vcxproj projects)

| Scope | Command |
|-------|---------|
| All tests | `vstest.console.exe x64\Debug\MyTests.dll` |
| Filter by name | `vstest.console.exe MyTests.dll /TestCaseFilter:"Name~CalculatorTest"` |
| Filter by trait | `vstest.console.exe MyTests.dll /TestCaseFilter:"Priority=1"` |
| List tests | `vstest.console.exe MyTests.dll /ListTests` |
| TRX output | `vstest.console.exe MyTests.dll /Logger:trx` |

#### Bazel

| Scope | Command |
|-------|---------|
| Run test target | `bazel test //path/to:my_test` |
| With filter | `bazel test //path/to:my_test --test_filter=SuiteName.TestName` |
| All tests | `bazel test //...` |

### Step 4: Run filtered tests

**GoogleTest filter syntax** (for direct binary or `--test_filter` in Bazel):

| Pattern | Meaning |
|---------|---------|
| `SuiteName.*` | All tests in a suite |
| `SuiteName.TestName` | Exact test |
| `*Foo*` | Any test containing "Foo" |
| `Suite1.*:Suite2.*` | Multiple patterns (OR) |
| `-SlowSuite.*` | Exclude pattern |
| `Suite.*-Suite.Slow` | Include suite, exclude one test |
| `*/*` | Parameterized tests (slash separates instantiation) |

**CTest filter** uses POSIX regex with `-R` (include) and `-E` (exclude):

```bash
# Run tests whose CTest name contains "Calculator"
ctest --test-dir build -R "Calculator" --output-on-failure

# Exclude integration tests
ctest --test-dir build -E "Integration" --output-on-failure

# Combine include and exclude
ctest --test-dir build -R "Unit" -E "Slow" --output-on-failure
```

**vstest filter** uses `TestCaseFilter` expressions:

```bash
# By name pattern
vstest.console.exe MyTests.dll /TestCaseFilter:"Name~Calculator"

# By trait (MS Native Test Framework)
vstest.console.exe MyTests.dll /TestCaseFilter:"Priority=1"

# Combined
vstest.console.exe MyTests.dll /TestCaseFilter:"Name~Calculator&Priority=1"
```

## Validation

- [ ] Build system was correctly identified (CMake, MSBuild, Bazel, Make)
- [ ] Test framework was correctly identified (GoogleTest, MS Native, both)
- [ ] Test binary was built before attempting to run
- [ ] Correct runner was used for the detected build system
- [ ] Filter syntax matched the runner (not mixing --gtest_filter with CTest)
- [ ] Test results were clearly reported to the user

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Running CTest without building first | Always `cmake --build` before `ctest` |
| Using `--gtest_filter` with CTest | CTest uses `-R` for name regex filtering |
| Tests not discovered by CTest | Ensure `gtest_discover_tests(target)` is in CMakeLists.txt (not just `add_executable`) |
| Wrong platform in vstest | Match x64/x86 between build configuration and DLL path |
| Binary not found after build | Check build output directory — CMake uses `build/`, vcxproj uses `x64/Debug/` |
| `gtest_discover_tests` fails at configure | Use `DISCOVERY_MODE PRE_TEST` to defer discovery until test execution |
| Tests pass individually but fail together | Use `--gtest_shuffle` to detect ordering dependencies |
| Parameterized test filter not matching | Use `*/*` pattern — parameterized tests have format `InstantiationName/SuiteName.TestName/ParamIndex` |

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `No tests were found` (vstest) | Test adapter not installed or DLL is wrong platform | Ensure Google Test Adapter is enabled in VS; verify platform matches |
| `ctest: No tests found` | Tests not registered with CTest | Add `gtest_discover_tests(target)` or `add_test()` to CMakeLists.txt |
| `error while loading shared libraries: libgtest.so` | GoogleTest built as shared but not in library path | Set `LD_LIBRARY_PATH` or build with `-DBUILD_SHARED_LIBS=OFF` |
| Binary crashes without output | Test crashes in fixture setup | Run with `--gtest_break_on_failure` under debugger, or use `--gtest_output=xml:` to capture partial results |
| `unknown flag --gtest_filter` | Binary not linked to gtest_main, or custom main doesn't call `::testing::InitGoogleTest` | Ensure `InitGoogleTest(&argc, argv)` is called in main, or link `GTest::gtest_main` |
| All tests skipped | `GTEST_SKIP()` called in `SetUpTestSuite` | Check fixture's static setup for early skip conditions |
