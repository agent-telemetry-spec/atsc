# Agent Telemetry Semantic Conventions (ATSC)
**Version:** 0.1.0  
**Status:** Draft  
**Schema ID:** `https://agent-telemetry-spec.github.io/atsc/schemas/v0.1.0/agent-telemetry-event.json`  
**License:** Apache 2.0  

---

## Abstract

Agent Telemetry Semantic Conventions (ATSC) defines a vendor-neutral, transport-agnostic schema for observability of AI agent systems. It specifies how agent operations — including LLM calls, tool invocations, memory access, retrieval, human-in-the-loop interactions, multi-agent handoffs, and evaluations — are represented as structured telemetry records.

Every ATSC record is a valid OpenTelemetry span. ATSC adds a semantic layer on top of the OTel base specification and the OTel GenAI Semantic Conventions, defining the agent-specific attributes, span kinds, and span events that those specifications do not address.

ATSC is designed to be ingested by any OTLP-compatible receiver without custom plugins. It is implementable by any agent framework, observability platform, or custom instrumentation. It is source-agnostic — Langfuse, LangSmith, LangGraph, Microsoft Agent Framework, or hand-rolled agents all emit the same canonical shape.

ATSC has first-class support for .NET Aspire. Aspire applications emit OTLP natively. ATSC spans are valid OTLP spans and are visualized in the Aspire local developer dashboard with zero additional configuration. The ATSC adapter for Microsoft Agent Framework on Aspire is a reference implementation maintained in the ATSC repository. This adapter also supports Semantic Kernel and AutoGen, which are converging into Microsoft Agent Framework.

---

## Status of This Document

This document is a draft specification. It is open for community review and contribution. The authors intend to propose this specification to the OpenTelemetry Semantic Conventions working group once it achieves critical implementation mass.

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## Table of Contents

