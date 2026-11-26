---
title: Mocking currentDay in loot-core tests does not move the day for schedule status
date: 2026-11-26
category: test-failures
module: loot-core schedules
problem_type: test_failure
component: testing_framework
symptoms:
  - A schedule test that moves the day forward sees no auto-posted transaction for the new month
  - getStatus reports an occurrence that is already past as scheduled instead of missed
root_cause: test_isolation
resolution_type: test_fix
severity: medium
tags: [vitest, vi-mock, currentday, schedules, getstatus, test-setup]
---

# Mocking currentDay in loot-core tests does not move the day for schedule status

## Problem

`packages/loot-core/src/server/schedules/app.test.ts` mocks `#shared/months` so a test can call `vi.mocked(monthUtils.currentDay).mockReturnValue(day)`. The mock moves the day only for modules that first load after the mock is registered. `packages/loot-core/src/shared/schedules.ts` (which holds `getStatus` and `getNextDate`) is already loaded by then, so it keeps the fixed test day `2017-01-01`. A test that moves the day forward and expects the auto-post service to catch up silently tests the wrong day.

## Symptoms

- After moving the day from 2017-01-01 to 2017-02-10, a monthly auto-posted schedule advanced its `next_date` to 2017-02-01 but posted nothing for February, so the February budget showed 0 spent.
- A probe call `getStatus('2017-02-01', false, false, '7')` inside `advanceSchedulesService` returned `scheduled`, which is only possible if `getStatus` still saw 2017-01-01; with the mocked 2017-02-10 it would be `missed`.

## What Didn't Work

- Treating the 0 spent as an app defect in the auto-post service. Tracing the service showed it advanced the schedule correctly with the mocked day while `getStatus` used the real test day, so the two halves disagreed only inside the test.

## Solution

Keep `currentDay()` at the fixed test day (2017-01-01) and put the earlier month in the past instead of moving today forward: create the schedule, then rewind its `schedules_next_date` row to the first due date, as the catch-up tests in `app.test.ts` do. Opening the budget on 2017-01-01 then posts every missed occurrence. The test "one-time and monthly expenses in the budget" uses this pattern to assert December and January totals.

## Why This Works

`packages/loot-core/src/mocks/setup.ts` imports `#server/transactions/transaction-rules`, which imports `#shared/months` directly and `#server/rules`, whose `packages/loot-core/src/server/rules/rule-utils.ts` imports `#shared/schedules`, which imports `./months`. Setup files run before the test file registers `vi.mock('#shared/months')`, so every module loaded during setup, `shared/schedules` included, keeps the real `months` module, whose `currentDay()` returns `2017-01-01` under `IS_TESTING`. `packages/loot-core/src/server/schedules/app.ts` is first loaded by the test file itself and receives the mock. Rewinding `next_date` needs no second clock, so every module agrees on the day.

## Prevention

- In loot-core tests, do not rely on `vi.mocked(currentDay)` to change what `getStatus`, `getNextDate`, or any other module loaded during test setup sees. Model elapsed time by moving dates into the past relative to 2017-01-01.
- The existing mocked-day test in `app.test.ts` ("uses the posting day when the day changes during the post") only depends on the day as `packages/loot-core/src/server/schedules/app.ts` reads it, which is why the mock works there.
- A flow that genuinely spans two real days (open the budget in one month, open it again the next) needs a browser check with a moved system clock, not this unit-test mock.
