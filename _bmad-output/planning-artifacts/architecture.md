---
stepsCompleted: [1, 2, 3, 4, 5, 6, 7, 8]
lastStep: 8
status: complete
completedAt: '2026-04-24'
inputDocuments:
  - "_bmad-output/planning-artifacts/prd.md"
  - "_bmad-output/planning-artifacts/product-brief-ensemble-session-inspection-agent.md"
  - "_bmad-output/planning-artifacts/product-brief-ensemble-session-inspection-agent-distillate.md"
  - "_bmad-output/planning-artifacts/research/technical-ensemble-session-inspection-agent-research-2026-04-24.md"
  - "_bmad-output/planning-artifacts/research/technical-ensemble-session-agent-ui-integration-research-2026-04-24.md"
workflowType: 'architecture'
project_name: 'Ensemble Session Inspection Agent'
user_name: 'Developer'
date: '2026-04-24'
---

# Architecture Decision Document

## Ensemble Session Inspection Agent

_This document builds collaboratively through step-by-step discovery. Sections are appended as we work through each architectural decision together._

## Project Context Analysis

### Requirements Overview

**Functional Requirements — 37 FRs across 8 capability areas:**

| Area | FRs | Architectural implication |
|---|---|---|
| Session Diagnostic Chat (FR1-9) | 9 | Multi-turn agent engine, persistent session state, reset/lifecycle management |
| Session Data Access (FR10-16) | 7 | SQL-first read-only query layer; 7 tools targeting 3 Ens tables + SuperSession + SearchTable |
| Message & Body Inspection (FR17-20) | 4 | Runtime polymorphic dispatch — must handle 5+ body class variants without hardcoded schemas |
| Business Process Inspection (FR21-23) | 3 | In-process class metadata reading via `%Dictionary.*`; stream-based source extraction |
| Error & Log Analysis (FR24-26) | 3 | %Status decoding, pattern recognition, severity-filtered event log queries |
| Portal Integration (FR27-30) | 4 | Zen page subclassing, ZenMethod AJAX bridge, HTTP-stateless-safe session persistence |
| Deployment & Configuration (FR31-34) | 4 | HSCUSTOM + package mapping; IRIS Wallet; read-only verifiability |
| Audit & Governance (FR35-37) | 3 | Persistent audit log; RBAC inheritance; non-PHI scoping |

**Non-Functional Requirements — architectural drivers:**

- **Performance:** ≤30s chat turn, ≤10s single SQL, ≤15s compound UNION ALL, ≤5s agent init — requires time-bound joins and query size caps in tool implementations
- **Security:** Three-layer read-only structural enforcement — cannot be a convention, must be verifiable by code review; IRIS Wallet as the only acceptable key storage
- **Integration:** LLM provider decoupled from tool layer — provider change must require zero tool changes; AI Hub EAP build 141 dependency with documented churn risk
- **Reliability:** Tool failure returns structured error, not exception — agent must be able to narrate partial results; no state corruption risk from any failure mode

**Scale & Complexity:**

- Project complexity: **High** — AI Hub EAP, dynamic body dispatch, Zen portal subclassing, healthcare domain
- Primary domain: **ObjectScript native tool / Management Portal extension**
- Estimated architectural components: **8 distinct deliverable units** across 3 layers

### Technical Constraints & Dependencies

- **Platform lock:** IRIS for Health 2026.2, AI Hub EAP build 141 — no fallback if SDK unavailable
- **Language:** ObjectScript exclusively for v1; no Python/Java runtimes
- **Deployment topology:** HSCUSTOM single-compile; per-namespace availability via package mapping to `%ALL`; bookmark URL access only (no portal nav registration)
- **Chat modality:** Blocking synchronous ZenMethod per turn — no streaming, no WebSockets; `%AI.Agent.Session` `%Save()`/`Load()` bridges HTTP request boundary
- **PHI boundary:** Non-PHI namespaces only in v1 — enforced by deployment convention; no technical enforcement in v1
- **Hackathon build constraint:** Incremental delivery required; terminal bot must be independently functional before portal work begins; parallel work tracks must be viable after Phase 1 gate

### Cross-Cutting Concerns

1. **Read-only enforcement** — spans all 13 tool implementations; must be verifiable at static-review time; cannot rely on runtime checks alone
2. **Agent session lifecycle** — `%AI.Agent.Session` `%Save()`/`Load()` pattern applies across Phase 1 (REPL turns) and Phase 2 (HTTP requests); same pattern, different contexts
3. **Graceful degradation** — all tools must return structured `{error, hint}` objects, never surface raw exceptions to the LLM
4. **Package mapping context** — all classes compile once in HSCUSTOM; `$NAMESPACE` at runtime is the portal's target namespace (correct by default via CSP app); no ZN switching in code
5. **Audit trail** — every chat turn across all phases must write an `AuditLog` row; cross-cutting across all phase boundaries
6. **Incremental build gate** — Phase 1 (5 core tools + terminal working) is the explicit gate before Phase 2/3 begins; architecture must make this gate testable

