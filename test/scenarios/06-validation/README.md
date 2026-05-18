# Scenario 06: Validation Rules

**Spec reference:** §3, §8, §9, §10

---

## Overview

A conformant implementation must reject invalid files before attempting to schedule them. Accepting an invalid file and producing a schedule silently corrupts the output in ways that may not be visible to the user. Validation must be the first step after parsing.

The test cases here are grouped into two categories:

- **invalid/**: files that MUST be rejected with an error. The implementation MUST NOT silently accept these and proceed to schedule.
- **valid/**: files that contain unusual but explicitly permitted patterns. The implementation MUST accept these without error.

Each invalid file contains exactly one violation, isolated from all other correctness concerns. This allows the test to confirm that the implementation detects the specific rule being tested, not some other incidental error.

---

## invalid/ - Files that MUST be rejected

### e01-multiple-top-level-projects.pac.toml

**Why this case exists:** A PAC file represents one project tree. Two `[[project]]` entries without a `parent_project_id` would mean two independent root projects in the same file. The rule is: exactly one project entry may have no `parent_project_id`.

**Violation:** Two `[[project]]` entries, both without `parent_project_id`.

**Expected:** validation error.

---

### e02-duplicate-task-id.pac.toml

**Why this case exists:** Task IDs are used as dependency references throughout the file. A duplicate ID makes dependency resolution ambiguous.

**Violation:** Two `[[project.task]]` entries with the same `id` in the same project.

**Expected:** validation error.

---

### e03-unknown-time-unit.pac.toml

**Why this case exists:** Valid `time_unit` values are a closed set: `"hours"` and `"days"`. Proceeding with an unknown value would either crash conversion code or silently use a default, both of which are wrong.

**Violation:** `time_unit = "sprints"`.

**Expected:** validation error.

---

### e05-circular-dependency.pac.toml

**Why this case exists:** The forward pass processes tasks in dependency order. A circular dependency makes topological ordering impossible. Detection must happen before scheduling.

**Violation:** Task `01VRFY00000000000000000041` depends on `01VRFY00000000000000000042`, and vice versa.

**Expected:** validation error.

---

### e07-progress-out-of-range.pac.toml

**Why this case exists:** `actual.progress` is defined as an integer in the range [0, 100]. Values outside this range are semantically undefined.

**Violation:** `actual.progress = 101`.

**Expected:** validation error.

---

## valid/ - Files that MUST be accepted

### v03-empty-dependencies.pac.toml

**Why this case exists:** The `dependencies` field is always present in the file, even when empty. An empty array `dependencies = []` is the explicit statement that a task has no predecessors and should start at ES=0.

**Pattern:** Task with `dependencies = []`.

**Expected:** accepted, task scheduled at ES=0.

---

### v04-negative-priority.pac.toml

**Why this case exists:** The `priority` field range is [-100, 100]. Negative priority is a valid way to mark tasks that should be deferred relative to other tasks of the same track.

**Pattern:** Task with `priority = -50`.

**Expected:** accepted, no error. The task participates in track parallelism resolution with priority -50 (lower than the default 0).

---

## Additional validation rules (no dedicated .pac.toml files)

These rules should be covered by unit tests of the validation function rather than complete project files:

| Rule | Condition | Expected result |
|---|---|---|
| `parallelism = -1` | Negative parallelism | Error: must be >= 0 |
| `actual.progress = -1` | Below range | Error: must be integer 0-100 |
| `actual.progress = 50.5` | Non-integer | Error: must be integer 0-100 |
| `plan.duration = 0` | Zero | Error: must be > 0 |
| `plan.duration = -1` | Negative | Error: must be > 0 |
| `plan.duration = 1.234` | More than 2 decimal places | Error: at most 2 decimals |
| `emoji = "AB"` | Multiple grapheme clusters | Error: must be exactly one grapheme cluster |
| `color = "#3b82f6"` | Lowercase hex | Error: must be uppercase hex (#RRGGBB) |
| `color = "#FFF"` | 3-digit shorthand | Error: must be 6-digit #RRGGBB format |
| Mixed `time_unit` across projects in the same file | Sub-project has different `time_unit` than parent | Error: v0.0.1 requires uniform time_unit |
| Milestone with `dependencies = []` | Empty milestone | Error: milestone must have at least one predecessor |

**Why the color rules matter:** Color values are stored in a canonical format (`#RRGGBB` uppercase) to ensure that two files containing the same color produce the same bytes and therefore the same git diff.

**Why emoji validation matters:** An emoji field displayed in a UI pill must occupy exactly the visual space of one character. A field containing two separate emoji or a letter sequence would break layout.
