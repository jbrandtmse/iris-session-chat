---
stepsCompleted: [step-01-init, step-02-discovery, step-02b-vision, step-02c-executive-summary, step-03-success, step-04-journeys, step-05-domain, step-06-innovation, step-07-project-type, step-08-scoping, step-09-functional, step-10-nonfunctional, step-11-polish, step-12-complete]
status: complete
classification:
  projectType: developer_tool
  domain: healthcare
  complexity: high
  projectContext: greenfield
inputDocuments:
  - "_bmad-output/planning-artifacts/product-brief-ensemble-session-inspection-agent.md"
  - "_bmad-output/planning-artifacts/product-brief-ensemble-session-inspection-agent-distillate.md"
  - "_bmad-output/planning-artifacts/research/technical-ensemble-session-inspection-agent-research-2026-04-24.md"
  - "_bmad-output/planning-artifacts/research/technical-ensemble-session-agent-ui-integration-research-2026-04-24.md"
workflowType: 'prd'
---

# Product Requirements Document - Ensemble Session Inspection Agent

**Author:** Developer
**Date:** 2026-04-24

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Project Classification](#project-classification)
3. [Success Criteria](#success-criteria)
4. [Product Scope](#product-scope)
5. [User Journeys](#user-journeys)
6. [Domain-Specific Requirements](#domain-specific-requirements)
7. [Innovation & Novel Patterns](#innovation--novel-patterns)
8. [Developer Tool Specific Requirements](#developer-tool-specific-requirements)
9. [Project Scoping & Phased Development](#project-scoping--phased-development)
10. [Functional Requirements](#functional-requirements)
11. [Non-Functional Requirements](#non-functional-requirements)

---

## Executive Summary

The Ensemble Session Inspection Agent is a read-only, AI-powered diagnostic tool that gives IRIS/Ensemble integration engineers conversational access to session trace intelligence. When a production session fails — or succeeds in a way that's hard to explain — operators currently spend 20-30 minutes manually correlating five separate data sources (message headers, message bodies, event log, rule log, Business Process runtime state) across multiple Management Portal tabs. This agent replaces that process with a question: *"What happened in session 42751?"*

The answer comes back in under 30 seconds: a plain-English narrative of the full session arc, decoded errors, rule decisions, body contents, and Business Process behavior — available as a multi-turn conversation that persists context and responds to follow-up questions. The tool deploys in three phases across a 4-hour hackathon build: a terminal REPL bot, a Chat tab injected into the existing Visual Trace page, and a custom Message Viewer that routes the full search-to-chat workflow. All three phases are demonstrable milestones; each is independently valuable if time runs short.

**Target users:** Integration engineers and operators on IRIS/Ensemble productions — from on-call staff diagnosing failures at 11pm to junior developers who previously needed to escalate. Secondary: QA teams and engineering leads who need plain-English session narratives for incident reports and post-mortems.

**Platform:** IRIS for Health 2026.2, AI Hub SDK EAP build 141. Compiled in HSCUSTOM; available in target namespaces via package mapping and bookmark URL. Non-PHI namespaces in v1.

### What Makes This Special

No AI diagnostic tool purpose-built for Ensemble exists. General-purpose AIOps platforms (Datadog, Dynatrace, Splunk) treat IRIS as a black box — Ensemble produces no OpenTelemetry instrumentation, so they can't see inside a session. This agent was built from the source: it knows where errors live (on response headers, not requests), how to resolve dynamically-typed message bodies at runtime, how to read BPL XML from `%Dictionary.XDataDefinition`, and how to decode raw `%Status` binary values. 13 specialized read-only tools cover every diagnostic question an operator would ask; tool schemas are generated at compile time from ObjectScript method signatures; conversation state persists across HTTP requests via `%AI.Agent.Session`, a built-in SDK primitive.

The core insight: the Ensemble session data model is a well-defined, queryable graph. `Ens.MessageHeader` is the spine; every other artifact (bodies, logs, rules, BP state) ties to it via three keys. The InterSystems AI Hub SDK — launched EAP in 2026.2 precisely to enable native IRIS agents — provides the execution primitives that make this buildable in an afternoon. No competitor can replicate the domain knowledge without rebuilding it from scratch.

**Defensible position:** First-mover domain authority in a structural market gap, open-source publication path post-hackathon, and a natural expansion roadmap (proactive anomaly surfacing, HealthShare-specific workflows, MCP ecosystem exposure) that compounds value over time.

## Project Classification

- **Project Type:** Developer tool — specialized diagnostic instrumentation for integration engineers
- **Domain:** Healthcare (HealthShare/IRIS For Health interoperability environments)
- **Complexity:** High — AI Hub EAP APIs, complex Ensemble data model, dynamic body type dispatch, Zen portal subclassing, regulated-adjacent data environment
- **Project Context:** Greenfield

## Success Criteria

### User Success

- An integration engineer enters a namespace and session ID and receives a plain-English narrative of what happened in that session — without opening any Management Portal tabs
- The agent correctly answers all four core diagnostic questions: "what happened," "what's in the body," "what did the BP do," "what does this error mean"
- A junior engineer independently diagnoses a failed session that would previously have required senior escalation
- The diagnostic conversation is multi-turn: follow-up questions build on earlier context
- Operators using the portal can open any session from the Message Viewer and chat with it in the Visual Trace — the full integrated workflow

### Business Success

- All three phases delivered within the hackathon build window, each independently demonstrable
- Published as open source post-hackathon, usable by any IRIS operator with HSCUSTOM access
- Time-to-diagnosis reduced by ≥50% vs. manual tab-switching in post-hackathon user testing
- Junior engineers independently resolving sessions that previously required escalation

### Technical Success

- Read-only enforcement holds across all phases: no `%Save`, `%DeleteId`, or queue-mutation calls in any tool
- Terminal bot multi-turn state persistent across REPL turns via `%AI.Agent.Session`
- Chat tab ZenMethod response time ≤30 seconds per turn (demo threshold; no production SLA defined)
- All 13 tools return graceful structured responses on edge cases (purged bodies, missing classes, errors)
- Package mapping to HSCUSTOM working in at least one target namespace
- Phase 2 portal subclassing stable: `Custom.EnsPortal.VisualTrace` inherits all existing Visual Trace behavior with no regressions

### Measurable Outcomes

- 4-hour build window yields all three phases functional
- Agent answers all four core diagnostic questions for any realistic session in the demo environment
- ≥50% time-to-diagnosis reduction vs. manual tab-switching (post-hackathon validation)

## Product Scope

### MVP — Phase 1: Terminal Bot

Build the terminal REPL first with an incremental tool rollout strategy — start with the core 5 tools that answer the most common questions, verify the agent is working, then expand to the full catalog before moving to portal integration:

**Core tool subset (start here):** `GetSessionSummary`, `GetSessionTimeline`, `GetEventLog`, `ExplainError`, `GetMessageBody`

**Full catalog (expand before Phase 2):** Add `GetRuleLog`, `GetMessageHeaders`, `GetMessageDetail`, `GetBusinessProcessSource`, `GetBusinessProcessInstance`, `ListBusinessProcessMethods`, `FindRelatedSessions`, `FindSessionsByBody`

**Deliverable:** `Custom.EnsSession.Shell` — `Do ##class(Custom.EnsSession.Shell).Run("NAMESPACE", sessionId)` launches a multi-turn diagnostic conversation with the full 13-tool catalog.

### Growth Features — Phase 2: Chat Tab in Visual Trace

Subclass `EnsPortal.VisualTrace` to add a fourth "Chat" tab alongside Header, Body, and Contents. The agent is context-aware: it knows which session is displayed and which message is selected. Multi-turn state persists across HTTP requests via `%AI.Agent.Session`.

**Deliverable:** `Custom.EnsPortal.VisualTrace` — accessible via bookmark URL `https://<host>/csp/healthshare/<NS>/Custom.EnsPortal.VisualTrace.zen?SESSIONID=<id>`.

### Phase 3: Custom Message Viewer

Subclass `EnsPortal.MessageViewer` to route session-link clicks to the Chat-enabled Visual Trace. Operators bookmark one URL per namespace and get the full search → select → chat workflow.

**Deliverable:** `Custom.EnsPortal.MessageViewer` — one-line `showTrace()` override; bookmark URL `https://<host>/csp/healthshare/<NS>/Custom.EnsPortal.MessageViewer.zen`.

### Vision (Post-Hackathon)

*See [Project Scoping & Phased Development](#project-scoping--phased-development) for the full post-hackathon expansion plan.*

## User Journeys

### Journey 1: Alex — The On-Call Engineer (Primary User, Happy Path)

Alex is a mid-level integration engineer at a regional health system. It's 11:42pm. Her phone pings — a monitoring alert: "XCPD message delivery failures, 17 errors in the last 10 minutes." She opens her laptop, bleary-eyed.

The alert includes a session ID. Six months ago, she would have opened four browser tabs — Visual Trace, Event Log, Rule Log, and the Business Process source code in Studio — and spent the next half hour piecing together what happened. Tonight, she opens a terminal.

```
Do ##class(Custom.EnsSession.Shell).Run("CQGATEWAY", 38903)
```

*"What happened in this session?"*

In 22 seconds, the agent responds with a plain narrative: the incoming XDS query arrived, the router correctly identified the target operation, but the outbound call timed out because the downstream registry was unreachable. The error was on the response header — she'd have missed it in the Visual Trace. The agent decoded the `%Status` value into readable text.

*"Why did the router choose that path?"* — The agent reads the rule log: the routing rule matched on the message type and sent it to the correct operation. The rule logic is quoted back to her.

It's 11:53pm. She's already filing the incident ticket. The downstream system is the problem, not the integration. She pages the infrastructure team and goes back to sleep.

**Requirements revealed:** Terminal REPL, multi-turn state, GetSessionSummary, GetSessionTimeline, GetEventLog, ExplainError (Priority 1), GetRuleLog (Priority 2).

---

### Journey 2: Jordan — The Junior Developer (Primary User, Edge Case / First Use)

Jordan joined the team three months ago. He knows ObjectScript basics but has never diagnosed a production session failure on his own. His senior engineer is on PTO. An ADT feed has been silent for two hours and the clinical team is calling.

The senior engineer texts: "Look at session 42751 in CENGATEWAY. Use the chat tab."

Jordan opens the Management Portal, finds the session in the Message Viewer, and clicks through to the Visual Trace. He sees a fourth tab he hasn't noticed before: **Chat**.

He clicks it and types: *"What happened?"*

The agent responds with a paragraph he can actually read — no `%Status` codes, no ObjectScript class names he doesn't recognize. It tells him the message arrived, was routed to the correct business operation, and failed because a configuration property was pointing to a decommissioned endpoint. It even tells him which property and which class.

Jordan copies the agent's response into a Slack message to the clinical team, then opens the production configuration to check the endpoint value. He files the ticket with the root cause already written. For the first time, he didn't have to escalate.

**Requirements revealed:** Chat tab in Visual Trace (Phase 2), Management Portal bookmark, plain-English output, selected message context awareness, GetMessageBody (Priority 1), GetBusinessProcessSource (Priority 3).

---

### Journey 3: Sam — The IRIS Administrator (Admin/Operations User)

Sam manages IRIS infrastructure for a healthcare network with eight active namespaces. She's heard about the Session Inspection Agent and wants to deploy it.

Her setup checklist:
1. Compile all `Custom.EnsSession.*` and `Custom.EnsPortal.*` classes in **HSCUSTOM** — one compilation, no per-namespace duplication
2. Add package mapping from HSCUSTOM to the two most active namespaces (CENGATEWAY, CQGATEWAY) — she'll expand to others after validation
3. Create the IRIS Wallet entry `AISecrets.AnthropicKey` with the API key
4. Distribute bookmark URLs to the engineering team: `https://<host>/csp/healthshare/CENGATEWAY/Custom.EnsPortal.MessageViewer.zen`

She opens a terminal and runs validation: `Do ##class(Custom.EnsSession.Shell).Run("CENGATEWAY", 12345)`. The agent responds. She verifies read-only behavior — no new rows in any Ens table. Two hours from first compile to team-wide availability.

**Requirements revealed:** HSCUSTOM single-compile deployment, package mapping to target namespaces, IRIS Wallet credential management, bookmark URL per namespace, CSP session inheritance (no second login), read-only verifiable.

---

### Journey 4: Morgan — The Engineering Lead (Secondary User, Post-Incident Review)

Morgan leads the integration engineering team at a payer organization. After a critical XCA failure disrupted provider directory lookups for three hours, she's running the post-incident review.

She has the session ID from the monitoring ticket. She opens the Management Portal, finds the session in the Message Viewer, and clicks through to the Visual Trace. She clicks the Chat tab.

*"Summarize what happened in this session in plain English, suitable for a non-technical incident report."*

The agent produces three paragraphs: what triggered the session, the sequence of messages, where the failure occurred, and why. Morgan asks a follow-up: *"Was this a code issue or a configuration issue?"* — the agent reads the Business Process source, confirms no code changes in the last 30 days, and concludes the root cause was an endpoint configuration mismatch.

She closes the review in 20 minutes instead of two hours. The agent's narrative becomes the official incident summary.

**Requirements revealed:** Plain-English narrative output, GetBusinessProcessSource (Priority 3), multi-turn reasoning, copy-paste artifact quality output.

---

### Journey Requirements Summary

| Journey | Key Capabilities Required | Tool Priorities Needed |
|---|---|---|
| Alex (on-call) | Terminal REPL, multi-turn, decoded errors, rule log | Priority 1 + Priority 2 (#6) |
| Jordan (junior dev) | Chat tab, bookmark URL, plain-English, message context | Priority 1 + Phase 2 + Phase 3 |
| Sam (admin) | HSCUSTOM compile, package mapping, Wallet, read-only | Deployment infrastructure |
| Morgan (eng lead) | BP source reasoning, narrative output, follow-up questions | Priority 1 + Priority 3 (#8) |

### Tool Build Priority

| Priority | Tools | Unlocks |
|---|---|---|
| **P1** | GetSessionSummary, GetSessionTimeline, GetEventLog, ExplainError, GetMessageBody | Terminal bot demo-ready; Phase 2 + 3 portal work can begin in parallel |
| **P2** | GetRuleLog, GetMessageHeaders | Routing explanation; Alex's full journey |
| **P3** | GetBusinessProcessSource, GetBusinessProcessInstance, GetMessageDetail | Code reasoning; Morgan's and Jordan's full journeys |
| **P4** | ListBusinessProcessMethods, FindRelatedSessions, FindSessionsByBody | Depth and discovery features |

### Parallel Build Strategy

Once Priority 1 tools are validated end-to-end in the terminal:

```
Track A (tool expansion):  P2 → P3 → P4 (deepen terminal catalog)
Track B (portal):          Phase 3 (15 min) → Phase 2a skeleton → Phase 2b wiring
```

Track B requires only a working agent — not the full tool catalog. Phase 2 chat tab demo is compelling with Priority 1 tools alone. Priority 2-4 tools enrich answer quality on both tracks simultaneously. No single point of failure: any combination of completed tracks yields a demonstrable product.

## Domain-Specific Requirements

*This is a healthcare IT operations tool, not a clinical system. FDA/medical device classification, clinical validation, and patient safety certification do not apply. The following concerns are scoped to the actual deployment context.*

### Compliance & Regulatory

- **HIPAA-adjacent data handling (primary constraint):** Ens.MessageHeader, event logs, and message bodies in production Ensemble namespaces routinely contain PHI (HL7 PID segments, MRNs, dates of service). Sending session content to a cloud LLM endpoint may constitute "disclosure" under HIPAA if PHI is present.
  - **v1 mitigation:** Scoped to non-PHI namespaces only — enforced by deployment convention, not by technical controls
  - **PHI deployment options (post-v1):** Self-hosted LLM (Ollama/NIM — strongest control, data never leaves IRIS host); BAA-covered cloud provider (AWS Bedrock + Anthropic with BAA, or Azure OpenAI with BAA); body-field redaction at tool boundary (defense-in-depth only, not a substitute for the above)

### Audit Trail

- Every chat interaction with session data should be auditable: who asked what, about which session, when, and what tools were invoked
- `Custom.EnsSession.AuditLog` (persistent class) records: timestamp, IRIS username, Ens session ID, chat session ID, tool calls made, token counts, and duration
- Designated a v1 requirement (not optional); first item to cut only if time runs critically short

### Access Control

- The tool inherits IRIS RBAC via the portal CSP session — users can only access sessions they could already access in the Management Portal (`%Ens_MessageTrace:USE` resource)
- No new privilege grants required beyond those already held by the operator
- Read-only enforcement is structural (three layers): method-level, AI Hub policy, IRIS SQL grants

### Technical Constraints

- **LLM provider:** Anthropic claude-sonnet-4-5 for v1 (non-PHI); provider must be swappable at the agent class level without code changes to tools
- **Data transmission:** Session body content is deserialized server-side; only decoded/rendered content — not raw binary — is sent to the LLM endpoint
- **No patient-facing surface:** This tool is used exclusively by integration engineers and operators — never exposed to patients or clinical workflows
- **Deployment isolation:** HSCUSTOM namespace compilation; target namespace access via package mapping; no cross-namespace data bleed

## Innovation & Novel Patterns

### Detected Innovation Areas

**1. First native AI agent purpose-built for the Ensemble session data model**
No AI tool has been built for the IRIS/Ensemble session trace model. Competitors treat IRIS as a black box because Ensemble produces no OpenTelemetry — they cannot see inside a session without custom instrumentation. This agent is the first to encode Ensemble domain knowledge (session graph structure, dynamic body dispatch, %Status encoding, BPL XML introspection) into an AI tool catalog. The domain knowledge is the moat.

**2. First production use of the AI Hub SDK `<Query>` declarative tool pattern**
The tool catalog uses the `<Query>` element in `%AI.ToolSet` XData — a new SDK primitive that validates SQL at compile time and auto-generates JSON schemas for the LLM. This product is among the first to use this pattern in a real-world domain, demonstrating the SDK's productivity claims with a concrete, complex use case.

**3. Conversational session introspection as a new interaction paradigm**
The existing interaction paradigm is: open tabs, mentally join tables, read code. The new paradigm is: ask a question, get a narrative. This is a genuine interface shift — not a dashboard or a query builder, but a diagnostic conversation with context that persists across turns and understands which message is currently selected.

**4. Multi-turn agent state spanning HTTP requests via built-in SDK persistence**
`%AI.Agent.Session` is a `%Persistent` class with `Load(id, provider)` and `%Save()` — enabling multi-turn conversations across stateless HTTP requests inside a portal page with no external infrastructure. This solves a non-trivial problem that general-purpose web frameworks require message queues or websockets to address.

### Market Context & Competitive Landscape

- **Structural gap:** No AIOps tool has built IRIS/Ensemble integration because Ensemble produces no OpenTelemetry. General-purpose tools (Datadog, Dynatrace, Splunk) cannot see inside a session without custom connectors that don't exist.
- **Closest analog validated:** Boomi Integration Advisor Agent (May 2025) — natural-language process review on a competing iPaaS — demonstrates market readiness. Critical difference: Boomi addresses design-time process review; this addresses run-time session diagnosis. Different problem, same buyer insight.
- **Platform provider as enabler:** InterSystems launched AI Hub EAP in 2026.2 — their own platform investment validates that native IRIS agents are the right architecture. The SDK was built precisely for this kind of use case.
- **First-mover window:** The gap is structural, not a lag. But it won't stay open indefinitely — InterSystems could ship something similar natively, or a large AIOps vendor could commission a custom IRIS connector.

### Validation Approach

- **Hackathon demo:** Agent answers all four core diagnostic questions for a realistic session — observable in real time by judges. No hidden assumptions; the output is verifiable against the actual session data.
- **Post-hackathon:** Structured time-to-diagnosis comparison — same engineer, same class of failed sessions, with and without the agent. Target: ≥50% reduction. Secondary: junior engineer first-use study.
- **Technical validation:** Read-only enforcement verified by absence of mutation calls; session state persistence verified by multi-turn coherence test (follow-up question references prior turn correctly).

### Risk Mitigation

| Innovation Risk | Mitigation |
|---|---|
| AI Hub EAP API churn before GA | Thin wrappers around `%AI.Provider.Create` and `%AI.Agent.%New`; re-verify on each IRIS build |
| LLM hallucination on BP source interpretation | Low temperature (0.2); factual system prompt; structured tool outputs (JSON, not prose); explicit "say uncertain" instruction |
| Portal subclassing breaks on IRIS upgrade | InterSystems ships `Ens.Enterprise.Portal.VisualTrace` using the same pattern — low risk |
| No production sessions available for demo | Seed the demo environment with scripted test sessions with known failure patterns before hackathon day |

## Developer Tool Specific Requirements

### Project-Type Overview

The Ensemble Session Inspection Agent is distributed as a set of ObjectScript classes compiled into the HSCUSTOM namespace and made available to target namespaces via IRIS package mapping. Distribution is compile-from-source for v1 (hackathon); IPM module packaging is deferred to post-hackathon as a first-priority follow-on task.

### Language Matrix

| Language | Role | Status |
|---|---|---|
| ObjectScript | Primary — all tool classes, agent, shell, portal subclasses | v1 |
| Python (%AI bindings) | Not in scope | Post-v1 if needed |
| Java | Not in scope | — |

All tools are `%AI.ToolSet`/`%AI.Tool` subclasses in ObjectScript. The AI Hub SDK manages the LLM interaction layer; no Python or external runtime is required for v1.

### Installation Methods

**v1 — Manual compilation:**

1. Clone/copy source to HSCUSTOM namespace
2. Compile `Custom.EnsSession.*` and `Custom.EnsPortal.*` package trees
3. Create IRIS Wallet entry: `AISecrets.AnthropicKey`
4. Add package mapping from HSCUSTOM to target interop namespaces (via Management Portal or programmatically)
5. Distribute per-namespace bookmark URLs to engineering team

**Post-hackathon — IPM module:**
Package as a ZPM/IPM module with `module.xml` descriptor; single-command install: `zpm install ens-session-agent`. Wallet key setup remains a manual post-install step.

### API Surface (The 13 Tools as Public Interface)

The tool catalog is the product's public API. Each tool is a named, documented, callable endpoint that the LLM invokes during a conversation. The API surface is defined by the `XData Definition` block in each tool class — JSON schemas are auto-generated from ObjectScript method signatures.

**Tool categories:**

- **Session overview:** `GetSessionSummary`, `GetSessionTimeline`, `GetMessageHeaders`
- **Event & rule data:** `GetEventLog`, `GetRuleLog`
- **Message detail:** `GetMessageBody`, `GetMessageDetail`
- **Business process:** `GetBusinessProcessSource`, `GetBusinessProcessInstance`, `ListBusinessProcessMethods`
- **Error handling:** `ExplainError`
- **Discovery:** `FindRelatedSessions`, `FindSessionsByBody`

### Documentation Plan

Documentation is built incrementally alongside story development — each story that ships a tool should include its documentation artifact.

| Artifact | Priority | When |
|---|---|---|
| README — installation steps, prerequisites, wallet setup, first run | **P1 — build early** | Before any demo |
| Per-tool descriptions (embedded in `///` doc comments + `<Description>` XML) | **P1 — per story** | As each tool is built |
| Example queries (common diagnostic questions + expected output) | P2 — incremental | After Priority 1 tools complete |
| Tool API reference (full parameter/return schema per tool) | P2 — incremental | After core tools stable |
| IPM install guide | Post-hackathon | After IPM packaging |

*Test data / demo sessions: out of scope — handled by a separate team member with a dedicated demo solution.*

### Implementation Considerations

- **No external runtime dependencies** — the agent runs fully inside IRIS; no Node.js, Python venv, or external process management
- **AI Hub EAP dependency** — requires IRIS 2026.2 with AI Hub EAP build 141 or later; document this prominently in the README
- **Wallet prerequisite** — the LLM API key must be in the IRIS Wallet before the agent initializes; installation fails gracefully with a clear error if the key is missing
- **Package mapping scope** — mapping to `%ALL` is the simplest deployment; per-namespace mapping gives more control; README should document both options
- **IRIS version lock** — EAP APIs may change; README should note the tested IRIS build and recommend re-verification after IRIS upgrades

## Project Scoping & Phased Development

### MVP Strategy & Philosophy

**MVP Approach:** Problem-solving MVP — the minimum code that proves the core diagnostic capability works. A terminal conversation that answers "what happened in this session?" is the atomic unit of value. Everything else builds on top of that.

**Resource Requirements:** 2-person team optimal for parallel execution (Track A tools + Track B portal); solo execution feasible with the milestone ladder as the fallback plan. No external infrastructure required — runs entirely inside IRIS.

### MVP Feature Set — Phase 1: Terminal Bot

**Core user journey supported:** Alex (on-call engineer) — enters namespace + session ID, asks diagnostic questions, gets plain-English answers.

**Must-have capabilities (Priority 1 tools, first):**

| Capability | Tool | Why essential |
|---|---|---|
| Session shape overview | `GetSessionSummary` | First question every operator asks |
| Full interleaved narrative | `GetSessionTimeline` | Core value proposition |
| Event log access | `GetEventLog` | Operational context |
| Error decoding | `ExplainError` | Most common follow-up |
| Message body contents | `GetMessageBody` | "What's in this message?" |
| Multi-turn conversation state | `%AI.Agent.Session` | Without this it's single-Q&A, not a diagnostic session |
| README with install steps | Documentation | Required before any demo |

**Success gate:** Agent answers all 4 core diagnostic questions for a realistic session in the demo environment. Once this gate is passed — Phase 2 + 3 development can begin in parallel.

### Post-MVP Features — Phases 2 and 3 (Same Hackathon Day)

**Phase 2 expansion (Track A — tool catalog enrichment):**
- Priority 2: `GetRuleLog`, `GetMessageHeaders` — routing explanation
- Priority 3: `GetBusinessProcessSource`, `GetBusinessProcessInstance`, `GetMessageDetail` — code reasoning
- Priority 4: `ListBusinessProcessMethods`, `FindRelatedSessions`, `FindSessionsByBody` — depth and discovery

**Phase 2 parallel (Track B — portal integration):**
- `Custom.EnsPortal.VisualTrace` — 4th Chat tab in Visual Trace
- `Custom.EnsPortal.MessageViewer` — session-link handoff override (~15 min)

### Post-Hackathon Expansion (Vision)

- IPM module packaging for one-command install
- PHI namespace support (LLM deployment decision)
- Proactive anomaly surfacing
- HealthShare-specific workflows (XCPD, XCA, XDS)
- MCP ecosystem exposure
- Open source publication → commercial licensing path

### Risk Mitigation Strategy

**Technical risks:**
- *AI Hub EAP API churn* → Thin wrappers around `%AI.Provider` and `%AI.Agent`; re-verify on each build; document tested IRIS version in README
- *Portal subclassing instability* → InterSystems ships the same pattern in `Ens.Enterprise.Portal.VisualTrace`; if they break it, they break their own code

**Resource risks:**
- *Phase 2b (ZenMethod wiring) slips* → Terminal bot (Phase 1) + MessageViewer handoff (Phase 3) + chat tab skeleton (Phase 2a) is still a compelling demo. No single point of failure.
- *Solo builder* → Build order (P1 → P3 → P2a → P2b) gives maximum demonstrable output at every checkpoint

**Market/demo risk:**
- *No realistic failed sessions available* → Separate team member is building demo solution and test data; explicitly out of scope for the engineering team

## Functional Requirements

*This is the capability contract. Every downstream design, architecture, and story must trace to a requirement listed here.*

### Session Diagnostic Chat

- **FR1:** Operator can initiate a multi-turn diagnostic conversation about a specific session by providing a namespace and session ID
- **FR2:** Operator can ask follow-up questions within a conversation that correctly reference and build on prior turns
- **FR3:** Operator can ask "what happened in this session?" and receive a plain-English narrative covering the full session arc — messages, errors, log events, and routing decisions
- **FR4:** Operator can ask "what does this error mean?" and receive a decoded, human-readable explanation with error category
- **FR5:** Operator can ask "what does the body of this message contain?" and receive decoded message body contents
- **FR6:** Operator can ask "why did the router take this path?" and receive a rule-based explanation
- **FR7:** Operator can ask "what did the Business Process do?" and receive a source-code-informed explanation of BP behavior
- **FR8:** Operator can reset the conversation to start fresh on the same or a different session
- **FR9:** Agent conversation state persists correctly across turns — each follow-up question has full access to prior exchanges within the same conversation

### Session Data Access

- **FR10:** Operator can retrieve a session summary — message count, error count, session duration, root message class — by session ID
- **FR11:** Operator can retrieve a chronologically-ordered interleaved timeline of all messages, event log entries, and rule evaluations for a session
- **FR12:** Operator can retrieve all message headers for a session with human-readable status, type, invocation, and component names
- **FR13:** Operator can retrieve event log entries for a session, filtered by message ID and minimum severity level
- **FR14:** Operator can retrieve rule evaluation results for a session, including rule name, component, reason for firing, and return value
- **FR15:** Operator can retrieve sessions correlated to a given cross-instance session identifier (SuperSession)
- **FR16:** Operator can find sessions by indexed body field values (e.g., patient MRN, organization ID)

### Message & Body Inspection

- **FR17:** Operator can retrieve decoded contents of any message body in a session regardless of body class type
- **FR18:** The system handles all Ensemble body class variants: JSON-native bodies, HL7/FHIR virtual documents, stream bodies, and arbitrary custom persistent classes
- **FR19:** The system returns a structured, informative response when a body class is missing (purged, renamed, or deleted) without terminating the conversation
- **FR20:** Operator can retrieve a combined view of a message's header fields, body summary, and related log entries in a single request

### Business Process Inspection

- **FR21:** Operator can retrieve the ObjectScript source code or BPL XML of any Business Process class referenced in a session
- **FR22:** Operator can enumerate the methods defined on any production class
- **FR23:** Operator can retrieve the persistent runtime state of a Business Process instance active during a specific session, including context variables and thread state

### Error & Log Analysis

- **FR24:** Operator can decode any IRIS `%Status` error value into plain-English text with an identified error category
- **FR25:** The system recognizes common Ensemble error patterns (BP termination cascades, timeouts, unhandled exceptions, permission errors) and annotates them with known-cause context
- **FR26:** Operator can filter the event log by severity level to focus on errors, warnings, or specific trace categories

### Portal Integration (Phases 2 and 3)

- **FR27:** Operator can access the diagnostic chat interface as a tab within the Management Portal's Visual Trace page, alongside the existing Header, Body, and Contents tabs
- **FR28:** The chat interface in the Visual Trace is aware of which session is displayed and which message is currently selected, and uses this as context for the conversation
- **FR29:** Operator can initiate the full search → session selection → diagnostic chat workflow from a single saved bookmark URL
- **FR30:** The portal chat interface maintains multi-turn conversation state across HTTP requests within a browser session

### Deployment & Configuration

- **FR31:** System administrator can make the product available in any target interop namespace without compiling code in that namespace (via package mapping from HSCUSTOM)
- **FR32:** System administrator can configure the LLM provider API key via the IRIS Wallet without embedding secrets in source code or environment variables
- **FR33:** Operator accessing the product via Management Portal bookmark uses their existing portal authentication — no additional login required
- **FR34:** System administrator can verify the product makes no writes to Ensemble data — no message mutations, resends, or queue operations

### Audit & Governance

- **FR35:** The system records every diagnostic conversation turn — IRIS user, Ensemble session ID, timestamp, tools invoked, and token counts — in a queryable audit log
- **FR36:** All product operations respect existing IRIS RBAC — operators can only access session data they are already permitted to view in the Management Portal
- **FR37:** The product operates exclusively on non-PHI namespaces in v1; deployment to PHI-bearing namespaces requires explicit configuration of a compliant LLM provider

## Non-Functional Requirements

### Performance

- **NFR1:** Each chat turn completes within ≤30 seconds in the demo environment; no production SLA defined for v1
- **NFR2:** Individual SQL tool calls (single-table queries) complete within 10 seconds on a standard production namespace
- **NFR3:** Compound queries (session timeline UNION ALL across three tables) complete within 15 seconds on namespaces with up to 100k messages/day
- **NFR4:** Agent initialization (provider init + toolset load + session reconstitution) completes within 5 seconds
- **NFR5:** On high-volume namespaces, `CorrespondingMessageId` joins include time-window constraints to prevent timeout; the tool implementation enforces this
- **NFR6:** No concurrent user SLA is defined for v1; the expected usage pattern is single operator per namespace at a time

### Security

- **NFR7:** All session data sent to the LLM is scoped to what the authenticated IRIS user is already authorized to access via portal RBAC (`%Ens_MessageTrace:USE`)
- **NFR8:** The LLM API key is stored exclusively in the IRIS Wallet; it must not appear in source code, configuration files, environment variables, or log output
- **NFR9:** Read-only enforcement is structural: no `%Save`, `%DeleteId`, `ResubmitMessage`, or queue-mutation method is called in any tool implementation — verifiable by static code review before demo
- **NFR10:** Audit log entries are retained per the target namespace's standard data retention policy; no separate retention policy is required for v1
- **NFR11:** The product does not create new IRIS security resources or bypass existing portal resource checks; it inherits them

### Integration

- **NFR12:** The LLM provider is swappable by changing `Parameter PROVIDER` and `PROVIDERCONFIG` on the agent class — no tool implementation changes required
- **NFR13:** Tool implementations are fully decoupled from the LLM provider; they return structured `%DynamicObject` results; the AI Hub SDK layer handles all provider communication
- **NFR14:** The product requires IRIS 2026.2 with AI Hub EAP build 141 or later; the README must state the tested build; forward compatibility is not guaranteed until SDK GA

### Reliability

- **NFR15:** When an individual tool call fails, the system returns a structured error response enabling the agent to narrate a partial result rather than terminating the conversation
- **NFR16:** When the LLM provider is unreachable, the system surfaces a clear, actionable message to the operator rather than hanging or returning a generic exception
- **NFR17:** Read-only architecture ensures no possibility of data corruption or production state change from any system failure or unexpected code path