## Foundation & Source Structure

### Primary Technology Domain

**IRIS ObjectScript native tool** — no external frontend framework, build system, or package manager. The "starter" is the existing ZPM module structure in the repository: `module.xml` → `src/` → compiled into HSCUSTOM.

### Starter Baseline

The repository provides the correct foundation:
- `module.xml` — ZPM module descriptor (`SourcesRoot=src`)
- `src/Sample.*` — placeholder classes to be replaced with our packages
- `App.Installer.cls` — IRIS manifest-based namespace installer

**Initialization step:** Replace `Sample.*` references in `module.xml` with `SAgent.*` package resources and delete `src/Sample/`.

### Source Package Structure

The file tree is the architectural boundary. `SAgent.Main` + `SAgent.Tools` (Track A) and `SAgent.Portal` (Track B) are never mixed in the same directory. Any developer picking up the project sees the build track separation immediately.

```
src/
└── SAgent/
    ├── Main/                        ← TRACK A, Part 1: Core agent runtime
    │   ├── Agent.cls                  %AI.Agent subclass (declarative config)
    │   ├── Shell.cls                  Terminal REPL entry point
    │   ├── ReadOnlyPolicy.cls         %AI.Policy.Authorization subclass
    │   └── AuditLog.cls               %Persistent audit trail
    ├── Tools/                       ← TRACK A, Part 2: Tool catalog
    │   ├── Tools.cls                  %AI.ToolSet composition root
    │   ├── Trace.cls                  P1: GetSessionSummary, GetSessionTimeline,
    │   │                                   GetEventLog, GetMessageHeaders, GetRuleLog
    │   ├── Body.cls                   P1: GetMessageBody, GetMessageDetail
    │   ├── Errors.cls                 P1: ExplainError
    │   ├── Process.cls                P3: GetBusinessProcessSource,
    │   │                                   GetBusinessProcessInstance,
    │   │                                   ListBusinessProcessMethods
    │   └── Meta.cls                   P4: FindRelatedSessions, FindSessionsByBody
    └── Portal/                      ← TRACK B: UI layer (after P1 gate)
        ├── VisualTrace.cls            EnsPortal.VisualTrace subclass (Phase 2)
        └── MessageViewer.cls          EnsPortal.MessageViewer subclass (Phase 3)
```

### Package Dependency Rules

```
SAgent.Main   →  %AI.* (AI Hub SDK)       ✅ allowed
SAgent.Tools  →  Ens.* (Ensemble classes)  ✅ allowed
SAgent.Portal →  SAgent.Main              ✅ allowed
SAgent.Portal →  SAgent.Tools             ✅ allowed
SAgent.Main   →  SAgent.Portal            ❌ FORBIDDEN
SAgent.Tools  →  SAgent.Portal            ❌ FORBIDDEN
```

Track B uses Track A. Track A never knows Track B exists. This wall makes parallel work safe.

### module.xml Resource Declarations

```xml
<Module>
  <Name>ens-session-agent</Name>
  <Version>1.0.0</Version>
  <SourcesRoot>src</SourcesRoot>
  <!-- Track A: Core agent — must compile and pass gate before Track B starts -->
  <Resource Name="SAgent.Main.PKG"/>
  <Resource Name="SAgent.Tools.PKG"/>
  <!-- Track B: Portal UI — parallel after Track A gate -->
  <Resource Name="SAgent.Portal.PKG"/>
</Module>
```

### Architectural Decisions Made by This Foundation

| Decision | Choice | Rationale |
|---|---|---|
| Language | ObjectScript only | AI Hub SDK native; no external runtime deps |
| Root package | `SAgent` | Short, brandable; clearly project-specific |
| Track A packages | `SAgent.Main` + `SAgent.Tools` | Main = agent brain; Tools = diagnostic catalog |
| Track B package | `SAgent.Portal` | Portal integration layer; starts after Track A gate |
| Tool grouping | 5 classes in `SAgent.Tools.*` | Matches P1-P4 priority ladder exactly |
| Composition root | `SAgent.Tools.Tools` as single ToolSet | Agent references one class; tracks add tools without touching Agent.cls |
| Portal classes | Subclasses of `EnsPortal.*` | Proven pattern (`Ens.Enterprise.Portal.VisualTrace`); no new Zen app needed |
| Source root | `src/` (existing) | ZPM-compatible; ready for IPM packaging post-hackathon |

## Core Architectural Decisions

### Decision Priority Analysis

**Critical Decisions (Block Implementation):**
- Incremental build gate definition — exactly what must pass before Track B starts
- Agent session persistence pattern — bridges HTTP stateless request boundary
- Tool error contract — LLM must receive structured errors, never raw exceptions
- LLM provider configuration — Anthropic via IRIS Wallet, provider-swappable

