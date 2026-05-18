# Scenario 02: Track Parallelism Constraint

**Spec reference:** §5.2, §3.2

---

## Overview

The track parallelism constraint limits how many tasks in a track may run simultaneously. It is applied as an iterative pass after the unconstrained forward pass. When a task's unconstrained ES would cause the concurrent count to exceed the track's `parallelism` cap, the task is delayed until a slot opens.

The constraint uses two tiebreaking rules to determine which task runs first when multiple tasks compete for a slot:
1. **Priority:** the task with the **higher** `priority` value runs first.
2. **ID tiebreak:** when priorities are equal, the task with the **lexicographically smaller** `id` runs first.

The iteration loop repeats until no ES changes between full passes. Comparisons are exact (no epsilon needed). The loop is capped at 200 iterations.

---

## case-a-unlimited.pac.toml

**Track parallelism:** 0 (unlimited)

`parallelism = 0` is the sentinel for "no limit". Two tasks in the same track with no dependencies both start at ES=0; the constraint is simply not applied.

| Task id | ES | EF |
|---|---|---|
| 01TPAR00000000000000000003 | 0 | 3 |
| 01TPAR00000000000000000004 | 0 | 3 |

**What breaks if wrong:** If `parallelism = 0` is interpreted as "zero slots", every task is delayed indefinitely. If treated as "1 slot", one task would be unnecessarily delayed.

---

## case-b-sequential.pac.toml

**Track parallelism:** 1

Forces strictly sequential execution. The most common real-world constraint (a single person or team working on one thing at a time). Equal priority, so the smaller ID wins the slot.

| Task id | ES | EF |
|---|---|---|
| 01TPAR00000000000000000012 | 0 | 3 |
| 01TPAR00000000000000000013 | 3 | 6 |

---

## case-c-priority.pac.toml

**Track parallelism:** 1

Priority overrides the ID tiebreak. Task B has `priority = 10`; Task A has `priority = 0` (default, omitted). B runs first even though A has the smaller ID.

| Task id | priority | ES | EF |
|---|---|---|---|
| 01TPAR00000000000000000022 (A) | 0 | 3 | 6 |
| 01TPAR00000000000000000023 (B) | 10 | 0 | 3 |

---

## case-d-parallelism-2.pac.toml

**Track parallelism:** 2

Three tasks compete for two slots. The two smallest IDs fill the slots; the largest waits.

| Task id | ES | EF |
|---|---|---|
| 01TPAR00000000000000000032 | 0 | 4 |
| 01TPAR00000000000000000033 | 0 | 4 |
| 01TPAR00000000000000000034 | 4 | 8 |

---

## case-e-cascading.pac.toml

**Track parallelism:** 1

Combines dependency constraints, priority ordering, and the iterative convergence property of the parallelism algorithm.

Tasks: A, B, D have no dependencies; C depends on A. D has `priority = 10`; the rest are 0.

Resolution:
- t=0: D wins the slot (highest priority). A, B, C delayed.
- t=3 (D finishes): A and B are eligible (C still waits for A). A has smaller ID. A wins.
- t=6 (A finishes): B and C eligible. B has smaller ID. B wins.
- t=9 (B finishes): C runs.

| Task id | ES | EF |
|---|---|---|
| 01TPAR00000000000000000041 (A) | 3 | 6 |
| 01TPAR00000000000000000042 (B) | 6 | 9 |
| 01TPAR00000000000000000043 (C) | 9 | 12 |
| 01TPAR00000000000000000044 (D) | 0 | 3 |

**Note on C's ES:** C has ES=9, not ES=6 (when A finished). Even though C's dependency was satisfied at t=6, the track slot was occupied by B until t=9. The parallelism constraint delays C further than the dependency alone would require.

**What breaks if wrong:** An implementation that runs only one iteration of the parallelism loop will not converge to the correct answer. An implementation that ignores ongoing track occupancy (computing C's ES as 6) will produce wrong results.

---

## Implementation notes

- The parallelism constraint MUST be applied as an iterative loop. Each iteration redoes the full forward pass; the loop terminates when a full pass produces no changes.
- Tasks without a `track_id` are never subject to any parallelism constraint. Each track is independent; there are no cross-track constraints.
- Milestones are never subject to parallelism constraints (§5.2).
- The 200-iteration cap is a safeguard against pathological inputs. Reaching the cap MUST surface a scheduling error.
