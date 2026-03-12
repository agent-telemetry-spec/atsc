# Agent Telemetry Semantic Conventions (ATSC)

**Version:** 0.1.0 — Draft  
**License:** Apache 2.0  
**Schema:** `https://agent-telemetry-spec.github.io/atsc/schemas/v0.1.0/agent-telemetry-event.json`  
**Authors:** Jesse Williams, [thegoo](https://github.com/thegoo)

---

AI agent systems have an observability problem. LLM calls, tool invocations, multi-agent handoffs, human-in-the-loop interactions, retrieval pipelines, and session memory all happen inside agent runtimes — and none of it surfaces as structured, queryable telemetry in any consistent way.

OTel exists. OTel GenAI Semantic Conventions exist for LLM calls. What doesn't exist is the semantic layer above that — the one that knows what an agent *turn* is, what a *handoff* means, what a human review event should carry, how retrieval quality signals are represented, or how to correlate memory reads across a session.

Every observability platform defines its own shape. Every framework emits something different. Enterprises running LangGraph, Microsoft Agent Framework, and AutoGen simultaneously maintain bespoke translation layers for each.

ATSC defines the missing layer.

---

## What It Is

ATSC is a vendor-neutral, transport-agnostic schema for AI agent observability. Every ATSC record is a valid OpenTelemetry span — ingestible by any OTLP-compatible receiver without custom plugins. ATSC adds agent-specific semantics on top of the OTel base spec and OTel GenAI Semantic Conventions.

**21 span kinds** covering the full agent operation surface:
agent turns, LLM calls, tool invocations, MCP tool calls, retrieval, memory, human-in-the-loop, multi-agent handoffs, guardrails, evaluations, graph/workflow steps.

**40+ span events** for point-in-time occurrences within spans:
reasoning thoughts, decisions, streaming tokens, guardrail triggers, human notifications, graph node traversals.

**14 domain objects** with normative field definitions:
model, tool, retrieval, memory, handoff, human, guardrail, evaluation, session, run, caller, executor, error, resource.

**Three-tier conformance model:**
Core (12 MUST fields) → Standard (applicable SHOULD fields) → Full (MAY fields + documented redaction).

---

## What It Is Not

ATSC does not replace OpenTelemetry. It extends it.

ATSC does not define a wire format or transport. Records are transported via OTLP.

ATSC does not require any specific agent framework, observability platform, or cloud provider. LangGraph, Microsoft Agent Framework, Semantic Kernel, AutoGen, or hand-rolled agents all emit the same canonical shape.

---

## Framework and Platform Coverage

| Framework / Platform | Status |
|---|---|
| LangGraph | Reference adapter — roadmap |
| Microsoft Agent Framework | Reference adapter — roadmap |
| Semantic Kernel | Compatibility mapping in spec, adapter roadmap |
| AutoGen | Compatibility mapping in spec, adapter roadmap |
| .NET Aspire | First-class — ATSC spans visualized in Aspire dashboard, zero config |
| Azure Monitor / App Insights | Native via Aspire OTLP export pipeline |
| Langfuse | Source adapter — roadmap |
| LangSmith | Source adapter — roadmap |
| Any OTLP receiver | Compatible by design |

---

## Repository Structure

```
/
├── README.md
├── SPEC.md                          ← Full normative specification
├── schemas/
│   └── v0.1.0/
│       └── agent-telemetry-event.json   ← Normative JSON Schema
├── CONTRIBUTING.md
└── LICENSE
```

---

## Quick Start

### Minimal conformant span

Twelve fields. That's Core conformance.

```json
{
  "spec_version": "0.1.0",
  "event_id": "01HXYZ1234567890ABCDEF",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "parent_span_id": null,
  "span_kind": "llm.chat",
  "timestamp": "2026-03-10T10:00:00.000Z",
  "start_time": "2026-03-10T10:00:00.000Z",
  "end_time": "2026-03-10T10:00:01.243Z",
  "status": "ok",
  "resource": {
    "service.name": "my-agent"
  },
  "run": {
    "id": "run_abc123",
    "kind": "agent"
  }
}
```

### Validate against the schema

```bash
npx ajv-cli validate \
  -s https://agent-telemetry-spec.github.io/atsc/schemas/v0.1.0/agent-telemetry-event.json \
  -d your-span.json
```

---

## Relationship to OpenTelemetry

```
┌─────────────────────────────────────────────────┐
│         ATSC (this specification)               │
│   Agent-specific span kinds, attributes,        │
│   domain objects, span events                   │
├─────────────────────────────────────────────────┤
│      OTel GenAI Semantic Conventions            │
│   gen_ai.* attributes, LLM-call conventions     │
├─────────────────────────────────────────────────┤
│         OTel Base Specification                 │
│   Spans, traces, metrics, logs, OTLP            │
└─────────────────────────────────────────────────┘
```

The authors intend to propose ATSC to the OpenTelemetry Semantic Conventions working group once it achieves implementation adoption.

---

## Status and Contributing

This is a v0.1.0 draft. The schema and span taxonomy are stable enough to build against. Domain object field sets may be refined based on implementation feedback before v1.0.

Feedback welcome via GitHub Issues. Pull requests welcome for:
- Corrections or ambiguities in the spec
- Additional worked examples
- Framework-specific attribute mapping tables
- Reference adapter implementations

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

*Apache 2.0 License. See [LICENSE](LICENSE).*