**Important Decisions (Shape Architecture):**
- Read-only enforcement layers — three layers, all verifiable by code review
- Tool composition pattern — single `SAgent.Tools.Tools` ToolSet as the composition root
- ZenMethod AJAX pattern — synchronous blocking per turn, ≤30s threshold

**Deferred Decisions (Post-MVP):**
- PHI-bearing namespace LLM deployment (self-hosted vs BAA cloud)
- IPM module packaging and ZPM manifest finalization
- Streaming responses (WebSocket/SSE)
- Cross-namespace session inspection

### Data Architecture

| Decision | Choice | Rationale |
|---|---|---|
| Session data source | IRIS globals via SQL — `Ens.MessageHeader`, `Ens_Util.Log`, `Ens_Rule.Log` | Read-only; already structured; no separate DB needed |
| Data access pattern | `%SQL.Statement` for all Ens.* queries; `%OpenId` for body dispatch; `%Dictionary.*` for class metadata | Native IRIS; no ORM; compile-time SQL validation via `<Query>` elements |
| Agent state storage | `%AI.Agent.Session` (`^AI.Agent.SessionD`) — built-in `%Persistent` class | Single `%Save()`/`Load()` call bridges HTTP request boundary; CSP session stores the `%Id()` pointer |
| Audit trail storage | `SAgent.Main.AuditLog` as a custom `%Persistent` class in `^SAgent.Main.AuditLogD` | Per-turn record: IRIS user, Ens SessionId, chat turn, tools invoked, token counts |
| Caching | None — all queries scoped to a single SessionId; result sets are inherently bounded | No caching infrastructure needed; QUERYMAXROWS=500 on tools caps result size |
| Body dispatch | Runtime polymorphic dispatch: `%JSON.Adaptor` first → VDoc → `%Stream` → generic reflector | No hardcoded body schemas; handles all Ensemble body class variants at runtime |

### Authentication & Security

| Decision | Choice | Rationale |
|---|---|---|
| Authentication | IRIS portal CSP session inheritance — no separate login | Bookmark URL rides existing portal session cookie |
| Authorization | IRIS RBAC inherited from portal — `%Ens_MessageTrace:USE` + `%Ens_MessageHeader:USE` | Users access only what they can already access in the portal |
| Read-only enforcement | Three structural layers: (1) no mutation APIs in tool code, (2) `SAgent.Main.ReadOnlyPolicy`, (3) IRIS RBAC SELECT-only grants | Each layer independently prevents mutations; verifiable by code review |
| Secrets management | IRIS Wallet entry `AISecrets.AnthropicKey` referenced via `@{wallet.AISecrets.AnthropicKey}` in `SAgent.Main.Agent` | No secrets in source code, environment variables, or log output |
| PHI boundary | Non-PHI namespaces only in v1 — deployment convention | PHI support deferred post-hackathon |
| Data transmission | Session body content deserialized server-side; only decoded/rendered content sent to LLM | Raw binary never leaves IRIS |

### API & Communication Patterns

| Decision | Choice | Rationale |
|---|---|---|
| LLM provider | Anthropic `claude-sonnet-4-5` via `%AI.Provider.Create("anthropic", ...)` | Best tool-calling capability; Wallet-managed key |
| Provider abstraction | Swappable by changing `Parameter PROVIDER` + `PROVIDERCONFIG` on `SAgent.Main.Agent` — zero tool changes | Tools return `%DynamicObject`; SDK handles all provider communication |
| Tool API | `<Query>` element for SQL-first tools; ClassMethods for logic tools | `<Query>` gives compile-time SQL validation + auto JSON schema |
| Internal chat API | ZenMethod `SendChatMessage(input As %String, selectedMsgId As %String) As %String` — synchronous blocking | Simplest path; no WebSocket infrastructure; ≤30s response acceptable for demo |
| Error contract | All tools return `%DynamicObject` with `{error, hint, variant}` on failure — never raise exceptions | LLM receives partial results + explanation; conversation continues |
| Session state API | `%AI.Agent.Session.Load(id, provider)` per HTTP request; `%Save()` after each turn | Built-in SDK pattern; no external message queue |

### Frontend Architecture (Portal Integration)

