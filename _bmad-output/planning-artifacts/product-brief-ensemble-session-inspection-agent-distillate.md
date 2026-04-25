---
title: "Product Brief Distillate: Ensemble Session Inspection Agent"
type: llm-distillate
source: "product-brief-ensemble-session-inspection-agent.md"
created: "2026-04-24"
purpose: "Token-efficient context for downstream PRD creation. Dense bullets; each is self-contained. Do not assume the reader has the brief loaded."
---

# Product Brief Distillate: Ensemble Session Inspection Agent

## User & Problem Context

- **Primary users**: InterSystems IRIS/Ensemble integration engineers and operators — people who spend on-call hours debugging failed interoperability sessions
- **Deployment context**: IRIS for Health / HealthShare environments; many productions carry PHI (scoped to non-PHI namespaces for v1)
- **The core pain**: An Ensemble session leaves a trace across 5 separate data sources (message headers, message bodies, event log, rule log, BP runtime state); operators must manually join them in their heads, every time, starting from scratch
- **Secondary pain**: Junior engineers cannot do this at all without senior help; knowledge is not retained across incidents
- **Audience for this brief**: Hackathon judges AND internal team members for pitch alignment
- **Hackathon format**: Pre-research unlimited; 4-hour coding window; all 3 phases must be demonstrable to be competitive
- **PHI stance**: Deliberately excluded from v1 scope; addressed post-hackathon. User does not want PHI to slow down the build or pitch.
- **Value hierarchy**: (1) Time savings — primary; (2) Accessibility — junior engineers can do senior work; (3) Completeness — stop missing correlated errors across tabs

---

## Technical Architecture — Accepted Decisions

- **AI Hub SDK**: `%AI.Agent`, `%AI.ToolSet`, `%AI.Tool`, `%AI.Agent.Session`, `%AI.Policy.Authorization`, `%AI.MCP.Service`; EAP build 2026.2.0AI.141.0
- **Tool count**: 13 tools total — 9 SQL-driven `<Query>` tools, 4 ObjectScript method tools
- **SQL tools use `<Query>` element**: Validates SQL at compile time, auto-generates JSON schema, returns standard `{rows, row_count, truncated, elapsed_ms}` envelope — no extra boilerplate
- **Method tools reserved for**: Body dispatch (runtime class check + fallback chain), class introspection (`%Dictionary.*`), error decoding (`%Status` pattern matching)
- **Agent state persistence across HTTP**: `%AI.Agent.Session` is `%Persistent` — `Session.Load(id, provider)` restores Rust state; store `%Id()` in `%session.Data("EnsAgent","SessionId")`; call `%Save()` after each turn; reset if Ensemble session ID changes
- **Tool-schema state**: NOT stored in the session; the agent's ToolManager token is passed to every `Chat()` call separately — create the agent fresh each HTTP request, load only the session
- **Read-only enforcement**: 3 layers — (1) no `%Save`/`%DeleteId`/`ResubmitMessage` in method implementations; (2) `Custom.EnsSession.ReadOnlyPolicy extends %AI.Policy.Authorization` blocks tools with `mutates=1` metadata; (3) IRIS RBAC — SELECT-only grants on `Ens.*` tables for the agent runtime user
- **Deployment**: HSCUSTOM namespace; package mapping to `%ALL` (or each target NS); operators use per-namespace bookmark URLs; no Portal menu integration; no runtime `ZN` namespace switching in CSP context
- **CSP session inheritance**: Bookmark URL uses existing portal CSP app; same session cookie; no re-authentication
- **LLM provider**: Anthropic (claude-sonnet-4-5); API key in IRIS Wallet via `@{wallet.AISecrets.AnthropicKey}`; prompt caching enabled (`cache_system_prompt: 1`, `cache_tool_definitions: 1`)
- **Subclassing approach** (Phase 2/3): Subclass `EnsPortal.VisualTrace` and `EnsPortal.MessageViewer` — not parallel pages; prior art: `Ens.Enterprise.Portal.VisualTrace` already does this in IRIS source
- **Chat UI**: Server-rendered HTML skeleton via `OnDrawContent="DrawChatUI"`; client-side JavaScript calls ZenMethod `SendChatMessage(userInput, selectedMessageId)` — blocking, synchronous; ~15 sec max per turn acceptable for demo
- **Selected-message context**: `zenPage.currentId` updated by Visual Trace when user clicks a message; passed to `SendChatMessage`; server-side adds `[Context: inspecting Ens session X; user is focused on message Y]` prefix to the LLM prompt

