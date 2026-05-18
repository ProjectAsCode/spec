# PAC Specification

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)

Project As Code (PAC) is an open standard for storing project plans as version-controlled text files.

---

## What is PAC?

Infrastructure as Code transformed operations by replacing manual, undocumented server configuration with version-controlled, reviewable, reproducible definitions. PAC applies the same principle to project planning: instead of a plan locked inside Jira, Notion, or a spreadsheet, you have a `pac.toml` file in your repository. The plan evolves alongside the code in the same commits, with the same history, reviewed in the same pull requests.

The relationship to git is first-class. A PAC file is designed to produce small, readable diffs: adding a task is a few lines, changing an estimate is one line, and every change carries a commit message explaining why.

For the manifesto and broader context, visit [projectascode.org](https://projectascode.org).

---

## Quick look

A minimal `pac.toml` with two tasks and a dependency:

```toml
[meta]
spec_version = "0.0.1"

[[project]]
color = "#3B82F6"
id = "01HWXKB3NDEKTSV4RRFFQ69G5A"
name = "API Launch"
time_unit = "days"

[[project.task]]
dependencies = []
id = "01HWXKB3NDEKTSV4RRFFQ69G5B"
name = "Design review"

plan.duration = 3

[[project.task]]
dependencies = [
  "01HWXKB3NDEKTSV4RRFFQ69G5B",
]
id = "01HWXKB3NDEKTSV4RRFFQ69G5C"
name = "Implementation"

plan.duration = 5
```

Every field is on its own line. Keys are in strict alphabetical order. Dependencies are a multiline array so that adding or removing one dependency is always a one-line diff.

---

## Core properties of a PAC file

- **Text**: readable in any editor, no binary format
- **[TOML](https://toml.io)**: structured, typed, and git-diff-friendly by design
- **Fixed filename**: always `pac.toml`, never user-chosen (like `Cargo.toml` or `Makefile`)
- **Lives in the repository**: alongside the code it describes
- **Diffable**: small changes produce small, readable diffs
- **Git-blamed**: every field change has an author, a timestamp, and a commit message

---

## The specification

The full PAC file format specification is in [`SPEC.md`](SPEC.md). It covers:

- Data model (Project, Track, Task, Milestone, Sub-project) with all fields and types
- Forward-pass scheduling with track parallelism
- Progress and status model
- A single structural alert (Blocked) and its trigger conditions
- File format rules (TOML structure, key ordering, identifier format, versioning)
- Sub-project nesting and conversion rules
- Scale constraints that all conformant implementations must enforce

What is deliberately not in v0.0.1 (PERT three-point estimates, float / critical path, probability tables, etc.) is listed in [`FUTURE.md`](FUTURE.md) with non-breaking re-introduction paths.

The spec uses [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) language throughout: MUST, SHOULD, MAY.

For a practical comparison of PAC against alternatives like Jira, Linear, GitHub Projects, and Notion, see [`COMPARISON.md`](COMPARISON.md).

---

## Implementations

| Implementation | Language | Status | Link |
|---|---|---|---|
| PAC reference tool | TypeScript / Preact | Active | [github.com/projectascode/tool](https://github.com/projectascode/tool) |

If you have built a PAC-compatible tool, open a PR to add it here.

---

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md) for the RFC-style process used to propose and review changes to the specification.

For broader discussion, use [GitHub Discussions](https://github.com/projectascode/spec/discussions).

---

## License

This specification is published under the [Creative Commons Attribution 4.0 International (CC-BY 4.0)](LICENSE) license. You are free to implement, share, and adapt it for any purpose with attribution.