| Decision | Choice | Rationale |
|---|---|---|
| UI extension pattern | Subclass `EnsPortal.VisualTrace` and `EnsPortal.MessageViewer` | InterSystems uses same pattern in `Ens.Enterprise.Portal.VisualTrace`; proven, low-risk |
| Tab addition | Override `XData allTabs` in `SAgent.Portal.VisualTrace` to add `<tab id="chatTab">` as 4th tab | Three existing tabs unchanged; 4th added declaratively |
| Chat UI rendering | Server-rendered HTML skeleton via `OnDrawContent="DrawChatUI"` + ~30 lines client JavaScript | No frontend framework; inherits portal CSS; ZenMethod provides AJAX bridge |
| Session context | `zenPage.currentId` passed to `SendChatMessage` on each turn | Agent knows which message is selected; prepended to LLM prompt server-side |
| MessageViewer handoff | Override `ClientMethod showTrace()` — change URL to `SAgent.Portal.VisualTrace.zen` | Single-line change; all other MessageViewer behavior unchanged |
| Visual design | Inherited from portal application styles | No new CSS; portal visual language automatically applied |

### Infrastructure & Deployment

| Decision | Choice | Rationale |
|---|---|---|
| Compilation target | HSCUSTOM namespace | Single compile; no per-namespace duplication |
| Class availability | Package mapping from HSCUSTOM to `%ALL` | Standard HealthShare pattern; zero runtime overhead |
| Entry point (Phase 1) | `Do ##class(SAgent.Main.Shell).Run("NAMESPACE", sessionId)` | Simplest possible invocation; validates agent end-to-end |
| Entry point (Phase 2-3) | Bookmark URL: `https://<host>/csp/healthshare/<NS>/SAgent.Portal.MessageViewer.zen` | One bookmark per namespace |
| Monitoring | `SAgent.Main.AuditLog` rows — built-in per-turn telemetry | No external observability infrastructure needed for v1 |
| CI/CD | Out of scope for hackathon — manual compile-and-test | ZPM module enables one-command install post-hackathon |

### Incremental Build Gate (Critical Decision)

**Gate Definition — all criteria must pass before Track B begins:**

```
✅ SAgent.Main.Agent compiles and initializes (provider + toolset load in <5s)
✅ SAgent.Main.Shell.Run() launches interactive REPL session
✅ SAgent.Tools.Trace + SAgent.Tools.Errors compile (P1 tools)
✅ Agent correctly answers: "What happened in session X?"
✅ Agent correctly answers: "What does this error mean?"
✅ Multi-turn confirmed: follow-up question references prior context correctly
✅ Read-only verified: no Ens.* rows added/modified after a chat turn
```

One developer validates the gate; others proceed to `SAgent.Portal.*` work in parallel once all criteria pass.

## Implementation Patterns & Consistency Rules

### Critical Conflict Points Identified

8 areas where parallel developers could make different choices and produce incompatible code:
1. Tool return type and key naming
2. Error response structure
3. SQL dialect and query patterns
4. Agent session lifecycle sequence
5. ZenMethod signature and return format
6. ObjectScript naming conventions
7. AuditLog write pattern
8. Phase 1 gate validation method

### Naming Conventions

**ObjectScript Class and Method Naming:**

```objectscript
// Classes: PascalCase package segments, PascalCase class name
Class SAgent.Tools.Trace Extends %AI.Tool    // ✅
Class sAgent.tools.trace Extends %AI.Tool    // ❌

// ClassMethods on Tool classes: PascalCase verb-noun, "Get" prefix for reads
ClassMethod GetSessionSummary(...)           // ✅
ClassMethod fetchSession(...)                // ❌

// Local variables: camelCase
Set sessionId = ...    // ✅
Set SessionId = ...    // ❌

// %DynamicObject keys: camelCase
Set result.sessionId = ...   // ✅
Set result.SessionId = ...   // ❌
```

**SQL Column Aliases:**
```sql
-- camelCase aliases throughout — consistent with DynamicObject keys
SELECT hdr.%ID AS messageId          -- ✅
SELECT hdr.%ID AS MessageId          -- ❌
SELECT hdr.%ID AS message_id         -- ❌
```

### Tool Return Format (Universal Contract)

**All 13 tools MUST return `%DynamicObject`. Never return `%String`, `%Integer`, or raw values.**

**Success pattern:**
```objectscript
Set result = {}
Set result.sessionId = sessionId
Set result.messageCount = count
Return result
// Query tools: standard envelope auto-produced by <Query> element:
// {rows: [...], row_count: N, truncated: bool, elapsed_ms: N}
```

**Error pattern — ALL tools MUST use this exact structure on failure:**
```objectscript
Set result = {}
Set result.error    = "Human-readable description"
Set result.errorKey = "not_found"      // machine-readable: not_found | purged |
                                        // missing_class | invalid_input |
                                        // internal | access_denied
Set result.hint     = "Actionable suggestion for the operator"
Return result
```

**FORBIDDEN in tool implementations:**
```objectscript
$$$ThrowOnError(tSC)  // ❌ — use Try/Catch + error object
Quit tSC              // ❌ — always return %DynamicObject
Quit ""               // ❌ — always return %DynamicObject
```

### SQL Patterns (IRIS Dialect Rules)

