---
name: filter-syntax
description: "Reference data for test filter syntax across all platform and framework combinations: VSTest --filter expressions, MTP filters for MSTest/NUnit/xUnit v3/TUnit, GoogleTest adapter filters via vstest, Microsoft Native Test Framework filters, and VSTest-to-MTP filter translation. DO NOT USE directly — loaded by run-tests, run-cpp-tests, mtp-hot-reload, and migrate-vstest-to-mtp when they need filter syntax."
user-invocable: false
license: MIT
---

# Test Filter Syntax Reference

Filter syntax depends on the **platform** and **test framework**.

## VSTest filters (MSTest, xUnit v2, NUnit on VSTest)

```bash
dotnet test --filter <EXPRESSION>
```

Expression syntax: `<Property><Operator><Value>[|&<Expression>]`

**Operators:**

| Operator | Meaning |
|----------|---------|
| `=` | Exact match |
| `!=` | Not exact match |
| `~` | Contains |
| `!~` | Does not contain |

**Combinators:** `|` (OR), `&` (AND). Parentheses for grouping: `(A|B)&C`

**Supported properties by framework:**

| Framework | Properties |
|-----------|-----------|
| MSTest | `FullyQualifiedName`, `Name`, `ClassName`, `Priority`, `TestCategory` |
| xUnit | `FullyQualifiedName`, `DisplayName`, `Traits` |
| NUnit | `FullyQualifiedName`, `Name`, `Priority`, `TestCategory` |

An expression without an operator is treated as `FullyQualifiedName~<value>`.

**Examples (VSTest):**

```bash
# Run tests whose name contains "LoginTest"
dotnet test --filter "Name~LoginTest"

# Run a specific test class
dotnet test --filter "ClassName=MyNamespace.MyTestClass"

# Run tests in a category
dotnet test --filter "TestCategory=Integration"

# Exclude a category
dotnet test --filter "TestCategory!=Slow"

# Combine: class AND category
dotnet test --filter "ClassName=MyNamespace.MyTestClass&TestCategory=Unit"

# Either of two classes
dotnet test --filter "ClassName=MyNamespace.ClassA|ClassName=MyNamespace.ClassB"
```

## MTP filters — MSTest and NUnit

MSTest and NUnit on MTP use the **same `--filter` syntax** as VSTest (same properties, operators, and combinators). The only difference is how the flag is passed:

```bash
# .NET SDK 8/9 (after --)
dotnet test -- --filter "Name~LoginTest"

# .NET SDK 10+ (direct)
dotnet test --filter "Name~LoginTest"
```

## MTP filters — xUnit (v3)

xUnit v3 on MTP uses **framework-specific filter flags** instead of the generic `--filter` expression:

| Flag | Description |
|------|-------------|
| `--filter-class "name"` | Run all tests in a given class |
| `--filter-not-class "name"` | Exclude all tests in a given class |
| `--filter-method "name"` | Run a specific test method |
| `--filter-not-method "name"` | Exclude a specific test method |
| `--filter-namespace "name"` | Run all tests in a namespace |
| `--filter-not-namespace "name"` | Exclude all tests in a namespace |
| `--filter-trait "name=value"` | Run tests with a matching trait |
| `--filter-not-trait "name=value"` | Exclude tests with a matching trait |

Multiple values can be specified with a single flag: `--filter-class Foo Bar`.

```bash
# .NET SDK 8/9
dotnet test -- --filter-class "MyNamespace.LoginTests"

# .NET SDK 10+
dotnet test --filter-class "MyNamespace.LoginTests"

# Combine: namespace + trait
dotnet test --filter-namespace "MyApp.Tests.Integration" --filter-trait "Category=Smoke"
```

### xUnit v3 query filter language

For complex expressions, use `--filter-query` with a path-segment syntax:

```text
/<assemblyFilter>/<namespaceFilter>/<classFilter>/<methodFilter>[traitName=traitValue]
```

Each segment matches against: assembly name, namespace, class name, method name. Use `*` for "match all" in any segment. Documentation: <https://xunit.net/docs/query-filter-language>

```shell
# xUnit.net v3 MTP — using query language (assembly/namespace/class/method[trait])
dotnet test -- --filter-query "/*/*/*IntegrationTests*/*[Category=Smoke]"
```

## MTP filters — TUnit

TUnit uses `--treenode-filter` with a path-based syntax:

```text
--treenode-filter "/<Assembly>/<Namespace>/<ClassName>/<TestName>"
```

Wildcards (`*`) are supported in any segment. Filter operators can be appended to test names for property-based filtering.

| Operator | Meaning |
|----------|---------|
| `*` | Wildcard match |
| `=` | Exact property match (e.g., `[Category=Unit]`) |
| `!=` | Exclude property value |
| `&` | AND (combine conditions) |
| `\|` | OR (within a segment, requires parentheses) |

**Examples (TUnit):**

```bash
# All tests in a class
dotnet run --treenode-filter "/*/*/LoginTests/*"

# A specific test
dotnet run --treenode-filter "/*/*/*/AcceptCookiesTest"

# By namespace prefix (wildcard)
dotnet run --treenode-filter "/*/MyProject.Tests.Api*/*/*"

# By custom property
dotnet run --treenode-filter "/*/*/*/*[Category=Smoke]"

# Exclude by property
dotnet run --treenode-filter "/*/*/*/*[Category!=Slow]"

# OR across classes
dotnet run --treenode-filter "/*/*/(LoginTests)|(SignupTests)/*"

# Combined: namespace + property
dotnet run --treenode-filter "/*/MyProject.Tests.Integration/*/*/*[Priority=Critical]"
```

