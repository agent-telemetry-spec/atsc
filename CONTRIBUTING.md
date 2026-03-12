# Contributing to ATSC

Thank you for your interest in contributing to the Agent Telemetry Semantic Conventions. ATSC is an open specification and community input is essential to making it useful across the agent ecosystem.

---

## What We're Looking For

**High value contributions at this stage:**

- Corrections or ambiguities in the normative spec language
- Implementation feedback — fields that are missing, underspecified, or impractical to emit
- Framework-specific attribute mapping tables (LangGraph, AutoGen, CrewAI, etc.)
- Reference adapter implementations
- Additional worked examples in Section 13
- Identified conflicts or overlaps with OTel GenAI Semantic Conventions

**Not in scope for v0.1.x:**

- New span kinds — the 21-kind taxonomy is stable for v0.1. Proposals for new kinds should target v0.2 and be opened as a discussion, not a PR.
- Changes to the 12 MUST fields — the Core conformance tier is intentionally minimal and stable.
- Wire format or transport definitions — ATSC is schema only.

---

## How to Contribute

### Issues

Open an issue for:
- Spec ambiguities or errors
- Missing field definitions
- Framework compatibility questions
- Proposals for new span kinds or domain object fields (discussion first)

Please include the relevant section number and a concrete example where possible.

### Pull Requests

1. Fork the repository
2. Create a branch named descriptively — `fix/retrieval-result-count-definition`, `feat/crewai-attribute-mapping`, etc.
3. Make your changes
4. Open a PR with a clear description of what changed and why
5. Reference any related issues

For spec changes, please use RFC 2119 language consistently (MUST, SHOULD, MAY) and match the existing prose style.

For schema changes, validate your changes against at least one of the worked examples in Section 13 before submitting.

### Discussions

Use GitHub Discussions for:
- Proposals that need design input before becoming a PR
- Questions about intended behavior
- Framework-specific implementation questions
- Interest in building a reference adapter

---

## Spec Language Guidelines

ATSC uses RFC 2119 keywords. When contributing to the normative spec:

- **MUST / MUST NOT** — absolute requirements. Use sparingly. Every MUST is a conformance burden.
- **SHOULD / SHOULD NOT** — strong recommendation with acknowledged exceptions. The default for most guidance.
- **MAY** — genuinely optional. Use when the field has real value but emission depends on runtime capability.

Avoid inflating SHOULD to MUST. The Core conformance tier is intentionally small — resist the pull to expand it.

---

## Schema Contributions

The normative JSON Schema lives at `schemas/v0.1.0/agent-telemetry-event.json`.

When modifying the schema:
- All nullable fields use `oneOf: [{type: "..."}, {type: "null"}]` — not `type: ["...", "null"]`
- New optional fields go in the appropriate domain object, not in `attributes`
- `attributes` is the escape hatch for data that genuinely has no core schema home
- Do not add `required` constraints to SHOULD or MAY fields

---

## Code of Conduct

Be direct. Be technical. Assume good faith. Feedback on the spec should be about the spec — concrete, specific, and constructive.

---

## Governance

ATSC v0.1.0 is authored and maintained by Jesse Hawkins / [Range Point AI](https://rangepointai.com). The intent is to propose this specification to the OpenTelemetry Semantic Conventions working group once it achieves implementation adoption, at which point governance will transfer to the OTel community process.

Until then, final decisions on spec changes rest with the authors. PRs that meet the bar above will be reviewed promptly.

---

*Apache 2.0 License. Contributions submitted via pull request are accepted under the same license.*
