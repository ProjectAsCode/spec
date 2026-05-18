# Contributing to the PAC Specification

Thank you for your interest in improving the PAC specification. This is a community standard and every contribution (from fixing a typo to proposing a new section) makes it better for everyone.

---

## How changes are made

The PAC specification follows an RFC-style process. Changes are not merged silently; they are proposed, discussed, and reviewed in the open.

### 1. Open an Issue first

Before writing any changes, open an Issue describing what you want to change and why. Use the [Spec Change template](.github/ISSUE_TEMPLATE/spec-change.md). This lets the community discuss the idea before anyone invests time writing it up. Many good ideas need refinement before they are ready to become spec language.

### 2. Discuss

Use the Issue thread to refine the proposal. For broader conversations that span multiple issues or are exploratory in nature, use [GitHub Discussions](https://github.com/projectascode/spec/discussions) in this repository.

### 3. Open a Pull Request

Once the Issue has reached rough consensus, open a PR against `main` with your changes to `SPEC.md`. Reference the Issue number in the PR description. Keep PRs focused: one change per PR makes review faster and history cleaner.

### 4. Review and merge

A maintainer will review the PR. All substantive changes require at least one approval before merging. Clarifications and typo fixes may be merged with lighter review.

---

## Versioning

The specification uses [Semantic Versioning](https://semver.org/):

| Change type | Version bump | Examples |
|---|---|---|
| Clarification: no behaviour change | **patch** (0.0.x) | Rewording, fixing ambiguity, adding examples |
| Additive change: backwards compatible | **minor** (0.x.0) | New optional field, new SHOULD-level rule |
| Breaking change: incompatible with prior files | **major** (x.0.0) | Removing a field, changing a MUST rule |

When in doubt, treat a change as more significant rather than less. It is easier to relax a version bump than to explain why a "clarification" broke existing implementations.

---

## Licensing

All contributions to this repository are made under the terms of the [Creative Commons Attribution 4.0 International (CC-BY 4.0)](LICENSE) license. By submitting a contribution, you agree that your work may be used, shared, and adapted under those terms, provided attribution is given.

---

## Style guide

- Write in the present tense: "A conformant implementation MUST…", not "A conformant implementation will need to…"
- Use [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) keywords (MUST, MUST NOT, SHOULD, SHOULD NOT, MAY) consistently and correctly
- Be precise: ambiguity in a specification is a bug
- Include examples wherever a rule might be misread
- Link to the relevant section when one rule depends on another

---

We are glad you are here. The best specifications are written by people who are actually building with them.
