# Scenario 01: Forward Pass

**Spec reference:** §5.1

---

## Overview

The forward pass computes Early Start (ES) and Early Finish (EF) for every task and milestone. These values define the v0.0.1 schedule.

For each task:

```
duration(task) = actual.duration if set, otherwise plan.duration

ES(task) = max(EF(predecessor)) for all predecessors
         = 0 if no predecessors

ES(task) = max(0, ES(task) + actual.offset)   -- apply offset, clamp at 0

EF(task) = ES(task) + duration(task)
```

Tie-break: when multiple tasks become eligible at the same point with no ordering constraint between them, they MUST be processed in `id` ascending order. The forward pass is deterministic: the same inputs always produce the same ES and EF values.

---

## linear-chain.pac.toml

Three tasks in sequence: A → B → C. Durations 3, 2, 4. Expected:

| Task | ES | EF |
|---|---|---|
| A | 0 | 3 |
| B | 3 | 5 |
| C | 5 | 9 |

---

## diamond.pac.toml

Diamond dependency: A → {B, C} → D. Durations 3, 4, 2, 2. Expected:

| Task | ES | EF |
|---|---|---|
| A | 0 | 3 |
| B | 3 | 7 |
| C | 3 | 5 |
| D | 7 | 9 (max(B.EF, C.EF) = 7) |

---

## id-tiebreak.pac.toml

Three independent tasks with identical durations, listed in non-ID order in the file. The forward pass MUST be deterministic regardless of file order; all three get ES=0, EF=2. The tiebreak rule (`id` ascending) is what gives the deterministic order when downstream tasks reference these.

---

## Implementation notes

- `plan.duration` is the canonical task duration in v0.0.1. Future versions may add optional `plan.optimistic` and `plan.pessimistic`; v0.0.1 implementations ignore them.
- `actual.duration`, if present, overrides `plan.duration` in scheduling (§3.3).
- `actual.offset`, if present, shifts ES (§3.3); negative offsets MAY break the predecessor dependency clamp.
- Milestones participate in the forward pass exactly like zero-duration tasks (EF = ES always).
