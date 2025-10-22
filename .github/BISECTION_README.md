# Test Timeout Bisection Guide

This directory contains configuration for bisecting test timeouts on `ubuntu-latest`.

## Problem

CI tests were timing out on `ubuntu-latest` without clear indication of which test(s) were causing the issue.

## Solution

### 1. Global Timeout
Added `timeout-minutes: 20` to the test job to ensure fast iterations during bisection.

### 2. Per-Test Timeout
Added `pytest-timeout` dependency and configured `timeout = 300` (5 minutes) in `pyproject.toml` to identify hanging tests.

### 3. Test Deselection for Bisection
Created `pytest_deselect.txt` to selectively disable tests during bisection.

## How to Use pytest_deselect.txt

The file contains one test path per line that will be deselected (skipped) during test runs:

```
# Comment out tests to re-enable them
tests/test_cli.py::test_name_to_disable
```

### Bisection Process

1. **Start with all suspected tests disabled** (currently all 21 tests using `mock_github` fixture)
2. **Run CI** - if tests pass, the issue is in one of the disabled tests
3. **Comment out half the tests** in `pytest_deselect.txt` to re-enable them
4. **Run CI again**:
   - If tests still pass → issue is in the other half (uncommented tests)
   - If tests timeout → issue is in this half (newly enabled tests)
5. **Repeat** until you narrow down to specific test(s)

### Example Bisection Steps

Starting state: 21 tests disabled (all pass)

Step 1: Enable 10 tests (comment out first 10 lines)
- If timeout → issue in those 10 tests
- If pass → issue in remaining 11 tests

Step 2: Continue bisecting the problematic subset
- Enable 5 of the 10 problematic tests
- Repeat until you identify the specific test(s)

## Current Status

All 21 tests using the `mock_github` fixture are currently disabled to verify if the mock GitHub server initialization is causing the timeout.

## Identified Potential Issues

The `ensure_mock_github()` function in `jupyter_releaser/util.py` has an infinite loop (lines 745-750) that waits for the mock GitHub server without a timeout mechanism. This could hang indefinitely if the server fails to start properly.
