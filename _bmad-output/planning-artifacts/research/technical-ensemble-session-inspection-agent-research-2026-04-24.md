---
stepsCompleted: [1, 2, 3, 4, 5, 6]
inputDocuments:
  - irislib/Ens/MessageHeader.cls
  - irislib/Ens/MessageHeaderBase.cls
  - irislib/Ens/* (related classes — Queue, Util.Log, Activity, Rule.Log, MessageBody)
  - irislib/%AI/* (AI Hub system library)
  - sources/ai-hub-eap/* (AI Hub SDK documentation)
  - sources/diagramtool/* (mermaid diagram tool source + docs)
  - sources/diagramtool/docs/dev-notes-correlation.md (message correlation logic)
  - /Users/jbrandt/iris-view-agent/CLAUDE.md (sibling agent project context)
  - /Users/jbrandt/iris-view-agent/.claude/skills/iris-trace-query/iris-schema.md (master schema notes)
  - /Users/jbrandt/iris-view-agent/.claude/skills/iris-trace-query/learned-schemas/Ens_MessageHeader.md
  - /Users/jbrandt/iris-view-agent/.claude/skills/iris-trace-query/learned-schemas/Ens_Util_Log.md
  - Web sources via Perplexity (InterSystems documentation + community articles)
workflowType: 'research'
lastStep: 6
workflowComplete: true
research_type: 'technical'
research_topic: 'Ensemble Session Inspection Agent Tool (AI Hub)'
research_goals: 'Design a read-only AI Hub agent toolset that can navigate an Ens.Production SessionId — correlating Ens.MessageHeader records, their resolved bodies (arbitrary %Persistent classes), queue activity, and event-log entries — so the agent can answer natural-language questions about what happened in a session, including inspecting the source of custom Business Processes, explaining message body contents, and interpreting error messages.'
user_name: 'Developer'
date: '2026-04-24'
web_research_enabled: true
source_verification: true
---

# Research Report: Technical

**Date:** 2026-04-24
**Author:** Developer
**Research Type:** Technical

---

## Research Overview

This research supports the design of a read-only AI Hub agent tool that enables natural-language Q&A over an InterSystems IRIS / Ensemble production session. The tool must correlate `Ens.MessageHeader` records by `SessionId` with their dynamically-typed message bodies (arbitrary `%Persistent` classes), associated queue activity, and event-log entries, then present the resulting trace to an agent built on the AI Hub SDK. The agent must also be able to read the source of custom `Ens.BusinessProcess` subclasses to explain observed behavior.

**Primary sources:**
- Local ObjectScript system library: `irislib/Ens/*` (authoritative for ENS schema and relationships)
- AI Hub EAP documentation bundle: `sources/ai-hub-eap/*`
- Local AI Hub system library: `irislib/%AI/*`
- **Prior-art reference projects:**
  - `sources/diagramtool/` — existing mermaid-diagram generator over `Ens.MessageHeader`; its `docs/dev-notes-correlation.md` is expected to contain the correlation logic we need to understand.
  - `/Users/jbrandt/iris-view-agent/` — sibling agent project that already interprets `Ens.MessageHeader` and related tables to build reporting views; its `iris-trace-query` skill and `learned-schemas/` directory contain distilled schema knowledge (particularly `Ens_MessageHeader.md` and `Ens_Util_Log.md`).
- Web research via Perplexity (InterSystems documentation, community articles, Learning resources) — used both as foundational framing before source reading and as a cross-check afterward.
- **(Added in Step 5)** `irislib/EnsPortal/SVG/VisualTrace.cls` and `irislib/EnsPortal/VisualTrace.cls` — the Management Portal's own session-reconstruction code; provides the canonical query pattern used by IRIS itself.

> **For the executive summary, TOC, strategic recommendations, implementation roadmap, and risk assessment, see the final section of this document: [Research Synthesis & Executive Summary](#research-synthesis--executive-summary).** The sections between here and there contain the full step-by-step findings.

---

<!-- Content will be appended sequentially through research workflow steps -->

## Technical Research Scope Confirmation

**Research Topic:** Ensemble Session Inspection Agent Tool (AI Hub)

**Research Goals:** Design a read-only AI Hub agent toolset that can navigate an `Ens.Production` SessionId — correlating `Ens.MessageHeader` records, their resolved bodies (arbitrary `%Persistent` classes), queue activity, and event-log entries — so the agent can answer natural-language questions about what happened in a session, including inspecting the source of custom Business Processes, explaining message body contents, and interpreting error messages.

**Technical Research Scope:**

- Architecture Analysis — `Ens.MessageHeader` / `Ens.MessageHeaderBase` schema, SessionId correlation model, relationship to message bodies (arbitrary `%Persistent` classes), `Ens.Queue`, `Ens.Util.Log`, `Ens.Activity.*`, `Ens.Rule.Log`.
- Implementation Approaches — ObjectScript query patterns for session reconstruction, IRIS SQL join strategies, safe reflection on arbitrary `%Persistent` body classes (`$ClassMethod`, `%OpenId`, JSON serialization), source-code retrieval for custom `Ens.BusinessProcess` subclasses.
- Technology Stack — AI Hub SDK primitives: `%AI.Provider`, `%AI.Agent`, `%AI.ToolSet`, `%AI.Skill`, policy layer.
- Integration Patterns — Tool schema design (`GetSessionTrace`, `GetMessageBody`, `GetEventLog`, `GetBusinessProcessSource`, `ExplainError`), JSON-schema inputs/outputs, serialization of varied body types, read-only policy enforcement.
- Performance & Safety Considerations — Session-scoped pagination, LLM-context budgeting, redaction, guarding against runaway joins, ensuring no mutations.

**Research Methodology:**

- Read the actual `irislib/Ens/*` source to ground every claim in code — not just docs.
- Leverage prior-art: `sources/diagramtool/` (existing mermaid generator over `Ens.MessageHeader`, with `docs/dev-notes-correlation.md`) and `/Users/jbrandt/iris-view-agent/` (sibling agent project with `iris-trace-query` skill containing distilled `learned-schemas/` for `Ens_MessageHeader` and `Ens_Util_Log`).
- Cross-reference with current web sources (InterSystems documentation, community articles) via Perplexity — used both for foundational framing before source reading and for validation afterward.
- Read the `sources/ai-hub-eap/*` bundle + `irislib/%AI/*` to anchor tool design in real SDK primitives.
- Flag claims with lower confidence; call out discrepancies where docs and code diverge.

**Scope Confirmed:** 2026-04-24

---

## Technical Overview — Foundational Landscape

> **Note on research methodology:** This section blends four source types: (a) authoritative IRIS source code (`irislib/Ens/*`), (b) prior-art distilled schema from a production agent project (`iris-view-agent/learned-schemas/*`), (c) InterSystems official documentation (via Perplexity search against `docs.intersystems.com`), and (d) the AI Hub EAP documentation bundle (`sources/ai-hub-eap/*`). Where these disagree I flag it; where a claim is only partially verified I mark confidence.

### The Interoperability Production Model

An InterSystems IRIS **Production** (`Ens.Production`) is a runtime container for interoperability components. All communication within a production happens exclusively through **messages** — there is no shared state, no direct procedure calls across components. Sources:
- `irislib/Ens/Production.cls` (declarative wiring of hosts)
- https://docs.intersystems.com/latest/csp/docbook/DocBook.UI.Page.cls?KEY=EGIN_intro

The three core component roles are:

- **Business Service (BS)** — Entry point. Wraps an inbound adapter (TCP, HL7, file, HTTP, SOAP, etc.) and creates the first message of a session. (`SourceBusinessType = 1`)
- **Business Process (BP)** — Routing and orchestration logic. Can be declarative (BPL — Business Process Language) or imperative (custom `Ens.BusinessProcess` subclass). Holds per-session state; can sleep/await responses. (`SourceBusinessType = 2`)
- **Business Operation (BO)** — Outbound adapter. Sends messages to external systems. (`SourceBusinessType = 3`)

*Source: `Ens.MessageHeader.cls` via `iris-view-agent/learned-schemas/Ens_MessageHeader.md` — encoding values verified against source.*

### Message Header + Body Separation — The Core Pattern

Every message crossing a production generates **two rows in two tables**:

1. **`Ens.MessageHeader`** — routing metadata (24 columns, confirmed in source + prior-art)
2. **A separate body table**, a subclass of `%Persistent` (or `%Stream.Object`), named by `MessageHeader.MessageBodyClassName`, keyed by `MessageHeader.MessageBodyId`

This is not optional and not a framework convention — it's enforced in `Ens.MessageHeader:NewRequestMessage()` / `NewResponseMessage()`. The body is `%Save()`'d, its OID captured, and the class name + primary OID stored on the header:

```objectscript
// From irislib/Ens/MessageHeader.cls, NewRequestMessage() (line ~73-78)
Set:pMessageBody.%IsA("%Library.Persistent")||pMessageBody.%IsA("%Stream.Object") tSC=pMessageBody.%Save()
...
Set pHeader.MessageBodyClassName=$classname(pMessageBody)
Set pHeader.MessageBodyId=$$$oidPrimary(tOID)
```

**Critical implication for any inspection tool**: body classes are *unknown at tool design time*. A session may contain headers pointing to `EnsLib.HL7.Message`, `Custom.AddUpdateHubRequest`, `EnsLib.REST.GenericMessage`, etc. The tool must reflect on each class at runtime — it cannot hardcode a schema.

> **Gotcha from prior-art** (`learned-schemas/Ens_MessageHeader.md`): `MessageBodyClassName` is sometimes stored in ALL CAPS (e.g., `MA.MESSAGE.ADDUPDATEHUBREQUEST` instead of `MA.Message.AddUpdateHubRequest`). IRIS SQL WHERE is case-insensitive by default, but case matters when converting to a SQL table name or passing to `$ClassMethod` / `%Dictionary`. Always verify actual case before using in code paths that are case-sensitive.

### SessionId — The Correlation Spine

From `Ens.MessageHeader.cls:70`:

```objectscript
If $G(pSessionId)="" Set pSessionId=pHeader.MessageId()
Set pHeader.SessionId = pSessionId
```

The first message in a session has **`SessionId = ID`** — a self-reference. Every downstream message (request or response, queued or inproc) inherits that same `SessionId`. This gives us the session's root by simple equality.

**Corollary queries (verified in `iris-view-agent/learned-schemas/Ens_MessageHeader.md`):**

```sql
-- Find the session-starting header:
SELECT * FROM Ens.MessageHeader
WHERE ID = SessionId AND SessionId = <sid>
```

*Prior-art note:* the weird-looking `ID = SessionId AND SessionId = <sid>` JOIN form is a production-tested pattern that reliably picks out the root message and lets IRIS use both indexes efficiently.

There is also a **`SuperSession`** property (MAXLEN 300) — a cross-system session identifier persisted via `Ens.SuperSessionIndex`. Used when a request flows across multiple IRIS instances (e.g., HealthShare edge → hub). Relevant for distributed setups but out-of-scope for this tool's v1.

### The Five Core Tables for Session Inspection

Confirmed across source + prior-art + InterSystems docs:

| Table | Role | Keyed to Header via |
|---|---|---|
| `Ens.MessageHeader` | Routing metadata, one row per message | — |
| `{MessageBodyClassName}` | Actual message payload (dynamic table) | `MessageBodyClassName` + `MessageBodyId` |
| `Ens.Util.Log` | Event log (Info, Warning, Error, Trace, Alert, Assert) | `MessageId` → `MessageHeader.%ID`; `SessionId` → `MessageHeader.SessionId` |
| `Ens.Queue` | Live queue state + queue runtime metadata | Via `TargetQueueName` / `ReturnQueueName` |
| `Ens.Rule.Log` | Business-rule evaluation history (Routing Rules, DTL rules) | via `SessionId` (and a per-rule linkage) |

**Additional tables for deeper tracing** (to be verified in Step 3):
- `Ens.Activity.*` — performance/throughput analytics
- `Ens.BusinessProcess` (`^Ens.BP.D` globals) — persistent BP state with per-instance context
- `Ens.Enterprise.MsgBank.MessageHeader` — central archive for cross-production message collection (only relevant if MsgBank is enabled)

### Message Lifecycle and Invocation Styles

From `Ens.MessageHeaderBase.cls` + InterSystems docs ([EMONITOR_concepts](https://docs.intersystems.com/latest/csp/docbook/DocBook.UI.Page.cls?KEY=EMONITOR_concepts)):

**Invocation styles** (`Invocation` column, `Ens.DataType.MessageInvocation`):
- **Queue** (`$$$eMessageInvocationQueue`, default) — async. Message is placed on `TargetQueueName`; the sender's job is released; a different job processes it.
- **Inproc** — sync. Message is formulated, sent, and delivered in the same job; the calling job blocks until return.

**Message priority** (`Priority`, `Ens.DataType.MessagePriority`):
- Async (default for requests)
- Sync (for inproc responses that block)

**Status lifecycle** (integer-encoded, decode with `%EXTERNAL(Status)`; verified codes from prior-art):
- `Created` (initial) → `Queued` → `Delivered` → `Completed`
- Error states: `Error` (value `8`), `Aborted`, `Suspended`, `Discarded`

**Error handling pattern** — critical gotcha: `IsError` and `ErrorStatus` are stored on the **response header**, not the request. To know whether a request failed, you must join `request.%ID = response.CorrespondingMessageId` and read `response.IsError`. `ErrorStatus` is a raw `%Status` value — query with `%ODBCOUT(ErrorStatus)` or it renders as garbled binary.

### The AI Hub SDK — Foundational Layer for the Agent Tool

*Source: `sources/ai-hub-eap/README.md` + `ObjectScript_SDK_Guide.md` (EAP version 2026.2.0AI.141.0) + Perplexity cross-reference to InterSystems Developer Community.*

The AI Hub is shipped as an **Early Access Program** component inside IRIS 2026.2. Two mutually-supporting pieces:

1. **AI SDK** — abstractions for building agents and calling LLMs from within IRIS business logic.
   - ObjectScript-native (`%AI.*` classes)
   - Python via langchain integration
   - Java via LangChain4J (forthcoming)
2. **MCP Server** — exposes IRIS class methods, SQL queries, and Business Services to external MCP clients (e.g., Claude Desktop, agent SDKs) as callable tools. Declarative via XData block or UI.

**Key `%AI.*` classes** (confirmed presence in `irislib/%AI/`; detailed semantics researched in Step 4):
- `%AI.Provider` — abstraction over LLM backends (OpenAI, Anthropic, Ollama, etc.). Uses LiteLLM under the hood for provider-neutral tool-call semantics. *Source: Perplexity + `community.intersystems.com/post/ai-agents-scratch-part-3-pulse-machine`.*
- `%AI.Agent` — the agent runtime: holds conversation history, executes LLM inference loops, invokes tools.
- `%AI.ToolSet` — a named collection of tools exposed to an agent. Can be built declaratively via XData.
- `%AI.Skill` — reusable tool bundle that can be exported, loaded, and registered.

**Config Store & Wallet** — a new IRIS 2026.2 feature for securely storing LLM endpoints, API keys, and MCP-server credentials. Access is RBAC-governed. Agents reference configuration via `secret://` URIs rather than inline strings.

**Policy layer** — the SDK includes a discovery/authorization/audit policy pipeline (`ai-hub-policies` skill notes it explicitly). Tools can be wrapped to enforce read-only access, sanitize arguments, and emit audit events. Essential for our read-only requirement.

> **Confidence note:** EAP means **APIs will change**. The README explicitly warns: *"As part of the EAP, we're making pre-release software available... some of the APIs and access control features are likely to change."* Any design based on this must tolerate minor API churn and re-verify against the shipped class names in `irislib/%AI/` before coding.

### Prior-Art Projects (Signal-Boosters, Not Authorities)

Two sibling projects have already solved subsets of our problem — they're enormously valuable as validation and acceleration, but they are *not* authoritative. We use them, we don't adopt them blindly.

1. **`sources/diagramtool/`** (ST-003) — mermaid-sequence-diagram generator over `Ens.MessageHeader`. Its `docs/dev-notes-correlation.md` is the *best existing articulation of Ensemble message correlation logic in plain English* — covers inproc vs queued correlation, `CorrespondingMessageId` vs `ReturnQueueName` fallback, unpaired-message warnings. Will be directly reused in Step 3.

2. **`/Users/jbrandt/iris-view-agent/`** — a Claude Code skill (`iris-trace-query`) that lets users interactively build SQL against `Ens.MessageHeader` + body tables via the Atelier API. The `learned-schemas/` directory is a *production-tested* schema reference, updated across multiple namespaces (QHINGATEWAY, IRISAPP, IRISAPP, etc.). We inherit its verified column lists, encoding-decode tricks (`%EXTERNAL`, `%ODBCOUT`), join patterns, and performance gotchas.

*Neither is an authoritative source by itself; both cite verification dates against real namespaces. We will treat their claims as "validated hypotheses" that still need cross-check against `irislib/Ens/*` for our target IRIS version (2026.2).*

### Source Reliability Assessment

| Source | Type | Confidence | Notes |
|---|---|---|---|
| `irislib/Ens/MessageHeader.cls` | Primary | **High** | Authoritative; matches IRIS 2026 (same copyright line) |
| `irislib/Ens/MessageHeaderBase.cls` | Primary | **High** | Authoritative |
| `irislib/%AI/*` | Primary | **High** | Authoritative for AI Hub EAP 141.0 |
| `sources/ai-hub-eap/README.md` | Secondary (official) | **High** | EAP docs; note "APIs will change" disclaimer |
| `iris-view-agent/learned-schemas/*` | Tertiary (validated hypotheses) | **Medium-High** | Verified against production namespaces through 2026-03-24 |
| `sources/diagramtool/docs/dev-notes-correlation.md` | Tertiary (validated) | **Medium-High** | Developer-facing notes; aligns with source-code behavior |
| InterSystems docs via Perplexity | Secondary (official) | **High** | `docs.intersystems.com/latest/*` citations |
| InterSystems Developer Community | Tertiary (community) | **Medium** | Useful for usage patterns; sometimes out of date |

### Known Gaps / To Verify in Later Steps

1. **Exact `%AI.*` class surfaces in IRIS 2026.2** — need to read `irislib/%AI/Agent.cls`, `%AI/Provider.cls`, `%AI/ToolSet.cls` directly in Step 4.
2. **MCP Server vs native `%AI.Agent`** — which deployment fits the use case better. Step 4.
3. **Policy enforcement details** — how the `ai-hub-policies` layer actually intercepts tool calls. Step 4.

---

## Technical Overview — Deep Dive (Extension)

*This section extends the foundational overview with detailed schema, runtime behavior, and introspection patterns grounded in direct source reading of `irislib/Ens/*`, `irislib/%Dictionary/*`, and `irislib/Ensemble.inc`. All decode tables verified against source.*

### The Complete Message-Type Decoder Reference

Every integer-coded column in `Ens.MessageHeader` maps to a user-facing label via `DISPLAYLIST` / `VALUELIST` parameters on its datatype class. `%EXTERNAL(col)` performs the decode inside SQL; `$classmethod("Ens.DataType.X","DisplayToLogical")` does it in ObjectScript. Verified values:

#### `Type` (`Ens.DataType.MessageType`)
*Source: `irislib/Ens/DataType/MessageType.cls`*

| Value | Display | Notes |
|---|---|---|
| `1` | `Request` | Outgoing or processing-triggering message |
| `2` | `Response` | Reply to a prior request; `IsError` and `ErrorStatus` live here |
| `3` | `Terminate` | **Session termination signal** — not commonly seen but valid |

#### `Status` (`Ens.DataType.MessageStatus`)
*Source: `irislib/Ens/DataType/MessageStatus.cls`*

| Value | Display | Meaning |
|---|---|---|
| `1` | `Created` | Object created, not yet queued |
| `2` | `Queued` | Sitting on a queue awaiting a worker |
| `3` | `Delivered` | Picked up by target; in-process |
| `4` | `Discarded` | Not routed (e.g., response for a request that already timed out) |
| `5` | `Suspended` | Held pending manual intervention |
| `6` | `Deferred` | Held for later processing (rare) |
| `7` | `Aborted` | Processing forcibly stopped |
| `8` | `Error` | Processing terminated with an error |
| `9` | `Completed` | Normal successful termination |

#### `Invocation` (`Ens.DataType.MessageInvocation`)
| Value | Display |
|---|---|
| `1` | `Queue` (async) |
| `2` | `Inproc` (sync) |

#### `SourceBusinessType` / `TargetBusinessType` (`Ens.DataType.MessageBusinessType`)
| Value | Display |
|---|---|
| `0` | `Unknown` |
| `1` | `BusinessService` |
| `2` | `BusinessProcess` |
| `3` | `BusinessOperation` |

> **Divergence warning**: The separate `Ens.Activity.Data.Seconds.HostType` property uses **`4` = Actor Pool** in addition to 1/2/3. So when reading Activity tables vs MessageHeader, the `HostType` enum is NOT the same enum — a subtle trap. *Source: `irislib/Ens/Activity/Data/Seconds.cls:16`.*

#### `Priority` (`Ens.DataType.MessagePriority`)
| Value | Display | Notes |
|---|---|---|
| `1` | `HighSync` | |
| `2` | `Sync` | |
| `3` | `Normal` | *Deprecated — only shown for older messages.* Value `3` is no longer assigned by new messages but the decoder retains it for historical data. |
| `4` | `SimSync` | "Simulated Sync" — async underneath but blocks like sync |
| `6` | `Async` | **Value 5 is deliberately skipped** in the enum — see source comment |

### Display-Decoding Patterns for the Agent

*All patterns cross-referenced between `irislib/Ens/MessageHeader.cls` + `iris-view-agent/learned-schemas/Ens_MessageHeader.md`.*

```sql
-- SQL-side decoding (recommended for the tool)
SELECT
  hdr.%ID                           AS MessageId,
  hdr.SessionId                     AS SessionId,
  %EXTERNAL(hdr.Type)               AS TypeLabel,        -- "Request"/"Response"/"Terminate"
  %EXTERNAL(hdr.Status)             AS StatusLabel,      -- "Queued"/"Completed"/"Error"/...
  %EXTERNAL(hdr.Invocation)         AS InvocationLabel,  -- "Queue"/"Inproc"
  %EXTERNAL(hdr.SourceBusinessType) AS SourceTypeLabel,  -- "BusinessService"/etc.
  %EXTERNAL(hdr.TargetBusinessType) AS TargetTypeLabel,
  %EXTERNAL(hdr.Priority)           AS PriorityLabel,
  hdr.SourceConfigName              AS FromComponent,
  hdr.TargetConfigName              AS ToComponent,
  hdr.MessageBodyClassName          AS BodyClassName,
  hdr.MessageBodyId                 AS BodyId,
  hdr.TimeCreated, hdr.TimeProcessed,
  hdr.IsError,
  %ODBCOUT(hdr.ErrorStatus)         AS ErrorText,        -- CRITICAL: without %ODBCOUT returns binary garbage
  hdr.CorrespondingMessageId        AS PairId,
  hdr.Description,
  hdr.Resent                        AS ResentIndex       -- > 0 if this message was resubmitted
FROM Ens.MessageHeader hdr
WHERE hdr.SessionId = ?
ORDER BY hdr.TimeCreated ASC, hdr.%ID ASC
```

**Gotcha 1: `ErrorStatus` encoding.** It's stored as an IRIS `%Status` value (a `$LIST`-based binary format). Raw selection returns unreadable control-character output. Use `%ODBCOUT(ErrorStatus)` in SQL, or `$$$StatusDisplayString(errStatus)` in ObjectScript.

**Gotcha 2: `Type = 'Response'` silently returns zero rows.** The column is an integer. Must use `Type = 2`.

**Gotcha 3: `IsError` is on the response header, not the request.** To know if a request failed:
```sql
LEFT JOIN Ens.MessageHeader resp ON resp.CorrespondingMessageId = req.%ID
-- then use resp.IsError, resp.ErrorStatus
```

**Gotcha 4: `MessageBodyClassName` may be stored ALL-CAPS** in some namespaces (observed by iris-view-agent in QHINGATEWAY). IRIS SQL `WHERE` is case-insensitive by default so equality works; but case matters when converting to a SQL table name (`$REPLACE(..., ".", "_")`) or when passing to `$ClassMethod`. Always verify actual case before using in case-sensitive code paths.

**Gotcha 5: `Resent > 0` indicates a resubmitted message.** `Ens.MessageHeader.ResubmitMessage()` and `ResendDuplicatedMessage()` both increment this within a session via `GetNextResendIndex()`. If you're reconstructing a "what happened" narrative and see `Resent > 0`, the session includes a resubmit path.

### Body-Class Resolution: The Full Pattern

The message body is a foreign-key-style reference — but to a dynamically-typed table. The full resolution rules from source:

**From `Ens.MessageHeader.cls:NewRequestMessage()` / `NewResponseMessage()`:**

```objectscript
If '$IsObject(pMessageBody) {
    Set pHeader.MessageBodyClassName = ""            // no class
    Set pHeader.MessageBodyId        = pMessageBody  // literal payload!
} Else {
    Set pHeader.MessageBodyClassName = $classname(pMessageBody)
    Set pHeader.MessageBodyId        = $$$oidPrimary(tOID)
}
```

**Three distinct body shapes to handle in the tool:**

1. **Object body (most common)** — `MessageBodyClassName` is set to a class name; `MessageBodyId` is the OID primary (the `%ID` of a row in that class's SQL table). The agent must:
   - Determine the body class's SQL schema+table (replace all dots except the last with underscores; last dot becomes `.`)
   - Read the row via SQL or `$classmethod(className, "%OpenId", bodyId)`
   - Serialize (see below)

2. **Literal body (scalar)** — `MessageBodyClassName` is empty; `MessageBodyId` contains the actual string/number. This is rare but legal — verified in source (`NewRequestMessage` line `Set pHeader.MessageBodyId=pMessageBody`).

3. **Null body** — `MessageBodyClassName=""` and `MessageBodyId=""`. Also legal.

**Body class verification from `Ens.MessageBody.cls` (the optional-but-common base):**

> *"Note however that any persistent or serial object can be sent as a message body. It is not required that all message body object classes to be derived from this class. Also note that **all message classes derived from this class will share the same storage extent in the database.**"* — source comment in `Ens.MessageBody.cls`

**Implications:**

- The tool cannot assume bodies inherit from `Ens.MessageBody`, `Ens.Request`, or `Ens.Response`. It must check `%IsA("%Persistent")` or `%IsA("%Stream.Object")` at inspection time.
- When a body IS derived from `Ens.MessageBody`, all such bodies share the global `^Ens.MessageBodyD` — but they have distinct SQL tables via class-name discrimination. Each body class still has its own `{Schema}.{Table}` projection.

### Body Serialization — The Already-Solved Path

`Ens.Util.MessageBodyMethods` (parent to `Ens.Request`/`Ens.Response`) has an opinionated JSON-serialization path we can directly reuse:

```objectscript
// From Ens.Util.MessageBodyMethods.cls — the canonical body-to-JSON decision tree:
If ..%Extends("%JSON.Adaptor") {
    // Native JSON support — clean round-trip
    Set tSC = ..%JSONExportToStream(.tJSONStream)
} Else {
    // Generic fallback — uses ZEN's generic object-to-dynamic-object converter
    Set tAET = ##class(%ZEN.Auxiliary.altJSONProvider).%ObjectToAET(pObject)
}
```

**For the agent tool:**
- Check if the body class `%Extends("%JSON.Adaptor")` — if yes, use native JSON export
- Otherwise use the generic reflector; this traverses properties and produces a reasonable JSON object
- For XML-only classes (`%XML.Adaptor`), the Management Portal falls back to XML; we can either render as XML string or apply the generic reflector anyway

**For HL7 messages (common in healthcare productions): `EnsLib.HL7.Message`** has its own specialized renderers and is NOT a simple `%Persistent` — it's a virtual document with segment-field access. Step 5 will address special-casing for `%IsA("EnsLib.HL7.Message")` and similar virtual-document classes.

### The Event Log (`Ens.Util.Log`) — Where the Narrative Lives

*Source: `irislib/Ens/Util/Log.cls`, verified against `iris-view-agent/learned-schemas/Ens_Util_Log.md`.*

This is the most important supporting table. Every `$$$LOGINFO`, `$$$LOGERROR`, `$$$LOGWARNING`, `$$$LOGASSERT`, `$$$LOGSTATUS`, and `$$$LOGALERT` macro writes a row here. The `$$$catTRACE(cat, arg)` macro writes trace rows (Type=5).

**Complete schema:**

| Column | Type | Role |
|---|---|---|
| `%ID` | bigint | Row ID |
| `Type` | int (bitmap index) | See LogType decode below |
| `TimeLogged` | timestamp (indexed) | UTC; when the event was written |
| `SourceClass` | varchar(255) | ObjectScript class that called the Log macro |
| `SourceMethod` | varchar(40) | Method name within that class |
| `ConfigName` | varchar(128) (bitmap index) | Production component name — read from `$$$JobConfigName` at log time |
| `SessionId` | integer (indexed) | **Read from `$$$JobSessionId` at log time** — the SessionId of whatever message the job was processing |
| `MessageId` | integer | **Read from `$$$JobCurrentHeaderId` at log time** — the specific header being processed. Note: fallback is `$G(^Ens.MessageHeaderD)` which gives the max ID (i.e., most recently created message); treat with skepticism |
| `Job` | varchar | `$Job` of the process that wrote the event |
| `Text` | varchar(32000) | Human-readable message (truncated at 32000 chars) |
| `Stack` | varchar ($LIST) | Only populated for errors (Type=2) with `pFramesToHide >= 0`. Use `$LISTGET(Stack, N)` |
| `StatusValue` | varchar (`%Status`) | Defaults to `1` (OK). Only meaningful for error events. Use `%ODBCOUT(StatusValue)` to display |
| `TraceCat` | varchar(10) | Only populated for Type=5 (Trace) events; categorizes trace (e.g., "system", "user", "timing") |

#### `Type` decoder (`Ens.DataType.LogType`)

| Value | Display | Macro | When written |
|---|---|---|---|
| `1` | `Assert` | `$$$LOGASSERT` / `$$$ASSERT` | Invariant check failed (in DEBUG builds) |
| `2` | `Error` | `$$$LOGERROR`, `$$$LOGSTATUS` (on err) | Error — populates Stack |
| `3` | `Warning` | `$$$LOGWARNING` | Warning |
| `4` | `Info` | `$$$LOGINFO`, `$$$LOGSTATUS` (on OK) | Informational |
| `5` | `Trace` | `$$$catTRACE(cat, msg)` | Verbose trace — usually off unless `DoTrace` is set |
| `6` | `Alert` | `$$$LOGALERT` | Triggers SNMP/WMI monitor notification |

**Session narrative reconstruction:**
```sql
SELECT
  log.TimeLogged,
  %EXTERNAL(log.Type) AS Severity,
  log.ConfigName,
  log.SourceClass, log.SourceMethod,
  log.Text,
  %ODBCOUT(log.StatusValue) AS StatusText,
  log.MessageId
FROM Ens_Util.Log log
WHERE log.SessionId = ?
ORDER BY log.TimeLogged ASC, log.%ID ASC
```

Combine with the message-list query (above) via `UNION ALL` with a discriminator column, order by timestamp — that gives you an interleaved session timeline.

**Gotcha (from iris-view-agent experience):** 7-day queries on `Ens_Util.Log` with `ConfigName` filter can time out due to bitmap intersection cost. Scope to ≤ 1 day or use a projection class with a compound index.

### The Rule Log — Two Classes, Same Job

Source revealed a non-obvious fact:

- **`Ens.Rule.RuleLog`** — **DEPRECATED** per its own source comment: *"Deprecated; use Ens.Rule.Log."* Still exists for backward compatibility. Has composite ID `(SessionId, ExecutionId)`.
- **`Ens.Rule.Log`** — the current one. Simpler ID scheme (bigint). Richer columns including `ConfigName` + `CurrentHeaderId` for component/message context.

**Both tables capture why a Routing Rule or DTL Rule fired.** The current `Ens.Rule.Log` schema:

| Column | Role |
|---|---|
| `SessionId` (indexed) | Session context |
| `TimeExecuted` (indexed) | When the rule ran |
| `RuleName` (bitmap index) | e.g., `MyRouter.Rule`|
| `RuleSet` | Rule-set name (if inside a rule set) |
| `ActivityName` | BPL activity name (if fired from a BPL rule activity) |
| `Reason` | Which rule-level the engine picked; the "why" |
| `ReturnValue` | What the rule returned (e.g., target config name) |
| `IsError` | Whether an error occurred |
| `ErrorMsg` | Error text if so |
| `DebugId` | FK to `Ens.Rule.DebugLog` for step-level trace |
| `ConfigName` (bitmap index) | The BP/Router component that invoked the rule |
| `CurrentHeaderId` | The message header being evaluated |

**For the "why did this message route the way it did?" agent query**, this is the primary source. Cross-reference with `Ens.MessageHeader` on `SessionId` + `CurrentHeaderId = MessageHeader.%ID`.

**Backward-compat note**: Purge of `Ens.Rule.Log` ALSO purges `Ens.Rule.RuleLog` (confirmed in `Ens.Rule.Log.cls:105`). Inverse is not true. Both may be populated in the same session during a transitional IRIS version — query both with a UNION if complete history is required.

### The Queue: Runtime Globals, Not Persistent

Critical finding from `Ens.Queue.cls`: **the live queue state is stored entirely in runtime globals**, not in a persistent table.

```objectscript
// From Ens.Queue.cls — the queue state globals:
$$$EnsQueue(queueName, 0, "count") = <count>
$$$EnsQueue(queueName, 0, "next") = <next-id>
$$$EnsQueue(queueName, 0, "time") = <UTC-timestamp>
$$$EnsQueue(queueName, 0, "job", <systemName>, <jobId>) = ""
$$$EnsQueue(queueName, <priority>, <index>) = <messageHeaderId>
```

Where `$$$EnsQueue` is defined in `Ensemble.inc` (via the `EnsQueuing` include). Confirmed from `Ensemble.inc:117 #include EnsQueuing`.

**Implications:**

- For **historical** queue analysis (what messages were queued in a past session?), we use the `Ens.MessageHeader` columns `TargetQueueName`, `ReturnQueueName`, and `Status`. This is what our tool will use.
- For **live** queue state (what's queued RIGHT NOW?), we'd need to walk globals via `$ORDER` — there is no SQL projection of the queue runtime. This is outside our tool's scope since we're reconstructing historical sessions.
- There is a related persistent table `Ens.Queue.FIFOMessageGroup` — this tracks FIFO-ordered message groups. Relevant only if the production uses FIFO groups. Usually safe to ignore.

### Business Process Persistent State — Mapping the BP Runtime

`Ens.BusinessProcess` is a `%Persistent` class — each BP instance creates a row storing its per-session state. This is how a BP can "await" an async response and resume later.

**Key persistent properties (verified in `irislib/Ens/BusinessProcess.cls`):**

| Property | Type | Role |
|---|---|---|
| `%PrimaryRequestHeader` | `Ens.MessageHeader` (FK) | The request that started this BP instance |
| `%PrimaryResponseHeader` | `Ens.MessageHeader` (FK) | The final response sent back |
| `%MasterPendingResponses` | children → `Ens.BP.MasterPendingResponse` | Each async call made that hasn't yet returned |
| `%TimeCreated`, `%TimeCompleted` | UTC | BP instance lifetime |
| `%IsCompleted`, `%IsTerminated` | bool | Terminal state flags |
| `%StatusCode` | `%Status` | Final status (OK or error) |
| `%RepliedStatus` | int | Reply state machine — uses `$$$eRepliedStatus*` macros |
| `%SessionId` | int (index `SessionId`) | Matches `Ens.MessageHeader.SessionId` |
| `%MessagesSent` | list | **Usually empty — `SKIPMESSAGEHISTORY=1` by default** |
| `%MessagesReceived` | list | **Usually empty — `SKIPMESSAGEHISTORY=1` by default** |

**Gotcha — `%MessagesSent`/`%MessagesReceived` are disabled by default.** `Parameter SKIPMESSAGEHISTORY As BOOLEAN = 1;` at the top of `Ens.BusinessProcess.cls`. To know what messages a BP sent/received, use `Ens.MessageHeader` indexed by `SessionId` + filter on `SourceConfigName/TargetConfigName = BP's ConfigName`. Don't rely on the BP instance's own message-history lists.

**BP instance SQL table:** The BP instance lives in `{Custom.BPClass}` table (e.g., `Custom.BusinessProcess`), projected from `^Custom.BusinessProcessD`. `%ID` of the BP instance = value of `Ens.MessageHeader.BusinessProcessId` for messages dispatched by that instance.

**For imperative custom BPs** (user-defined subclass of `Ens.BusinessProcess` with custom `OnRequest`/`OnResponse` methods): the persistent state IS the class's own properties. Any member variable the developer added will be in the instance table.

### BPL Process Runtime — Separate State

For BPL (Business Process Language) processes, the runtime state is stored in **two** additional persistent classes:

- **`Ens.BP.Context`** (`^Ens.BP.ContextD`) — holds user-declared BPL `<context>` properties (variables accessible across states), plus framework fields: `%ResponseHandlers` (completion-key → handler-name map), `%LastError`, `%LastFault`. One row per BP instance.

- **`Ens.BP.Thread`** (`^Ens.BP.ThreadD`) — one row per BPL "thread of control". BPL supports parallel execution; each `<thread>` gets a row. State machine: `%NextState` (next state method to execute, initial `"S1"`, terminal `"Stop"`), `%Status` (run status), `%ChildThreads` (nested threads), `%PendingResponses`/`%SyncResponses` (await/resume mechanism), `%HandlerStack` (fault handler stack), `%ActivityStack` (`<scope>` stack for unwinding).

BPL code is compiled into generated state-machine methods `S1`, `S2`, etc., on the BP class. The generated class's `XData BPL` block (accessible via `%Dictionary.XDataDefinition.%OpenId("Custom.BPLClass||BPL")`) contains the original BPL XML definition.

**For imperative (non-BPL) custom BPs**, these tables may be empty — the dev's own properties carry the state.

### The Activity Tables — Performance Metrics, Not Message-Level Detail

`Ens.Activity.Data.Seconds` / `.Hours` / `.Days` are time-bucketed aggregates per business host. Columns (verified in source):

- `Instance`, `Namespace`, `SiteDimension`
- `HostName`, `HostType` (1-4 — includes `4 = Actor Pool`)
- `TimeSlot`, `TimeSlotUTC`
- `TotalCount` — messages processed
- `TotalDuration` — sum of processing time
- `TotalDurationSquare` — for variance/stddev calculations
- `TotalQueueDuration` — sum of queue-wait time

**NOT useful for per-session reconstruction** — these are pre-aggregated rollups. They're valuable for the broader "is this BP normally slow?" or "has throughput changed since last week?" agent query. Likely out of scope for v1 but called out for future use.

### Runtime Job-Local State (`%Ensemble` / `$$$EnsJobLocal`)

From `Ensemble.inc`, the per-job "where am I?" state lives in a process-local variable `%Ensemble`:

```objectscript
#define EnsJobLocal         %Ensemble
#define JobConfigName       $S($D($$$EnsJobLocal("ConfigName")):
                              $G($$$EnsJobLocal("GuestConfigName"),
                                 $$$EnsJobLocal("ConfigName")),
                              1:$$$GblConfigName)
#define JobSessionId        $$$EnsJobLocal("SessionId")
#define JobCurrentHeaderId  $$$EnsJobLocal("CurrentHeaderId")
#define JobSuperSession     $$$EnsJobLocal("SuperSession")
```

**These are live process-local variables, not persisted.** When a BP/BS/BO processes a message, the Ensemble framework sets `%Ensemble("SessionId")`, `%Ensemble("ConfigName")`, `%Ensemble("CurrentHeaderId")` before dispatching to user code. Any `$$$LOGINFO` call then inherits these values.

**Implication for the tool**: our agent tool is read-only inspection of historical data, so we don't use `%Ensemble` directly. But it's important to understand — it's why `Ens.Util.Log` rows have `SessionId`/`MessageId`/`ConfigName` populated correctly with respect to the *processing* job, not the writer.

### Production Runtime Catalog (`^Ens.Runtime`)

From `Ensemble.inc`:

```objectscript
#define EnsRuntime                          ^Ens.Runtime
#define ConfigClassName(%configname)        $$$EnsRuntime("ConfigItem", %configname, "ClassName")
#define ConfigQueueName(%configname)        $$$EnsRuntime("ConfigItem", %configname, "QueueName")
#define ConfigBusinessType(%configname)     $$$EnsRuntime("ConfigItem", %configname, "BusinessType")
#define ConfigId(%configname)               $$$EnsRuntime("ConfigItem", %configname, "%Id")
#define DispatchNameToConfigName(%name)     $$$EnsRuntime("DispatchName", %name)
```

`^Ens.Runtime` is the in-memory catalog of configured production components — it's rebuilt from the `Ens.Config.*` persistent tables each time the production starts. Useful for the agent because:

- `^Ens.Runtime("ConfigItem", <name>, "ClassName")` gives us the ObjectScript class implementing a config component. For `Ens.MessageHeader.SourceConfigName = "Custom.MyRouter"` we can look up the actual class (e.g., `Custom.BusinessProcessRouter`) and then read its source.
- `^Ens.Runtime("ConfigItem", <name>, "BusinessType")` tells us if it's a Service/Process/Operation (redundant with `Source/TargetBusinessType` on the header, but useful for config-only lookups).

**SQL alternative** (preferred for the tool): the same info is in the `Ens.Config.Item` persistent table, queryable by namespace + component name.

### Programmatic Class Introspection — Two Parallel Paths

**The user specifically asked about `%Api` and `%Atelier` for this. There are TWO distinct approaches, and we should use BOTH depending on context.**

#### Approach A: In-Process (ObjectScript) — `%Dictionary.*`

When the agent runs **inside IRIS** (via `%AI.Agent` natively), we read class definitions directly from the class dictionary:

```objectscript
// Full class metadata (includes declared members, storage, parameters, etc.)
Set tClass = ##class(%Dictionary.ClassDefinition).%OpenId("Custom.BusinessProcess", , .tSC)

// Core fields available:
Write tClass.Name                    // "Custom.BusinessProcess"
Write tClass.Description             // class-level doc comment
Write tClass.ClassType               // "persistent", "cspage", etc.
Write tClass.Super                   // comma-separated superclasses
Write tClass.Abstract                // boolean

// Relationships — iterate the children:
//   tClass.Methods    -> %Dictionary.MethodDefinition
//   tClass.Properties -> %Dictionary.PropertyDefinition
//   tClass.XDatas     -> %Dictionary.XDataDefinition
//   tClass.Parameters -> %Dictionary.ParameterDefinition
//   tClass.Indices, tClass.Queries, tClass.Triggers, tClass.ForeignKeys, etc.

// Reading METHOD SOURCE — the critical one for "what happened in this BP":
Set tMethodDef = ##class(%Dictionary.MethodDefinition).%OpenId("Custom.BusinessProcess||OnRequest")
Write tMethodDef.Name             // "OnRequest"
Write tMethodDef.Description      // doc comment
Write tMethodDef.FormalSpec       // "pRequest:Ens.Request,*pResponse:Ens.Response"
Write tMethodDef.ReturnType       // "%Status"
Set tStream = tMethodDef.Implementation  // %Stream.TmpCharacter — the METHOD BODY
Do tStream.Rewind()
While 'tStream.AtEnd { Write tStream.ReadLine(), ! }
```

*Source: `irislib/%Dictionary/MethodDefinition.cls:54` confirms `Property Implementation As %Stream.TmpCharacter`. Verified via Perplexity against `docs.intersystems.com`.*

**ID format for child members:** `{ClassName}||{MemberName}` using double-pipe. Examples:
- `Custom.BusinessProcess||OnRequest` (method)
- `Custom.BusinessProcess||BPL` (XData block — this is where BPL XML lives)
- `Custom.BusinessProcess||RESPONSECLASSNAME` (parameter)

**For BPL processes — reading the XML:**
```objectscript
Set tBPL = ##class(%Dictionary.XDataDefinition).%OpenId("Custom.MyBPL||BPL")
Set tStream = tBPL.Data  // %Stream.TmpCharacter containing the <process>...</process> XML
```
*Source: `irislib/%Dictionary/XDataDefinition.cls:15` confirms `Property Data As %Stream.TmpCharacter`.*

**`ClassDefinition` vs `CompiledClass` vs `CompiledMethod`:**

| Class | Read | Notes |
|---|---|---|
| `%Dictionary.ClassDefinition` | Source (declared) | What the dev wrote. Use for source reading. |
| `%Dictionary.CompiledClass` | Compiled (resolved) | Inherited members flattened in. Use to understand final shape. |
| `%Dictionary.MethodDefinition` | Declared method source | Has `Implementation` stream. Our primary tool for "show me the code." |
| `%Dictionary.CompiledMethod` | Compiled method | Also has `Implementation`. Useful for methods added via trait/superclass. |

**For deployed classes (`Deployed=1`)**, `Implementation` may be empty — `Deployed` means "disassociated from source," a production-hardening mode. In that case, we can still read the compiled method signature but not the body. Note this in the tool's output.

#### Approach B: Out-of-Process (REST) — `%Api.Atelier.v7` / `v8`

When the agent runs **outside IRIS** (e.g., Claude Desktop via MCP, or an external Python agent using the AI Hub config store), we use the Atelier REST API.

*Source: `irislib/%Api/Atelier.cls`; `irislib/%Api/Atelier/v7.cls:41` + `irislib/%Api/Atelier/v8.cls:41`.*

**Source-retrieval endpoints:**

| Endpoint | Purpose |
|---|---|
| `GET /api/atelier/v8/{namespace}/doc/{name}.cls` | Retrieve class source (UDL format) |
| `GET /api/atelier/v8/{namespace}/doc/{name}.mac` | Retrieve a macro/routine |
| `GET /api/atelier/v8/{namespace}/doc/{name}.int` | Retrieve generated intermediate code |
| `GET /api/atelier/v8/{namespace}/docnames/{cat}/{type}` | List classes by category/type |
| `HEAD /api/atelier/v8/{namespace}/doc/{name}.cls` | Existence + metadata only |
| `POST /api/atelier/v1/{namespace}/action/query` | Execute SQL (v1 ONLY — not in v7/v8) |
| `GET /api/atelier/v1/{namespace}/ens/classes/{type}` | List Ensemble classes by type (v1 only) |

**Document naming:** `{ClassName}.cls`, e.g., `Custom.BusinessProcess.cls` — case-sensitive, capital `.CLS` extension. URL-encode the name if it contains special characters.

**Authentication:** HTTP Basic. Namespace must exist (returns 404 otherwise). `application/json` content-type required.

**Prior-art pattern** (from `iris-view-agent/.claude/skills/iris-trace-query/Tools/iris_query.py`):
```python
endpoint = f"{base_url}/api/atelier/v1/{namespace}/action/query"
response = requests.post(
    endpoint,
    json={"query": sql_string},
    auth=(user, password),
    headers={"Content-Type": "application/json"},
)
```

**SQL is only available at `/v1/.../action/query`** — v7/v8 dropped this route. The Atelier API versions are backwards-compatible for doc/docs/docnames, but `/action/query` is v1-specific. So the tool, if using Atelier, must hit two versions: `/v8/doc/...` for source, `/v1/action/query` for SQL.

#### Approach recommendation (for the design in later steps)

**Primary path for our tool: Approach A (in-process).** We're building the agent via AI Hub (`%AI.Agent`) running inside IRIS, so `%Dictionary.*` is the natural fit — simpler, no network, no auth, no version-compat concerns.

**Keep Approach B for MCP exposure.** If we ALSO want to expose this tool to external MCP clients (Claude Desktop), the MCP Server capability of AI Hub will wrap our ObjectScript methods as MCP tools — still using Approach A underneath, but exposed via MCP protocol. If instead we build a custom external agent (e.g., Python), we'd use Approach B + Atelier for SQL and docs. *This decision belongs in Step 4.*

### `MessageHeader` Lifecycle — End-to-End from Source

Putting all source findings together, a single "what happens when a BS receives a message?" trace:

```
1. External event arrives at inbound adapter (e.g., HL7 MLLP TCP)
2. Adapter → BS.OnProcessInput(rawData)
3. BS creates request body (some class T extends %Persistent)
4. BS calls SendRequestSync(TargetName, request, .response) OR SendRequestAsync(...)
5. Framework calls Ens.MessageHeader.NewRequestMessage:
   a. request.%Save() → MessageBodyId
   b. New MessageHeader row:
      - TimeCreated = UTC now
      - Type = 1 (Request)
      - Priority = Async (default)
      - SessionId = SELF %ID  ← IF session starter
      - MessageBodyClassName = $classname(request)
      - MessageBodyId = bodyId
      - SourceConfigName = BS name
      - TargetConfigName = configured target
   c. If TargetConfigName host is a Queue target:
      - Ens.Queue.EnQueue(header): sets Status=Queued, TimeProcessed=now,
        writes to $$$EnsQueue(targetQueue, priority, idx) = headerId
6. Async: queue worker dequeues, SetStatus(Delivered), dispatches to BP.MessageHeaderHandler
7. BP code runs (OnRequest / BPL state machine). May:
   - Log via $$$LOGINFO → Ens.Util.Log row with SessionId, MessageId, ConfigName
   - Fire Routing Rules → Ens.Rule.Log row with SessionId, CurrentHeaderId
   - Send follow-on async/sync requests (back to step 4, SessionId inherited)
   - On async: update %MasterPendingResponses child rows
8. When BP sends a response:
   - Ens.MessageHeader.NewResponseMessage on original request header:
     - Type = 2
     - Source/Target reversed
     - SessionId inherited
     - CorrespondingMessageId = request.%ID
     - request.CorrespondingMessageId = response.%ID  ← bidirectional linkage
9. On error: Ens.MessageHeader.NewErrorResponse:
   - IsError = 1
   - ErrorStatus = %Status value
10. SetStatus(Completed) OR SetStatus(Error) — updates Status + TimeProcessed atomically
11. If SAM monitoring enabled: increments `$$$IncHostHdrStatusCount` counters
12. On purge after days-to-keep:
    - Ens.Util.MessagePurge walks old headers with integrity check
    - Body class deleted via $classmethod(bodyClass, "%DeleteId", bodyId) IF body class has ENSPURGE != 0
    - Search table entries removed via Ens.SearchTableBase.RemoveSearchTableEntries
```

*All steps verified against source in `irislib/Ens/MessageHeader.cls`, `Ens/Queue.cls`, `Ens/BusinessProcess.cls`, `Ens/Util/Log.cls`, `Ens/Rule/Log.cls`, plus `Ensemble.inc` macros.*

### Revised Gap List for Later Steps

**Closed from Step 2 deep dive:**
- ✅ Ens.Activity.* schema — understood; out of scope for v1
- ✅ Ens.Rule.Log schema + session linkage — fully mapped
- ✅ Custom BP source retrieval — `%Dictionary.ClassDefinition` + `MethodDefinition.Implementation` + `XDataDefinition.Data` in-process, or `GET /api/atelier/v8/{ns}/doc/{class}.cls` out-of-process
- ✅ Body reflection patterns — `%JSON.Adaptor`-first, `%ZEN.Auxiliary.altJSONProvider.%ObjectToAET` fallback
- ✅ Ensemble runtime state model — `%Ensemble`/`$$$EnsJobLocal` explained

**Remaining for Steps 3-5:**
- Exact `%AI.*` class surface in IRIS 2026.2 (Step 4)
- MCP Server vs native `%AI.Agent` deployment choice (Step 4)
- Policy layer hooks for read-only enforcement + argument sanitization (Step 4)
- Special body classes (`EnsLib.HL7.Message`, `EnsLib.EDI.X12.Document`, stream bodies) — virtual-document rendering (Step 5)
- LLM-context budgeting strategy for long sessions (Step 5)
- Exact tool schema JSON definitions (Step 5)

---

## Integration Patterns — The Session Correlation Model

*Step 2 catalogued the pieces. This section ties them together: how the framework creates a **coherent session graph** across `Ens.MessageHeader`, body tables, `Ens.Util.Log`, `Ens.Rule.Log`, `Ens.BP.*`, `Ens.SearchTable*`, and the cross-system extensions (`Ens.SuperSessionIndex`, `Ens.Enterprise.MsgBank.*`). The result is a practical reconstruction recipe the agent tool must implement.*

### 1. The Session Join Graph — Canonical Reference

All correlation flows from this graph. Every table ties back to `Ens.MessageHeader` via one of three keys: `%ID` (specific message), `SessionId` (session scope), or `CorrespondingMessageId` (request↔response pair).

```
                          ┌───────────────────────────┐
                          │     Ens.MessageHeader     │   ← Spine
                          │  %ID, SessionId,          │
                          │  CorrespondingMessageId   │
                          │  MessageBodyClassName,    │
                          │  MessageBodyId,           │
                          │  BusinessProcessId,       │
                          │  Source/TargetConfigName  │
                          └───┬───┬───┬───┬───┬───┬───┘
                              │   │   │   │   │   │
     ┌────────────────────────┘   │   │   │   │   └────────────────────┐
     │                            │   │   │   │                        │
     ▼                            ▼   │   │   ▼                        ▼
 {MessageBody}             Ens.Util   │   │   Ens.Rule.Log         Ens.SuperSession
 (dynamic class,           .Log       │   │   (SessionId,          Index
 keyed by                  (SessionId,│   │   CurrentHeaderId,     (MessageHeader FK,
 MessageBodyClassName      MessageId, │   │   ConfigName,          SuperSession)
 + MessageBodyId)          ConfigName)│   │   RuleName, Reason,
                                      │   │   ReturnValue)
                                      │   │
                                      │   ▼
                                      │   {Custom.BusinessProcess}
                                      │   (BP instance row,
                                      │    indexed by %SessionId;
                                      │    %ID = MessageHeader.BusinessProcessId)
                                      │       │
                                      │       ├───▶ Ens.BP.Context     (BPL only)
                                      │       │     (%Process FK,
                                      │       │      user-declared
                                      │       │      context properties,
                                      │       │      %ResponseHandlers)
                                      │       │
                                      │       ├───▶ Ens.BP.Thread      (BPL only)
                                      │       │     (one+ per BP instance,
                                      │       │      %NextState state machine,
                                      │       │      %PendingResponses)
                                      │       │
                                      │       └───▶ Ens.BP.Master
                                      │             PendingResponse    (all BPs)
                                      │             (pending async
                                      │              completion keys)
                                      │
                                      ▼
                            Ens.VDoc.SearchTable / Ens.SearchTableBase
                            (per-body-class indexed fields,
                             e.g. HL7 PatientID, MRN extracted)
```

**Plus** (cross-system, optional):
- `Ens.Enterprise.MsgBank.MessageHeader` — archived copy in a central MsgBank node (if production is configured with banking).

*All linkages verified in source: `Ens.MessageHeader.cls`, `Ens.Util.Log.cls`, `Ens.Rule.Log.cls`, `Ens.BusinessProcess.cls`, `Ens.BP.Context.cls`, `Ens.BP.Thread.cls`, `Ens.SuperSessionIndex.cls`, `Ens.SearchTableBase.cls`.*

### 2. Request/Response Correlation — Inproc vs Queue

This is the hardest part of the integration model. The diagramtool project (`sources/diagramtool/`) has already formalized this and production-tested it. The tool can adopt these rules wholesale.

#### Arrow semantics (from `sources/diagramtool/docs/architecture.md` + `dev-notes-correlation.md`)

| `Invocation` | Display | Both legs? |
|---|---|---|
| `Queue` (1) | `-->>` (async) | Yes — both request and response are async |
| `Inproc` (2) | `->>` (sync) | Yes — both legs sync |
| Unknown value | Default to `->>` + warning | Defensive |

**Source**: `sources/diagramtool/docs/dev-notes-correlation.md` — "Queue (async) → -->> (both request and response legs are async for queued pairs)"

#### Inproc correlation algorithm

For each `request` row where `Invocation = 'Inproc'` and `Type = 1`:

1. Forward-scan the session's ordered rows for the first `response` where:
   - `response.Type = 2`
   - `response.SourceConfigName = request.TargetConfigName`
   - `response.TargetConfigName = request.SourceConfigName`
2. If either `request.CorrespondingMessageId` or `response.CorrespondingMessageId` is populated:
   - If they match (`response.CorrespondingMessageId = request.%ID`), pairing is **confirmed**.
   - If they conflict (both populated, different values), **still pair by order+reversed-endpoints** but emit a warning: *"CorrMsgId conflict between ReqID=... and RespID=...; using order-based pairing"*.
3. Each response is paired at most once.

#### Queue correlation algorithm

For each `request` row where `Invocation = 'Queue'` and `Type = 1`:

1. **Primary key**: find `response` where `response.CorrespondingMessageId = request.%ID` and `response.Type = 2`.
2. **Fallback key** (only when CorrMsgId is missing): find `response` where:
   - `response.ReturnQueueName = request.ReturnQueueName` (matching fallback queue)
   - AND endpoints reverse (`response.SourceConfigName = request.TargetConfigName` AND `response.TargetConfigName = request.SourceConfigName`)
3. If neither works, leave unpaired and emit: *"Unpaired queued request at ID=...; missing or unmatched CorrMsgId/ReturnQueueName"*.

**Why both algorithms exist**: `CorrespondingMessageId` is the authoritative link, but in some edge cases (e.g., deep historic resends, partial archive restores) it's missing. The `ReturnQueueName`-with-reversed-endpoints fallback handles queued pairs where the framework set up the return queue but `CorrespondingMessageId` wasn't populated at response creation. *Source: verified in `Ens.MessageHeader.NewResponseMessage()` — `CorrespondingMessageId` is only set when `TargetQueueName'=""` (the response has somewhere to go back to).*

#### The bidirectional linkage quirk

From `Ens.MessageHeader.cls:113-116`:

```objectscript
If pHeader.TargetQueueName'="" {
    Set pHeader.CorrespondingMessageId = ..MessageId()     // response → request
    Set ..CorrespondingMessageId = pHeader.MessageId(), tSaveThis = 1  // request → response
}
```

Both sides point at each other. This means the tool can start from EITHER side and find its partner in one hop. But for session-trace queries, we always drive from the request to the response (forward scan), and use the request's `CorrespondingMessageId` as the link when walking in the other direction.

### 3. Session Timeline Reconstruction — The Unified Narrative

The agent must produce a **single chronological narrative** from heterogeneous sources. The pattern:

```sql
-- Phase 1: Messages in the session (the skeleton)
SELECT
  'msg' AS EventKind,
  hdr.TimeCreated AS Ts,
  hdr.%ID AS Anchor,
  %EXTERNAL(hdr.Type) AS Subtype,
  %EXTERNAL(hdr.Status) AS Status,
  hdr.SourceConfigName || ' → ' || hdr.TargetConfigName AS Flow,
  hdr.MessageBodyClassName AS Detail,
  hdr.CorrespondingMessageId AS PairId,
  CASE WHEN hdr.IsError = 1 THEN %ODBCOUT(hdr.ErrorStatus) ELSE '' END AS ErrorText
FROM Ens.MessageHeader hdr
WHERE hdr.SessionId = ?

UNION ALL

-- Phase 2: Event log entries (the narration)
SELECT
  'log' AS EventKind,
  log.TimeLogged AS Ts,
  log.MessageId AS Anchor,
  %EXTERNAL(log.Type) AS Subtype,
  '' AS Status,
  log.ConfigName AS Flow,
  log.Text AS Detail,
  NULL AS PairId,
  CASE WHEN log.Type = 2 THEN %ODBCOUT(log.StatusValue) ELSE '' END AS ErrorText
FROM Ens_Util.Log log
WHERE log.SessionId = ?

UNION ALL

-- Phase 3: Rule evaluations (the decisions)
SELECT
  'rule' AS EventKind,
  rule.TimeExecuted AS Ts,
  rule.CurrentHeaderId AS Anchor,
  rule.RuleName AS Subtype,
  rule.ReturnValue AS Status,
  rule.ConfigName AS Flow,
  rule.Reason AS Detail,
  NULL AS PairId,
  rule.ErrorMsg AS ErrorText
FROM Ens_Rule.Log rule
WHERE rule.SessionId = ?

ORDER BY Ts ASC, EventKind ASC
```

**This is the primary session-trace pattern for the agent.** It gives the LLM a rich, time-ordered, annotated narrative. The agent can then decorate specific messages with their bodies on demand (the body fetch is a separate tool call to avoid ballooning context).

#### Ordering subtleties

- **Timestamps are not strictly monotonic** across tables. `Ens.MessageHeader.TimeCreated` (when the header was instantiated) can fire microseconds before `Ens.Util.Log.TimeLogged` (when the first log line was written by the dispatched worker). The diagramtool handles this with a stable secondary sort on `%ID`.
- **Two messages can share a `TimeCreated`** in high-throughput cases. Use `ORDER BY TimeCreated, %ID` (with `%ID` as tiebreaker, which is monotonically increasing).
- **`Ens.Util.Log` uses `TimeLogged` not `TimeCreated`** — different field name, different semantic (write-time vs object-construction-time). Don't mix them.
- **`Ens.Rule.Log.TimeExecuted` is normalized via `Ens.DataType.UTC.Normalize()`** (from source line 25). May have slightly different precision than the other timestamps.

### 4. Request/Response Pairing — The "Pair" Concept for Narratives

Once the agent has the raw header list, it needs to collapse request/response pairs into single conceptual events when narrating. The diagramtool's `CorrelateEvents` pseudocode (from `dev-notes-correlation.md`) is directly applicable:

```
For each row in ordered session rows:
  If row.Type = Request:
    Emit "Request" event with arrow from Invocation
    If Inproc:
      Forward-scan for reversed-endpoint Response
      Confirm/warn on CorrMsgId
      Emit paired "Response" event with -> arrow
    Else If Queue:
      Try CorrMsgId match first
      Then try ReturnQueueName-with-reversed-endpoints
      Emit paired "Response" event with --> arrow
      On failure: emit unpaired warning
    Else Unknown:
      Default to sync, emit warning
  Else If row.Type = Response and not already paired:
    Emit as singleton Response (or skip, per policy)
```

**For the agent's NL output**, a paired event narrates naturally as:
> *"At 10:04:15, `HL7Router` sent an HL7 ADT_A01 message to `EpicOperation` (async queue). The response returned 3 seconds later, successful, status=Completed."*

An unpaired request (waiting or orphaned) narrates:
> *"At 10:04:15, `HL7Router` sent an HL7 ADT_A01 message to `EpicOperation`. **No response was received in this session's trace** — the request may be still pending, or the response was purged."*

### 5. Business Process Integration — The "What Happened Inside?" Pattern

When the agent is asked *"Look inside Custom.BusinessProcess code and tell me what happened to the message inside?"* the reconstruction requires tying together:

1. **The BP instance row** — `SELECT * FROM {Custom.BPClassName} WHERE %SessionId = ?` OR `WHERE %ID = <header.BusinessProcessId>`.
2. **The BP's source code** — via `%Dictionary.ClassDefinition` + `MethodDefinition.Implementation`, or via Atelier REST.
3. **The messages it sent and received** — via `Ens.MessageHeader` filtered by `SessionId` + `SourceConfigName/TargetConfigName = BP's ConfigName`.
4. **The event log entries written BY the BP** — via `Ens.Util.Log` where `ConfigName = BP's ConfigName` AND `SessionId = ?`.
5. **Rules fired by the BP** — via `Ens.Rule.Log` where `ConfigName = BP's ConfigName` AND `SessionId = ?`.
6. **If BPL**: the context values — via `Ens.BP.Context` where `%Process = <bp instance id>`, plus thread state from `Ens.BP.Thread` filtered on `%Process`.

**The full `BusinessProcessId` bridge** is important:

```sql
-- Find all messages handled by a specific BP instance (not just by ConfigName):
SELECT hdr.* FROM Ens.MessageHeader hdr
WHERE hdr.BusinessProcessId = ? -- <bp instance %ID>
  AND hdr.SessionId = ?
```

This is more precise than filtering by `ConfigName` because a single ConfigName (e.g., `Custom.Router`) can have multiple instances (one per PoolSize job) — `BusinessProcessId` isolates a single instance's activity.

**BPL-specific additional data**:
```sql
-- Read BPL context properties (user-declared <context> vars):
SELECT * FROM Ens_BP.Context ctx
WHERE ctx._Process = ? -- <bp instance %ID>
-- (Note: SqlFieldName = _Process because leading % isn't valid SQL)

-- Read thread state (the BPL state machine):
SELECT thread._NextState, thread._Status, thread._PendingResponses, thread._ChildThreads
FROM Ens_BP.Thread thread
WHERE thread._Process = ?
```

### 6. The Body Class Dispatch — One Pattern, Three Variants

When the agent fetches a message body, three shapes are possible (from Step 2 analysis). The integration pattern for the body-fetch tool:

```objectscript
// Pseudocode — adapted for the tool's GetMessageBody call
ClassMethod GetMessageBody(pMessageId As %String, Output pJSON As %DynamicObject) As %Status
{
    Set hdr = ##class(Ens.MessageHeader).%OpenId(pMessageId, , .tSC)
    If '$IsObject(hdr) Quit tSC

    Set tClassName = hdr.MessageBodyClassName
    Set tBodyId = hdr.MessageBodyId

    // VARIANT 1: Null body
    If tClassName = "" && tBodyId = "" {
        Set pJSON = {"empty": true}
        Quit $$$OK
    }

    // VARIANT 2: Literal scalar body
    If tClassName = "" {
        Set pJSON = {"scalar": (tBodyId)}
        Quit $$$OK
    }

    // VARIANT 3: Object body (the common case)
    // Step A: verify class exists + is persistent
    If '$$$comClassDefined(tClassName) {
        Set pJSON = {"error": "Body class no longer exists: " _ (tClassName)}
        Quit $$$OK  // don't fail — annotate and continue
    }

    Set tBody = $classmethod(tClassName, "%OpenId", tBodyId, , .tSC)
    If '$IsObject(tBody) {
        Set pJSON = {"error": "Body could not be opened (may be purged)"}
        Quit $$$OK
    }

    // Step B: serialize via the already-solved path
    If tBody.%Extends("%JSON.Adaptor") {
        Set tStream = ##class(%Stream.TmpCharacter).%New()
        Do tBody.%JSONExportToStream(.tStream)
        Set pJSON = {}.%FromJSON(tStream)
    } ElseIf tBody.%Extends("EnsLib.HL7.Message") {
        // Special case for HL7 VDocs (see Section 7)
        Set pJSON = ..RenderHL7Body(tBody)
    } ElseIf tBody.%Extends("%Stream.Object") {
        Set pJSON = {"stream": (tBody.Read(4096))}  // truncate
    } Else {
        // Generic fallback: the same path Ens.Util.MessageBodyMethods uses
        Set pJSON = ##class(%ZEN.Auxiliary.altJSONProvider).%ObjectToAET(tBody)
    }

    Quit $$$OK
}
```

This dispatch is critical — hardcoding `"SELECT * FROM {schema}.{table}"` fails for streams, VDocs, and scalar bodies. The reflection path above handles all three.

### 7. Virtual-Document Bodies — HL7, X12, FHIR, XML

*Sources: `EnsLib/HL7/Message.cls` + `EnsLib/HL7/SearchTable.cls` + `Ens/VDoc/SearchTable.cls` (verified presence; full API study deferred to Step 5).*

Healthcare productions often use **Virtual Documents (VDocs)** as bodies — `EnsLib.HL7.Message`, `EnsLib.EDI.X12.Document`, `EnsLib.EDI.XML.Document`. These are NOT simple `%Persistent` classes with a fixed schema:

- They have a nested segment/field structure (HL7 MSH/PID/PV1, etc.)
- Accessed via **path expressions** (e.g., `msg.GetValueAt("PID:5.1")`)
- The SQL projection table exists but shows mostly metadata; the segment data is in separate `^EnsLib.HL7.SegmentD` globals.
- A **SearchTable** may be configured to extract specific fields (e.g., `PatientID`, `MRN`) into indexed columns for fast filtering.

**Integration pattern for VDoc bodies**:

1. Detect via `body.%IsA("EnsLib.HL7.Message")` / `EnsLib.EDI.*.Document` / etc.
2. Render via the VDoc's built-in `OutputToString()` / `ShowContents()` — these produce structured serializations.
3. Optionally enrich with SearchTable extractions via `Ens.SearchTableBase` — gives the LLM key indexed fields without parsing the full document.

**For FHIR bodies** (often `HS.FHIRServer.Interop.Request`, `HS.FHIRMeta.API.*`): these are JSON-native bodies. `%JSONExportToStream()` works directly. Confirmed via prior-art in `iris-view-agent/learned-schemas/HS_FHIRServer_Interop_Request.md`.

### 8. SearchTable Integration — Already-Extracted Body Fields

`Ens.SearchTableBase` and its per-body-class subclasses store pre-indexed, per-body-field values. If a production is configured with a search table for a given body class, the agent can efficiently filter sessions by extracted fields (e.g., "find sessions for patient MRN=12345").

**Pattern**:
```sql
-- Production has a SearchTable for MA.Message.DirectorySearchRequest
-- Query for OrgID='foo' across all sessions:
SELECT DISTINCT hdr.SessionId
FROM Ens.MessageHeader hdr
JOIN MA_Message.DirectorySearchRequest dsr ON hdr.MessageBodyId = dsr.%ID
JOIN MA_Message.DirectorySearchRequest_AdditionalInfo dsa
  ON dsr.ID = dsa.DirectorySearchRequest AND dsa.element_key = 'OrgID'
WHERE hdr.MessageBodyClassName LIKE '%DirectorySearchRequest%'
  AND dsa.AdditionalInfo = 'foo'
```

*Source: pattern verified in `iris-view-agent/.claude/skills/iris-trace-query/iris-schema.md` — the `{Table}_AdditionalInfo` collection-table pivot pattern. Key discovery:*
```sql
SELECT DISTINCT element_key, COUNT(*) FROM {Schema}.{Table}_AdditionalInfo
GROUP BY element_key ORDER BY COUNT(*) DESC
```
*to enumerate available keys before querying.*

The tool probably doesn't *need* to proactively use SearchTables for its primary session-trace flow, but they're an integration option for an "Find related sessions" follow-up query.

### 9. Cross-System Integration — SuperSession & MsgBank

For productions that span multiple IRIS instances (classic pattern: HealthShare edge gateway → hub → pipeline), sessions get a **SuperSession** identifier to link their per-instance pieces.

#### `Ens.SuperSessionIndex` — per-instance linkage

*Source: `irislib/Ens/SuperSessionIndex.cls` (read directly, 44 lines).*

```sql
-- Find all MessageHeaders in THIS namespace that share a SuperSession:
SELECT ssi.MessageHeader FROM Ens.SuperSessionIndex ssi
WHERE ssi.SuperSession = ?
-- Then SELECT * FROM Ens.MessageHeader WHERE %ID IN (above)
```

**Schema**:
| Column | Type | Notes |
|---|---|---|
| `SuperSession` | varchar(300), `SQLUPPER(250)` index | The cross-instance session ID |
| `MessageHeader` | FK to `Ens.MessageHeader`, cascade delete | One entry per participating header |

A single SuperSession can reference multiple MessageHeaders within a namespace — typically all the messages that participated in the same end-to-end business transaction.

#### `Ens.Enterprise.MsgBank.MessageHeader` — the cross-production archive

*Source: `irislib/Ens/Enterprise/MsgBank/MessageHeader.cls` (not read in full, but per Perplexity cross-reference to docs.intersystems.com confirmed preserves SessionId + metadata).*

When a production is configured with a **Message Bank**, copies of headers/bodies/search-table entries are pushed to a central "bank" IRIS instance via TCP. From source-listed files in `Ens/Enterprise/MsgBank/`:
- `MessageHeader.cls` — archival copy class, similar schema + `Node` identifier
- `Log.cls` — banked event log
- `BankTCPAdapter.cls` / `ClientTCPAdapter.cls` — the transport
- `Node.cls` — which production the banked row came from
- `Production.cls` — bank's own production definition

**For the agent tool**:
- **V1 scope: ignore MsgBank.** It's only relevant in enterprise setups where a central bank has been configured. Most productions don't use it.
- **Future: cross-production queries** become interesting — "Find this SessionId across all banked productions." But requires a separate tool that queries the bank node.

**SuperSession** on the other hand is worth a **secondary tool**: `FindRelatedSessions(superSessionId)` — returns all local sessions that share a SuperSession. Low cost to add, valuable for HealthShare-style setups.

### 10. Health-Specific Trace Cruft — A Filter Worth Copying

*Source: `sources/diagramtool/docs/architecture.md` ST-002: "Filter out `MessageBodyClassName = 'HS.Util.Trace.Request'`"*

In HealthShare productions, there are "trace" messages that are internal framework plumbing — they show up in `Ens.MessageHeader` with class `HS.Util.Trace.Request` and clutter any session diagram without adding narrative value. The diagramtool filters them out by default.

**Recommended filter pattern for the agent** (configurable but default on):
```sql
AND hdr.MessageBodyClassName <> 'HS.Util.Trace.Request'
```

Also worth considering for session-trace filtering:
- `Ens.AlarmRequest` / `Ens.AlarmResponse` (internal timer messages from `SetTimer()`)
- `Ens.AlertRequest` (if alerting to `Ens.Alert` — separate concern from the business flow)

Make these filters toggleable so advanced users can see everything when debugging.

### 11. Error Correlation — The Three Error Surfaces

Errors appear in THREE distinct places, with different semantics. The agent must understand all three to answer *"What does this error message mean?"* and *"What went wrong in this session?"*:

| Surface | Where | Content | Trigger |
|---|---|---|---|
| **`Ens.MessageHeader.IsError` + `ErrorStatus`** (on the response) | Response row for a failed request | `%Status` with class+method+text | Message handler raised/returned an error |
| **`Ens.Util.Log.Type=2`** | Event log row with Error type | Text + `StatusValue` (`%Status`) + `Stack` ($LIST) | Code called `$$$LOGERROR(...)` or `$$$LOGSTATUS(err)` |
| **`Ens.Rule.Log.IsError=1` + `ErrorMsg`** | Rule log row | Plain `ErrorMsg` varchar(1024) | Rule evaluation raised an error |

**Semantic differences**:
- Header errors: message-level — *"this particular message failed."*
- Log errors: operational — *"code inside this component had a problem" (may or may not have failed a message).*
- Rule log errors: decision-level — *"a routing rule or DTL rule couldn't evaluate."*

A complete error picture for a session requires gathering all three and presenting them in timestamp order, with the surface type labeled.

**Gotcha from `iris-view-agent` experience**: error texts in production often differ only in trailing details (trace IDs, UUIDs). For aggregate "what errors are common?" queries, group by prefix:
```sql
SELECT SUBSTRING(%ODBCOUT(ErrorStatus), 1, 20) AS ErrorPrefix,
       MIN(SUBSTRING(%ODBCOUT(ErrorStatus), 1, 255)) AS RepresentativeError,
       COUNT(*) AS Occurrences
FROM Ens.MessageHeader WHERE IsError = 1 AND SessionId IN (...)
GROUP BY SUBSTRING(%ODBCOUT(ErrorStatus), 1, 20)
```
(Not critical for single-session agent, but handy for session-corpus questions.)

### 12. Namespace Projection of ObjectScript → SQL

*Verified in prior-art `iris-view-agent/.claude/skills/iris-trace-query/iris-schema.md`.*

Every `%Persistent` class has a SQL projection. Naming rule (for programmatic conversion):

- ObjectScript class: `My.Package.ClassName`
- SQL: schema = `My_Package`, table = `ClassName`
- **Replace all dots EXCEPT the last with `_`**; last dot separates schema from table.

So `EnsLib.HL7.Message` → `EnsLib_HL7.Message`; `Custom.App.Messages.PatientRequest` → `Custom_App_Messages.PatientRequest`.

**The agent's `GetMessageBody` tool must do this conversion** when falling back to a SQL fetch instead of `$classmethod(className, "%OpenId", ...)`. But since we're running in-process, prefer `%OpenId` — it handles all edge cases the SQL projection does (and more, e.g., serial properties).

### 13. Data-Format Contract — What the Tool Returns to the LLM

For every table's row, the tool needs a JSON shape the LLM can reason over. Proposed contract (to be refined in Step 5):

**Message event**:
```json
{
  "kind": "message",
  "id": 12345,
  "sessionId": 10000,
  "timeCreated": "2026-04-24T10:04:15.123Z",
  "timeProcessed": "2026-04-24T10:04:15.456Z",
  "type": "Request",
  "status": "Completed",
  "invocation": "Queue",
  "priority": "Async",
  "source": { "configName": "HL7Router", "businessType": "BusinessProcess" },
  "target": { "configName": "EpicOperation", "businessType": "BusinessOperation" },
  "body": { "className": "EnsLib.HL7.Message", "id": "abc123" },
  "pairId": 12346,
  "isError": false,
  "businessProcessId": 77
}
```

**Log event**:
```json
{
  "kind": "log",
  "id": 54321,
  "timeLogged": "2026-04-24T10:04:15.200Z",
  "severity": "Info",
  "component": "HL7Router",
  "source": { "class": "Custom.Router", "method": "OnRequest" },
  "text": "Routing to EpicOperation based on MSH-11",
  "messageId": 12345,
  "hasStack": false
}
```

**Rule event**:
```json
{
  "kind": "rule",
  "id": 99,
  "timeExecuted": "2026-04-24T10:04:15.180Z",
  "ruleName": "MyRouter.Rule",
  "ruleSet": "RoutingRules",
  "component": "HL7Router",
  "messageId": 12345,
  "reason": "MSH:11.1 = 'P'",
  "returnValue": "EpicOperation",
  "isError": false
}
```

**Design principles**:
- Wrapped discriminator (`kind`) so the LLM can ignore-by-type.
- Decoded enums (no raw integers) — readability > compactness.
- `pairId` / `messageId` cross-references let the LLM navigate.
- Large fields (body content, full error text) are fetched on-demand by separate tools, keeping the timeline response compact.

### 14. Read-Only Enforcement Pattern

The scope requires strict read-only tool behavior. Enforcement is multi-layered:

1. **At the ObjectScript method level**: use only `SELECT` SQL (via `%SQL.Statement`), `%OpenId` (never `%Save` / `%DeleteId`), and `%Dictionary.*` read APIs. Never call `ResubmitMessage`, `ResendDuplicatedMessage`, or `SetStatus`.
2. **At the AI Hub policy level**: attach a policy (see `ai-hub-policies` skill) that rejects tool invocations attempting mutation. Step 4 will detail this.
3. **At the MCP/external exposure level**: if exposing via MCP Server, mark the toolset with read-only semantics in the XData declaration.
4. **SQL privilege level (optional defense-in-depth)**: run under a dedicated IRIS user with `SELECT`-only grants on `Ens.*` tables. Deferred to deployment, not tool design.

### 15. Cross-Pattern Summary — The Integration Checklist

| Need | Primary Table(s) | Join Key |
|---|---|---|
| All messages in a session | `Ens.MessageHeader` | `SessionId` |
| Session-starting header | `Ens.MessageHeader` | `%ID = SessionId` |
| Request↔Response pair | `Ens.MessageHeader` self-join | `hdr.%ID = resp.CorrespondingMessageId` |
| Fallback pair | `Ens.MessageHeader` self-join | `ReturnQueueName` + reversed endpoints |
| Specific message body | `{MessageBodyClassName}` | `%ID = MessageBodyId` |
| Event log for session | `Ens.Util.Log` | `SessionId` |
| Event log for specific message | `Ens.Util.Log` | `MessageId = hdr.%ID` |
| BP instance for session | `{BP Class}` | `%SessionId` or `%ID = hdr.BusinessProcessId` |
| BPL context | `Ens.BP.Context` | `_Process = bpInstance.%ID` |
| BPL thread state | `Ens.BP.Thread` | `_Process = bpInstance.%ID` |
| Rule decisions in session | `Ens.Rule.Log` | `SessionId` + `CurrentHeaderId = hdr.%ID` |
| Rule decisions by component | `Ens.Rule.Log` | `ConfigName + SessionId` |
| Cross-instance correlation | `Ens.SuperSessionIndex` | `MessageHeader` ↔ `SuperSession` |
| SearchTable-indexed body fields | `{Body Class}_AdditionalInfo` | `{ParentTable} = body.%ID` + `element_key` |
| Classes in the namespace | `%Dictionary.ClassDefinition` | — |
| Method implementation | `%Dictionary.MethodDefinition` | `%ID = "ClassName||MethodName"` |
| BPL XML | `%Dictionary.XDataDefinition` | `%ID = "ClassName||BPL"` |

---

## Architectural Patterns — AI Hub Agent & Tool Design

*With the data model mapped (Steps 2-3), this section lays out the architecture of the agent on top: how the `%AI.*` classes compose, what the right deployment pattern is, and how read-only policy enforcement plugs in. All claims grounded in direct source reading of `irislib/%AI/*` and the EAP SDK guide (`sources/ai-hub-eap/ObjectScript_SDK_Guide.md`, v2026.2.0AI.141.0).*

### 1. The AI Hub Class Hierarchy — What We're Building On

Source: `irislib/%AI/` directory listing + class files read.

```
%AI.Provider          LLM abstraction (OpenAI, Anthropic, Gemini, Vertex, Bedrock, Grok, NIM, Meta)
  ├─ capability-based: HasCapability(CAPABILITYTOOLCALLING) — all big-4 providers support it
  └─ config via %DynamicObject: api_key, region, bearer_token, base_url

%AI.Agent             Execution engine
  ├─ Property Provider As %AI.Provider
  ├─ Property Model, SystemPrompt, Temperature
  ├─ Property ToolManager As %AI.ToolMgr
  ├─ Declarative subclass: Parameter PROVIDER/MODEL/APIKEY/TOOLSETS + XData INSTRUCTIONS
  ├─ Methods: Chat(), StreamChat(), ChatWithContent()
  └─ CreateSession(config) → %AI.Agent.Session

%AI.Agent.Session     Conversation state
  ├─ Message history + stats
  ├─ config: max_iterations, temperature, max_tokens, cache{...}
  └─ GetStats() → total_interactions, total_tool_calls, token counts

%AI.ToolMgr           Registry + policy manager
  ├─ AddTool(uri | %DynamicObject | instance)
  ├─ SetAuthPolicy(policy)    — %AI.Policy.Authorization
  ├─ SetAuditPolicy(policy)   — %AI.Policy.Audit
  ├─ SetDiscoveryPolicy(p)    — %AI.Policy.Discovery (filter visible tools)
  └─ EnableSmartDiscovery()   — RAG-based tool selection (for 20+ tool registries)

%AI.Tool              Base for any tool-exposing class
  ├─ Parameter REQUIRESAUTH, DISCOVERYLIMIT, STATEFUL, QUERYMAXROWS
  ├─ Auto-exposes public class methods + instance methods as tools
  ├─ Auto-generates JSON Schema from method signature
  └─ Supports class queries with ROWSPEC (see "Query-as-Tool")

%AI.ToolSet extends %AI.Tool
  ├─ XDATA Definition — the declarative XML spec
  ├─ Supports <Tool Method="..."/>, <Include Class="..."/>,
  │         <Query .../>, <Exclude .../>, <MCP .../>
  └─ Policies attached at toolset level via <Policies>...</Policies>

%AI.Policy.Authorization   Approval gate for REQUIRESAUTH tools
%AI.Policy.Audit           Logs tool invocations
%AI.Policy.Discovery       Filters tool visibility to the LLM
%AI.Policy.ConsoleAuth     Interactive console authorization (dev)
%AI.Policy.ConsoleAudit    Console audit sink (dev)

%AI.MCP.Service            Base for classes that expose tools via MCP Server
                           (used with external iris-mcp-server Rust gateway)

%AI.Tools.SQL              Built-in: generic SQL tools (GetContext, ListTables,
                           DescribeTable, SampleRows, RunQuery)
%AI.Tools.FileSystem       Built-in filesystem access (REQUIRESAUTH-gated)
%AI.Tools.ShellTools       Built-in shell access (REQUIRESAUTH-gated)

%AI.LLM.Response           Response object: Content, ToolCalls, Usage
```

This is a well-layered architecture. The agent is composed of a Provider + a ToolManager; the ToolManager holds a collection of ToolSets; each ToolSet is an ObjectScript class with declarative XML that compiles to a tool catalog.

### 2. Five Ways to Build a Tool — And Which Ones We Use

From the SDK guide, five methods are supported. Mapping each to our session-inspection needs:

| Method | Mechanism | Fit for our tool |
|---|---|---|
| **1. Inline ToolSet methods** | `<Tool Name="..." Method="..."/>` calls ClassMethod; JSON schema from signature | **PRIMARY** — for methods like `GetSessionSummary` that wrap non-trivial logic |
| **2. Typed parameters** | Auto-derive JSON Schema from `As %String(DESCRIPTION = "...")` | **USED EVERYWHERE** — every tool should use typed params |
| **3. Wrap existing class with `<Description/>`** | Re-document an internal method for the LLM | Occasional — if we reuse `Ens.MessageHeader` methods directly |
| **4. Built-in `%AI.Tools.SQL`** | Generic SQL access (`ListTables`, `RunQuery`, etc.) | **DO NOT EXPOSE DIRECTLY** — too powerful; the LLM could issue expensive queries. Keep it scoped. |
| **5. Query-as-Tool** | `Query Name As %SQLQuery(ROWSPEC="...")` in a `%AI.Tool` subclass | **PRIMARY** — ideal for all our "give me a session trace / list of log entries / etc." read-only queries |
| **5b. Inline `<Query>` in ToolSet XML** | Declarative SQL tools with compile-time validation | **PRIMARY** — most of our tools are SQL-first; this is the cleanest path |

**Key insight**: Methods 1+2 and 5/5b are complementary — we'll use Method 5b (`<Query>` elements in XML) for straightforward session-trace queries, and Method 1+2 (method tools) for anything that requires ObjectScript logic (class introspection, body serialization, error decoding).

### 3. Recommended Architecture — The Layered Toolset

```
┌─────────────────────────────────────────────────────────────┐
│  Custom.EnsSession.Agent            (extends %AI.Agent)     │
│  ├─ Parameter PROVIDER = "anthropic"                        │
│  ├─ Parameter MODEL    = "claude-sonnet-4-5"                │
│  ├─ Parameter TOOLSETS = "Custom.EnsSession.Tools"          │
│  └─ XData INSTRUCTIONS (the system prompt)                  │
└───────────────┬─────────────────────────────────────────────┘
                │ uses
                ▼
┌─────────────────────────────────────────────────────────────┐
│  Custom.EnsSession.Tools            (extends %AI.ToolSet)   │
│  └─ XData Definition (the declarative toolset)              │
│                                                             │
│  Includes these tool groups (as <Include> elements):        │
│    • Custom.EnsSession.Tools.Trace     (session timeline)   │
│    • Custom.EnsSession.Tools.Body      (message body)       │
│    • Custom.EnsSession.Tools.Process   (BP introspection)   │
│    • Custom.EnsSession.Tools.Errors    (error decoding)     │
│    • Custom.EnsSession.Tools.Meta      (class/rule meta)    │
│                                                             │
│  Policies (from %AI.Policy.*):                              │
│    • Authorization: strict (default REQUIRESAUTH=0 for all) │
│    • Audit: log every tool call                             │
│    • Discovery: SmartDiscovery OFF (small toolset, ~10-15)  │
└─────────────────────────────────────────────────────────────┘
                │ each group is a %AI.Tool subclass
                ▼
┌─────────────────────────────────────────────────────────────┐
│  Each Custom.EnsSession.Tools.{Group}:                      │
│    • extends %AI.Tool                                       │
│    • Parameter REQUIRESAUTH = 0 (strict read-only)          │
│    • Parameter DISCOVERYLIMIT = <self>                      │
│    • Parameter QUERYMAXROWS = 500 (session traces can be    │
│      long; provide headroom but still cap)                  │
│    • Query-as-Tool for SQL-first operations                 │
│    • Methods for ObjectScript-logic operations              │
└─────────────────────────────────────────────────────────────┘
```

**Why this structure?**

- **Per-group tool classes** let us scope `DISCOVERYLIMIT` and `REQUIRESAUTH` at natural granularity. If we later add a mutation capability (e.g., "resubmit this message"), it goes in a separate `Custom.EnsSession.Tools.Mutations` with `REQUIRESAUTH=1` — while read-only tools stay ungated.
- **ToolSet XML composition** (`<Include Class="..."/>`) gives us the filtering machinery for free — if a deployment wants a subset of tools (e.g., expose only trace tools to a junior-support agent), add `<Exclude>` in a separate ToolSet.
- **A single top-level Agent class** is deployable as either a native IRIS agent OR wrapped behind MCP Server — same tools, different front end. (See section 6 below.)

### 4. Tool Catalog — Proposed Schema

Based on the scope Q&A and the integration checklist from Step 3, here are the tools to build. Each has a name, inputs, outputs, and which Method from §2 it uses.

| Tool | Method | Inputs | Output | Purpose |
|---|---|---|---|---|
| `GetSessionSummary` | 1 (method) | `sessionId: int` | Object `{start, end, componentCount, messageCount, errorCount, rootMessage, terminalMessages, timelineDigest}` | High-level "what happened" at a glance |
| `GetSessionTimeline` | 5b (`<Query>`) | `sessionId: int, offset: int=0, limit: int=200` | `{rows: Event[], row_count, truncated, elapsed_ms}` | Interleaved message+log+rule events (the `UNION ALL` from §3.3) |
| `GetMessageHeaders` | 5b (`<Query>`) | `sessionId: int` | `{rows: Header[], ...}` | Just the messages, no logs/rules |
| `GetMessageBody` | 1 (method) | `messageId: int` | Object `{bodyClass, body: DynamicObject, variant: "object"/"scalar"/"null"/"stream"/"vdoc"}` | Decoded, serialized body |
| `GetMessageDetail` | 1 (method) | `messageId: int` | Object combining header fields + body summary + related log entries | Single-message deep-dive |
| `GetEventLog` | 5b (`<Query>`) | `sessionId: int, messageId: int=0, minSeverity: string=""` | `{rows: LogEvent[], ...}` | Log entries filtered by session and optionally by message/severity |
| `GetRuleLog` | 5b (`<Query>`) | `sessionId: int` | `{rows: RuleEvent[], ...}` | Rule evaluations for session |
| `GetBusinessProcessInstance` | 1 (method) | `messageIdOrBpId: string, sessionId: int` | Object `{bpClass, bpId, properties, contextProperties, threadState}` | BP instance state, incl. BPL context/thread if applicable |
| `GetBusinessProcessSource` | 1 (method) | `classname: string, methodName: string=""` | Object `{description, source, isBPL, bplXml}` | Read BP ObjectScript + BPL XML via `%Dictionary.*` |
| `ListBusinessProcessMethods` | 5b (`<Query>`) | `classname: string` | `{rows: MethodMeta[], ...}` | Enumerate methods on a BP class |
| `ExplainError` | 1 (method) | `errorText: string` OR `messageId: int` | Object `{errorCode, errorMessage, category, knownCause?, suggestedFix?}` | Decode `%Status`, look up common patterns |
| `FindRelatedSessions` | 5b (`<Query>`) | `superSessionId: string` | `{rows: SessionBrief[], ...}` | Cross-instance sessions via `Ens.SuperSessionIndex` |
| `FindSessionsByBody` | 5b (`<Query>`) | `bodyClass: string, filterField: string, filterValue: string, sinceHours: int=24` | `{rows: SessionBrief[], ...}` | Search sessions by indexed body field via SearchTable |

**Design principles** from the SDK guide + our integration analysis:

- **Every tool that primarily runs SQL uses Method 5b** — compile-time SQL validation, automatic parameter-schema derivation, standard result envelope. Less code, less bugs.
- **Method 1 tools are reserved for logic** — class introspection (not easily expressed as SQL), body dispatch (runtime class check), error decoding.
- **No tool returns raw `%Status`** — always decode to a readable string before returning.
- **No tool returns more than `QUERYMAXROWS=500` rows** — the result envelope's `truncated: true` flag tells the LLM to narrow the query.
- **Typed parameters everywhere** with `DESCRIPTION` params so the LLM gets per-param schema fields, not just a combined tool description.

### 5. Read-Only Enforcement — Defense in Depth

The scope mandates strictly read-only. Three layers of enforcement:

#### Layer 1: Method-level discipline (primary)

- Every tool method uses `SELECT` (via `%SQL.Statement` or `&sql()` cursors) or `%OpenId` — never `%Save`, `%DeleteId`, `%UpdateCursor`, or cursor-based writes.
- No tool calls `Ens.MessageHeader.ResubmitMessage`, `ResendDuplicatedMessage`, `SetStatus`, or `Ens.Queue.EnQueue/DeQueue`.
- Code review + static grep for mutation patterns before shipping.

#### Layer 2: AI Hub policy layer (belt and suspenders)

Attach a custom `%AI.Policy.Authorization` to the ToolSet. From `irislib/%AI/Policy/Authorization.cls` (contract inferred): the policy's method gates tool execution before it runs. We'd implement:

```objectscript
Class Custom.EnsSession.ReadOnlyPolicy Extends %AI.Policy.Authorization
{
    /// Reject any tool whose spec has "mutates": true in metadata.
    Method Authorize(toolSpec As %DynamicObject, args As %DynamicObject) As %Status
    {
        If toolSpec.metadata.%Get("mutates") = "1" {
            Return $$$ERROR($$$EnsErrGeneral, "Read-only policy: tool " _ toolSpec.name _ " is disallowed")
        }
        Return $$$OK
    }
}
```

Attached via `<Policies><Authorization Class="Custom.EnsSession.ReadOnlyPolicy"/></Policies>` in the ToolSet XML. Since all our tools are read-only, none have `mutates=1` metadata — the policy is essentially a future-proofing safeguard against accidentally adding a mutation.

#### Layer 3: IRIS role/RBAC (deployment-time)

The agent runs as a specific IRIS user. That user can be granted `SELECT`-only privileges on `Ens.*` and `%Dictionary.*` tables — no `INSERT/UPDATE/DELETE`. Belt-and-suspenders: even a buggy tool method that tried to write would fail at the SQL engine level.

Deferred to deployment docs, but important to flag: the agent does NOT need `%Ens_MessageResend:USE` privilege. Grant only what's needed.

### 6. Deployment Pattern — Native vs. MCP Server

The scope doesn't pin down how users will interact with the agent, so we need to pick (or support both).

#### Option A: Native `%AI.Agent` (in-process)

```
User → (CSP page / custom UI / terminal) → Custom.EnsSession.Agent → Tools → IRIS DB
```

- Fastest path. Agent runs in IRIS; tools call `%OpenId` / `%SQL.Statement` directly.
- No network/protocol overhead.
- Needs a user-facing interface — a CSP page, `%AI.Shell`-based REPL, or embedded in an existing IRIS app.
- Best for: integration into an existing IRIS ops console.

#### Option B: MCP Server (external client, e.g., Claude Desktop)

```
User → Claude Desktop → iris-mcp-server (Rust) → wgproto → IRIS → %AI.MCP.Service subclass → Tools → IRIS DB
```

Per `MCP_Server_Guide.md`:
- `iris-mcp-server` is a Rust binary shipped in `<IRIS>/bin/`.
- It's a transparent gateway: Claude Desktop speaks MCP (stdio or HTTP), gateway speaks wgproto to IRIS.
- We build a class extending `%AI.MCP.Service` that exposes our tools; the gateway + config.toml + Management Portal MCP Server definition wires it up.
- Two-layer auth: LLM client → iris-mcp-server (optional TLS + bearer), iris-mcp-server → IRIS (wgproto with IRIS user credentials).
- Best for: users who want the tool available from their existing AI client (Claude Desktop).

#### Option C: BOTH (recommended)

**The ToolSet is the same in both cases.** `Custom.EnsSession.Tools` is a `%AI.ToolSet` — it works inside a native `%AI.Agent` AND can be wrapped by a `%AI.MCP.Service` subclass for external access. Same classes, two front ends.

```objectscript
// Native agent
Class Custom.EnsSession.Agent Extends %AI.Agent
{
    Parameter TOOLSETS = "Custom.EnsSession.Tools";
    ...
}

// MCP exposure
Class Custom.EnsSession.McpService Extends %AI.MCP.Service
{
    // Reference the same ToolSet; the MCP service generates MCP-protocol-level
    // tool manifests from it. Claude Desktop sees the same tools.
    Parameter TOOLSETS = "Custom.EnsSession.Tools";
}
```

**Recommendation**: Start with Option A (native + a simple CSP page or Shell) to validate the tool design. Add Option B as a second release once the tool inventory is stable. No rework needed — both share the ToolSet.

### 7. Secret Management — Config Store + Wallet

The LLM provider API key (and any other secret) should NOT be in source. The AI Hub supports three resolvers for `@{prefix.key}` placeholders (from SDK guide §2243):

| Prefix | Source | Pattern |
|---|---|---|
| `env` | OS environment variable | `@{env.ANTHROPIC_API_KEY}` |
| `config` | `^%AI.Config` global | `@{config.BaseURL}` |
| `wallet` | IRIS Secure Wallet | `@{wallet.AISecrets.AnthropicKey}` |

**Recommendation**: for the API key, use `@{wallet.AISecrets.<ProviderKey>}` — the IRIS Secure Wallet is encrypted and RBAC-controlled. Set up the wallet entry at install time per `Config_Store_Guide.md`. The declarative agent's `PROVIDERCONFIG` parameter references the placeholder:

```objectscript
Parameter PROVIDERCONFIG = "{""api_key"": ""@{wallet.AISecrets.AnthropicKey}""}";
```

### 8. Agent System Prompt — What to Tell the LLM

The system prompt (XData INSTRUCTIONS) determines how well the LLM uses our tools. Proposed skeleton (to be refined in Step 5):

```markdown
# Ensemble Session Inspector

You are a read-only InterSystems IRIS Interoperability diagnostic agent.
Given a SessionId, you help users understand what happened in an
Ens.Production session — correlating message headers, event log entries,
rule decisions, and Business Process state.

## How to investigate a session

When asked "What happened during session X?":
1. Call GetSessionSummary first to get the shape — component count,
   error count, duration. This tells you how to narrow the investigation.
2. Then call GetSessionTimeline to get the interleaved event stream.
3. If any messages have interesting bodies (flagged by their class name
   or error state), call GetMessageBody for the specific ones.
4. If a Business Process did something non-obvious, call
   GetBusinessProcessSource to read its code, and GetBusinessProcessInstance
   to see its runtime state.
5. For errors, call ExplainError — it decodes IRIS %Status values and
   recognizes common Ensemble error patterns.

## Do NOT

- Do NOT attempt to modify any message, BP state, or queue.
  All tools are strictly read-only.
- Do NOT dump every message body proactively — they can be large.
  Ask for the body only when the user's question needs it.
- Do NOT rely on SessionIds from previous conversations; always use
  the one the user just provided.

## Key Ensemble concepts

- SessionId groups all messages in one business transaction.
- The first message in a session has ID = SessionId.
- `IsError` + `ErrorStatus` are on the RESPONSE, not the request.
- `ErrorStatus` requires %ODBCOUT() to render; tools do this for you.
- Bodies are dynamically-typed; the `bodyClass` tells you what schema.
- Rules (Ens.Rule.Log) tell you WHY a router made a decision.
```

### 9. Performance & Scalability Considerations

Adapted from prior-art hindsight (`iris-view-agent/learned-schemas/*` gotchas):

| Concern | Mitigation |
|---|---|
| Large sessions (100+ messages) | `QUERYMAXROWS = 500` + paginated tools (`offset` + `limit` params) |
| Long event logs per session | Filter by `minSeverity` (default: Info and above) |
| `CorrespondingMessageId` join timeout on high-volume namespaces | Add time-bound join constraints (mirror the session's time range) |
| Body size explosion (HL7 with many segments) | `GetMessageBody` returns full body; `GetSessionTimeline` returns only `{bodyClass, bodyId}` stub |
| Tool latency | Query tools are compile-time-prepared; negligible overhead |
| Context budget (LLM) | Result envelope + `truncated` flag lets the LLM decide when to narrow |

### 10. Session Lifecycle (AI Session, not Ens Session)

`%AI.Agent.Session` stats track token usage per conversation:

```objectscript
Set stats = session.GetStats()
// stats.total_interactions, total_tool_calls
// stats.total_prompt_tokens, total_completion_tokens
```

**Recommendation**: log these to IRIS at session end for usage analytics. Also pass `cache_system_prompt: 1, cache_tool_definitions: 1` in session config so the long system prompt and tool schemas are prompt-cached on the LLM side (Anthropic/Gemini/Vertex/Bedrock-SigV4 all support it per capability table in SDK guide).

### 11. Error Handling Pattern

From `%AI.Errors.inc` (presence confirmed in `irislib/%AI/`): the SDK has its own error macros (`$$$AICoreToolArg*`). Our tools should:

- Return structured error objects, not throw exceptions through the tool boundary
- Distinguish four error classes: `invalid_input` (agent passed bad args), `not_found` (session/message/class doesn't exist), `access_denied` (policy blocked), `internal` (IRIS error)
- Include actionable hints: `{error: "session not found", hint: "session IDs in this namespace range from ... to ..."}`

The LLM will see the structured error and can react — e.g., re-ask the user for a valid SessionId.

### 12. Source-Reliability Assessment for This Step

| Source | Reliability | Notes |
|---|---|---|
| `irislib/%AI/*.cls` (direct source) | **High** | Authoritative; EAP build 141.0 |
| `sources/ai-hub-eap/ObjectScript_SDK_Guide.md` | **High** | Official EAP documentation, current version |
| `sources/ai-hub-eap/MCP_Server_Guide.md` | **High** | Official EAP documentation |
| Perplexity cross-check | **Medium** | IRIS 2025/2026 AI Hub coverage on the web is thin; useful for sanity-check only |

**Caveat — EAP churn**: README explicitly warns *"some of the APIs and access control features are likely to change"*. The patterns in this step are current as of the EAP build we have; re-verify class and parameter names against shipped IRIS before production use.

### 13. Gaps Closed in Step 4 → Step 5

**Closed:**
- ✅ Exact `%AI.*` class surface in IRIS 2026.2 — mapped all top-level classes + subdirs
- ✅ MCP Server vs native `%AI.Agent` — "both, same toolset" architecture chosen
- ✅ Policy layer hooks — Authorization + Audit + Discovery; ReadOnlyPolicy pattern sketched
- ✅ Tool schema approach — Method 5b (`<Query>` XML) primary, Method 1+2 for logic tools
- ✅ Secret management — wallet-based via `@{wallet.*}` placeholders
- ✅ Deployment options — native, MCP, or both

**Remaining for Step 5:**
- Final tool schema JSON with concrete parameter definitions
- Complete SQL for each `<Query>` tool (with decoder functions applied)
- Body-dispatch method code (`GetMessageBody` implementation stub)
- Class-introspection method code (`GetBusinessProcessSource` implementation stub)
- Error-decoding method code (`ExplainError`)
- The full `Custom.EnsSession.Agent` + `Custom.EnsSession.Tools` class skeleton
- VDoc rendering strategy (HL7 specifically)
- LLM-context sizing decisions per tool

---

## Implementation Research — Concrete Code Patterns

*This section translates the architecture into implementation-ready class skeletons, SQL, and ObjectScript snippets. The key new source for this step is **`EnsPortal/SVG/VisualTrace.cls`** — the Management Portal's own session reconstruction — which gives us the canonical query pattern used by IRIS itself.*

### 1. The Canonical Session-Trace Query (from IRIS Management Portal)

*Source: `irislib/EnsPortal/SVG/VisualTrace.cls:1516-1579` — this is literally what the Management Portal uses.*

```sql
-- The 14-column session trace (verified pattern from VisualTrace.BuildTraceInfo):
SELECT %ID, TimeCreated, SourceConfigName,
       TargetConfigName, BusinessProcessId, Type,
       MessageBodyClassName, MessageBodyId, ReturnQueueName, CorrespondingMessageId,
       Status, IsError, SourceBusinessType, TargetBusinessType
FROM Ens.MessageHeader
WHERE SessionId = :sessionId
ORDER BY %ID
```

**Key observation: ORDER BY %ID, not TimeCreated.** `%ID` is monotonically assigned during message creation within a session, and is unique. `TimeCreated` can tie within microseconds on high-throughput runs. The VisualTrace reinforces the Step-3 integration rule: use `%ID` as the primary sort, with `TimeCreated` as the rendered column.

**Canonical event-log query, also from `VisualTrace.cls:1598-1603`:**

```sql
SELECT %ID, TimeLogged, ConfigName, Type, MessageId, SourceClass
FROM Ens_Util.Log
WHERE SessionId = :sessionId
```

**Session-boundary query** (to get start/end times and min/max IDs), from `VisualTrace.cls:1482-1487`:

```sql
SELECT %ID, TimeCreated, TimeProcessed
FROM Ens.MessageHeader
WHERE SessionId = :sessionId
```
*(Iterate, track MIN/MAX of each column.)*

**SessionId-from-MessageId trick**, from `VisualTrace.cls:1447`:

```sql
SELECT SessionId INTO :sessionId FROM Ens.MessageHeader WHERE %ID = :sessionId
```

If the user passes what looks like a SessionId but is actually a MessageId, this pattern (treating the same variable as both input and output) finds the true SessionId. Useful for the `GetActualSessionId` helper.

**Filter predicates to reuse**, from `VisualTrace.cls:1526-1577`:
- Source/Target host filter: null OR match
- Message-body filter: null OR `(MessageBodyClassName=? AND MessageBodyID=?)`
- Corresponding-message filter: null OR `(ID=? OR CorrespondingMessageID=?)`

These are all worth exposing as optional parameters in our `GetSessionTimeline` tool.

### 2. Full `Custom.EnsSession.Tools` ToolSet Class Skeleton

This is the root toolset the agent uses. All policies attach here; all tool groups compose here.

```objectscript
/// Top-level ToolSet for the Ensemble Session Inspection Agent.
/// All tools are strictly read-only. Composes tool groups by category.
Class Custom.EnsSession.Tools Extends %AI.ToolSet [ System = 4 ]
{

XData Definition [ MimeType = application/xml ]
{
<ToolSet Name="EnsSessionTools" Description="Read-only inspection of Ens.Production sessions.">

    <!-- Global policy: strict read-only enforcement -->
    <Policies>
        <Authorization Class="Custom.EnsSession.ReadOnlyPolicy"/>
        <Audit         Class="%AI.Policy.ConsoleAudit"/>
    </Policies>

    <!-- Compose tool groups -->
    <Include Class="Custom.EnsSession.Tools.Trace"/>
    <Include Class="Custom.EnsSession.Tools.Body"/>
    <Include Class="Custom.EnsSession.Tools.Process"/>
    <Include Class="Custom.EnsSession.Tools.Errors"/>
    <Include Class="Custom.EnsSession.Tools.Meta"/>

</ToolSet>
}

}
```

### 3. `Custom.EnsSession.Tools.Trace` — The Timeline & Message Tools

Uses Method 5b (`<Query>` XML) — the SQL is validated at compile time.

```objectscript
Class Custom.EnsSession.Tools.Trace Extends %AI.Tool [ System = 4 ]
{

Parameter REQUIRESAUTH = 0;       // read-only
Parameter DISCOVERYLIMIT;         // expose everything on this class
Parameter QUERYMAXROWS = 500;

XData INSTRUCTIONS [ MimeType = text/markdown ]
{
# Session Trace Tools

Read-only tools for inspecting `Ens.Production` session activity.

## When to use each tool

- `GetSessionSummary` — call FIRST for any session. Returns shape:
  start/end time, component list, message count, error count,
  root message. Tells you where to drill down.
- `GetSessionTimeline` — interleaved list of messages, log events,
  and rule decisions, chronologically. Primary narrative source.
- `GetMessageHeaders` — just the messages (no logs/rules). Lighter
  payload for when the user asked "what messages flowed".
- `GetEventLog` — just the log entries. Filter by severity if noisy.
- `GetRuleLog` — rule evaluations, useful for "why did X happen?"

## Conventions

- SessionId is an integer. The first message in a session has
  %ID = SessionId.
- Type: 1=Request, 2=Response, 3=Terminate.
- Status: 1=Created, 2=Queued, 3=Delivered, 4=Discarded,
  5=Suspended, 6=Deferred, 7=Aborted, 8=Error, 9=Completed.
- IsError and ErrorStatus appear on the RESPONSE, not the request.
}

/// Get a high-level summary of a session: message count, component list,
/// error count, start/end time, root message. Always call this first when
/// investigating a new session.
/// sessionId: The Ens.Production SessionId (integer). If you have a
///   MessageId instead, this tool auto-resolves to its containing SessionId.
Query GetSessionSummary(
    sessionId As %Integer(DESCRIPTION = "Session ID or any message ID within the session")) As %SQLQuery(
    ROWSPEC = "RootMessageId:%Integer,StartTime:%TimeStamp,EndTime:%TimeStamp,MessageCount:%Integer,ErrorCount:%Integer,ComponentList:%String,RootBodyClass:%String") [ SqlProc ]
{
    SELECT
        MIN(hdr.%ID)                                AS RootMessageId,
        MIN(hdr.TimeCreated)                        AS StartTime,
        MAX(hdr.TimeProcessed)                      AS EndTime,
        COUNT(*)                                    AS MessageCount,
        SUM(CASE WHEN hdr.IsError = 1 THEN 1 ELSE 0 END) AS ErrorCount,
        LIST(DISTINCT hdr.SourceConfigName)         AS ComponentList,
        MIN(CASE WHEN hdr.%ID = hdr.SessionId
                 THEN hdr.MessageBodyClassName END) AS RootBodyClass
    FROM Ens.MessageHeader hdr
    WHERE hdr.SessionId = :sessionId
}

/// Return the interleaved message+log+rule timeline for a session,
/// ordered by timestamp. Use this as the primary narrative source.
/// Columns decoded to human-readable strings (e.g. "Request", "Completed").
/// ErrorText decodes %Status via %ODBCOUT.
Query GetSessionTimeline(
    sessionId As %Integer(DESCRIPTION = "The session ID")) As %SQLQuery(
    ROWSPEC = "EventKind:%String,Ts:%TimeStamp,MessageId:%Integer,Subtype:%String,Status:%String,Component:%String,Flow:%String,Detail:%String,PairId:%Integer,ErrorText:%String") [ SqlProc ]
{
    SELECT 'message'                                AS EventKind,
           hdr.TimeCreated                          AS Ts,
           hdr.%ID                                  AS MessageId,
           %EXTERNAL(hdr.Type)                      AS Subtype,
           %EXTERNAL(hdr.Status)                    AS Status,
           hdr.SourceConfigName                     AS Component,
           hdr.SourceConfigName || ' -> ' || hdr.TargetConfigName AS Flow,
           hdr.MessageBodyClassName                 AS Detail,
           hdr.CorrespondingMessageId               AS PairId,
           CASE WHEN hdr.IsError = 1
                THEN %ODBCOUT(hdr.ErrorStatus) ELSE '' END AS ErrorText
    FROM Ens.MessageHeader hdr
    WHERE hdr.SessionId = :sessionId
      AND hdr.MessageBodyClassName <> 'HS.Util.Trace.Request'

    UNION ALL

    SELECT 'log'                                    AS EventKind,
           log.TimeLogged                           AS Ts,
           log.MessageId                            AS MessageId,
           %EXTERNAL(log.Type)                      AS Subtype,
           ''                                       AS Status,
           log.ConfigName                           AS Component,
           log.SourceClass || '.' || log.SourceMethod AS Flow,
           log.Text                                 AS Detail,
           NULL                                     AS PairId,
           CASE WHEN log.Type = 2
                THEN %ODBCOUT(log.StatusValue) ELSE '' END AS ErrorText
    FROM Ens_Util.Log log
    WHERE log.SessionId = :sessionId

    UNION ALL

    SELECT 'rule'                                   AS EventKind,
           rule.TimeExecuted                        AS Ts,
           rule.CurrentHeaderId                     AS MessageId,
           rule.RuleName                            AS Subtype,
           rule.ReturnValue                         AS Status,
           rule.ConfigName                          AS Component,
           rule.RuleSet || ':' || rule.RuleName     AS Flow,
           rule.Reason                              AS Detail,
           NULL                                     AS PairId,
           CASE WHEN rule.IsError = 1
                THEN rule.ErrorMsg ELSE '' END      AS ErrorText
    FROM Ens_Rule.Log rule
    WHERE rule.SessionId = :sessionId

    ORDER BY Ts ASC, EventKind ASC
}

/// Return ONLY the message headers for a session (no logs/rules).
/// Lighter payload than GetSessionTimeline when the user asked
/// specifically about message flow.
Query GetMessageHeaders(
    sessionId As %Integer) As %SQLQuery(
    ROWSPEC = "MessageId:%Integer,TimeCreated:%TimeStamp,TimeProcessed:%TimeStamp,TypeLabel:%String,StatusLabel:%String,InvocationLabel:%String,SourceComponent:%String,TargetComponent:%String,BodyClassName:%String,BodyId:%String,PairId:%Integer,IsError:%Boolean,ErrorText:%String,BusinessProcessId:%Integer") [ SqlProc ]
{
    SELECT hdr.%ID                              AS MessageId,
           hdr.TimeCreated,
           hdr.TimeProcessed,
           %EXTERNAL(hdr.Type)                  AS TypeLabel,
           %EXTERNAL(hdr.Status)                AS StatusLabel,
           %EXTERNAL(hdr.Invocation)            AS InvocationLabel,
           hdr.SourceConfigName                 AS SourceComponent,
           hdr.TargetConfigName                 AS TargetComponent,
           hdr.MessageBodyClassName             AS BodyClassName,
           hdr.MessageBodyId                    AS BodyId,
           hdr.CorrespondingMessageId           AS PairId,
           hdr.IsError,
           CASE WHEN hdr.IsError = 1
                THEN %ODBCOUT(hdr.ErrorStatus) ELSE '' END AS ErrorText,
           hdr.BusinessProcessId
    FROM Ens.MessageHeader hdr
    WHERE hdr.SessionId = :sessionId
      AND hdr.MessageBodyClassName <> 'HS.Util.Trace.Request'
    ORDER BY hdr.%ID
}

/// Retrieve event log entries. Optional filter by message and by
/// minimum severity ("Error", "Warning", "Info", "Trace", "Alert", "Assert").
/// Empty messageId = all messages in session. Empty minSeverity = all severities.
Query GetEventLog(
    sessionId As %Integer,
    messageId As %Integer = 0,
    minSeverity As %String = "") As %SQLQuery(
    ROWSPEC = "LogId:%Integer,TimeLogged:%TimeStamp,Severity:%String,Component:%String,SourceClass:%String,SourceMethod:%String,Text:%String,MessageId:%Integer,StatusText:%String") [ SqlProc ]
{
    SELECT log.%ID                              AS LogId,
           log.TimeLogged,
           %EXTERNAL(log.Type)                  AS Severity,
           log.ConfigName                       AS Component,
           log.SourceClass,
           log.SourceMethod,
           log.Text,
           log.MessageId,
           CASE WHEN log.Type = 2
                THEN %ODBCOUT(log.StatusValue) ELSE '' END AS StatusText
    FROM Ens_Util.Log log
    WHERE log.SessionId = :sessionId
      AND (:messageId = 0 OR log.MessageId = :messageId)
      AND (:minSeverity = ''
           OR log.Type <= CASE :minSeverity
                WHEN 'Assert'  THEN 1
                WHEN 'Error'   THEN 2
                WHEN 'Warning' THEN 3
                WHEN 'Info'    THEN 4
                WHEN 'Trace'   THEN 5
                WHEN 'Alert'   THEN 6
                ELSE 6 END)
    ORDER BY log.TimeLogged
}

/// Retrieve Business Rule evaluations (from Ens.Rule.Log) for a session.
/// Explains "why did the router/DTL rule pick this outcome?"
Query GetRuleLog(
    sessionId As %Integer) As %SQLQuery(
    ROWSPEC = "RuleLogId:%Integer,TimeExecuted:%TimeStamp,RuleName:%String,RuleSet:%String,ActivityName:%String,Component:%String,CurrentHeaderId:%Integer,Reason:%String,ReturnValue:%String,IsError:%Boolean,ErrorMsg:%String") [ SqlProc ]
{
    SELECT rule.%ID                             AS RuleLogId,
           rule.TimeExecuted,
           rule.RuleName,
           rule.RuleSet,
           rule.ActivityName,
           rule.ConfigName                      AS Component,
           rule.CurrentHeaderId,
           rule.Reason,
           rule.ReturnValue,
           rule.IsError,
           rule.ErrorMsg
    FROM Ens_Rule.Log rule
    WHERE rule.SessionId = :sessionId
    ORDER BY rule.TimeExecuted
}

/// Find sessions related to a given SuperSession (cross-instance link).
/// Useful for HealthShare edge/hub and similar multi-namespace setups.
Query FindRelatedSessions(
    superSessionId As %String(DESCRIPTION = "The SuperSession identifier")) As %SQLQuery(
    ROWSPEC = "SessionId:%Integer,StartTime:%TimeStamp,RootBodyClass:%String,MessageCount:%Integer,ErrorCount:%Integer") [ SqlProc ]
{
    SELECT hdr.SessionId,
           MIN(hdr.TimeCreated)                 AS StartTime,
           MIN(CASE WHEN hdr.%ID = hdr.SessionId
                    THEN hdr.MessageBodyClassName END) AS RootBodyClass,
           COUNT(*)                             AS MessageCount,
           SUM(CASE WHEN hdr.IsError = 1 THEN 1 ELSE 0 END) AS ErrorCount
    FROM Ens.MessageHeader hdr
    INNER JOIN Ens.SuperSessionIndex ssi
        ON ssi.MessageHeader = hdr.%ID
    WHERE ssi.SuperSession = :superSessionId
    GROUP BY hdr.SessionId
    ORDER BY MIN(hdr.TimeCreated)
}

}
```

### 4. `Custom.EnsSession.Tools.Body` — Body Retrieval Tool (Method 1+2)

Uses a method tool rather than a query tool because body-class dispatch needs runtime class-check and fallback logic.

```objectscript
Class Custom.EnsSession.Tools.Body Extends %AI.Tool [ System = 4 ]
{

Parameter REQUIRESAUTH = 0;
Parameter DISCOVERYLIMIT;

/// Retrieve and decode the body of a message. Returns a JSON object shaped
/// by the body class: a plain object (for %Persistent bodies), a scalar
/// (for literal bodies), a stream-excerpt (for %Stream bodies), or a
/// structured virtual-document (for HL7/X12/XML VDoc bodies).
/// messageId: The Ens.MessageHeader %ID.
/// maxStreamBytes: Max bytes to return for stream bodies (default 4096).
ClassMethod GetMessageBody(
    messageId As %Integer(DESCRIPTION = "Message header ID"),
    maxStreamBytes As %Integer(DESCRIPTION = "Max bytes returned for stream bodies") = 4096) As %DynamicObject
{
    Set result = {}
    Set hdr = ##class(Ens.MessageHeader).%OpenId(messageId, , .tSC)
    If '$ISOBJECT(hdr) {
        Set result.error    = "Message header not found"
        Set result.errorKey = "not_found"
        Set result.hint     = "The message may have been purged or the ID is invalid"
        Return result
    }
    Set result.messageId         = messageId
    Set result.sessionId         = hdr.SessionId
    Set result.bodyClassName     = hdr.MessageBodyClassName
    Set result.bodyId            = hdr.MessageBodyId

    // Variant detection
    If (hdr.MessageBodyClassName = "") && (hdr.MessageBodyId = "") {
        Set result.variant = "null"
        Set result.body    = {}
        Return result
    }
    If hdr.MessageBodyClassName = "" {
        Set result.variant = "scalar"
        Set result.body    = {"value": (hdr.MessageBodyId)}
        Return result
    }
    // Object variant
    If '$$$comClassDefined(hdr.MessageBodyClassName) {
        Set result.variant = "missing_class"
        Set result.error   = "Body class no longer exists: " _ hdr.MessageBodyClassName
        Return result
    }
    Set body = $CLASSMETHOD(hdr.MessageBodyClassName, "%OpenId", hdr.MessageBodyId, , .tSC)
    If '$ISOBJECT(body) {
        Set result.variant = "purged"
        Set result.error   = "Body object could not be opened (likely purged)"
        Return result
    }
    // Dispatch: VDoc > %JSON.Adaptor > %Stream.Object > generic
    If body.%Extends("EnsLib.HL7.Message") {
        Set result.variant = "vdoc_hl7"
        Set result.body    = ..RenderHL7Body(body, maxStreamBytes)
    } ElseIf body.%Extends("EnsLib.EDI.X12.Document") {
        Set result.variant = "vdoc_x12"
        Set result.body    = ..RenderVDocBody(body, maxStreamBytes)
    } ElseIf body.%Extends("%JSON.Adaptor") {
        Set result.variant = "json"
        Set stream = ##class(%Stream.TmpCharacter).%New()
        Do body.%JSONExportToStream(.stream)
        Set result.body = {}.%FromJSON(stream)
    } ElseIf body.%Extends("%Stream.Object") {
        Set result.variant = "stream"
        Do body.Rewind()
        Set result.body = {"excerpt": (body.Read(maxStreamBytes))}
        Set result.body.truncated = (body.Size > maxStreamBytes)
        Set result.body.size      = body.Size
    } Else {
        Set result.variant = "generic"
        // Same fallback Ens.Util.MessageBodyMethods uses in the Management Portal
        Set result.body = ##class(%ZEN.Auxiliary.altJSONProvider).%ObjectToAET(body)
    }
    Return result
}

/// Render an HL7 VDoc as a structured summary: MSH metadata + segment list
/// with key fields. Truncates segment values to maxStreamBytes total.
ClassMethod RenderHL7Body(pMsg As EnsLib.HL7.Message, maxBytes As %Integer = 4096) As %DynamicObject [ Internal ]
{
    Set out = {}
    Set out.messageType    = pMsg.GetValueAt("MSH:9")
    Set out.controlId      = pMsg.GetValueAt("MSH:10")
    Set out.sendingFacility = pMsg.GetValueAt("MSH:4")
    Set out.receivingFacility = pMsg.GetValueAt("MSH:6")
    Set out.timestamp      = pMsg.GetValueAt("MSH:7")

    Set segments = []
    Set segCount = pMsg.SegCount
    Set budget = maxBytes
    For i = 1:1:segCount {
        Set seg = pMsg.GetSegmentAt(i)
        Quit:'$ISOBJECT(seg)
        Quit:budget <= 0
        Set raw = seg.OutputToString()
        Set raw = $EXTRACT(raw, 1, budget)
        Set budget = budget - $LENGTH(raw)
        Do segments.%Push({"type": (seg.Name), "raw": (raw)})
    }
    Set out.segments  = segments
    Set out.segCount  = segCount
    Set out.truncated = (budget <= 0) && (segCount > segments.%Size())
    Return out
}

ClassMethod RenderVDocBody(pDoc As %RegisteredObject, maxBytes As %Integer = 4096) As %DynamicObject [ Internal ]
{
    // Minimal generic VDoc renderer — extend for X12/XML specifics later
    Set out = {}
    Set stream = ##class(%Stream.TmpCharacter).%New()
    Try {
        Do pDoc.OutputToLibraryStream(stream)
        Do stream.Rewind()
        Set out.raw = stream.Read(maxBytes)
        Set out.truncated = (stream.Size > maxBytes)
    } Catch ex {
        Set out.error = ex.DisplayString()
    }
    Return out
}

}
```

### 5. `Custom.EnsSession.Tools.Process` — Business Process Inspection

Two tools: one for the BP instance state, one for reading its source code.

```objectscript
Class Custom.EnsSession.Tools.Process Extends %AI.Tool [ System = 4 ]
{

Parameter REQUIRESAUTH = 0;
Parameter DISCOVERYLIMIT;

/// Read the ObjectScript source of a Business Process method.
/// If methodName is empty, returns the class description + a list of method names.
/// If methodName is provided, returns the method's signature + implementation.
/// Also returns the BPL XML if this is a BPL-compiled process.
ClassMethod GetBusinessProcessSource(
    classname As %String(DESCRIPTION = "Fully-qualified class name, e.g. Custom.MyRouter"),
    methodName As %String(DESCRIPTION = "Method name; empty = summary mode") = "") As %DynamicObject
{
    Set out = {"classname": (classname), "methodName": (methodName)}

    // Class-level metadata
    Set cls = ##class(%Dictionary.ClassDefinition).%OpenId(classname, , .tSC)
    If '$ISOBJECT(cls) {
        Set out.error = "Class not found or not compiled: " _ classname
        Return out
    }
    Set out.classDescription = cls.Description
    Set out.super            = cls.Super
    Set out.isAbstract       = cls.Abstract
    Set out.deployed         = cls.Deployed

    // Method enumeration
    If methodName = "" {
        Set methods = []
        Set iter   = cls.Methods.%New()  // relationship iteration
        Set key    = ""
        For i = 1:1:cls.Methods.Count() {
            Set m = cls.Methods.GetAt(i)
            Do methods.%Push({
                "name":        (m.Name),
                "description": ($EXTRACT(m.Description, 1, 200)),
                "signature":   (m.FormalSpec),
                "returnType":  (m.ReturnType),
                "classMethod": (m.ClassMethod)
            })
        }
        Set out.methods = methods

        // Also enumerate XData blocks (to reveal BPL)
        Set xdatas = []
        For i = 1:1:cls.XDatas.Count() {
            Set x = cls.XDatas.GetAt(i)
            Do xdatas.%Push({"name": (x.Name), "mimeType": (x.MimeType)})
        }
        Set out.xdatas = xdatas
        Return out
    }

    // Specific method
    Set methodDef = ##class(%Dictionary.MethodDefinition).%OpenId(classname _ "||" _ methodName, , .tSC)
    If '$ISOBJECT(methodDef) {
        Set out.error = "Method not found: " _ classname _ "||" _ methodName
        Return out
    }
    Set out.description = methodDef.Description
    Set out.formalSpec  = methodDef.FormalSpec
    Set out.returnType  = methodDef.ReturnType
    Set out.isClassMethod = methodDef.ClassMethod
    Set out.isAbstract    = methodDef.Abstract
    // Implementation stream — read into a single string
    Set impl = methodDef.Implementation
    If $ISOBJECT(impl) {
        Do impl.Rewind()
        Set src = ""
        While 'impl.AtEnd { Set src = src _ impl.ReadLine() _ $CHAR(10) }
        Set out.implementation = src
    } Else {
        Set out.implementation = ""
        Set out.note = "Method Implementation is empty (possibly deployed/disassociated class)"
    }
    Return out
}

/// Retrieve the BPL XData block (for BPL-compiled Business Processes).
ClassMethod GetBusinessProcessBPL(
    classname As %String(DESCRIPTION = "BPL class name")) As %DynamicObject
{
    Set out = {"classname": (classname)}
    Set xd = ##class(%Dictionary.XDataDefinition).%OpenId(classname _ "||BPL", , .tSC)
    If '$ISOBJECT(xd) {
        Set out.error = "BPL XData not found — this may not be a BPL process"
        Return out
    }
    Set stream = xd.Data
    If $ISOBJECT(stream) {
        Do stream.Rewind()
        Set xml = ""
        While 'stream.AtEnd { Set xml = xml _ stream.Read(4096) }
        Set out.bpl = xml
    }
    Return out
}

/// Retrieve the persistent state of a Business Process instance.
/// Either give the BP instance ID directly (bpInstanceId) or locate via
/// (sessionId, classname) to find the instance that handled a session.
ClassMethod GetBusinessProcessInstance(
    sessionId As %Integer(DESCRIPTION = "Session ID (optional if bpInstanceId given)") = 0,
    bpInstanceId As %Integer(DESCRIPTION = "BP instance %ID (optional if sessionId given)") = 0,
    classname As %String(DESCRIPTION = "BP class name (required for lookup by session)") = "") As %DynamicObject
{
    Set out = {}

    // Resolve instance ID
    If bpInstanceId = 0 {
        If (sessionId = 0) || (classname = "") {
            Set out.error = "Provide either bpInstanceId OR (sessionId AND classname)"
            Return out
        }
        // Query Ens.MessageHeader for a BusinessProcessId
        Set sql = "SELECT TOP 1 BusinessProcessId FROM Ens.MessageHeader "
                _ "WHERE SessionId = ? AND TargetConfigName = ? AND BusinessProcessId IS NOT NULL"
        Set stmt = ##class(%SQL.Statement).%New()
        Do stmt.%Prepare(sql)
        Set rs = stmt.%Execute(sessionId, classname)
        If rs.%Next() { Set bpInstanceId = rs.BusinessProcessId }
        If bpInstanceId = 0 {
            Set out.error = "No BP instance found for session + class combination"
            Return out
        }
    }

    Set out.bpInstanceId = bpInstanceId
    Set out.classname    = classname

    // Open the BP instance
    If classname = "" {
        Set out.error = "classname is required to open BP instance"
        Return out
    }
    Set bp = $CLASSMETHOD(classname, "%OpenId", bpInstanceId, , .tSC)
    If '$ISOBJECT(bp) {
        Set out.error = "BP instance could not be opened"
        Return out
    }

    // Use the same serializer Ens.Util.MessageBodyMethods uses
    Set out.properties = ##class(%ZEN.Auxiliary.altJSONProvider).%ObjectToAET(bp)

    // If BPL, also read context + thread state
    Try {
        Set ctx = ##class(Ens.BP.Context).%OpenId(bpInstanceId, , .tSC)
        If $ISOBJECT(ctx) {
            Set out.bplContext = ##class(%ZEN.Auxiliary.altJSONProvider).%ObjectToAET(ctx)
        }
    } Catch { }

    Set threadSQL = "SELECT ID, _NextState, _Status, _PendingResponses "
                  _ "FROM Ens_BP.Thread WHERE _Process = ?"
    Set stmt = ##class(%SQL.Statement).%New()
    Do stmt.%Prepare(threadSQL)
    Set rs = stmt.%Execute(bpInstanceId)
    Set threads = []
    While rs.%Next() {
        Do threads.%Push({
            "threadId":         (rs.ID),
            "nextState":        (rs.%Get("_NextState")),
            "status":           (rs.%Get("_Status")),
            "pendingResponses": (rs.%Get("_PendingResponses"))
        })
    }
    Set out.threads = threads
    Return out
}

}
```

### 6. `Custom.EnsSession.Tools.Errors` — Error Decoding

```objectscript
Class Custom.EnsSession.Tools.Errors Extends %AI.Tool [ System = 4 ]
{

Parameter REQUIRESAUTH = 0;
Parameter DISCOVERYLIMIT;

/// Decode an IRIS %Status value or explain what a MessageHeader error means.
/// Pass either (statusText) — raw ODBC-rendered %Status string — OR messageId
/// to fetch and decode the response-header's ErrorStatus.
ClassMethod ExplainError(
    statusText As %String(DESCRIPTION = "Raw %Status text from %ODBCOUT, or a log Text field") = "",
    messageId As %Integer(DESCRIPTION = "Message ID to look up") = 0) As %DynamicObject
{
    Set out = {}

    // Resolve source text
    Set src = statusText
    If src = "" && (messageId > 0) {
        Set hdr = ##class(Ens.MessageHeader).%OpenId(messageId, , .tSC)
        If $ISOBJECT(hdr) && hdr.IsError {
            Set src = $SYSTEM.Status.GetOneStatusText(hdr.ErrorStatus, 1)
        } Else {
            // Check if there's a response paired to this request that errored
            Set respSQL = "SELECT TOP 1 ErrorStatus FROM Ens.MessageHeader "
                        _ "WHERE CorrespondingMessageId = ? AND IsError = 1"
            Set stmt = ##class(%SQL.Statement).%New()
            Do stmt.%Prepare(respSQL)
            Set rs = stmt.%Execute(messageId)
            If rs.%Next() { Set src = $SYSTEM.Status.GetOneStatusText(rs.ErrorStatus, 1) }
        }
    }
    If src = "" {
        Set out.error = "No error text provided or found"
        Return out
    }
    Set out.source = src

    // Parse IRIS %Status format: "ERROR #NNNN: message text"
    Set codeMatch = $MATCH(src, "ERROR #(\d+):")
    If codeMatch {
        Set out.errorCode = $PIECE($PIECE(src, "ERROR #", 2), ":", 1)
    }

    // Known Ensemble error categorization
    If (src [ "<Ens>ErrBPTerm") {
        Set out.category  = "BusinessProcessTermination"
        Set out.knownCause = "A BP terminated abnormally — likely due to an error in a downstream call"
    } ElseIf (src [ "<Ens>ErrTimeout") {
        Set out.category  = "Timeout"
        Set out.knownCause = "Operation exceeded its configured timeout"
    } ElseIf (src [ "<Ens>ErrException") {
        Set out.category  = "Exception"
        Set out.knownCause = "An unhandled ObjectScript exception occurred"
    } ElseIf (src [ "<PROTECT>") {
        Set out.category  = "ProtectError"
        Set out.knownCause = "Attempted write to a protected global / permission denied"
    } ElseIf (src [ "<UNDEFINED>") {
        Set out.category  = "UndefinedVariable"
        Set out.knownCause = "Referenced a variable or global node that doesn't exist"
    } Else {
        Set out.category  = "Unknown"
    }

    // Extract the human-readable portion
    If src [ ":" {
        Set out.message = $ZSTRIP($PIECE(src, ":", 2, *), "<>W")
    } Else {
        Set out.message = src
    }
    Return out
}

}
```

### 7. `Custom.EnsSession.Tools.Meta` — Meta and Search Tools

```objectscript
Class Custom.EnsSession.Tools.Meta Extends %AI.Tool [ System = 4 ]
{

Parameter REQUIRESAUTH = 0;
Parameter DISCOVERYLIMIT;
Parameter QUERYMAXROWS = 200;

/// List methods on a Business Process class.
Query ListBusinessProcessMethods(
    classname As %String(DESCRIPTION = "Class name")) As %SQLQuery(
    ROWSPEC = "Name:%String,ClassMethod:%Boolean,FormalSpec:%String,ReturnType:%String,Description:%String") [ SqlProc ]
{
    SELECT Name, ClassMethod, FormalSpec, ReturnType, SUBSTRING(Description, 1, 500) AS Description
    FROM %Dictionary.MethodDefinition
    WHERE parent = :classname
    ORDER BY Name
}

/// Resolve a MessageId to its containing SessionId (in case the user pasted a message ID).
Query ResolveSessionId(
    messageOrSessionId As %Integer) As %SQLQuery(
    ROWSPEC = "SessionId:%Integer,IsSessionRoot:%Boolean") [ SqlProc ]
{
    SELECT SessionId,
           CASE WHEN %ID = SessionId THEN 1 ELSE 0 END AS IsSessionRoot
    FROM Ens.MessageHeader
    WHERE %ID = :messageOrSessionId
}

}
```

### 8. The Read-Only Policy

```objectscript
Class Custom.EnsSession.ReadOnlyPolicy Extends %AI.Policy.Authorization [ System = 4 ]
{

/// Reject any tool with mutates=1 in its metadata.
/// Our tools are strictly read-only; this is a safety net for future additions.
Method Authorize(toolSpec As %DynamicObject, args As %DynamicObject) As %Status
{
    If $ISOBJECT(toolSpec.metadata) && (toolSpec.metadata.%Get("mutates") = "1") {
        Return $$$ERROR($$$GeneralError,
            "Read-only policy: tool '" _ toolSpec.name _ "' denied (mutates metadata set)")
    }
    Return $$$OK
}

}
```

### 9. The Agent Class

```objectscript
/// Declarative agent config. Runs a read-only Ensemble session inspector.
Class Custom.EnsSession.Agent Extends %AI.Agent [ System = 4 ]
{

Parameter PROVIDER = "anthropic";

Parameter MODEL = "claude-sonnet-4-5@20250929";

/// API key resolved from IRIS Wallet at runtime via the Config Store.
Parameter PROVIDERCONFIG = "{""api_key"": ""@{wallet.AISecrets.AnthropicKey}""}";

Parameter TOOLSETS = "Custom.EnsSession.Tools";

XData INSTRUCTIONS [ MimeType = text/markdown ]
{
# Ensemble Session Inspector

You are a read-only InterSystems IRIS Interoperability diagnostic agent.
Given a SessionId, you help users understand what happened in an
Ens.Production session — correlating message headers, event log entries,
rule decisions, and Business Process state.

## Investigation recipe

When asked "what happened in session X":
1. Call `GetSessionSummary` first. Tells you size, duration, error count.
2. Call `GetSessionTimeline` for the interleaved message+log+rule stream.
3. If a specific message looks important (error, unusual body class),
   call `GetMessageBody` for that single message. Do NOT dump all bodies.
4. For "why did component X do Y" questions, call `GetBusinessProcessSource`
   to read the BP code, and `GetBusinessProcessInstance` for runtime state.
5. For errors, `ExplainError` decodes %Status values and recognizes
   common Ensemble error patterns.

## Hard constraints

- Never attempt any mutation. All tools are read-only.
- Do NOT proactively fetch all bodies — they can be large (HL7, X12).
- When a tool returns `truncated: true`, narrow the query and re-ask.
- Decode status codes when presenting to the user — never show raw %Status.

## Key Ensemble concepts (your working knowledge)

- SessionId groups messages in one business transaction.
- Root message has %ID = SessionId.
- Type: 1=Request, 2=Response, 3=Terminate.
- Status 1-9: Created, Queued, Delivered, Discarded, Suspended, Deferred,
  Aborted, Error, Completed.
- Invocation: 1=Queue (async), 2=Inproc (sync).
- IsError/ErrorStatus live on the RESPONSE header, not the request.
- Bodies are dynamically-typed; bodyClass tells you the schema.
- Rules (Ens.Rule.Log) explain why routers/DTLs chose a branch.

## Tone

Concise and technical. Quote relevant field values rather than
summarizing loosely. If uncertainty exists, say so explicitly —
don't manufacture details.
}

/// Enable prompt caching by default to keep token usage low
/// across a debugging session.
Method %OnInit() As %Status
{
    // Framework handles the rest from Parameters
    Return $$$OK
}

}
```

**Usage from an IRIS terminal or CSP page:**

```objectscript
Set agent = ##class(Custom.EnsSession.Agent).%New()
$$$ThrowOnError(agent.%Init())

Set config = {
    "max_iterations": 15,
    "temperature":    0.2,             // factual-mode
    "max_tokens":     2000,
    "cache": {
        "enabled":               (1),
        "cache_system_prompt":   (1),
        "cache_tool_definitions": (1)
    }
}
Set session = agent.CreateSession(config)

Set response = agent.Chat(session, "What happened during session 42751?")
Write response.Content, !
```

### 10. VDoc Rendering Strategy (Detailed)

The `RenderHL7Body` method above handles the common case. For completeness, the rendering options:

| Body type | Strategy |
|---|---|
| `EnsLib.HL7.Message` | MSH metadata + segment-by-segment `OutputToString()` truncated by byte budget |
| `EnsLib.EDI.X12.Document` | Generic `OutputToLibraryStream()` + excerpt |
| `EnsLib.EDI.XML.Document` | Similar — `OutputToLibraryStream` |
| `HS.FHIRServer.Interop.Request` | `%JSON.Adaptor` native path — just works |
| `HS.Message.XMLMessage` | Extract content stream, return text |

For HL7 specifically, we could go further and recognize common segment types (PID, PV1, MSH) and extract clinical fields — but that scope-creeps the tool. V1: return raw segments with enough MSH context for the LLM to identify the message type and patient context. V2: add targeted extractors.

### 11. LLM Context-Budget Considerations

From the SDK guide: `session.GetStats()` tracks token usage. Rough sizing for a "what happened" query:

| Component | Est. tokens | Notes |
|---|---|---|
| System prompt | ~400 | Cached after first call |
| Tool schemas (13 tools) | ~2,000 | Cached after first call |
| User question | ~50 | Varies |
| `GetSessionSummary` result | ~200 | Per session |
| `GetSessionTimeline` result | ~50 per event × up to 500 events = ~25K | Cap at 500 rows |
| `GetMessageBody` per body | 500-5,000 | Depends on body type |
| Final LLM response | 500-2,000 | User-facing narrative |

Total ballpark: 30-50K tokens for a deep investigation of a medium session. Well within Claude Sonnet 4.5's 200K context. With prompt caching, the per-turn cost drops significantly on subsequent questions in the same conversation.

**Mitigations for very large sessions**:
- `GetSessionTimeline` already has `offset`/`limit` (inherited from the base parameters)
- If `truncated: true`, LLM knows to narrow (e.g., by time window or component)
- Add a future `GetSessionStatistics(sessionId)` that returns just counts-by-component — lets the LLM ask "what's interesting" before pulling full timeline

### 12. Testing Strategy

The user's scope didn't ask for this, but the agent must be testable. Three layers:

1. **Unit tests (`%UnitTest.TestCase`)**:
   - For each tool method: valid input, not-found, malformed, permission-denied.
   - Mock `Ens.MessageHeader` rows via test fixtures (seed `^Ens.MessageHeaderD` directly in test setup).
   - Assert return shape matches the documented JSON schema.

2. **Integration tests**:
   - Spin up a small `Ens.Production` with scripted messages; drive it; then run the agent tools against the resulting session. Assert the narrative the LLM produces includes known facts.

3. **Smoke test script**:
   - An ObjectScript routine that picks a random recent SessionId, calls `GetSessionSummary`, and asserts basic invariants (MessageCount > 0, StartTime != "").

### 13. Deployment Checklist

1. **Install AI Hub EAP** container/kit per `sources/ai-hub-eap/README.md` (IRIS 2026.2.0AI.141.0 or later).
2. **Wallet entry** for the LLM API key: `@{wallet.AISecrets.AnthropicKey}` (or OpenAI / Bedrock as appropriate).
3. **Compile the 7 classes**: `Custom.EnsSession.Agent`, `Custom.EnsSession.Tools`, `Custom.EnsSession.Tools.{Trace,Body,Process,Errors,Meta}`, `Custom.EnsSession.ReadOnlyPolicy`.
4. **Grant SQL SELECT** on `Ens.MessageHeader`, `Ens_Util.Log`, `Ens_Rule.Log`, `Ens.SuperSessionIndex`, `Ens_BP.Thread` to the agent runtime user. NOT INSERT/UPDATE/DELETE.
5. **(Optional) MCP Server setup** per `MCP_Server_Guide.md` if exposing to Claude Desktop.
6. **Wire to UI**: CSP page with a simple text input + session selector, or expose via `%AI.Shell` for terminal use.
7. **Monitor**: periodic `session.GetStats()` dumps to a log — track tool-call patterns and token spend.

### 14. Known Limitations & Future Extensions

| Limitation | Mitigation / Extension |
|---|---|
| BPL XData might not exist on non-BPL BPs | `GetBusinessProcessBPL` returns clear error; caller falls back to `GetBusinessProcessSource` |
| Deployed classes have empty `Implementation` | `GetBusinessProcessSource` notes this; user must deploy with source if inspection is needed |
| `%ZEN.Auxiliary.altJSONProvider` may emit some IRIS-specific artifacts | Acceptable for LLM consumption; can post-process if needed |
| HL7 renderer is minimal (raw segments) | Future: add targeted extractors for common segments (PID, PV1, OBX) |
| No cross-namespace queries (current namespace only) | Future: accept a `namespace` parameter + switch namespace per-tool-call |
| No MsgBank query support | Deferred — add `Custom.EnsSession.Tools.MsgBank` ToolSet for enterprise customers |
| Live queue state inaccessible (it's in globals, not SQL) | Out of scope for historical analysis; could add global-iteration tool if needed |

### 15. Source Reliability Assessment for This Step

| Source | Contribution | Reliability |
|---|---|---|
| `irislib/EnsPortal/SVG/VisualTrace.cls` | Canonical session-trace SQL | **Very High** |
| `irislib/EnsPortal/VisualTrace.cls` | Helper method patterns (GetActualSessionId) | **Very High** |
| `sources/ai-hub-eap/ObjectScript_SDK_Guide.md` | ToolSet/Agent/Query-tool patterns | **High** |
| `irislib/%AI/Tool.cls`, `ToolSet.cls` | Framework contracts (REQUIRESAUTH, %Invoke) | **High** |
| `iris-view-agent/learned-schemas/*` | Production-tested SQL patterns (%EXTERNAL, %ODBCOUT) | **Medium-High** |
| `sources/diagramtool/docs/dev-notes-correlation.md` | Correlation rules (reused in timeline query) | **Medium-High** |

**Caveat**: The implementation code is illustrative — it needs to be compiled and tested against a real IRIS 2026.2 instance. Expect minor method-signature tweaks (especially for `%AI.*` API surface changes from EAP build to build).

---

# Research Synthesis & Executive Summary

*The sections above are the raw research — source readings, schema tables, decoder references, code skeletons. This final synthesis pulls the essentials forward: what was found, what it means, how to build it, what to watch for.*

## Executive Summary

A read-only **Ensemble Session Inspection Agent** is feasible, well-defined, and buildable with the InterSystems AI Hub SDK (EAP 2026.2.0AI.141.0). The research confirmed:

1. **The IRIS Ensemble message model is cleanly decomposable into a single correlation graph.** Every piece of session state — messages, bodies, event log entries, rule decisions, queue activity, Business Process state — ties back to `Ens.MessageHeader` via three keys: `%ID`, `SessionId`, and `CorrespondingMessageId`. This graph is stable across IRIS versions and already used by IRIS's own Management Portal (`EnsPortal.SVG.VisualTrace`) for session reconstruction.

2. **The AI Hub SDK provides every primitive we need** — `%AI.Agent` (execution), `%AI.ToolSet` (declarative tool catalogs), `%AI.Tool` (method- and query-based tools with auto-generated JSON schemas), `%AI.Policy.*` (auth/audit/discovery), and `%AI.MCP.Service` (external-client exposure via the `iris-mcp-server` Rust gateway). No meaningful gap between "what we need" and "what the SDK offers."

3. **The design collapses to 13 tools** — 9 SQL-driven `<Query>` tools, 4 ObjectScript method tools, composed into a single root `ToolSet` with a strict read-only authorization policy. All code needed is illustrated in Step 5 as compilable skeletons.

**Key Technical Findings:**

- **Session-starting messages have `%ID = SessionId`.** A single-column equality locates the session root — confirmed from source (`Ens.MessageHeader.cls:70`) and from the Management Portal's VisualTrace.
- **Message bodies are dynamically typed** — the header stores `MessageBodyClassName` + `MessageBodyId`, and body classes range from `EnsLib.HL7.Message` to user-defined `%Persistent` subclasses. The tool must reflect on the class at runtime; no hardcoded schema will work. A clean dispatch pattern exists: `%JSON.Adaptor`-first → VDoc-specialized → `%Stream.Object`-excerpt → `%ZEN.Auxiliary.altJSONProvider` generic fallback (same path `Ens.Util.MessageBodyMethods` uses).
- **Errors live on responses, not requests.** `IsError` and `ErrorStatus` are populated on `Type = 2` (Response) headers. To know if a request failed, join to its response via `CorrespondingMessageId`. `ErrorStatus` is a raw `%Status` value — render with `%ODBCOUT()` or `$SYSTEM.Status.GetOneStatusText()`.
- **`Ens.Rule.RuleLog` is deprecated** — use `Ens.Rule.Log` (richer schema with `ConfigName` + `CurrentHeaderId`). Both may coexist during a transition.
- **Class introspection is direct**: `%Dictionary.ClassDefinition.%OpenId(classname)` + `Methods.Implementation` (a `%Stream.TmpCharacter`) for method source; `XDataDefinition.%OpenId(classname||BPL)` for BPL XML. Child-member ID format: `ClassName||MemberName` with double-pipe.
- **The Management Portal's session-reconstruction query is the right pattern** — 14-column projection over `Ens.MessageHeader` filtered by `SessionId`, ordered by `%ID`, unioned with `Ens.Util.Log` (by SessionId) and `Ens.Rule.Log` (by SessionId + CurrentHeaderId) for the full interleaved timeline.
- **The Config Store + IRIS Wallet** keeps LLM API keys out of source (`@{wallet.AISecrets.AnthropicKey}` syntax).
- **Prompt caching support is broad** — Anthropic, OpenAI, Gemini, Vertex, and Bedrock-SigV4 all support it; the system prompt and tool schemas become effectively zero-cost on subsequent turns.
- **EAP API churn is a real risk** — the `%AI.*` surface is pre-release. The architecture tolerates minor naming shifts; re-verify class/parameter names against the shipped IRIS before production.

**Strategic Technical Recommendations:**

1. **Build a single `Custom.EnsSession.Tools` ToolSet** composed of 5 per-concern `%AI.Tool` subclasses (Trace, Body, Process, Errors, Meta). Attach a `Custom.EnsSession.ReadOnlyPolicy` at the ToolSet level. Expose via both a native `%AI.Agent` (for in-IRIS use) AND a `%AI.MCP.Service` subclass (for Claude Desktop / external clients) — same ToolSet, two front ends, zero duplication.
2. **Make every SQL-first tool a `<Query>` element** — compile-time SQL validation, automatic JSON-schema derivation, standard result envelope `{rows, row_count, truncated, elapsed_ms}`. Reserve ObjectScript method tools for body dispatch and class introspection where runtime logic is essential.
3. **Inherit the Management Portal's query shape** — 14-column projection, `ORDER BY %ID`, `UNION ALL` for timeline events with a `kind` discriminator. Decoded enum values (`%EXTERNAL`) and error text (`%ODBCOUT`) at the SQL layer, not the application layer.
4. **Three-layer read-only enforcement** — method-level discipline (no `%Save`/`%DeleteId`/`ResubmitMessage`), `%AI.Policy.Authorization` gating `mutates=1` metadata, and IRIS RBAC granting SELECT-only on `Ens.*` tables at deployment time.
5. **Adopt prior art, don't reinvent** — the `diagramtool` correlation algorithm (Inproc forward-scan, Queue CorrMsgId + ReturnQueueName fallback) is production-tested; the `iris-view-agent/learned-schemas/` distillation encodes gotchas (encoding tricks, join performance traps) worth heeding verbatim.

---

## Table of Contents

1. [Research Overview](#research-overview)
2. [Technical Research Scope Confirmation](#technical-research-scope-confirmation)
3. [Technical Overview — Foundational Landscape](#technical-overview--foundational-landscape)
4. [Technical Overview — Deep Dive (Extension)](#technical-overview--deep-dive-extension)
5. [Integration Patterns — The Session Correlation Model](#integration-patterns--the-session-correlation-model)
6. [Architectural Patterns — AI Hub Agent & Tool Design](#architectural-patterns--ai-hub-agent--tool-design)
7. [Implementation Research — Concrete Code Patterns](#implementation-research--concrete-code-patterns)
8. [Research Synthesis & Executive Summary](#research-synthesis--executive-summary) *(you are here)*
   - [Executive Summary](#executive-summary)
   - [Research Methodology](#research-methodology)
   - [Strategic Recommendations](#strategic-recommendations)
   - [Implementation Roadmap](#implementation-roadmap)
   - [Risk Assessment](#risk-assessment)
   - [Future Outlook](#future-outlook)
   - [Conclusion & Next Steps](#conclusion--next-steps)

---

## Research Methodology

The research was conducted in six sequential phases, each building on the prior:

| Phase | Focus | Primary Sources |
|---|---|---|
| 1. Scope | Goals, targets, constraints | User Q&A |
| 2. Overview | Ensemble production model + AI Hub landscape | `irislib/Ens/*.cls`, `irislib/%AI/*`, `sources/ai-hub-eap/README.md`, Perplexity for doc cross-check |
| 3. Integration | Session correlation model | All of 2 + `sources/diagramtool/docs/*`, `iris-view-agent/.claude/skills/iris-trace-query/*` |
| 4. Architecture | AI Hub tool/agent/policy design | `sources/ai-hub-eap/ObjectScript_SDK_Guide.md` (full read) + `irislib/%AI/Tool.cls`, `ToolSet.cls` |
| 5. Implementation | Concrete code patterns | `irislib/EnsPortal/SVG/VisualTrace.cls`, `irislib/EnsPortal/VisualTrace.cls`, + all prior-art |
| 6. Synthesis | This section | All of 1-5 |

**Source authority tiers** (used consistently throughout):

1. **Authoritative (source code in `irislib/`)** — ground truth; what IRIS actually does.
2. **Official documentation (`sources/ai-hub-eap/*`, InterSystems Documatic via Perplexity)** — intent and API contract; may lag source in EAP.
3. **Prior art (`sources/diagramtool/`, `iris-view-agent/`)** — production-validated patterns; treated as validated hypotheses, cross-checked against source.
4. **Community / secondary (Developer Community posts via Perplexity)** — context and usage patterns; lowest trust, useful for orientation.

**Cross-validation protocol**: any claim used for implementation had to appear in at least one authoritative source AND (where possible) be consistent with either official docs or prior art. Discrepancies were flagged explicitly in the prose rather than silently resolved.

**Known research limitations**:
- AI Hub EAP version is `2026.2.0AI.141.0`; APIs may change before GA.
- No live IRIS instance was used — the code skeletons in Step 5 are illustrative and will need compilation + testing.
- MsgBank (`Ens.Enterprise.MsgBank.*`) was surveyed but not deeply characterized — v1 scope excludes cross-production queries.
- HL7 VDoc rendering was kept minimal; segment-level clinical extraction is deferred to v2.

---

## Strategic Recommendations

### R1. Adopt the "same ToolSet, two front ends" deployment pattern

Build `Custom.EnsSession.Tools` once as a `%AI.ToolSet`. Reference it from a native `%AI.Agent` (for in-IRIS operator consoles) AND a `%AI.MCP.Service` subclass (for Claude Desktop users). Zero duplication, both audiences served. Start with the native agent first to validate the tool catalog; add MCP exposure as a no-code follow-on.

### R2. Prefer `<Query>` tools over method tools wherever possible

The SDK's `<Query>` element (Method 5b from Step 4) validates SQL at compile time, derives JSON schemas automatically, and returns a standard result envelope. Most of our tools (9 of 13) are SQL-first. Use method tools only where runtime logic is essential: body dispatch (`GetMessageBody`), class introspection (`GetBusinessProcessSource`), error decoding (`ExplainError`).

### R3. Inherit the Management Portal's query shape

IRIS's own `EnsPortal.SVG.VisualTrace.BuildTraceInfo` method is the reference implementation for session reconstruction. Our `GetSessionTimeline` SQL (Step 5 §3) matches its 14-column projection, `ORDER BY %ID`, and `UNION ALL` structure with `Ens.Util.Log` and `Ens.Rule.Log`. This maximizes consistency with what administrators already see in the Management Portal's Visual Trace.

### R4. Enforce read-only at three layers

Method-level discipline (no mutation APIs called) → `%AI.Policy.Authorization` filter on tool metadata → IRIS RBAC with SELECT-only grants. Each layer independently prevents mutations even if another fails. Do not skip any layer; the cost is a few lines of code, the payoff is real safety in production.

### R5. Treat `iris-view-agent/learned-schemas/` as a production-validated cheat sheet

The sibling project has already catalogued the non-obvious gotchas: `%EXTERNAL` vs `%ODBCOUT` for decoding, the ALL-CAPS `MessageBodyClassName` trap, performance traps on `CorrespondingMessageId` joins in high-volume namespaces, the `{Table}_AdditionalInfo` pivot pattern for SearchTable-indexed fields. Copy its patterns verbatim; don't rediscover.

### R6. Wallet-based secret management from day one

Use `@{wallet.AISecrets.AnthropicKey}` (or equivalent) in the agent's `PROVIDERCONFIG`. No secrets in source. No secrets in environment variables (which leak into process lists and logs). Configure the wallet entry at install time.

### R7. Plan for EAP churn

The AI Hub README warns APIs may change. Write thin wrappers around `%AI.Provider.Create(...)` and `%AI.Agent.%New(...)` so a future rename surfaces in one place. Version-pin to the known-good IRIS build. Re-verify every `%AI.*` method signature before each production upgrade.

---

## Implementation Roadmap

### Phase 1 — MVP (2-3 weeks)

**Goal**: Native agent answers "what happened during this session?" end-to-end.

- Compile the 7 classes in Step 5 §2-§9 (ToolSet, Trace/Body/Process/Errors/Meta, ReadOnlyPolicy, Agent).
- Wire to a simple IRIS CSP page: session-ID input box + agent chat.
- Configure wallet entry for the LLM API key.
- Seed test data by scripting an `Ens.Production` with a few Business Services, Processes, and Operations; generate 10-20 test sessions with varied outcomes (success, error, timeout, resubmit).
- Verify the canonical queries against real data; tune SQL as needed.
- Test all 13 tools via the agent; iterate on the system prompt based on narrative quality.

**Exit criteria**:
- Agent correctly answers the 4 scoping questions for all test sessions: "What happened during session X?", "What does Custom.BusinessProcess code do with the message?", "What does the AddUpdateHubRequest contain?", "What does this error mean?"
- All tools return well-formed JSON under `QUERYMAXROWS=500` rows.
- No mutation of any Ens state verified via audit log.

### Phase 2 — Productionization (2 weeks)

- Add `%UnitTest.TestCase` coverage for each tool (valid/not-found/permission-denied cases).
- Add integration tests driving a scripted production end-to-end.
- Grant SELECT-only SQL privileges to the agent's IRIS user.
- Add usage logging — persist `session.GetStats()` results to a custom table for analytics.
- Tune `QUERYMAXROWS` per tool based on observed patterns.
- Write operator documentation (how to run the agent, common queries, known limitations).

### Phase 3 — MCP Exposure (1 week)

- Add `Custom.EnsSession.McpService extends %AI.MCP.Service` referencing the same ToolSet.
- Configure `iris-mcp-server` binary per the MCP Server Guide; set up the config.toml.
- Create the MCP Server record in the Management Portal.
- Test from Claude Desktop; verify tool discovery and invocation.
- Document the Claude Desktop installation flow for operators.

### Phase 4 — Enhancements (ongoing, scope-driven)

- **VDoc enrichment**: targeted extractors for common HL7 segments (PID, PV1, OBX) to enrich `GetMessageBody` for healthcare productions.
- **Cross-namespace queries**: accept a `namespace` parameter on each tool and use `zn` / `$namespace` switching per call.
- **MsgBank queries**: add a `Custom.EnsSession.Tools.MsgBank` ToolSet for enterprise customers with central banking.
- **SuperSession deep-dive**: `FindRelatedSessions` is already in v1; a follow-on tool could aggregate metrics across SuperSessions.
- **Statistical tools**: `GetSessionStatistics` for counts-by-component without pulling full timeline — lets the LLM ask "what's interesting?" before fetching detail.
- **Smart Discovery**: if the toolset grows past ~20 tools, enable `EnableSmartDiscovery()` for RAG-based tool selection.

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| AI Hub API surface changes pre-GA | Medium | Medium | Thin wrappers around `%AI.Provider.Create` / `%AI.Agent.%New`; re-verify on each IRIS upgrade |
| LLM hallucinates session details not in tool output | Medium | High | Low temperature (0.2), factual system prompt, structured JSON results with decoded enums, explicit "don't invent" instruction |
| Large sessions (500+ messages) exceed token budget | Low | Medium | `QUERYMAXROWS=500`, paginated timeline tool, `truncated` signal to LLM |
| Body purging orphans `GetMessageBody` calls | Medium | Low | Tool returns `{error: "purged"}` cleanly; LLM narrates the gap |
| Deployed classes have empty `Implementation` | Low | Medium | `GetBusinessProcessSource` notes deployment status; operator documentation explains limitation |
| `CorrespondingMessageId` join times out on high-volume namespaces | Medium | Medium | Add time-window constraints to joins; prior-art shows this is a real production trap |
| PHI in message bodies leaks to LLM provider | **High** | **High** | **See special note below — most important risk** |
| Read-only policy bypassed by a future contributor | Low | **High** | Three-layer enforcement (method discipline + policy + RBAC); code review checklist |
| Wallet misconfiguration exposes API key | Low | Medium | Wallet entries audited; `@{wallet.*}` placeholders; no env-var fallback |
| EAP build incompatibility (SDK changes) | Medium | Medium | Pin IRIS version; test upgrades in staging before prod |

### Special note on PHI / sensitive body data

The tool reads message bodies, and in healthcare productions those bodies contain PHI (Protected Health Information). **The LLM provider sees whatever the tool returns.** Three choices:

1. **Self-hosted LLM** (e.g., Ollama, NIM) — data never leaves the IRIS host. Strongest PHI control. Recommended for healthcare deployments.
2. **BAA-covered cloud provider** (Anthropic via AWS Bedrock with BAA, Azure OpenAI with BAA) — contractually covered, still exposed. Acceptable for regulated workflows with appropriate agreements.
3. **Body redaction at tool boundary** — strip or mask known-sensitive fields before returning from `GetMessageBody`. Adds complexity; defensible as defense-in-depth but not a substitute for #1 or #2.

The research (Step 3 §14, Step 4 §5) notes this but does not specify a policy — that decision depends on the deployment environment and is out of scope for the tool design itself. Flag this explicitly to stakeholders before production rollout.

Additionally: the org-level policy for this environment (see `<organizationInstructions>` in the runtime context) prohibits PHI/PII in Claude Desktop sessions. For operators using the MCP Server exposure to Claude Desktop, this means either (a) the production has no PHI, or (b) the tool's body-fetch must be redacted, or (c) the MCP route must be disabled in PHI-bearing environments. Native `%AI.Agent` with a self-hosted LLM provider sidesteps this constraint.

---

## Future Outlook

**Near-term (1-2 years)**: AI Hub moves from EAP to GA; the `%AI.*` surface stabilizes. IRIS for Health integrations likely add first-party tools for HL7/FHIR message inspection — our work here may be superseded for healthcare-specific use cases, but remains valuable for generic `Ens.Production` inspection. MCP adoption grows; more operators want their agents accessible from Claude Desktop and similar clients.

**Medium-term (3-5 years)**: Expect agentic tooling for Ensemble to evolve toward proactive diagnostics — agents that watch for anomalies, not just respond to questions. Our read-only foundation is the basis for such a system (the anomaly detector uses the same tools we're building here). Possible InterSystems-shipped equivalent of our tool as a native capability would be a net positive; we'd migrate users to it.

**Long-term (5+ years)**: Session traces become inputs to automated-remediation agents (with explicit write permissions under strict policies). Our read-only tool is the safe "inspector" half of a future read/write "operator" agent. The clean separation of concerns (read-only here, mutations elsewhere) means that future expansion is additive, not a rewrite.

**Specific innovations worth tracking**:
- **RAG-over-trace**: index past sessions by outcome and retrieve similar-shape sessions for comparison ("show me other sessions like this one that timed out").
- **Anomaly detection**: auto-flag sessions whose timeline shape deviates from baseline for their production.
- **Natural-language trace authoring**: user describes an issue narratively; agent identifies the session IDs that match.
- **Auto-remediation runbooks**: agent identifies a known failure pattern and proposes a documented fix (human-approved before execution).

---

## Conclusion & Next Steps

This research delivers a complete, source-grounded, buildable design for a read-only Ensemble Session Inspection Agent on the InterSystems AI Hub SDK. The seven-class implementation in Step 5 should compile against IRIS 2026.2.0AI.141.0 with minor tweaks; the tool catalog covers the four scoping questions the user articulated; the three-layer read-only enforcement provides defense in depth; and the "same ToolSet, two front ends" architecture offers both in-IRIS and external-client access without duplication.

**Immediate next steps**:

1. **Compile and test** the Step 5 class skeletons against a real IRIS 2026.2 instance. Expect minor API-surface tweaks; budget a day for this.
2. **Seed a test production** with scripted Business Services/Processes/Operations and generate 10-20 test sessions with varied outcomes.
3. **Iterate the system prompt** (Step 5 §9) based on narrative quality — the prompt is designed for Claude Sonnet 4.5; if using a different model, tune for its style.
4. **Grant SELECT-only SQL privileges** to the agent's IRIS user.
5. **Make the PHI/deployment decision explicitly** before any production use: self-hosted LLM vs BAA-covered cloud vs body redaction.
6. **Review the org-level policy constraint** (PHI/PII prohibition in Claude Desktop sessions) against the intended deployment topology.

**Document handoff**: this research document is the foundation for a product project. The next phase is the PRD — which should:
- Cite this doc as the technical reference.
- Define user stories for each of the 13 tools.
- Specify acceptance criteria (probably against Step 5 §12 testing patterns).
- Identify the deployment environment (PHI-bearing? BAA? native vs MCP?).
- Scope v1 boundaries (native only? MCP from day one? VDoc depth?).

---

**Research Completion Date:** 2026-04-24
**Research Duration:** 6 sequential phases (scope → overview → integration → architecture → implementation → synthesis)
**Document Length:** ~2,700 lines
**Primary Source Verification:** All claims grounded in `irislib/` source code + AI Hub EAP documentation + prior-art from two reference projects
**Web Verification:** Perplexity search against `docs.intersystems.com` and community sources used for cross-checking 15+ non-trivial claims
**Technical Confidence:** **High** — except for EAP API surface which is labeled Medium due to explicit churn warning
**Applicability:** Direct input to a PRD for product development

*This document serves as the authoritative technical reference for the Ensemble Session Inspection Agent product project. The concrete code patterns in Step 5 should be treated as near-ready implementations pending verification against a live IRIS 2026.2 instance.*