---

## The 13 Tools (Full Catalog)

| # | Tool | Kind | Primary purpose |
|---|---|---|---|
| 1 | `GetSessionSummary` | Query | Shape, duration, error count, root message class |
| 2 | `GetSessionTimeline` | Query | UNION ALL of Ens.MessageHeader + Ens.Util.Log + Ens.Rule.Log, ordered by timestamp |
| 3 | `GetMessageHeaders` | Query | All messages in session with decoded status/type/invocation |
| 4 | `GetEventLog` | Query | Filterable by session, message, and minimum severity |
| 5 | `GetRuleLog` | Query | Rule evaluations with reason, return value, component, currentHeaderId |
| 6 | `FindRelatedSessions` | Query | Cross-instance sessions via `Ens.SuperSessionIndex` |
| 7 | `FindSessionsByBody` | Query | Sessions by indexed body fields via `{Table}_AdditionalInfo` pivot |
| 8 | `GetMessageBody` | Method | Runtime dispatch: %JSON.Adaptor → VDoc → %Stream → generic fallback |
| 9 | `GetMessageDetail` | Method | Combined header + body summary + related log entries |
| 10 | `GetBusinessProcessSource` | Method | %Dictionary.ClassDefinition + MethodDefinition.Implementation stream; BPL via XDataDefinition |
| 11 | `GetBusinessProcessInstance` | Method | BP persistent instance row + Ens.BP.Context + Ens.BP.Thread state |
| 12 | `ListBusinessProcessMethods` | Query | Enumerate methods on any class via %Dictionary.MethodDefinition |
| 13 | `ExplainError` | Method | Decode %Status; recognizes `<Ens>ErrBPTerm`, `<PROTECT>`, `<UNDEFINED>`, etc. |

---

## Key SQL Patterns (from IRIS Management Portal source)