## VSTest → MTP filter translation (for migration)

**MSTest, NUnit, and xUnit.net v2 (with `YTest.MTP.XUnit2`)**: The VSTest `--filter` syntax is identical on both VSTest and MTP. No changes needed.

**xUnit.net v3 (native MTP)**: xUnit.net v3 does NOT support the VSTest `--filter` syntax on MTP. Translate filters using xUnit.net v3's native options:

| VSTest `--filter` syntax | xUnit.net v3 MTP equivalent | Notes |
|---|---|---|
| `FullyQualifiedName~ClassName` | `--filter-class *ClassName*` | Wildcards required for substring match |
| `FullyQualifiedName=Ns.Class.Method` | `--filter-method Ns.Class.Method` | Exact match on fully qualified method |
| `Name=MethodName` | `--filter-method *MethodName*` | Wildcards for substring match |
| `Category=Value` (trait) | `--filter-trait "Category=Value"` | Filter by trait name/value pair |
| Complex expressions | `--filter-query "expr"` | Uses xUnit.net query filter language (see above) |

## VSTest filters — GoogleTest (via Google Test Adapter)

When running C++ GoogleTest tests through `vstest.console.exe` or `dotnet test` (via the Google Test Adapter included with "Desktop development with C++" workload), tests are exposed as vstest test cases using standard `TestCaseFilter` syntax.

### Test name mapping

GoogleTest names map to vstest properties as follows:

| GoogleTest concept | vstest property | Example value |
|---|---|---|
| `TestSuiteName.TestName` | `FullyQualifiedName` | `CalculatorTest.AddsPositiveNumbers` |
| `TestName` | `Name` | `AddsPositiveNumbers` |
| `TestSuiteName` | `ClassName` | `CalculatorTest` |
| Parameterized: `Prefix/Suite.Test/N` | `FullyQualifiedName` | `MyInstantiation/ParamTest.Works/0` |
| `TEST_F` fixture | `ClassName` | `CalculatorFixture` |

### Filter syntax

```bash
# Run all tests in a suite
vstest.console.exe MyTests.dll /TestCaseFilter:"ClassName=CalculatorTest"

# Run a specific test
vstest.console.exe MyTests.dll /TestCaseFilter:"FullyQualifiedName=CalculatorTest.AddsPositiveNumbers"

# Substring match (any test containing "Add")
vstest.console.exe MyTests.dll /TestCaseFilter:"Name~Add"

# Exclude a suite
vstest.console.exe MyTests.dll /TestCaseFilter:"ClassName!=SlowTests"

# Combine: suite AND name pattern
vstest.console.exe MyTests.dll /TestCaseFilter:"ClassName=CalculatorTest&Name~Negative"

# Via dotnet test (solution containing vcxproj with GoogleTest)
dotnet test MySolution.sln --filter "ClassName=CalculatorTest"
dotnet test MySolution.sln --filter "Name~Boundary"
```

### Supported filter properties

| Property | Description | Operators |
|----------|-------------|----------|
| `FullyQualifiedName` | `SuiteName.TestName` (or `Prefix/Suite.Test/Param`) | `=`, `!=`, `~`, `!~` |
| `Name` | Test name only (without suite) | `=`, `!=`, `~`, `!~` |
| `ClassName` | Suite/fixture name | `=`, `!=`, `~`, `!~` |
| `TestCategory` | Mapped from `TYPED_TEST`, `TEST_P` instantiation name, or custom traits (adapter-dependent) | `=`, `!=` |

### GoogleTest adapter configuration

The adapter can be configured via Tools → Options → Test Adapter for Google Test, or via a `.runsettings` file:

```xml
<RunSettings>
  <GoogleTestAdapterSettings>
    <SolutionSettings>
      <Settings>
        <TestDiscoveryRegex>.*[Tt]est.*\.exe</TestDiscoveryRegex>
        <WorkingDir>$(SolutionDir)</WorkingDir>
        <AdditionalTestExecutionParams>--gtest_shuffle</AdditionalTestExecutionParams>
      </Settings>
    </SolutionSettings>
  </GoogleTestAdapterSettings>
</RunSettings>
```

Use with: `vstest.console.exe MyTests.dll /Settings:my.runsettings /TestCaseFilter:"Name~Integration"`

### Microsoft Native Test Framework filters (vstest)

Microsoft Native Test Framework tests (CppUnitTest.h) also use vstest `TestCaseFilter` syntax:

```bash
# By method name
vstest.console.exe NativeTests.dll /TestCaseFilter:"Name~Calculator"

# By class (TEST_CLASS name)
vstest.console.exe NativeTests.dll /TestCaseFilter:"ClassName=CalculatorTests"

# By trait (from BEGIN_TEST_METHOD_ATTRIBUTE)
vstest.console.exe NativeTests.dll /TestCaseFilter:"Priority=1"
vstest.console.exe NativeTests.dll /TestCaseFilter:"Owner=TeamA"

# Combine
vstest.console.exe NativeTests.dll /TestCaseFilter:"ClassName=CalculatorTests&Priority=1"
```

**Trait properties for MS Native tests:**

| Trait macro | Filter property |
|-------------|----------------|
| `TEST_OWNER(L"name")` | `Owner` |
| `TEST_PRIORITY(n)` | `Priority` |
| `TEST_METHOD_ATTRIBUTE(L"Category", L"value")` | `Category` |
| `TEST_METHOD_ATTRIBUTE(L"key", L"value")` | `key` |