1. [Motivation](#1-motivation)
2. [Design Principles](#2-design-principles)
3. [Relationship to OpenTelemetry](#3-relationship-to-opentelemetry)
4. [Record Model](#4-record-model)
5. [Span Kind Taxonomy](#5-span-kind-taxonomy)
6. [Top-Level Fields](#6-top-level-fields)
7. [Domain Objects](#7-domain-objects)
   - 7.1 [resource](#71-resource)
   - 7.2 [session](#72-session)
   - 7.3 [run](#73-run)
   - 7.4 [caller and executor](#74-caller-and-executor)
   - 7.5 [model](#75-model)
   - 7.6 [tool](#76-tool)
   - 7.7 [retrieval](#77-retrieval)
   - 7.8 [memory](#78-memory)
   - 7.9 [handoff](#79-handoff)
   - 7.10 [human](#710-human)
   - 7.11 [guardrail](#711-guardrail)
   - 7.12 [evaluation](#712-evaluation)
   - 7.13 [error](#713-error)
   - 7.14 [input and output](#714-input-and-output)
8. [Span Events](#8-span-events)
9. [Conformance](#9-conformance)
10. [Extension and Vendor Namespaces](#10-extension-and-vendor-namespaces)
11. [Privacy and Redaction](#11-privacy-and-redaction)
12. [JSON Schema](#12-json-schema)
13. [Examples](#13-examples)
14. [Changelog](#14-changelog)

---

## 1. Motivation

AI agent systems present observability requirements that general-purpose distributed tracing specifications do not address. Current gaps include:

- No standard representation for agent reasoning cycles, tool invocations, or multi-agent handoffs
- No canonical schema for human-in-the-loop interactions and the audit trail they require
- No standard for capturing retrieval quality signals in RAG pipelines
- No common model for session-scoped memory and its lineage across runs
- No consistent span hierarchy for graph-based agent frameworks

Without a shared standard, every observability platform (Langfuse, LangSmith, Arize Phoenix) defines its own schema. Agent frameworks (LangGraph, Microsoft Agent Framework, AutoGen) emit incompatible telemetry shapes. Enterprises running multiple frameworks and multiple observability backends are forced to maintain bespoke translation layers.

A specific and common instance of this problem: .NET developers building agents on Microsoft Agent Framework (or its predecessors Semantic Kernel and AutoGen) with .NET Aspire have a first-class OTLP pipeline available out of the box, but no standard semantic layer for agent-specific operations. Aspire surfaces raw OTel spans in its developer dashboard and routes to Azure Monitor / Application Insights in production — but without ATSC, tool calls, agent turns, HITL events, and retrieval operations are invisible as structured data. They appear as generic undifferentiated spans with no agent-specific meaning.

ATSC defines the missing semantic layer. It does not replace OTel — it extends it for the agent domain in the same way that OTel GenAI Semantic Conventions extended it for LLM calls.

---

## 2. Design Principles

**Transport-agnostic.** ATSC defines a semantic schema, not a wire format. Records are transported via OTLP, but the schema is valid JSON and implementable over any transport.

**Source-agnostic.** ATSC does not care where telemetry originates. Langfuse, LangSmith, custom instrumentation, and MCP-native tool calls all emit the same canonical record shape. Source-specific attributes belong in vendor extension namespaces.

**Complete spans only.** ATSC records represent complete spans with both start and end times. Partial or streaming records are an adapter implementation concern and MUST NOT be emitted as conformant ATSC records. Adapters receiving start/end events from upstream sources MUST buffer and assemble complete spans before emission.

**OTel receiver compatible.** Every ATSC record MUST be ingestible by any OTLP-compatible receiver without custom plugins. Semantic enrichment is additive — receivers that do not understand ATSC-specific attributes degrade gracefully to standard OTel span behavior.

**Low barrier to conformance.** The MUST tier is intentionally minimal. Twelve fields. All universally available from any agent runtime without special instrumentation. Developers MUST NOT be pressured to emit dummy values to satisfy conformance.

**Progressive enrichment.** Conformance is tiered. Core conformance is fast. Full conformance is the mature pipeline goal. Telemetry improves over time without re-instrumentation.

---

## 3. Relationship to OpenTelemetry

ATSC sits on top of the OTel stack:

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

ATSC records are OTel spans. The following OTel concepts apply directly and without modification:

- `trace_id` and `span_id` follow OTel W3C TraceContext formatting
- `parent_span_id` establishes the OTel parent/child span relationship
- `links[]` follows the OTel span links model for cross-trace correlation
- `resource` follows OTel resource conventions
- `span_events` follows the OTel span events model

Where ATSC defines attributes that overlap with OTel GenAI Semantic Conventions (e.g. `model.name` overlapping with `gen_ai.request.model`), ATSC exporters targeting OTLP backends SHOULD map ATSC attributes to their OTel GenAI equivalents. The canonical mapping table is maintained in the ATSC repository.

### Relationship to .NET Aspire and Microsoft Agent Framework

.NET Aspire provides a cloud-native application framework with first-class OpenTelemetry support. Microsoft Agent Framework (AF) is Microsoft's converged agent development platform, consolidating Semantic Kernel and AutoGen into a single framework. Both SK and AutoGen remain supported during the migration period — the ATSC adapter covers all three.

> **Migration note:** Semantic Kernel and AutoGen are actively converging into Microsoft Agent Framework. The ATSC adapter and attribute namespace target AF as the primary surface. SK and AutoGen attribute mappings are maintained as compatibility aliases. Implementations built on SK or AutoGen today will migrate to AF without ATSC schema changes — the canonical fields do not change, only the `executor.framework` value and any `semantic_kernel.*` / `autogen.*` extension attributes.

The relationship between ATSC and Aspire operates at three levels:

**Local development dashboard.** The Aspire developer dashboard ingests OTLP spans natively. Since every ATSC record is a valid OTLP span, ATSC-instrumented agents running in an Aspire AppHost are automatically visualized in the dashboard — traces, spans, and span events all rendered without additional configuration. This is the zero-friction local development story for .NET agent developers regardless of whether they are on SK, AutoGen, or AF.

**Azure Monitor / Application Insights in production.** Aspire's production deployment path routes OTLP to Azure Monitor via the `OpenTelemetry.Exporter.AzureMonitor` package. ATSC spans flow through this pipeline unmodified. Azure Monitor receives structured agent telemetry — tool calls, LLM calls, HITL events — as queryable telemetry data in Application Insights and Log Analytics.

**ATSC enrichment as an Aspire-compatible processor.** The ATSC adapter for Microsoft Agent Framework is implemented as an OTel `ActivityProcessor` — the standard .NET OTel extension point — and registered in the Aspire `IDistributedApplicationBuilder` pipeline. This means it composes naturally with all other Aspire telemetry configuration without framework-specific coupling.

```
Aspire AppHost
  └─ Microsoft Agent Framework agent
       │  (migrating from Semantic Kernel / AutoGen)
       └─ .NET OTel pipeline
            ├─ ATSC ActivityProcessor    ← enriches AF/SK/AutoGen spans with ATSC attributes
            ├─ Aspire dashboard          ← local visualization, zero config
            └─ AzureMonitorExporter      ← production: App Insights / Log Analytics
```

The `aspire.*` and `agent_framework.*` vendor namespaces in `attributes` are reserved for Aspire and AF-specific telemetry attributes that do not map to core ATSC fields. See Section 10.

---

## 4. Record Model

One ATSC record represents one complete span. A span is a discrete unit of agent work with a start time, end time, and a single `span_kind`.

```
atsc_record
  │
  ├── identity          trace_id, span_id, parent_span_id, event_id
  ├── classification    span_kind, otel_span_kind, status
  ├── timing            timestamp, start_time, end_time
  ├── infrastructure    resource
  ├── context           session, run, caller, executor
  ├── domain object     model | tool | retrieval | memory | handoff |
  │                     human | guardrail | evaluation | error
  ├── content           input, output
  ├── events            span_events[]
  ├── links             links[]
  └── extension         attributes{}
```

Exactly one domain object SHOULD be populated per record, corresponding to the `span_kind`. A `llm.chat` span populates `model`. A `tool.call` span populates `tool`. Populating multiple domain objects on a single span is a conformance warning — use child spans instead.

### Span Hierarchy

```
session  (logical container — shared across runs)
  └─ run  (one agent execution)
       └─ agent.invoke  (root span — MUST be present)
            ├─ agent.turn
            │    ├─ llm.chat
            │    │    └─ span_events: thought, token.limit.warning
            │    ├─ tool.call
            │    │    └─ span_events: tool.input.validated, guardrail.triggered
            │    └─ retrieval.query
            ├─ agent.step  (graph/workflow frameworks)
            │    └─ agent.turn
            ├─ guardrail.check
            ├─ evaluation.run
            ├─ memory.read
            ├─ memory.write
            └─ agent.handoff
                 └─ agent.invoke  (subagent — orchestrator pattern)
```

---

## 5. Span Kind Taxonomy

`span_kind` is the primary classification of every ATSC record. Every span has exactly one `span_kind`. The taxonomy is stable. New kinds require a spec revision.

### OTel SpanKind Mapping

Every ATSC `span_kind` maps to an OTel `SpanKind`. This mapping is normative. Exporters MUST apply it.

| ATSC span_kind | OTel SpanKind | Description |
|---|---|---|
| `agent.invoke` | INTERNAL | Top-level agent run. The root span for any agent execution. |
| `agent.turn` | INTERNAL | One reasoning cycle — from receiving input to producing output or deciding next action. Always involves or leads to an LLM call. |
| `agent.step` | INTERNAL | One node execution in a graph or workflow. May or may not involve an LLM call. OPTIONAL — see conformance. |
| `agent.handoff` | CLIENT | Transfer of control from one agent to another. |
| `llm.chat` | CLIENT | Chat completion call to any model provider. |
| `llm.completion` | CLIENT | Text completion call to any model provider. |
| `llm.embed` | CLIENT | Embedding generation call. |
| `tool.call` | CLIENT | Invocation of any tool — function, API, or service. |
| `tool.mcp` | CLIENT | MCP-native tool invocation. Carries MCP-specific attributes. |
| `retrieval.query` | CLIENT | Semantic or keyword search against an external knowledge source. |
| `retrieval.fetch` | CLIENT | Direct fetch of a document or chunk by identifier. |
| `memory.read` | INTERNAL | Read from an agent-managed memory store. |
| `memory.write` | INTERNAL | Write to an agent-managed memory store. |
| `human.input` | SERVER | Agent requested input from a human to proceed. |
| `human.review` | INTERNAL | Agent output gated on human approval before proceeding. |
| `human.override` | INTERNAL | Human intervened to change an agent decision. |
| `guardrail.check` | INTERNAL | A guardrail running as a discrete operation with its own latency and result. |
| `evaluation.run` | INTERNAL | An evaluation scoring operation. |
| `infra.cache` | CLIENT | Cache read or write operation. |
| `infra.queue` | PRODUCER / CONSUMER | Async message enqueue or dequeue. |
| `error` | INTERNAL | Standalone error span for failures not attributable to a specific operation. |

### `agent.step` Conformance Note

`agent.step` is OPTIONAL. Frameworks with explicit graph or workflow execution models (e.g. LangGraph, Semantic Kernel pipelines, AutoGen workflows) SHOULD emit `agent.step` spans. Frameworks without this concept MAY omit them entirely. Receivers MUST NOT require `agent.step` for a trace to be considered conformant. This field is expected to move to RECOMMENDED in a future version as graph-based framework adoption grows.

### Memory vs Retrieval Distinction

`memory.*` spans represent agent-managed state persistence — reading and writing the agent's own accumulated context, history, or learned associations. The agent owns the memory store.

`retrieval.*` spans represent queries against external knowledge sources — vector databases, document stores, search engines, or any corpus the agent does not own or manage. The agent queries but does not write to retrieval stores.

An agent reading from its own episodic memory store is `memory.read`. The same agent querying a shared enterprise knowledge base is `retrieval.query`. The distinction is ownership and write access, not storage technology.

---

## 6. Top-Level Fields

### Required Fields

| Field | Type | Description |
|---|---|---|
| `spec_version` | `string` | MUST be `"0.1.0"`. Enables versioned parsing by receivers. |
| `event_id` | `string` | Globally unique identifier for this record. Used for deduplication across retries and pipeline failures. |
| `trace_id` | `string` | 32 hex character W3C TraceContext trace ID. |
| `span_id` | `string` | 16 hex character span identifier. |
| `span_kind` | `string` | One value from the span kind taxonomy. See Section 5. |
| `timestamp` | `string` | ISO 8601 datetime. The canonical time of this record — typically equal to `start_time`. |
| `start_time` | `string` | ISO 8601 datetime. When this span began. |
| `end_time` | `string` | ISO 8601 datetime. When this span ended. |
| `status` | `string` | `"ok"` \| `"error"` \| `"cancelled"` \| `"timeout"` |
| `resource` | `object` | Service and infrastructure context. See Section 7.1. MUST include `resource.service.name`. |
| `run.id` | `string` | Stable run identifier shared across all spans in a logical run. See Section 7.3. |
| `run.kind` | `string` | Classification of the run. See Section 7.3. |

### Optional Top-Level Fields

| Field | Type | Description |
|---|---|---|
| `parent_span_id` | `string \| null` | 16 hex character parent span ID. NULL on root spans only. MUST be set on all non-root spans. |
| `otel_span_kind` | `string` | OTel SpanKind value. If omitted, receivers derive from `span_kind` using the normative mapping table. |
| `trace_state` | `string \| null` | W3C TraceContext tracestate header value. |
| `links` | `array` | Cross-trace span links. See links schema below. |
| `attributes` | `object` | Extension attributes. See Section 10. |

### `parent_span_id` Schema

```json
"parent_span_id": {
  "oneOf": [
    { "type": "string", "pattern": "^[A-Fa-f0-9]{16}$" },
    { "type": "null" }
  ],
  "description": "NULL on root spans only. MUST be set on all non-root spans."
}
```

### `links` Schema

```json
"links": {
  "type": "array",
  "items": {
    "type": "object",
    "required": ["trace_id", "span_id"],
    "additionalProperties": false,
    "properties": {
      "trace_id": { "type": "string", "pattern": "^[A-Fa-f0-9]{32}$" },
      "span_id":  { "type": "string", "pattern": "^[A-Fa-f0-9]{16}$" },
      "attributes": { "type": "object", "additionalProperties": true }
    }
  }
}
```

Use `links` for: pipeline pattern handoffs (cross-trace correlation), async operations, forked sessions, and any causal relationship that is not expressible as parent/child.

---

## 7. Domain Objects

### 7.1 `resource`

Follows OTel resource conventions. Receivers use resource attributes for service topology, environment filtering, and infrastructure correlation.

| Field | Type | Conformance | Description |
|---|---|---|---|
| `service.name` | `string` | MUST | Logical service name. |
| `service.version` | `string \| null` | SHOULD | Deployed version of the service. |
| `service.instance.id` | `string \| null` | MAY | Unique identifier of this service instance. |
| `deployment.environment` | `string \| null` | SHOULD | e.g. `production`, `staging`, `development` |
| `cloud.provider` | `string \| null` | MAY | e.g. `azure`, `aws`, `gcp` |
| `cloud.region` | `string \| null` | MAY | e.g. `eastus`, `us-east-1` |
| `host.name` | `string \| null` | MAY | Hostname of the machine running the agent. |

`resource` allows `additionalProperties: true` to support any OTel resource attributes not listed here.

---

### 7.2 `session`

The outermost logical container. A session spans multiple runs sharing continuous context — a multi-turn user conversation, a long-running research task, or a persistent background agent. Session is optional. Single-shot batch agents with no user context MAY omit it.

#### `session` Identity

| Field | Type | Conformance | Description |
|---|---|---|---|
| `session.id` | `string \| null` | SHOULD | Stable session identifier. Shared across all runs and spans in the session. Use to correlate across trace boundaries. |
| `session.name` | `string \| null` | MAY | Human readable session label. |
| `session.kind` | `string \| null` | MAY | `"interactive"` \| `"background"` \| `"scheduled"` \| `"event_driven"` \| `"other"` |
| `session.state` | `string \| null` | MAY | `"active"` \| `"idle"` \| `"paused"` \| `"completed"` \| `"abandoned"` \| `"expired"` |
| `session.started_at` | `string \| null` | MAY | ISO 8601. When the session was first created. |
| `session.ended_at` | `string \| null` | MAY | ISO 8601. When the session ended. NULL if still active. |
| `session.turn_count` | `integer \| null` | MAY | Agent turns completed in this session so far. |
| `session.run_count` | `integer \| null` | MAY | Runs completed in this session so far. |
| `session.resumed` | `boolean \| null` | MAY | Whether this session was resumed after a gap or interruption. |
| `session.resumed_from` | `string \| null` | MAY | `session.id` this session was resumed or forked from. Enables branched session history. |

#### `session.participant`

| Field | Type | Conformance | Description |
|---|---|---|---|
| `user_id` | `string \| null` | SHOULD | Opaque user identifier. Caller-defined. SHOULD be pseudonymized when emitting to third-party backends. |
| `user_role` | `string \| null` | SHOULD | `"end_user"` \| `"operator"` \| `"developer"` \| `"system"` \| `"other"` |
| `anonymous` | `boolean \| null` | MAY | Whether the participant is unauthenticated. |
| `organization_id` | `string \| null` | MAY | Tenant or organization identifier for multi-tenant systems. |
| `channel` | `string \| null` | MAY | `"web"` \| `"mobile"` \| `"api"` \| `"cli"` \| `"voice"` \| `"email"` \| `"slack"` \| `"teams"` \| `"other"` |
| `locale` | `string \| null` | MAY | BCP 47 language tag. e.g. `en-US` |
| `timezone` | `string \| null` | MAY | IANA timezone. e.g. `America/New_York` |

#### `session.context`

| Field | Type | Conformance | Description |
|---|---|---|---|
| `memory_store_id` | `string \| null` | MAY | Identifier of the memory store scoped to this session. |
| `variables` | `object \| null` | MAY | Session-scoped variables available to all runs. High sensitivity — apply redaction rules. |
| `tags` | `array \| null` | MAY | Freeform string labels for filtering and grouping. |
| `token_budget` | `integer \| null` | MAY | Total token budget for this session if constrained. |
| `tokens_consumed` | `integer \| null` | MAY | Cumulative tokens used across all runs in this session. |
| `cost_usd_accumulated` | `number \| null` | MAY | Cumulative cost across all runs. Adapter-computed. Accuracy not guaranteed. |

---

### 7.3 `run`

One logical agent execution. A run contains one or more spans sharing a `run.id`. All spans in a run MUST share the same `run.id`. A session contains one or more runs.

#### Run Kind Definitions

| Value | Definition |
|---|---|
| `"agent"` | A single autonomous agent instance executing a goal. Has its own reasoning loop, tools, and memory. A LangGraph graph with an LLM node is `agent`. An AutoGen AssistantAgent is `agent`. |
| `"workflow"` | A deterministic or semi-deterministic orchestration of multiple agents or steps. Control flow is defined by the system, not dynamically chosen by an LLM. A LangGraph StateGraph with fixed edges is `workflow`. A Semantic Kernel pipeline is `workflow`. |
| `"session"` | A multi-turn interaction with a persistent user or system spanning multiple agent or workflow runs. A session contains runs. Runs do not contain sessions. |
| `"task"` | A discrete bounded unit of work dispatched to an agent or workflow. Has a defined input, expected output, and completion criteria. A background job or queue-dispatched invocation is `task`. |

| Field | Type | Conformance | Description |
|---|---|---|---|
| `run.id` | `string` | MUST | Stable run identifier. Shared across all spans in this run. A UUID generated at run start is sufficient. |
| `run.kind` | `string` | MUST | One of the four run kind values above. |
| `run.name` | `string \| null` | SHOULD | Human readable run label. |
| `run.attempt` | `integer \| null` | SHOULD | Retry attempt number. 1 = first attempt. Allows correlation of retried runs sharing the same `run.id`. |

---

### 7.4 `caller` and `executor`

Every span has two actor roles. `caller` is the agent or system that initiated this span. `executor` is the agent or system performing the work. For most spans in single-agent systems these are identical. For handoffs and tool calls they differ.

Replaces the `actor` field from earlier drafts.

| Field | Type | Conformance | Description |
|---|---|---|---|
| `type` | `string \| null` | SHOULD | `"agent"` \| `"tool"` \| `"model"` \| `"system"` \| `"user"` |
| `id` | `string \| null` | SHOULD | Unique identifier of this actor. |
| `name` | `string \| null` | SHOULD | Human readable name. |
| `framework` | `string \| null` | SHOULD | e.g. `"langgraph"`, `"semantic_kernel"`, `"autogen"`, `"custom"` |
| `version` | `string \| null` | MAY | Version of the agent or framework. |

Both `caller` and `executor` share this field schema.

---

### 7.5 `model`

Populated on `llm.chat`, `llm.completion`, and `llm.embed` spans.

| Field | Type | Conformance | Description |
|---|---|---|---|
| `provider` | `string \| null` | SHOULD | e.g. `"openai"`, `"azure_openai"`, `"anthropic"`, `"google"` |
| `name` | `string \| null` | SHOULD | Model name as returned by the provider. e.g. `"gpt-4o"`, `"claude-3-5-sonnet"` |
| `request_id` | `string \| null` | MAY | Provider-assigned request identifier. |
| `temperature` | `number \| null` | MAY | Sampling temperature used. |
| `max_tokens` | `integer \| null` | MAY | Max tokens requested. |
| `input_tokens` | `integer \| null` | SHOULD | Prompt tokens consumed. Omit only if provider does not return usage. |
| `output_tokens` | `integer \| null` | SHOULD | Completion tokens generated. Omit only if provider does not return usage. |
| `total_tokens` | `integer \| null` | MAY | Total tokens. MAY be derived from input + output. |
| `cost_usd` | `number \| null` | MAY | Adapter-computed cost estimate. Accuracy not guaranteed. Not a billing source of truth. |

---

### 7.6 `tool`

Populated on `tool.call` and `tool.mcp` spans.

| Field | Type | Conformance | Description |
|---|---|---|---|
| `name` | `string \| null` | SHOULD | Tool name. SHOULD never be omitted — tool name is always known at invocation. |
| `type` | `string \| null` | SHOULD | `"function"` \| `"extension"` \| `"datastore"` — per OTel GenAI `gen_ai.tool.type` |
| `version` | `string \| null` | MAY | Tool version. |
| `target` | `string \| null` | MAY | The endpoint, function path, or resource the tool acts on. |
| `success` | `boolean \| null` | SHOULD | Whether the tool invocation succeeded. |

For `tool.mcp` spans, MCP-specific attributes (server, method, transport) SHOULD be placed in `attributes` under the `mcp.*` namespace.

---

### 7.7 `retrieval`

Populated on `retrieval.query` and `retrieval.fetch` spans.

#### `retrieval.store`

| Field | Type | Conformance | Description |
|---|---|---|---|
| `id` | `string \| null` | SHOULD | Identifier of the retrieval store. |
| `type` | `string \| null` | SHOULD | `"vector"` \| `"keyword"` \| `"hybrid"` \| `"graph"` \| `"sql"` \| `"document"` \| `"web"` \| `"other"` |
| `provider` | `string \| null` | SHOULD | e.g. `"pinecone"`, `"weaviate"`, `"opensearch"`, `"postgres"`, `"bing"` |
| `index` | `string \| null` | MAY | Index or collection name. |

#### `retrieval.query`

| Field | Type | Conformance | Description |
|---|---|---|---|
| `text` | `string \| null` | SHOULD | The query sent verbatim. Omit only if query contains PII and redaction is not configured. |
| `embedding_model` | `string \| null` | MAY | Model used to embed the query. |
| `top_k` | `integer \| null` | SHOULD | Number of results requested. |
| `similarity_threshold` | `number \| null` | MAY | Minimum similarity score for inclusion. |
| `filters` | `object \| null` | MAY | Metadata filters applied to the search. |
| `hybrid_alpha` | `number \| null` | MAY | 0–1 weighting between vector and keyword for hybrid search. |

#### `retrieval.results`

| Field | Type | Conformance | Description |
|---|---|---|---|
| `result_count` | `integer \| null` | SHOULD | Total results returned by the store. |
| `returned_count` | `integer \| null` | MAY | Results after threshold filtering. |
| `reranked` | `boolean \| null` | MAY | Whether results were reranked. |
| `rerank_model` | `string \| null` | MAY | Model used for reranking. |
| `top_score` | `number \| null` | MAY | Similarity score of the top result. |
| `min_score` | `number \| null` | MAY | Similarity score of the lowest-ranked returned result. |
| `mean_score` | `number \| null` | MAY | Mean similarity score across returned results. |
| `threshold_misses` | `integer \| null` | MAY | Results retrieved but dropped below similarity threshold. High values indicate threshold tuning opportunity. |
| `documents` | `array \| null` | MAY | Retrieved documents. MAY be omitted for volume or privacy. See document schema below. |

#### `retrieval.results.documents[]`

| Field | Type | Description |
|---|---|---|
| `id` | `string \| null` | Document identifier. |
| `score` | `number \| null` | Similarity score 0–1. |
| `rank` | `integer \| null` | Position in result set. 1-indexed. |
| `source` | `string \| null` | Document origin URI or identifier. |
| `chunk_id` | `string \| null` | Chunk identifier within source document. |
| `mime_type` | `string \| null` | Content type. |
| `token_count` | `integer \| null` | Size of document in tokens. |
| `redacted` | `boolean \| null` | Whether document content was redacted before recording. |

#### `retrieval.performance`

| Field | Type | Description |
|---|---|---|
| `embed_latency_ms` | `integer \| null` | Time to embed the query. |
| `search_latency_ms` | `integer \| null` | Time for the store search. |
| `rerank_latency_ms` | `integer \| null` | Time for reranking. |
| `total_latency_ms` | `integer \| null` | Total retrieval operation time. |

---

### 7.8 `memory`

Populated on `memory.read` and `memory.write` spans.

#### `memory.store`

| Field | Type | Conformance | Description |
|---|---|---|---|
| `id` | `string \| null` | SHOULD | Identifier of the memory store. |
| `type` | `string \| null` | SHOULD | `"in_context"` \| `"vector"` \| `"key_value"` \| `"document"` \| `"episodic"` \| `"semantic"` \| `"other"` |
| `provider` | `string \| null` | MAY | e.g. `"redis"`, `"pinecone"`, `"postgres"`, `"in_process"` |
| `scope` | `string \| null` | SHOULD | `"session"` \| `"user"` \| `"agent"` \| `"global"` \| `"run"` — lifetime and visibility of this store. |

#### `memory.operation`

| Field | Type | Conformance | Description |
|---|---|---|---|
| `key` | `string \| null` | MAY | Explicit key for key/value stores. |
| `query` | `string \| null` | MAY | Semantic query for similarity-based retrieval. |
| `namespace` | `string \| null` | MAY | Logical partition within the store. |
| `top_k` | `integer \| null` | MAY | Number of results requested. |
| `similarity_threshold` | `number \| null` | MAY | Minimum similarity for inclusion. |
| `ttl_seconds` | `integer \| null` | MAY | Time-to-live for write operations. |
| `overwrite` | `boolean \| null` | MAY | Whether write overwrote an existing value. |

#### `memory.result` (read operations only)

| Field | Type | Conformance | Description |
|---|---|---|---|
| `hit` | `boolean \| null` | SHOULD | Whether memory was found. |
| `result_count` | `integer \| null` | MAY | Number of results returned. |
| `token_count` | `integer \| null` | MAY | Size of retrieved memory in tokens. |
| `freshness` | `string \| null` | MAY | `"fresh"` \| `"stale"` \| `"expired"` |
| `source_run_id` | `string \| null` | MAY | `run.id` of the run that originally wrote this memory. Enables memory lineage tracing. |
| `source_span_id` | `string \| null` | MAY | `span_id` of the span that originally wrote this memory. |

#### `memory.content`

| Field | Type | Conformance | Description |
|---|---|---|---|
| `summary` | `string \| null` | MAY | Human readable description of what is stored or retrieved. |
| `mime_type` | `string \| null` | MAY | Content type of memory payload. |
| `token_count` | `integer \| null` | MAY | Size of memory content in tokens. |
| `redacted` | `boolean \| null` | MAY | Whether content was redacted before recording in telemetry. |

---

### 7.9 `handoff`

Populated on `agent.handoff` spans. See Section 5 for pattern definitions.

#### Handoff Patterns

| Pattern | Description | Span Structure |
|---|---|---|
| `orchestrator` | Caller retains control, delegates task, expects result. | `agent.handoff` span CONTAINS target `agent.invoke` as child. |
| `pipeline` | Caller completes, passes state forward, exits. No return expected. | Target `agent.invoke` is in a SEPARATE trace. Link via `links[]` and shared `run.id`. |
| `collaborative` | Peer agents contribute to shared task. No strict caller/callee. | All share same `run.id`. Link to each other via `links[]`. |
| `fallback` | Primary agent failed. Control transfers to backup agent. | `reason.code` MUST be `"error_recovery"`. |

#### `handoff` Fields

| Field | Type | Conformance | Description |
|---|---|---|---|
| `pattern` | `string \| null` | SHOULD | One of the four pattern values above. |
| `reason.code` | `string \| null` | SHOULD | `"specialization"` \| `"capacity"` \| `"policy"` \| `"error_recovery"` \| `"escalation"` \| `"task_complete"` \| `"capability_gap"` \| `"other"` |
| `reason.detail` | `string \| null` | MAY | Free text explanation. |
| `target.agent_id` | `string \| null` | MAY | Receiving agent identifier. |
| `target.agent_name` | `string \| null` | SHOULD | Receiving agent name. |
| `target.framework` | `string \| null` | MAY | Receiving agent framework. |
| `target.endpoint` | `string \| null` | MAY | Network address if target is remote. |
| `target.invoke_span_id` | `string \| null` | MAY | `span_id` of the target agent's `agent.invoke` span if known. |
| `context_transfer.strategy` | `string \| null` | SHOULD | `"full_history"` \| `"summary"` \| `"task_only"` \| `"state_snapshot"` \| `"none"` |
| `context_transfer.summary` | `string \| null` | MAY | Human readable description of transferred context. |
| `context_transfer.included_span_ids` | `array \| null` | MAY | `span_id` values of spans whose outputs were included in context transfer. |
| `context_transfer.token_count` | `integer \| null` | MAY | Size of transferred context in tokens. |
| `context_transfer.truncated` | `boolean \| null` | MAY | Whether context was truncated before transfer. |
| `outcome.accepted` | `boolean \| null` | SHOULD | Whether the target agent accepted the task. |
| `outcome.returned` | `boolean \| null` | SHOULD | Whether control returned to the calling agent. |
| `outcome.return_span_id` | `string \| null` | MAY | `span_id` where control returned — orchestrator pattern. |
| `outcome.result_summary` | `string \| null` | MAY | Brief summary of what the target agent produced. |
| `outcome.failure_reason` | `string \| null` | MAY | `"target_unavailable"` \| `"target_rejected"` \| `"timeout"` \| `"capability_mismatch"` \| `"context_too_large"` \| `"other"` |

---

### 7.10 `human`

Populated on `human.input`, `human.review`, and `human.override` spans.

HITL telemetry is high-sensitivity. See Section 11 for redaction requirements.

#### Per Span Kind Requirements

| Field Group | human.input | human.review | human.override |
|---|---|---|---|
| `trigger.reason` | SHOULD | SHOULD | MUST |
| `trigger.triggered_by` | MAY | SHOULD | MUST |
| `assignment.assignee_role` | SHOULD | MUST | SHOULD |
| `context.shared_content` | MAY | MAY | MAY |
| `response.action` | MUST | MUST | MUST |
| `response.latency_ms` | MUST | SHOULD | SHOULD |
| `response.modified_fields` | N/A | N/A | MUST |
| `response.modified_values` | N/A | N/A | SHOULD |
| `audit.*` | MAY | MAY | SHOULD |

#### `human.trigger`

| Field | Type | Description |
|---|---|---|
| `reason` | `string \| null` | `"confidence_threshold"` \| `"policy_rule"` \| `"tool_risk"` \| `"explicit_request"` \| `"error_recovery"` \| `"escalation"` \| `"scheduled_review"` \| `"other"` |
| `reason_detail` | `string \| null` | Free text elaboration. |
| `triggered_by` | `string \| null` | `span_id` of the span that caused HITL invocation. Links HITL to its cause. |
| `confidence_score` | `number \| null` | Agent confidence 0–1 if threshold-triggered. |

#### `human.assignment`

| Field | Type | Description |
|---|---|---|
| `assignee_id` | `string \| null` | Identifier of the assigned human. Opaque. SHOULD be pseudonymized. |
| `assignee_role` | `string \| null` | `"operator"` \| `"reviewer"` \| `"approver"` \| `"end_user"` \| `"supervisor"` \| `"other"` |
| `queue` | `string \| null` | Work queue or team the task was routed to. |
| `deadline` | `string \| null` | ISO 8601. When a response is required. |
| `notified_at` | `string \| null` | ISO 8601. When the human was notified. |

#### `human.context`

What was presented to the human when HITL was triggered.

| Field | Type | Description |
|---|---|---|
| `prompt` | `string \| null` | The instruction or question posed to the human. |
| `interface` | `string \| null` | `"chat"` \| `"form"` \| `"email"` \| `"dashboard"` \| `"api"` \| `"other"` |
| `options_presented` | `array \| null` | Discrete choices shown to human. |
| `shared_content.agent_output` | `string \| null` | What the agent produced that triggered review — verbatim. |
| `shared_content.agent_reasoning` | `string \| null` | Agent chain-of-thought or explanation if available. |
| `shared_content.tool_results` | `array \| null` | Tool outputs that contributed to this HITL trigger. Each item: `{ tool_name, span_id, output }` |
| `shared_content.conversation_context` | `string \| null` | Relevant conversation history shown to the human. |
| `shared_content.error_detail` | `string \| null` | Error or failure information shown if model or tool failed. |
| `shared_content.confidence_context` | `string \| null` | Confidence scores or uncertainty signals surfaced to human. |
| `shared_content.attachments` | `array \| null` | Files, images, or documents shared. Each item: `{ name, mime_type, uri, size_bytes }` |

#### `human.response`

| Field | Type | Description |
|---|---|---|
| `action` | `string \| null` | `"approved"` \| `"rejected"` \| `"modified"` \| `"escalated"` \| `"abandoned"` \| `"timed_out"` |
| `text` | `string \| null` | Free text response — comments, instructions, corrections. |
| `structured` | `object \| null` | Structured form submission or API response. Shape is caller-defined. |
| `selection` | `string \| null` | Which option was selected from `options_presented`. |
| `responded_at` | `string \| null` | ISO 8601. When the human responded. |
| `latency_ms` | `integer \| null` | Time from human notification to response submission. |
| `modified_fields` | `array \| null` | For `modified` action — which fields the human changed. |
| `modified_values` | `object \| null` | Before/after values: `{ "before": {}, "after": {} }` |

#### `human.audit`

| Field | Type | Description |
|---|---|---|
| `actor_verified` | `boolean \| null` | Whether the human's identity was verified before action. |
| `verification_method` | `string \| null` | `"sso"` \| `"mfa"` \| `"api_key"` \| `"none"` |
| `ip_address` | `string \| null` | Network origin. MAY be redacted. |
| `session_id` | `string \| null` | Human's application session identifier. |
| `signature` | `string \| null` | Cryptographic signature for tamper-evidence. |

---

### 7.11 `guardrail`

Populated on `guardrail.check` spans. For guardrails that fire as instantaneous events within another span, use the `guardrail.triggered` span event instead.

| Field | Type | Conformance | Description |
|---|---|---|---|
| `name` | `string \| null` | SHOULD | Guardrail identifier. |
| `policy` | `string \| null` | MAY | Policy or rule that was evaluated. |
| `triggered` | `boolean \| null` | SHOULD | Whether the guardrail fired. |
| `action` | `string \| null` | SHOULD | `"allow"` \| `"block"` \| `"warn"` \| `"redact"` \| `"escalate"` |
| `score` | `number \| null` | MAY | Confidence score of guardrail evaluation 0–1. |
| `categories` | `array \| null` | MAY | Policy categories evaluated. e.g. `["pii", "toxicity"]` |

---

### 7.12 `evaluation`

Populated on `evaluation.run` spans.

| Field | Type | Conformance | Description |
|---|---|---|---|
| `name` | `string \| null` | SHOULD | Evaluation name. e.g. `"faithfulness"`, `"relevance"`, `"groundedness"` |
| `score` | `number \| null` | SHOULD | Numeric score. |
| `label` | `string \| null` | MAY | Categorical label. e.g. `"pass"`, `"fail"`, `"hallucination"` |
| `passed` | `boolean \| null` | SHOULD | Whether the evaluation passed its threshold. |
| `threshold` | `number \| null` | MAY | Score threshold used for pass/fail determination. |
| `evaluator` | `string \| null` | MAY | What performed the evaluation. e.g. `"gpt-4o"`, `"human"`, `"heuristic"` |
| `explanation` | `string \| null` | MAY | Evaluator reasoning or justification. |

---

### 7.13 `error`

Populated on `error` spans and available as a field on any span for inline error capture.

| Field | Type | Conformance | Description |
|---|---|---|---|
| `type` | `string \| null` | SHOULD | Exception class or error category. |
| `message` | `string \| null` | SHOULD | Human readable error message. |
| `stacktrace` | `string \| null` | MAY | Full stack trace. |
| `code` | `string \| null` | MAY | Provider or system error code. |
| `retryable` | `boolean \| null` | MAY | Whether the operation that failed can be retried. |

---

### 7.14 `input` and `output`

Present on any span that has meaningful input or output content.

```json
"input": {
  "mime_type": "application/json",
  "value": {}
},
"output": {
  "mime_type": "text/plain",
  "value": ""
}
```

`value` is untyped — it MAY be a string, object, or array depending on the span kind. High sensitivity. Apply redaction rules before emitting to external backends.

---

## 8. Span Events

Span events are point-in-time occurrences within a span. They have a timestamp and attributes but no duration and no independent identity. They are not separately queryable spans.

**Decision rule:** If an occurrence has duration, needs to be independently queried, or represents a discrete operation that can fail independently — use a child span. If it is a moment within a span — use a span event.

### Span Event Schema

```json
{
  "name": "string",           // required
  "timestamp": "date-time",   // required  
  "sequence": 0,              // optional — monotonic order within span
  "attributes": {}            // optional — shape defined per event name
}
```

`sequence` provides guaranteed ordering within a span. Use when event ordering matters for debugging (e.g. thought → decision → tool.selected must be in order).

### Event Taxonomy

#### Lifecycle Events (attach to `agent.invoke`)

| Event | Key Attributes | Description |
|---|---|---|
| `agent.started` | — | Agent execution began. |
| `agent.stopped` | `message` | Agent execution ended cleanly. |
| `agent.paused` | `message` | Agent execution suspended awaiting external input. |
| `agent.resumed` | `message` | Agent execution resumed after pause. |

#### Reasoning Events (attach to `agent.turn`)

| Event | Key Attributes | Description |
|---|---|---|
| `thought` | `thought.type`, `thought.content` | Agent internal reasoning step. Captures chain-of-thought. `thought.type`: `"reasoning"` \| `"reflection"` \| `"planning"` \| `"critique"` |
| `plan.created` | `plan.steps[]` | Agent produced an execution plan. |
| `plan.updated` | `plan.steps[]`, `plan.revision_reason` | Agent revised its plan. |
| `decision` | `decision.choice`, `decision.rationale`, `decision.confidence`, `decision.alternatives[]` | Agent chose an action. Captures alternatives considered. |

#### Model Events (attach to `llm.*`)

| Event | Key Attributes | Description |
|---|---|---|
| `stream.first_token` | `latency_ms` | First token received in streaming response. |
| `stream.completed` | `latency_ms` | Streaming response completed. |
| `token.limit.warning` | `token.count`, `token.limit`, `token.percent_used` | Approaching context limit. |
| `content.filtered` | `message` | Model output was filtered by provider. |
| `retry.attempt` | `retry.attempt`, `retry.max_attempts`, `retry.delay_ms`, `retry.reason` | Retry attempted on this operation. |

#### Tool Events (attach to `tool.*`)

| Event | Key Attributes | Description |
|---|---|---|
| `tool.selected` | `tool.name`, `tool.reason` | Agent selected a tool before invocation. |
| `tool.input.validated` | `message` | Tool input passed validation. |
| `tool.output.validated` | `message` | Tool output passed validation. |
| `tool.rate_limited` | `retry.delay_ms` | Tool call was rate limited. |

#### Memory Events (attach to `memory.*`)

| Event | Key Attributes | Description |
|---|---|---|
| `memory.hit` | — | Memory was found. |
| `memory.miss` | — | Memory was not found. |
| `memory.evicted` | `message` | Memory was evicted from store. |

#### Retrieval Events (attach to `retrieval.*`)

| Event | Key Attributes | Description |
|---|---|---|
| `retrieval.reranked` | `message` | Results were reranked after initial retrieval. |
| `retrieval.threshold.miss` | `message` | No results met the similarity threshold. |

#### Guardrail Events (attach to any span)

| Event | Key Attributes | Description |
|---|---|---|
| `guardrail.triggered` | `guardrail.name`, `guardrail.policy`, `guardrail.action`, `guardrail.score` | A guardrail fired instantaneously within another operation. |
| `guardrail.bypassed` | `guardrail.name`, `message` | A guardrail was explicitly bypassed. |

#### Human Events (attach to `human.*`)

| Event | Key Attributes | Description |
|---|---|---|
| `human.notified` | `latency_ms` | Human was notified that action is required. |
| `human.reminded` | `latency_ms` | Follow-up notification sent. |
| `human.escalated` | `message` | Task escalated to a different human or queue. |

#### Graph Events (attach to `agent.step`)

| Event | Key Attributes | Description |
|---|---|---|
| `node.entered` | `node.id`, `node.type` | Execution entered a graph node. |
| `node.exited` | `node.id`, `node.type` | Execution exited a graph node. |
| `edge.traversed` | `edge.from`, `edge.to` | Execution traversed a graph edge. |
| `state.updated` | `state.keys_modified[]` | Graph state was modified. |

#### Evaluation Events (attach to `evaluation.run`)

| Event | Key Attributes | Description |
|---|---|---|
| `score.recorded` | `score.name`, `score.value`, `score.threshold`, `score.passed` | An evaluation score was recorded. |
| `threshold.failed` | `score.name`, `score.value`, `score.threshold` | Score did not meet threshold. |

#### Error Events (attach to any span)

| Event | Key Attributes | Description |
|---|---|---|
| `exception` | `exception.type`, `exception.message`, `exception.stacktrace`, `exception.escaped` | An exception occurred. Follows OTel exception event conventions. |
| `timeout` | `latency_ms`, `message` | Operation timed out. |
| `rate_limit` | `http.status_code`, `retry.delay_ms` | Rate limit encountered. |
| `context.overflow` | `token.count`, `token.limit` | Context window exceeded. |

### Event Naming Convention

| Namespace | Pattern | Example |
|---|---|---|
| Core events | flat name | `thought`, `decision` |
| Extension events | namespaced | `langgraph.checkpoint` |
| Vendor events | vendor prefix | `langfuse.score_updated` |
| Custom events | `x.` prefix | `x.my_custom_event` |

---

## 9. Conformance

### Design Philosophy

> A sparse but accurate trace is more valuable than a complete trace with dummy values. The MUST tier is intentionally minimal — twelve fields, all universally available from any agent runtime without special instrumentation. Developers MUST NOT be pressured to emit placeholder values to satisfy validation.

### Tier 1 — Core Conformant (MUST)

A trace is spec-compliant if and only if every emitted span contains all twelve of the following fields with accurate values:

```
Field                  Acceptable Value
─────────────────────────────────────────────────────────
spec_version           "0.1.0"
event_id               Any non-empty unique string
trace_id               32 hex characters (W3C TraceContext)
span_id                16 hex characters
span_kind              Any value from Section 5 taxonomy
timestamp              ISO 8601 datetime
start_time             ISO 8601 datetime
end_time               ISO 8601 datetime
status                 "ok" | "error" | "cancelled" | "timeout"
resource.service.name  Non-empty string
run.id                 Stable non-empty string (UUID is sufficient)
run.kind               "agent" | "workflow" | "session" | "task"
```

**That's it.** A developer who populates these twelve fields has a conformant trace. Everything else adds value. Nothing else is required.

### Tier 2 — Standard Conformant (SHOULD)

Standard conformance is Core conformance plus all SHOULD fields that are applicable to the span kind and technically available. "Applicable" means the field is relevant to the `span_kind`. "Technically available" means the data exists in the runtime without additional instrumentation.

`parent_span_id` MUST be set on all non-root spans. NULL is only valid on the `agent.invoke` root span.

### Tier 3 — Full Conformant (MAY)

Full conformance is Standard conformance plus MAY fields where technically feasible, with redaction rules documented for all sensitive fields. Suitable for regulated industries, fine-tuning pipelines, and enterprise compliance requirements.

### Conformance Levels Summary

| Level | Requirement | Suitable For |
|---|---|---|
| Core Conformant | All 12 MUST fields | Early-stage agents, batch pipelines, lightweight instrumentation |
| Standard Conformant | Core + applicable SHOULD fields | Production agents, SRE observability |
| Full Conformant | Standard + MAY fields + documented redaction | Regulated industries, fine-tuning, compliance |

### Custom Telemetry Conformance

Implementations that do not use a supported framework adapter and instead emit telemetry directly MUST still satisfy the MUST tier in full. There are no exemptions for custom implementations.

Specifically:

- `spec_version` MUST be set to the version of this spec being implemented.
- `span_kind` MUST be one of the defined taxonomy values. If no taxonomy value fits, use `attributes` for custom operation data and open a spec issue to propose a new kind.
- `run.id` MUST be generated and consistently applied across all spans in a logical run. A UUID generated at run start is sufficient.
- `run.kind` MUST reflect the actual nature of the run. Do not default to `"agent"` if the run is a `"workflow"` or `"task"`.

**On dummy values:** Implementations MUST NOT populate MUST fields with placeholder or fabricated values to satisfy validation. A span with `run.id: "unknown"` or `status: "ok"` on a failed operation is a conformance violation. If a value is genuinely unavailable, omit the field if it is SHOULD or MAY. If it is MUST and unavailable, document why in your adapter specification.

**On `attributes`:** The `attributes` object is the correct home for data that does not map to a defined schema field. Using `attributes` for well-defined fields — for example, placing model name in `attributes.model` instead of `model.name` — is a conformance violation for MUST and SHOULD fields.

---

## 10. Extension and Vendor Namespaces

ATSC is extensible without spec modification. Extension attributes belong in the `attributes` object.

### Namespacing Rules

| Namespace | Owner | Example |
|---|---|---|
| `atsc.*` | Reserved for future spec versions | `atsc.experimental.feature` |
| `gen_ai.*` | OTel GenAI SemConv | `gen_ai.request.model` |
| `aspire.*` | .NET Aspire / Microsoft | `aspire.app_host.name`, `aspire.resource.name` |
| `agent_framework.*` | Microsoft Agent Framework | `agent_framework.agent.name`, `agent_framework.step.name` |
| `semantic_kernel.*` | Semantic Kernel (migrating to AF) | `semantic_kernel.plugin.name`, `semantic_kernel.function.name` |
| `autogen.*` | AutoGen (migrating to AF) | `autogen.conversation_id`, `autogen.agent_name` |
| `{framework}.*` | Framework maintainers | `langgraph.node_id` |
| `{vendor}.*` | Observability platform vendors | `langfuse.trace_id`, `langsmith.run_id` |
| `x.*` | User-defined custom | `x.my_org.custom_field` |

### Microsoft Agent Framework Attribute Mapping

Microsoft Agent Framework is the converged destination for Semantic Kernel and AutoGen. The ATSC adapter targets AF as the primary surface. SK and AutoGen mappings are maintained as compatibility aliases during the migration period.

> Implementations SHOULD use `agent_framework.*` attributes for new AF-native instrumentation. `semantic_kernel.*` and `autogen.*` attributes are supported but considered legacy — they will be marked deprecated once AF reaches GA and the migration period ends.

#### Agent Framework / Semantic Kernel

| SK / AF Activity Tag | ATSC Field | Notes |
|---|---|---|
| `semantic_kernel.function.name` | `tool.name` | For SK/AF function tool calls |
| `agent_framework.function.name` | `tool.name` | AF-native equivalent — prefer over SK tag |
| `semantic_kernel.function.plugin_name` | `attributes.semantic_kernel.plugin_name` | No ATSC core equivalent |
| `agent_framework.plugin.name` | `attributes.agent_framework.plugin_name` | AF-native equivalent |
| `semantic_kernel.function.is_streaming` | `attributes.semantic_kernel.is_streaming` | |
| `gen_ai.system` | `model.provider` | Already OTel GenAI — pass through |
| `gen_ai.request.model` | `model.name` | Already OTel GenAI — pass through |
| `gen_ai.usage.prompt_tokens` | `model.input_tokens` | |
| `gen_ai.usage.completion_tokens` | `model.output_tokens` | |

#### AutoGen

| AutoGen Attribute | ATSC Field | Notes |
|---|---|---|
| `autogen.agent_name` | `executor.name` | |
| `autogen.conversation_id` | `session.id` | AutoGen conversation maps to ATSC session |
| `autogen.message_type` | `attributes.autogen.message_type` | No ATSC core equivalent |
| `autogen.round` | `attributes.autogen.round` | Maps loosely to turn count — use `session.turn_count` |

#### `executor.framework` Values

When emitting from AF, SK, or AutoGen, set `executor.framework` as follows:

| Runtime | `executor.framework` value |
|---|---|
| Microsoft Agent Framework (GA) | `"agent_framework"` |
| Semantic Kernel (pre-AF migration) | `"semantic_kernel"` |
| AutoGen (pre-AF migration) | `"autogen"` |

### Aspire Deployment Targets

ATSC spans are compatible with the following Aspire-connected backends without custom plugins:

| Backend | How | Notes |
|---|---|---|
| Aspire Dashboard | Built-in OTLP receiver | Local dev only. Full span visualization. |
| Azure Monitor / App Insights | `OpenTelemetry.Exporter.AzureMonitor` | Production. ATSC attributes queryable in Log Analytics. |
| Prometheus | `OpenTelemetry.Exporter.Prometheus.AspNetCore` | Metrics only — use for token/cost counters derived from ATSC spans. |
| Jaeger | `OpenTelemetry.Exporter.Jaeger` | Trace visualization. Full ATSC span hierarchy rendered. |
| OTLP Collector | `OpenTelemetry.Exporter.OpenTelemetryProtocol` | Routes to any downstream — Datadog, Honeycomb, Range Point AI, etc. |

### OTel GenAI Attribute Mapping

When emitting to OTLP backends, exporters SHOULD map ATSC attributes to OTel GenAI equivalents where defined. The canonical mapping is maintained in the ATSC repository. Key mappings:

| ATSC Field | OTel GenAI Attribute |
|---|---|
| `model.name` | `gen_ai.request.model` |
| `model.provider` | `gen_ai.system` |
| `model.input_tokens` | `gen_ai.usage.input_tokens` |
| `model.output_tokens` | `gen_ai.usage.output_tokens` |
| `tool.name` | `gen_ai.tool.name` |
| `tool.type` | `gen_ai.tool.type` |

---

## 11. Privacy and Redaction

ATSC telemetry can contain sensitive data. The following fields are classified as high-sensitivity and MUST be handled according to applicable privacy regulations:

### High-Sensitivity Fields

| Field | Risk | Default Recommendation |
|---|---|---|
| `input.value`, `output.value` | May contain PII, credentials, or proprietary content | Redact before external export |
| `human.context.shared_content.*` | Contains agent output and user-facing content | Redact before external export |
| `human.response.text`, `human.response.structured` | Contains human responses | Redact before external export |
| `session.participant.user_id` | User identity | Pseudonymize before external export |
| `session.context.variables` | Session state — may contain any sensitive data | Redact by default |
| `human.audit.ip_address` | Network identity | Redact or pseudonymize |
| `retrieval.results.documents[]` | May contain proprietary knowledge base content | Redact by default |
| `memory.content.*` | Agent memory — may contain any prior sensitive data | Redact by default |

### Redaction Principles

Implementations MUST document which high-sensitivity fields they populate. Exporters routing to external or third-party destinations SHOULD apply redaction rules to high-sensitivity fields before transmission. The `redacted: true` flag on applicable fields signals that content was present but removed before recording.

Redaction is an exporter concern, not a schema concern. The schema records what happened. The exporter decides what leaves the network.

---

## 12. JSON Schema

The full normative JSON Schema for ATSC v0.1.0 is maintained at:

```
https://agent-telemetry-spec.github.io/atsc/schemas/v0.1.0/agent-telemetry-event.json
```

The schema `$id` is:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://agent-telemetry-spec.github.io/atsc/schemas/v0.1.0/agent-telemetry-event.json",
  "title": "Agent Telemetry Semantic Conventions",
  "description": "ATSC v0.1.0 — vendor-neutral schema for AI agent telemetry. Every record is a valid OTel span.",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "spec_version",
    "event_id",
    "span_kind",
    "timestamp",
    "start_time",
    "end_time",
    "status",
    "trace_id",
    "span_id",
    "resource",
    "run"
  ]
}
```

The full schema with all domain objects is maintained separately from this narrative specification. Both are normative. In case of conflict, the JSON Schema is authoritative for field types and patterns. This document is authoritative for semantics and conformance rules.

---

## 13. Examples

### Minimal Core Conformant Span

The simplest possible conformant record — twelve fields, nothing more.

```json
{
  "spec_version": "0.1.0",
  "event_id": "01HXYZ1234567890ABCDEF",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "span_kind": "agent.invoke",
  "timestamp": "2026-03-06T10:00:00.000Z",
  "start_time": "2026-03-06T10:00:00.000Z",
  "end_time": "2026-03-06T10:00:04.823Z",
  "status": "ok",
  "resource": {
    "service.name": "customer-support-agent"
  },
  "run": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "kind": "agent"
  }
}
```

---

### Standard Conformant LLM Call

```json
{
  "spec_version": "0.1.0",
  "event_id": "01HXYZ9876543210FEDCBA",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "a2fb4a1d1a96d312",
  "parent_span_id": "00f067aa0ba902b7",
  "span_kind": "llm.chat",
  "timestamp": "2026-03-06T10:00:00.341Z",
  "start_time": "2026-03-06T10:00:00.341Z",
  "end_time": "2026-03-06T10:00:02.187Z",
  "status": "ok",
  "resource": {
    "service.name": "customer-support-agent",
    "deployment.environment": "production"
  },
  "run": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "kind": "agent",
    "attempt": 1
  },
  "session": {
    "id": "sess_8821_user_42",
    "participant": {
      "user_id": "usr_pseudonymized_x7k2",
      "user_role": "end_user",
      "channel": "web"
    }
  },
  "caller": {
    "type": "agent",
    "id": "support-agent-01",
    "name": "Customer Support Agent",
    "framework": "langgraph"
  },
  "executor": {
    "type": "model",
    "name": "gpt-4o",
    "id": "azure-openai-eastus"
  },
  "model": {
    "provider": "azure_openai",
    "name": "gpt-4o",
    "input_tokens": 842,
    "output_tokens": 156,
    "total_tokens": 998
  },
  "input": {
    "mime_type": "application/json",
    "value": { "role": "user", "content": "What is the status of my order?" }
  },
  "output": {
    "mime_type": "text/plain",
    "value": "I can help you check your order status. Could you provide your order number?"
  },
  "span_events": [
    {
      "name": "thought",
      "timestamp": "2026-03-06T10:00:00.612Z",
      "sequence": 1,
      "attributes": {
        "thought.type": "planning",
        "thought.content": "Need order number before I can query the order system"
      }
    }
  ]
}
```

---

### Human Override Span

```json
{
  "spec_version": "0.1.0",
  "event_id": "01HXYZOVERRIDE123456789",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "c3fb5b1e2b87e423",
  "parent_span_id": "00f067aa0ba902b7",
  "span_kind": "human.override",
  "timestamp": "2026-03-06T10:01:14.000Z",
  "start_time": "2026-03-06T10:01:14.000Z",
  "end_time": "2026-03-06T10:02:47.331Z",
  "status": "ok",
  "resource": {
    "service.name": "customer-support-agent"
  },
  "run": {
    "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "kind": "agent"
  },
  "human": {
    "trigger": {
      "reason": "tool_risk",
      "reason_detail": "Agent attempted to issue refund above automated approval threshold",
      "triggered_by": "b4fc6c2f3c98f534",
      "confidence_score": 0.43
    },
    "assignment": {
      "assignee_role": "approver",
      "queue": "refunds-tier2",
      "notified_at": "2026-03-06T10:01:14.220Z"
    },
    "context": {
      "prompt": "Agent is requesting a $340 refund. This exceeds the $250 automated approval limit. Please review and approve or modify.",
      "interface": "dashboard",
      "shared_content": {
        "agent_output": "I will process a full refund of $340 for order #88291.",
        "agent_reasoning": "Customer provided evidence of non-delivery. Policy allows full refund for non-delivery.",
        "tool_results": [
          {
            "tool_name": "order_lookup",
            "span_id": "b4fc6c2f3c98f534",
            "output": "Order #88291 — status: shipped, carrier: FedEx, last_scan: 14 days ago"
          }
        ],
        "confidence_context": "Confidence score 0.43 — below 0.70 threshold for automated approval"
      },
      "options_presented": ["Approve $340", "Approve $250", "Reject", "Modify amount"]
    },
    "response": {
      "action": "modified",
      "text": "Approving partial refund — carrier shows shipment in transit, not lost. Standard policy applies.",
      "selection": "Approve $250",
      "responded_at": "2026-03-06T10:02:47.000Z",
      "latency_ms": 93000,
      "modified_fields": ["refund.amount"],
      "modified_values": {
        "before": { "refund.amount": 340.00 },
        "after":  { "refund.amount": 250.00 }
      }
    },
    "audit": {
      "actor_verified": true,
      "verification_method": "sso"
    }
  }
}
```

---

### Multi-Agent Handoff (Orchestrator Pattern)

```json
{
  "spec_version": "0.1.0",
  "event_id": "01HXYZHANDOFF987654321",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "d5fd7d3e4da9b645",
  "parent_span_id": "e6fe8e4f5ebacd56",
  "span_kind": "agent.handoff",
  "timestamp": "2026-03-06T10:00:03.100Z",
  "start_time": "2026-03-06T10:00:03.100Z",
  "end_time": "2026-03-06T10:00:09.441Z",
  "status": "ok",
  "resource": {
    "service.name": "research-orchestrator"
  },
  "run": {
    "id": "run_research_q3_2026",
    "kind": "workflow"
  },
  "caller": {
    "type": "agent",
    "id": "orchestrator-01",
    "name": "Research Orchestrator",
    "framework": "langgraph"
  },
  "executor": {
    "type": "agent",
    "id": "analyst-01",
    "name": "Data Analyst Agent",
    "framework": "langgraph"
  },
  "handoff": {
    "pattern": "orchestrator",
    "reason": {
      "code": "specialization",
      "detail": "Task requires quantitative financial analysis"
    },
    "target": {
      "agent_name": "Data Analyst Agent",
      "agent_id": "analyst-01",
      "invoke_span_id": "f7gf9f5g6fbcde67"
    },
    "context_transfer": {
      "strategy": "summary",
      "summary": "Q3 revenue data retrieved. Needs trend analysis and YoY comparison.",
      "included_span_ids": ["a1bc2d3e4f567890", "b2cd3e4f56789012"],
      "token_count": 1840,
      "truncated": false
    },
    "outcome": {
      "accepted": true,
      "returned": true,
      "return_span_id": "g8hg0h6h7hcdef78",
      "result_summary": "YoY growth 12.3%, Q3 outperformed Q2 by 4.1%"
    }
  }
}
```

---

---

### Microsoft Agent Framework on Aspire — `agent.invoke` Root Span

A .NET agent running on Microsoft Agent Framework (migrating from Semantic Kernel) inside an Aspire AppHost, emitting ATSC via the AF adapter. Shows how AF/SK-native attributes map to ATSC fields and how Aspire resource context populates `resource`. The `executor.framework` value is `"semantic_kernel"` here — update to `"agent_framework"` once migrated to AF GA.

```json
{
  "spec_version": "0.1.0",
  "event_id": "01HXYZSKASPIRE123456789",
  "trace_id": "7cf03f4688c45eb7b4df040f1f1e5847",
  "span_id": "b1ec4a2f3b09e521",
  "parent_span_id": null,
  "span_kind": "agent.invoke",
  "timestamp": "2026-03-06T10:00:00.000Z",
  "start_time": "2026-03-06T10:00:00.000Z",
  "end_time": "2026-03-06T10:00:06.312Z",
  "status": "ok",
  "resource": {
    "service.name": "invoice-processing-agent",
    "service.version": "1.4.2",
    "deployment.environment": "production",
    "cloud.provider": "azure",
    "cloud.region": "eastus",
    "aspire.app_host.name": "InvoiceProcessingAppHost",
    "aspire.resource.name": "invoice-agent"
  },
  "run": {
    "id": "run_inv_88291_20260306",
    "kind": "agent",
    "name": "invoice-processing-run",
    "attempt": 1
  },
  "session": {
    "id": "sess_tenant_contoso_20260306",
    "kind": "background",
    "state": "active",
    "participant": {
      "organization_id": "tenant_contoso",
      "user_role": "system",
      "channel": "api"
    }
  },
  "caller": {
    "type": "system",
    "name": "InvoiceQueueProcessor",
    "framework": "aspire"
  },
  "executor": {
    "type": "agent",
    "id": "invoice-agent-01",
    "name": "Invoice Processing Agent",
    "framework": "semantic_kernel",
    "version": "1.30.0"
  },
  "input": {
    "mime_type": "application/json",
    "value": { "invoice_id": "INV-88291", "action": "process" }
  },
  "output": {
    "mime_type": "application/json",
    "value": { "status": "approved", "amount": 4250.00, "gl_code": "5200" }
  },
  "attributes": {
    "aspire.app_host.name": "InvoiceProcessingAppHost",
    "semantic_kernel.plugin.name": "InvoicePlugin",
    "semantic_kernel.function.name": "ProcessInvoice"
  },
  "span_events": [
    {
      "name": "agent.started",
      "timestamp": "2026-03-06T10:00:00.041Z",
      "sequence": 1
    },
    {
      "name": "thought",
      "timestamp": "2026-03-06T10:00:01.220Z",
      "sequence": 2,
      "attributes": {
        "thought.type": "planning",
        "thought.content": "Invoice INV-88291 requires GL code lookup and approval threshold check before processing"
      }
    },
    {
      "name": "decision",
      "timestamp": "2026-03-06T10:00:05.901Z",
      "sequence": 3,
      "attributes": {
        "decision.choice": "approve",
        "decision.rationale": "Amount within automated approval threshold. GL code verified.",
        "decision.confidence": 0.97
      }
    },
    {
      "name": "agent.stopped",
      "timestamp": "2026-03-06T10:00:06.310Z",
      "sequence": 4
    }
  ]
}
```

## 14. Changelog

### v0.1.0 — 2026-03-06

Initial draft specification. Defines:

- Span kind taxonomy (21 kinds across 10 categories)
- Domain objects: resource, session, run, caller, executor, model, tool, retrieval, memory, handoff, human, guardrail, evaluation, error
- Span event taxonomy (40+ events across 9 categories)
- Three-tier conformance model (Core / Standard / Full)
- Custom telemetry conformance clause
- Privacy and redaction guidance
- OTel GenAI attribute mapping table
- Extension and vendor namespace conventions — including `aspire.*`, `agent_framework.*`, `semantic_kernel.*` (legacy), and `autogen.*` (legacy)
- .NET Aspire integration: dashboard visualization, Azure Monitor / App Insights deployment path, AF/SK/AutoGen attribute mapping tables, `executor.framework` values, Aspire-compatible deployment targets
- Microsoft Agent Framework migration note: SK and AutoGen converging to AF, compatibility aliases maintained during migration period
- Five worked examples: minimal span, LLM call, human override, multi-agent handoff, Microsoft Agent Framework on Aspire

---

*Agent Telemetry Semantic Conventions is an independent open specification. Contributions welcome. Intent to propose to the OpenTelemetry Semantic Conventions working group pending implementation adoption.*

*Repository: https://github.com/rangepointai/agent-telemetry-spec*  
*Schema: https://agent-telemetry-spec.github.io/atsc/schemas/v0.1.0/agent-telemetry-event.json*  
*License: Apache 2.0*