- **Session root**: `SELECT * FROM Ens.MessageHeader WHERE %ID = SessionId` — first message has `%ID = SessionId`
- **Canonical trace query**: 14-column projection from `EnsPortal.SVG.VisualTrace.BuildTraceInfo` (IRIS's own Visual Trace); `ORDER BY %ID` (not TimeCreated — can tie in microseconds)
- **Timeline UNION ALL**: `Ens.MessageHeader` + `Ens_Util.Log` (join on SessionId) + `Ens_Rule.Log` (join on SessionId + CurrentHeaderId); discriminator column `EventKind`; filter `HS.Util.Trace.Request` bodies (HealthShare trace cruft)
- **Error location**: `IsError` and `ErrorStatus` are on `Type=2` (Response) headers — NOT on the request; join via `CorrespondingMessageId` to find
- **ErrorStatus rendering**: MUST use `%ODBCOUT(ErrorStatus)` in SQL or `$SYSTEM.Status.GetOneStatusText()` in ObjectScript — raw value is garbled binary
- **MessageBodyClassName case**: May be stored ALL-CAPS in some namespaces; IRIS SQL WHERE is case-insensitive but `$ClassMethod` and `%Dictionary` lookups are case-sensitive — always verify before using in code paths
- **%ID vs ID**: Always use `%ID` pseudo-column in JOINs — resolves correctly even when `ID` column is renamed
- **Timestamp decode**: `%EXTERNAL(Type)`, `%EXTERNAL(Status)`, `%EXTERNAL(Invocation)`, `%EXTERNAL(SourceBusinessType)` for human-readable values; `%ODBCOUT(ErrorStatus)` and `%ODBCOUT(StatusValue)` for %Status fields
- **CorrespondingMessageId join timeout**: On high-volume namespaces (e.g., 15k ops/day), joining on CorrespondingMessageId without a time-window constraint times out — add `AND resp.TimeCreated > DATEADD('hour', -1, CURRENT_TIMESTAMP)` to bound the scan

---

## Body Class Dispatch Chain

Order of operations in `GetMessageBody`:
1. If `MessageBodyClassName = ""` and `MessageBodyId = ""` → null body
2. If `MessageBodyClassName = ""` and `MessageBodyId ≠ ""` → literal scalar body
3. If class doesn't exist → return `{variant: "missing_class"}` gracefully (class may have been purged)
4. If `%OpenId` fails → return `{variant: "purged"}`
5. If class `%Extends("%JSON.Adaptor")` → `%JSONExportToStream()`
6. If class `%Extends("EnsLib.HL7.Message")` → segment-by-segment via `GetValueAt()` for key fields, `OutputToString()` per segment
7. If class `%Extends("%Stream.Object")` → `Read(4096)` excerpt with size and truncated flag
8. Else → `%ZEN.Auxiliary.altJSONProvider.%ObjectToAET(body)` generic fallback (same path the Management Portal uses)
9. All bodies from `Ens.MessageBody` subclasses share `^Ens.MessageBodyD` global — IDs are globally unique across the hierarchy

---

## Deployment Architecture

- **HSCUSTOM compilation**: All `Custom.EnsSession.*` and `Custom.EnsPortal.*` classes compile once in HSCUSTOM
- **Package mapping to `%ALL`**: Via Management Portal → System Administration → Configuration → Namespaces → target NS (or `%ALL`) → Package Mappings. Maps `Custom.EnsSession` and `Custom.EnsPortal` from HSCUSTOMCODE database. Standard HealthShare pattern.
- **Bookmark URL pattern**: `https://<host>/csp/healthshare/<TARGET_NS>/Custom.EnsPortal.MessageViewer.zen` — one bookmark per namespace; user selects namespace via which bookmark they use
- **Phase 2 tab**: Override `XData allTabs` in `Custom.EnsPortal.VisualTrace extends EnsPortal.VisualTrace` — exact same pattern used by `Ens.Enterprise.Portal.VisualTrace` (ships with IRIS). Add `<tab id="chatTab" caption="Chat" title="Chat about this session"><html id="chatUI" OnDrawContent="DrawChatUI" /></tab>` after the 3 existing tabs.
- **ZenMethod security**: `standardDialog.OnPreHyperEvent()` chains to `GetHyperEventResources()` — override to return `""` (no extra resource beyond page resource). Our custom class inherits `Parameter RESOURCE = "%Ens_MessageTrace:USE"`.
- **Phase 3 override** (entire change): Override `ClientMethod showTrace(sessionId, evt)` — change `'EnsPortal.VisualTrace.zen'` to `'Custom.EnsPortal.VisualTrace.zen'`. That's the complete implementation.
- **`$NAMESPACE` in CSP context**: Do NOT use `ZN` inside CSP/Zen hyperevents — it persists for the entire HTTP request. Package mapping makes this unnecessary.

---

## Hackathon Build Order & Time Budget

| Phase | Task | Budget | Milestone |
|---|---|---|---|
| 1 | Terminal shell (`Custom.EnsSession.Shell`) | 30-60 min | Terminal chat demo live |
| 3 | Custom MessageViewer (one method override) | 15-30 min | Search + handoff works |
| 2a | VisualTrace subclass skeleton + 4th tab | 60-90 min | Chat tab renders |
| 2b | `SendChatMessage` ZenMethod + JS wiring | 60-90 min | Full chat works end-to-end |
| — | Polish, markdown rendering, debug | 30-60 min | Demo-ready |

**Phase order rationale**: Build 1 → 3 → 2 so each milestone is independently demo-able. If Phase 2b slips, Phase 1 + Phase 3 + the chat tab skeleton (Phase 2a) is still a strong demo.

**Highest-risk step**: Phase 2b — `SendChatMessage` ZenMethod connecting agent state persistence to the UI. Have the terminal bot fallback ready as backup demo path.

---

## Rejected Decisions (Do Not Re-Propose)

| Idea | Rejected because |
|---|---|
| Build parallel Zen pages instead of subclassing | Subclassing proven by `Ens.Enterprise.Portal.VisualTrace` prior art; subclassing inherits all existing functionality free; parallel pages = full reimplementation |
| Runtime namespace switching via `ZN` in CSP | Dangerous in ZenMethod context — persists for entire HTTP request; can break framework code; package mapping to %ALL eliminates the need entirely |
| Portal menu integration | Requires modifying EnsPortal.Application (System=4, overwritten on upgrade); bookmark-based access is sufficient for operators and zero-risk |
| `%ALL` package map vs per-namespace | Per-namespace is more controlled but more ops burden; `%ALL` is standard HealthShare pattern and sufficient for the use case |
| Streaming LLM responses in Phase 2 chat | Requires WebSocket or SSE infrastructure; adds complexity; blocking ZenMethod acceptable for demo and post-demo production use |
| Cross-namespace inspection in a single turn | Requires `ZN` or `@ns` method syntax in CSP context; deferred; each bookmark targets one namespace |
| Portal navigation menu items | See "Portal menu integration" above |
| `%MessagesSent` / `%MessagesReceived` arrays on Ens.BusinessProcess for message history | `SKIPMESSAGEHISTORY = 1` is default — these arrays are typically empty; use `Ens.MessageHeader` filtered by SessionId + ConfigName instead |
| Using `Ens.Queue` for historical queue analysis | Ens.Queue is runtime globals, not a SQL table; use `Ens.MessageHeader.TargetQueueName + Status` for historical analysis |
| Querying Ens.Rule.RuleLog | Deprecated — use `Ens.Rule.Log` instead; richer schema with ConfigName + CurrentHeaderId + DebugId |

---

## Competitive Intelligence

- **Datadog Bits AI**: General-purpose cloud infra + APM; 51% observability market share; no IRIS instrumentation (Ensemble produces no OpenTelemetry)
- **Dynatrace Davis**: Causal correlation + anomaly detection; requires deep instrumentation; IRIS is a blind spot
- **Splunk AI**: NLQ over indexed logs; requires Splunk to ingest and index IRIS event logs first — significant pipeline overhead; still generic text search with no Ensemble session awareness
- **Boomi Integration Advisor Agent** (May 2025): Natural-language process *design* review; targets Boomi workflow *authors* not operators; operates at design-time not run-time — fundamentally different value prop from session trace diagnosis. Key differentiator soundbite: "Boomi helps you build a flow; this tells you why your flow failed at 2am."
- **IBM AIOps Log Analytics**: 60% of trial users saved 30+ minutes per incident on general IT logs — benchmark for ROI narrative; our domain specificity should produce even higher savings
- **InterSystems HealthShare AI Assistant** (November 2025) + IntelliCare (March 2025): Signals vendor AI roadmap and organizational readiness; creates AI Hub SDK as enabler; also the most credible future competitor
- **Market size**: Healthcare interoperability solutions $4.84B (2025) → $17.94B (2035); AI observability $X → $10.7B by 2033 at 22.5% CAGR; IRIS installed in majority of large U.S. health systems
- **Structural gap**: No AIOps tool purpose-built for IRIS/Ensemble. Gap is structural (no OTel), not a matter of feature parity.

---

## Requirements Hints (User-Stated Preferences)

- Multi-turn conversation — not single-turn Q&A; the agent must remember what was asked earlier in a session
- Chat must be aware of which message the user has selected in the Visual Trace (the `zenPage.currentId` value)
- Blocking (synchronous) chat is explicitly acceptable — no streaming required
- The 4-hour build window must yield all three phases or a graceful subset
- Read-only is non-negotiable — no writes, no resends, no queue mutations
- PHI: scoped out of v1; addressed in a follow-on deployment decision (self-hosted LLM vs. BAA-covered cloud provider)
- Open source publication: immediate next step post-hackathon before any commercial motion

---

## Open Questions (Unresolved)

- **LLM provider for production**: Anthropic is scoped for the demo. PHI-bearing environments need either (a) self-hosted LLM (Ollama/NIM — strongest PHI control), (b) Anthropic via AWS Bedrock with BAA, or (c) Azure OpenAI with BAA. Decision deferred to deployment.
- **Body redaction policy**: Even in "non-PHI" namespaces, Ensemble message bodies may carry PII or patient identifiers in HL7 headers (MSH, PID segments). The tool sends body content to the LLM. A body-field redaction layer may be needed even for nominally non-PHI environments. Currently unaddressed in the design.
- **Audit log implementation**: `Custom.EnsSession.AuditLog` persistent class to record every chat turn with timestamp, user, Ens session ID, tool calls, token counts, and duration. Designed but marked as first cut if time runs short.
- **Markdown → HTML rendering**: `DrawChatUI` response rendering uses a placeholder `MarkdownToHtml()` that does basic HTML escaping. A real Markdown renderer may be needed for production readability. Fallback: `<pre>` wrapper.
- **Session cleanup / purge policy**: `%AI.Agent.Session` rows accumulate in `^AI.Agent.SessionD`. No purge task is designed. For production: add a daily scheduled task or a `/reset` button in the chat UI that calls `session.%DeleteId()`.
- **EAP API churn**: AI Hub is pre-release (EAP 141.0). `%AI.*` API surface may change before GA. Recommendation: verify class/parameter names against installed IRIS before hackathon coding day.
- **Concurrent users**: Multiple operators on same portal user will share `%session.Data` → unexpected chat state collisions. Affects multi-user orgs. Currently unaddressed.

---

## Scope Signals

| Topic | Signal |
|---|---|
| PHI support | OUT for v1; IN post-hackathon pending LLM deployment decision |
| All 3 phases | IN for hackathon — treat as one cohesive product |
| Streaming responses | OUT — blocking is fine |
| Portal menu integration | OUT — bookmark only |
| Cross-namespace single conversation | OUT |
| Proactive alerting / anomaly detection | OUT for v1; noted as 2-3 year roadmap item |
| HL7 clinical field extraction | OUT — VDoc generic rendering only |
| Open source publication | IN — immediate next step post-hackathon |
| Commercial licensing | Future — via SI partner channel |
| MCP ecosystem exposure | Future — 2-3 year roadmap |
| Audit logging | IN if time allows; first to cut if not |

---

## Additional Technical Context for PRD

- **EnsPortal.VisualTrace parent chain**: `EnsPortal.Dialog.standardDialog → %CSP.Portal.standardDialog → EnsPortal.Template.base → %ZEN.Component.page`
- **EnsPortal.MessageViewer parent chain**: `EnsPortal.Template.filteredViewer → EnsPortal.Template.viewerPage`
- **MessageViewer tabs**: 4 tabs defined in `filteredViewer.XData detailsPane` (Header, Body, Contents, Trace); NOT defined in MessageViewer itself
- **VisualTrace tabs**: 3 tabs in `VisualTrace.XData allTabs` (Header, Body, Contents); Phase 2 adds a 4th
- **IRIS SQL dialect quirks**: `SELECT TOP N` (not LIMIT); `%ID` pseudo-column (not `ID`); `%NOLOCK` hint; `DATEADD('day', -N, CURRENT_TIMESTAMP)`; string comparison case-insensitive by default; `%EXACT()` to force case-sensitive; stream columns need `SUBSTRING(col, 1, N)` not `CONVERT`
- **Class introspection ID format**: `ClassName||MemberName` (double-pipe) for `%Dictionary.*` opens — e.g., `##class(%Dictionary.MethodDefinition).%OpenId("Custom.MyBP||OnRequest")`
- **BPL source location**: `##class(%Dictionary.XDataDefinition).%OpenId("Custom.MyBPL||BPL")` — the `.Data` property is a `%Stream.TmpCharacter` containing the BPL XML
- **Storage globals reference**: `^Ens.MessageHeaderD`, `^Ens.Util.LogD`, `^Ens.Rule.LogD`, `^Ens.BP.ContextD`, `^Ens.BP.ThreadD`, `^AI.Agent.SessionD`
- **Purge behavior**: `Ens.MessageHeader.Purge()` with `pKeepIntegrity=1` (default) will not delete headers from sessions that still have in-progress messages; body classes deleted via `$classmethod(bodyClass, "%DeleteId", bodyId)` if class exists and `ENSPURGE ≠ 0`
- **Prior-art reference projects in repo**: `sources/diagramtool/` (Mermaid diagram generator over Ens.MessageHeader, includes validated correlation algorithm in `docs/dev-notes-correlation.md`); `iris-view-agent/` at `/Users/jbrandt/iris-view-agent/` (sibling agent with production-tested `learned-schemas/` for Ens.MessageHeader and Ens.Util.Log)