```sql
-- SELECT TOP N (never LIMIT)
SELECT TOP 200 ...                                        -- ✅

-- Decode integers — always %EXTERNAL(), never raw
%EXTERNAL(hdr.Type)                                       -- ✅ "Request"/"Response"
hdr.Type                                                  -- ❌ returns 1 or 2

-- Decode %Status — always %ODBCOUT(), never raw
%ODBCOUT(hdr.ErrorStatus)                                 -- ✅ "ERROR #5001: ..."
hdr.ErrorStatus                                           -- ❌ garbled binary

-- Always %ID in JOINs (not ID)
ON resp.CorrespondingMessageId = hdr.%ID                  -- ✅

-- Time-bound CorrespondingMessageId joins — REQUIRED to prevent timeout
LEFT JOIN Ens.MessageHeader resp
  ON resp.CorrespondingMessageId = hdr.%ID
  AND resp.TimeCreated > DATEADD('hour', -2, CURRENT_TIMESTAMP)   -- ✅

-- ORDER BY %ID for session traces (not TimeCreated — can tie microseconds)
ORDER BY hdr.%ID                                          -- ✅

-- Always filter HS trace cruft
AND hdr.MessageBodyClassName <> 'HS.Util.Trace.Request'  -- ✅ always include

-- Optional SQL parameters: handle in SQL, not in ObjectScript defaults
WHERE (:status = '' OR hdr.Status = :status)             -- ✅
```

### Agent Session Lifecycle (HTTP Request Boundary Pattern)

**Both Phase 1 (REPL) and Phase 2 (ZenMethod) use the SAME sequence. Follow exactly.**

```objectscript
// 1. Create fresh agent each request (stateless)
Set agent = ##class(SAgent.Main.Agent).%New()
$$$ThrowOnError(agent.%Init())

// 2. Load existing session OR create new
Set chatId = $G(%session.Data("SAgent","SessionId"))
If chatId '= "" {
    Set chatSession = ##class(%AI.Agent.Session).Load(chatId, agent.Provider)
} Else {
    Set chatSession = agent.CreateSession({
        "temperature": 0.2,
        "cache": {"enabled": (1), "cache_system_prompt": (1)}})
}

// 3. Chat
Set response = agent.Chat(chatSession, userInput)

// 4. Persist — ALWAYS, even on apparent success
Do chatSession.%Save()
Set %session.Data("SAgent","SessionId") = chatSession.%Id()

// 5. Write audit log — ALWAYS
Do ##class(SAgent.Main.AuditLog).Write(
    chatSession.%Id(), ensSessionId, userInput,
    response.Content, chatSession.GetStats())
```

**CSP session data keys — use exactly these strings:**
```objectscript
%session.Data("SAgent","SessionId")      // %AI.Agent.Session %Id()
%session.Data("SAgent","EnsSessionId")   // Ens production SessionId (integer)
%session.Data("SAgent","NS")             // namespace when session was created
```

### ZenMethod Signatures (Phase 2 — Portal Chat)

```objectscript
// ✅ CORRECT — never add parameters (breaks generated client JS)
Method SendChatMessage(
    userInput As %String,
    selectedMessageId As %String = "") As %String [ ZenMethod ]

// ✅ CORRECT — reset uses ClassMethod
ClassMethod ResetChat() As %Boolean [ ZenMethod ]

// ✅ Return types: %String for chat (HTML content), %Boolean for actions
// ❌ NEVER return %DynamicObject from ZenMethod — Zen can't serialize it
```

### AuditLog Write Pattern

```objectscript
// Every chat turn in every phase — no exceptions
Do ##class(SAgent.Main.AuditLog).Write(
    chatSession.%Id(),   // string — %AI.Agent.Session %Id()
    ensSessionId,        // integer — Ens production SessionId
    userInput,           // string
    response.Content,    // string
    chatSession.GetStats())  // %DynamicObject from SDK

// Write() handles %Save() internally — callers MUST NOT %Save() the log row
```

### Track Boundary Rules

```objectscript
// ❌ FORBIDDEN in SAgent.Main.* or SAgent.Tools.*:
Set page = ##class(SAgent.Portal.VisualTrace).%New()
Set url  = "SAgent.Portal.VisualTrace.zen"

// ✅ Portal classes reference Main/Tools freely:
Set agent = ##class(SAgent.Main.Agent).%New()
Do ..UseToolSet("SAgent.Tools.Tools")
```

### Anti-Patterns (All Agents Must Avoid)

