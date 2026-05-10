---
name: write-unit-tests
description: Use when writing unit tests for any feature or change - discovers the test framework and patterns from the codebase, uses git diffs to identify what needs coverage
---

# Writing Unit Tests

## Overview

Discover context from the codebase first, then write tests that match existing patterns. Never assume the stack - read it.

## Step 1: Discover What Needs Testing

Run these before writing a single test:

```bash
# See what changed
git diff HEAD~1 --stat
git diff HEAD~1 -- <relevant files>

# Or for staged/unstaged changes
git diff --cached
git diff
```

Read the changed files to understand the new logic, edge cases, and failure modes.

## Step 2: Discover the Test Setup

```bash
# Find test framework config
ls pytest.ini pyproject.toml setup.cfg jest.config.* vitest.config.* package.json 2>/dev/null

# Find existing tests
find . -name "test_*.py" -o -name "*_test.py" -o -name "*.test.ts" -o -name "*.spec.ts" | head -20

# Find fixtures/helpers
find . -name "conftest.py" -o -name "fixtures.*" -o -name "helpers.*" | head -10
```

Read 1-2 existing test files and the fixture setup to understand:
- Naming conventions
- How test data is built (factories, fixtures, inline)
- How external dependencies are mocked
- How the test client / runner is set up
- Assertion style

## Step 3: Identify Coverage Gaps

For each changed function/method/endpoint, ask:
- Happy path covered?
- What are the distinct failure modes? (bad input, missing auth, dependency down, edge cases)
- What invariants must hold regardless of input?
- Are there boundary conditions (empty, zero, max, expired)?

## Step 4: Write Tests That Match Existing Patterns

Mirror the style of existing tests exactly:
- Same import structure
- Same fixture usage
- Same factory/builder pattern for test data
- Same assertion style
- Same file location and naming

**Class-based grouping** (if existing tests use it):
```python
class Test<Feature>:
    def test_<happy_path>(self): ...
    def test_<error_case>(self): ...
```

**Flat functions** (if existing tests use it):
```python
def test_<feature>_<scenario>(): ...
```

## Coverage Checklist

For any changed code, write tests for:
- [ ] Expected output for valid input
- [ ] Each distinct error/exception path
- [ ] Auth/permission failures (if applicable)
- [ ] External dependency failures (network down, DB error)
- [ ] Boundary values (empty string, zero, null, expired timestamps)
- [ ] Any logic branch not covered by the above

## Running Tests

Discover the command from config, then run:

```bash
# Python
pytest                        # all
pytest path/to/test_file.py  # one file
pytest -k "test_name"        # by pattern

# Node
npm test
npx vitest
npx jest

# Verify new tests pass and nothing else broke
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Writing tests before reading the diff | Read the diff first to know what you're testing |
| Ignoring existing test patterns | Read 1-2 existing tests and mirror them exactly |
| Testing implementation details | Test behavior and outcomes, not internal calls |
| Patching at the wrong module path | Patch where the name is used, not where it is defined |
| Not testing the failure path | Every external call can fail - test that too |
| Skipping boundary cases | Empty, null, zero, expired, max-length inputs break things |
