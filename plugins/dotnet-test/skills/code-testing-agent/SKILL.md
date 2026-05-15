---
name: code-testing-agent
description: >-
  Unit tests authoring for any programming language.
  Use when asked to write (generate) unit tests, improve test coverage, add test coverage, or create test files.
  Supports any language - C#, TypeScript, JavaScript, Python, Go, Rust, Java, and more.
  Produces tests that compile, pass, and follow project conventions.
  DO NOT USE FOR: running existing tests, executing dotnet test, applying
  test filters, detecting test platforms, or troubleshooting test execution
  (use run-tests for all of these); MSTest-specific assertion guidance,
  MSTest test pattern modernization, or fixing existing MSTest test code
  (use writing-mstest-tests for those).
license: MIT
---

# Code Testing Generation Skill

An AI-powered skill that generates comprehensive, workable unit tests for any programming language using a coordinated multi-agent pipeline.

## When to Use This Skill

Use this skill when you need to:

- Generate unit tests for an entire project or specific files or functions
- Improve test coverage for existing codebases
- Create test files that follow project conventions
- Write tests that actually compile and pass
- Add tests for new features or untested code

## When Not to Use

- Running or executing existing tests (use the `run-tests` skill)
- Migrating between test frameworks (use migration skills)
- Writing tests specifically for MSTest patterns (use `writing-mstest-tests`)
- Debugging failing test logic

## Step-by-Step Instructions

### Step 1: Determine the user request

Make sure you understand what user is asking and for what scope.
When the user does not express strong requirements for test style, coverage goals, or conventions, source the guidelines from [unit-test-generation.prompt.md](unit-test-generation.prompt.md). This prompt provides best practices for discovering conventions, parameterization strategies, coverage goals (aim for 80%), and language-specific patterns.

### Step 2: Determine scope and strategy

A small, self-contained request (e.g., tests for a single function or class) that you can complete without sub-agents should use the **Direct** strategy:

 - Follow the codebase conventions on test file structure, naming, style and testing approaches.
 - Reuse existing test projects and test files when possible. If the code under test already has tests, add new tests to the same file or test project. Only create a new test file when no canonical file is named or discoverable for the symbol under test.
 - Write the tests directly, then run them right away. If any test fails, read the production code, fix the assertion, and re-run before writing more tests.

You can skip rest of this skill for the Direct strategy.

For moderate or large scopes (you need to author tests for more than a single file, module, or class) proceed by calling the `code-testing-generator` agent with your test generation request:

```
task({ agent_type: "dotnet-test:code-testing-generator", name: "generator", prompt: "..." })
```

Sample prompt (make sure to pass full path to 'unit-test-generation.prompt.md' if you want to pass it as part of the prompt - as agent doesn't have access to your referencess and will not know how to find it otherwise):

```text
Generate unit tests for [path or description of what to test], following the [unit-test-generation.prompt.md](unit-test-generation.prompt.md) guidelines
```

The Test Generator will manage the entire pipeline automatically.

### Note on calling the code-testing subagents

The `code-testing` agents is a cascade of agents that will direct you to call each other. Make sure to call them as subagents (`task({ agent_type: "dotnet-test:code-testing...", ... })`)