| Anti-pattern | Correct pattern |
|---|---|
| `Quit tSC` from a tool method | Try/Catch + return error %DynamicObject |
| `ZN "<TARGET_NS>"` inside any class | Package mapping makes this unnecessary |
| `hdr.ID` in JOINs | Always `hdr.%ID` |
| `ORDER BY TimeCreated` in session trace | `ORDER BY %ID` |
| Raw `hdr.ErrorStatus` in SELECT | `%ODBCOUT(hdr.ErrorStatus)` |
| `hdr.Type` (raw integer) in SELECT | `%EXTERNAL(hdr.Type)` |
| Hardcoded body class names in dispatch | Runtime `$CLASSNAME(body)` dispatch |
| Skipping `%Save()` after chat turn | Always `%Save()` + store `%Id()` in `%session.Data` |
| CorrespondingMessageId JOIN without time bound | Always add `AND resp.TimeCreated > DATEADD(...)` |

## Project Structure & Boundaries

### Complete Project Directory Structure

```
ready-hackathon-dev-template/         ← repo root
│
├── module.xml                         ZPM module descriptor — update from Sample.*
├── App.Installer.cls                  IRIS namespace installer (existing)
├── CLAUDE.md                          AI agent coding instructions for this project
├── README.md                          Installation guide (write this FIRST — P1 gate)
│
├── src/                               SourcesRoot for ZPM
│   └── SAgent/
│       ├── Main/                      ← TRACK A, Part 1: Agent runtime
│       │   ├── Agent.cls              %AI.Agent declarative subclass
│       │   │                             PROVIDER=openai, MODEL=gpt-5.4
│       │   │                             PROVIDERCONFIG, TOOLSETS, INSTRUCTIONS XData
│       │   ├── Shell.cls              Terminal REPL entry point
│       │   │                             ClassMethod Run(namespace, sessionId) As %Status
│       │   ├── ReadOnlyPolicy.cls     %AI.Policy.Authorization subclass
│       │   │                             Method %CanExecute() — blocks mutates=1 tools
│       │   └── AuditLog.cls           %Persistent audit trail
│       │                                 ClassMethod Write(chatId, ensId, in, out, stats)
│       │
│       ├── Tools/                     ← TRACK A, Part 2: Tool catalog
│       │   ├── Tools.cls              %AI.ToolSet composition root
│       │   │                             XData Definition — includes all tool groups
│       │   │                             Policies: ReadOnlyPolicy, ConsoleAudit
│       │   ├── Trace.cls              %AI.Tool — P1 + P2 query tools
│       │   │                             Query GetSessionSummary(sessionId)
│       │   │                             Query GetSessionTimeline(sessionId)
│       │   │                             Query GetMessageHeaders(sessionId)
│       │   │                             Query GetEventLog(sessionId, messageId, minSeverity)
│       │   │                             Query GetRuleLog(sessionId)
│       │   │                             Query FindRelatedSessions(superSessionId)
│       │   │                             Query FindSessionsByBody(bodyClass, field, value)
│       │   ├── Body.cls               %AI.Tool — P1 method tools
│       │   │                             ClassMethod GetMessageBody(messageId, maxBytes)
│       │   │                             ClassMethod GetMessageDetail(messageId)
│       │   ├── Errors.cls             %AI.Tool — P1 method tools
│       │   │                             ClassMethod ExplainError(statusText, messageId)
│       │   ├── Process.cls            %AI.Tool — P3 method tools
│       │   │                             ClassMethod GetBusinessProcessSource(class, method)
│       │   │                             ClassMethod GetBusinessProcessInstance(sessionId, bpId, class)
│       │   │                             Query ListBusinessProcessMethods(classname)
│       │   └── Meta.cls               %AI.Tool — P4 query tools
│       │                                 (FindRelatedSessions, FindSessionsByBody moved here from Trace)
│       │
│       ├── Portal/                    ← TRACK B: Management Portal integration
│       │   ├── VisualTrace.cls        Extends EnsPortal.VisualTrace
│       │   │                             XData allTabs — adds chatTab (4th tab)
│       │   │                             Method DrawChatUI(pSeed) — HTML skeleton
│       │   │                             Method SendChatMessage(input, msgId) [ZenMethod]
│       │   │                             ClassMethod ResetChat() [ZenMethod]
│       │   └── MessageViewer.cls      Extends EnsPortal.MessageViewer
│       │                                 ClientMethod showTrace(sessionId, evt) — 1 line
│       │
│       └── Test/                      %UnitTest.TestCase subclasses
│           ├── AgentTest.cls          Agent init, session create, read-only verify
│           ├── TraceTest.cls          GetSessionSummary, GetSessionTimeline queries
│           ├── BodyTest.cls           GetMessageBody variants (JSON/VDoc/Stream/null)
│           ├── ErrorsTest.cls         ExplainError pattern recognition
│           └── GateTest.cls           Phase 1 gate: all 7 gate criteria
│
├── docs/                              Project documentation
│   ├── installation.md                Detailed setup (derived from README)
│   ├── tool-api-reference.md          Per-tool parameter/return schema (P2 doc)
│   ├── example-queries.md             Common questions + expected agent responses
│   └── deployment-guide.md            HSCUSTOM → package mapping → bookmark URLs
│
└── _bmad-output/                      Planning artifacts
    └── planning-artifacts/
        ├── prd.md
        ├── architecture.md
        └── research/
```

### LLM Provider Configuration

**Default (2026 — OpenAI):**

```objectscript
Class SAgent.Main.Agent Extends %AI.Agent
{
    Parameter PROVIDER       = "openai";
    Parameter MODEL          = "gpt-5.4";
    Parameter PROVIDERCONFIG = "{""api_key"": ""@{wallet.AISecrets.OpenAIKey}""}";
    Parameter TOOLSETS       = "SAgent.Tools.Tools";
    // ...
}
```

**Swap to Anthropic (change 3 parameters, recompile Agent.cls, zero tool changes):**

```objectscript
    Parameter PROVIDER       = "anthropic";
    Parameter MODEL          = "claude-sonnet-4-6";
    Parameter PROVIDERCONFIG = "{""api_key"": ""@{wallet.AISecrets.AnthropicKey}""}";
```

**Upgrade OpenAI model only (keep same provider + key):**

```objectscript
    Parameter MODEL = "gpt-5.5";   // PROVIDER and PROVIDERCONFIG unchanged
```

**Wallet key names by provider:**

| Provider | Wallet key | Model (2026) |
|---|---|---|
| OpenAI (default) | `AISecrets.OpenAIKey` | `gpt-5.4` |
| Anthropic | `AISecrets.AnthropicKey` | `claude-sonnet-4-6` |
| AWS Bedrock | credential chain / `AISecrets.BedrockToken` | provider-specific |

### Requirements to Structure Mapping

| FR Category | FRs | Primary File(s) |
|---|---|---|
| Session Diagnostic Chat | FR1-9 | `SAgent.Main.Agent`, `SAgent.Main.Shell`, `SAgent.Tools.Tools` |
| Session Data Access | FR10-16 | `SAgent.Tools.Trace` (7 Query tools) |
| Message & Body Inspection | FR17-20 | `SAgent.Tools.Body` |
| Business Process Inspection | FR21-23 | `SAgent.Tools.Process` |
| Error & Log Analysis | FR24-26 | `SAgent.Tools.Errors` + `SAgent.Tools.Trace` (GetEventLog) |
| Portal Integration | FR27-30 | `SAgent.Portal.VisualTrace`, `SAgent.Portal.MessageViewer` |
| Deployment & Configuration | FR31-34 | `module.xml`, `README.md` |
| Audit & Governance | FR35-37 | `SAgent.Main.AuditLog`, `SAgent.Main.ReadOnlyPolicy` |

**Build track to file mapping:**

- **Track A** (build first): `SAgent.Main.*` + `SAgent.Tools.*` + `SAgent.Test.GateTest`
- **Track B** (parallel after gate): `SAgent.Portal.*`
- **Cross-cutting**: `SAgent.Main.AuditLog` (called by both tracks); `SAgent.Test.*` validates both

### Architectural Boundaries

**Data boundaries — what each layer reads/writes:**

```
SAgent.Tools.*       READ:  Ens.MessageHeader, Ens_Util.Log, Ens_Rule.Log,
                             Ens.SuperSessionIndex, {MessageBodyClass},
                             %Dictionary.ClassDefinition, %Dictionary.MethodDefinition
                     WRITE: nothing (read-only by design)

SAgent.Main.AuditLog WRITE: ^SAgent.Main.AuditLogD (own global only)

%AI.Agent.Session    READ/WRITE: ^AI.Agent.SessionD (SDK-managed)

SAgent.Portal.*      READ/WRITE: %session.Data("SAgent",...)
```

**Data flow per chat turn:**

```
User input
  → SAgent.Main.Shell.Run()          [Phase 1 — terminal]
  → SAgent.Portal.VisualTrace        [Phase 2 — ZenMethod]
       ↓
  SAgent.Main.Agent.Chat(session, input)
       ↓
  %AI.Agent dispatches tool calls to SAgent.Tools.*
       ↓ returns %DynamicObject
  %AI.Agent + LLM (gpt-5.4) synthesizes response
       ↓
  SAgent.Main.AuditLog.Write(...)
  %AI.Agent.Session.%Save()
  %session.Data("SAgent","SessionId") = id
       ↓
  Response text → user/UI
```

**Integration points:**

| Boundary | Technology | Direction |
|---|---|---|
| Tool → Ens data | IRIS SQL via `%SQL.Statement` | Read-only |
| Agent → LLM | OpenAI gpt-5.4 via `%AI.Provider` | Outbound |
| Portal → Agent | In-process ObjectScript call | Both |
| Portal → CSP session | `%session.Data(...)` | Read/Write |
| Track A → Track B | One-way dependency only | Track B uses Track A |

## Architecture Validation Results

### Coherence Validation ✅

All decisions work together without conflict:
- ObjectScript + AI Hub EAP build 141 + IRIS Zen portal — all native IRIS, zero framework conflicts
- `gpt-5.4` with `%AI.Provider.Create("openai")` — AI Hub SDK uses LiteLLM; OpenAI support confirmed
- `%AI.Agent.Session` (`%Persistent`) + ZenMethod AJAX + `%session.Data` pointer — verified SDK pattern; each layer owns its boundary
- `XData allTabs` override pattern — proven by InterSystems' own `Ens.Enterprise.Portal.VisualTrace` subclass
- HSCUSTOM + package mapping + CSP session inheritance — standard HealthShare pattern
- camelCase `%DynamicObject` keys, `%EXTERNAL`/`%ODBCOUT` decode, `%ID` pseudo-column, `ORDER BY %ID` — all internally consistent
- `SAgent.Main` → `SAgent.Tools` → `SAgent.Portal` dependency hierarchy is one-directional and enforceable

### Requirements Coverage Validation ✅

All 37 FRs and 17 NFRs covered:

| FR Category | FRs | File |
|---|---|---|
| Session Diagnostic Chat | FR1-9 | `SAgent.Main.Agent`, `Shell`, `SAgent.Tools.Tools` |
| Session Data Access | FR10-16 | `SAgent.Tools.Trace` |
| Message & Body Inspection | FR17-20 | `SAgent.Tools.Body` |
| Business Process Inspection | FR21-23 | `SAgent.Tools.Process` |
| Error & Log Analysis | FR24-26 | `SAgent.Tools.Errors` + `Trace.GetEventLog` |
| Portal Integration | FR27-30 | `SAgent.Portal.VisualTrace` + `MessageViewer` |
| Deployment & Configuration | FR31-34 | `module.xml`, `README.md`, package mapping |
| Audit & Governance | FR35-37 | `SAgent.Main.AuditLog`, `ReadOnlyPolicy` |

NFR coverage: Performance constraints in SQL patterns; 3-layer security documented; provider-swap satisfies integration NFR; graceful degradation satisfies reliability NFR.

### Gap Analysis

**Moderate (address in first stories):**
1. `SAgent.Tools.Tools.cls` XData Definition body — the `<ToolSet>` XML including all 5 tool groups not yet specified; define in first Tools story
2. `SAgent.Main.AuditLog` property schema — `Write()` signature given; the 8-9 persistent properties need a complete definition; address in AuditLog story

**Minor (acceptable for hackathon):**
3. `SAgent.Main.Shell` exact REPL adaptation — captured in research distillate; sufficient for single developer
4. Test fixture strategy — deferred to demo team (separate team member)
5. `README.md` content structure — sufficient to say "write early"; content follows from installation steps

### Architecture Completeness Checklist

- [x] Project context analyzed (37 FRs, 17 NFRs, high complexity, healthcare domain)
- [x] Technical constraints documented (IRIS 2026.2, AI Hub EAP build 141, ObjectScript only)
- [x] All critical decisions documented with current model IDs and versions
- [x] Provider swap pattern explicit (3 parameters, zero tool changes)
- [x] Implementation patterns cover 8 conflict areas with examples and anti-patterns
- [x] Complete file tree: 14 source files, 4 test classes, 4 doc files
- [x] Track A/B boundary enforced by package dependency rules
- [x] All 37 FRs mapped to specific files
- [x] Phase 1 gate: 7 explicit criteria defined

### Architecture Readiness Assessment

**Overall Status: READY FOR IMPLEMENTATION**
**Confidence Level: High**

**Key strengths:**
- Incremental gate makes hackathon risk manageable — every milestone is independently demonstrable
- Track A/B parallel structure reduces critical path after gate
- All 14 source files specified — developers can start coding without further design discussions
- Provider swap trivially documented — 3 parameters, zero downstream changes
- Anti-patterns table prevents the most common IRIS ObjectScript mistakes

**Future enhancements:** IPM module packaging; PHI deployment decision; `SAgent.Tools.Tools` XData body to be locked in first story

### Implementation Handoff — Build Order

```
1. SAgent.Main.AuditLog          no dependencies — compile first
2. SAgent.Main.ReadOnlyPolicy    no dependencies
3. SAgent.Tools.Trace            P1 core SQL tools
   SAgent.Tools.Errors           P1 ExplainError
   SAgent.Tools.Body             P1 body dispatch
4. SAgent.Tools.Tools            composition root
   SAgent.Main.Agent             wires tools + provider
   SAgent.Main.Shell             REPL entry point
   README.md                     write before demo

── PHASE 1 GATE ── (all 7 criteria must pass) ──────────────

   Track A continues              Track B starts in parallel
5a. SAgent.Tools.Process  P3     5b. SAgent.Portal.MessageViewer
6a. SAgent.Tools.Meta     P4     6b. SAgent.Portal.VisualTrace
```
