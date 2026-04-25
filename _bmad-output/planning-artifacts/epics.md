---
stepsCompleted: [1, 2, 3, 4]
status: complete
completedAt: '2026-04-24'
inputDocuments:
  - "_bmad-output/planning-artifacts/prd.md"
  - "_bmad-output/planning-artifacts/architecture.md"
---

# Ensemble Session Inspection Agent - Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for the Ensemble Session Inspection Agent, decomposing the requirements from the PRD and Architecture into implementable stories organized around the incremental build strategy: Phase 1 Terminal Bot (Track A) → Phase 1 Gate → Parallel Track A (tool expansion) + Track B (portal integration).

## Requirements Inventory

### Functional Requirements

FR1: Operator can initiate a multi-turn diagnostic conversation about a specific session by providing a namespace and session ID
FR2: Operator can ask follow-up questions within a conversation that correctly reference and build on prior turns
FR3: Operator can ask "what happened in this session?" and receive a plain-English narrative covering the full session arc — messages, errors, log events, and routing decisions
FR4: Operator can ask "what does this error mean?" and receive a decoded, human-readable explanation with error category
FR5: Operator can ask "what does the body of this message contain?" and receive decoded message body contents
FR6: Operator can ask "why did the router take this path?" and receive a rule-based explanation
FR7: Operator can ask "what did the Business Process do?" and receive a source-code-informed explanation of BP behavior
FR8: Operator can reset the conversation to start fresh on the same or a different session
FR9: Agent conversation state persists correctly across turns — each follow-up question has full access to prior exchanges within the same conversation
FR10: Operator can retrieve a session summary — message count, error count, session duration, root message class — by session ID
FR11: Operator can retrieve a chronologically-ordered interleaved timeline of all messages, event log entries, and rule evaluations for a session
FR12: Operator can retrieve all message headers for a session with human-readable status, type, invocation, and component names
FR13: Operator can retrieve event log entries for a session, filtered by message ID and minimum severity level
FR14: Operator can retrieve rule evaluation results for a session, including rule name, component, reason for firing, and return value
FR15: Operator can retrieve sessions correlated to a given cross-instance session identifier (SuperSession)
FR16: Operator can find sessions by indexed body field values (e.g., patient MRN, organization ID)
FR17: Operator can retrieve decoded contents of any message body in a session regardless of body class type
FR18: The system handles all Ensemble body class variants: JSON-native bodies, HL7/FHIR virtual documents, stream bodies, and arbitrary custom persistent classes
FR19: The system returns a structured, informative response when a body class is missing (purged, renamed, or deleted) without terminating the conversation
FR20: Operator can retrieve a combined view of a message's header fields, body summary, and related log entries in a single request
FR21: Operator can retrieve the ObjectScript source code or BPL XML of any Business Process class referenced in a session
FR22: Operator can enumerate the methods defined on any production class
FR23: Operator can retrieve the persistent runtime state of a Business Process instance active during a specific session, including context variables and thread state
FR24: Operator can decode any IRIS %Status error value into plain-English text with an identified error category
FR25: The system recognizes common Ensemble error patterns (BP termination cascades, timeouts, unhandled exceptions, permission errors) and annotates them with known-cause context
FR26: Operator can filter the event log by severity level to focus on errors, warnings, or specific trace categories
FR27: Operator can access the diagnostic chat interface as a tab within the Management Portal's Visual Trace page, alongside the existing Header, Body, and Contents tabs
FR28: The chat interface in the Visual Trace is aware of which session is displayed and which message is currently selected, and uses this as context for the conversation
FR29: Operator can initiate the full search → session selection → diagnostic chat workflow from a single saved bookmark URL
FR30: The portal chat interface maintains multi-turn conversation state across HTTP requests within a browser session
FR31: System administrator can make the product available in any target interop namespace without compiling code in that namespace (via package mapping from HSCUSTOM)
FR32: System administrator can configure the LLM provider API key via the IRIS Wallet without embedding secrets in source code or environment variables
FR33: Operator accessing the product via Management Portal bookmark uses their existing portal authentication — no additional login required
FR34: System administrator can verify the product makes no writes to Ensemble data — no message mutations, resends, or queue operations
FR35: The system records every diagnostic conversation turn — IRIS user, Ensemble session ID, timestamp, tools invoked, and token counts — in a queryable audit log
FR36: All product operations respect existing IRIS RBAC — operators can only access session data they are already permitted to view in the Management Portal
FR37: The product operates exclusively on non-PHI namespaces in v1; deployment to PHI-bearing namespaces requires explicit configuration of a compliant LLM provider

### NonFunctional Requirements

NFR1: Each chat turn completes within ≤30 seconds in the demo environment; no production SLA defined for v1
NFR2: Individual SQL tool calls (single-table queries) complete within 10 seconds on a standard production namespace
NFR3: Compound queries (session timeline UNION ALL across three tables) complete within 15 seconds on namespaces with up to 100k messages/day
NFR4: Agent initialization (provider init + toolset load + session reconstitution) completes within 5 seconds
NFR5: On high-volume namespaces, CorrespondingMessageId joins include time-window constraints to prevent timeout; the tool implementation enforces this
NFR6: No concurrent user SLA is defined for v1; the expected usage pattern is single operator per namespace at a time
NFR7: All session data sent to the LLM is scoped to what the authenticated IRIS user is already authorized to access via portal RBAC (%Ens_MessageTrace:USE)
NFR8: The LLM API key is stored exclusively in the IRIS Wallet; it must not appear in source code, configuration files, environment variables, or log output
NFR9: Read-only enforcement is structural: no %Save, %DeleteId, ResubmitMessage, or queue-mutation method is called in any tool implementation — verifiable by static code review before demo
NFR10: Audit log entries are retained per the target namespace's standard data retention policy; no separate retention policy is required for v1
NFR11: The product does not create new IRIS security resources or bypass existing portal resource checks; it inherits them
NFR12: The LLM provider is swappable by changing Parameter PROVIDER and PROVIDERCONFIG on SAgent.Main.Agent — no tool implementation changes required
NFR13: Tool implementations are fully decoupled from the LLM provider; they return structured %DynamicObject results; the AI Hub SDK layer handles all provider communication
NFR14: The product requires IRIS 2026.2 with AI Hub EAP build 141 or later; the README must state the tested build; forward compatibility is not guaranteed until SDK GA
NFR15: When an individual tool call fails, the system returns a structured error response enabling the agent to narrate a partial result rather than terminating the conversation
NFR16: When the LLM provider is unreachable, the system surfaces a clear, actionable message to the operator rather than hanging or returning a generic exception
NFR17: Read-only architecture ensures no possibility of data corruption or production state change from any system failure or unexpected code path

### Additional Requirements

From Architecture decisions critical to story creation:

- **Project init**: Remove src/Sample/* and update module.xml before any story work begins
- **Package structure**: SAgent.Main (core agent), SAgent.Tools (13-tool catalog), SAgent.Portal (UI integration) — separate ZPM resources in module.xml
- **Default LLM provider**: OpenAI gpt-5.4; IRIS Wallet key AISecrets.OpenAIKey; provider swappable via 3 parameters in SAgent.Main.Agent with zero tool changes
- **Phase 1 Gate (7 criteria)**: SAgent.Main.Agent inits in <5s; Shell.Run() launches REPL; SAgent.Tools.Trace + Errors compile; agent answers "What happened in session X?"; agent answers "What does this error mean?"; multi-turn confirmed; read-only verified
- **Tool build priority**: P1 (Trace+Body+Errors), P2 (GetRuleLog+GetMessageHeaders), P3 (Process tools), P4 (Meta/discovery tools)
- **Deployment**: HSCUSTOM compilation + package mapping to %ALL; bookmark URL per namespace; no Portal menu integration
- **README required before any demo**: installation steps, prerequisites, wallet setup, first run
- **Moderate gap — SAgent.Tools.Tools XData Definition**: must define <ToolSet> XML composition root in first Tools story
- **Moderate gap — SAgent.Main.AuditLog properties**: must define 9 persistent properties in AuditLog story
- **Track B starts parallel after gate**: SAgent.Portal.MessageViewer (~15 min) then SAgent.Portal.VisualTrace; Track B never imports Track A by class name
- **IRIS SQL dialect rules**: ORDER BY %ID, %EXTERNAL(Type), %ODBCOUT(ErrorStatus), %ID in JOINs, time-bounded CorrespondingMessageId JOINs — enforced by patterns in all tool stories

### UX Design Requirements

No UX design document — portal UI inherits Management Portal styles via Zen subclassing pattern. No new visual design required. Chat UI rendered as server-side HTML skeleton within a Zen tab.

### FR Coverage Map

```
FR1:  Epic 1 — Initiate multi-turn diagnostic conversation
FR2:  Epic 1 — Follow-up questions reference prior turns
FR3:  Epic 1 — "What happened?" full narrative
FR4:  Epic 1 — "What does this error mean?" decoded answer
FR5:  Epic 1 — "What's in the body?" decoded contents
FR6:  Epic 2 — "Why did the router pick this path?" rule explanation
FR7:  Epic 3 — "What did the BP do?" source-code reasoning
FR8:  Epic 1 — Reset conversation (terminal /reset)
FR9:  Epic 1 — Agent state persists across turns
FR10: Epic 1 — GetSessionSummary (count, duration, root class)
FR11: Epic 1 — GetSessionTimeline (interleaved message/log/rule)
FR12: Epic 2 — GetMessageHeaders (all messages, decoded fields)
FR13: Epic 1 — GetEventLog (filtered by message + severity)
FR14: Epic 2 — GetRuleLog (rule name, reason, return value)
FR15: Epic 6 — FindRelatedSessions via SuperSession
FR16: Epic 6 — FindSessionsByBody (body field search)
FR17: Epic 1 — Decoded body contents regardless of class type
FR18: Epic 1 — Handle JSON/VDoc/Stream/custom body variants
FR19: Epic 1 — Graceful response when body is purged/missing
FR20: Epic 3 — GetMessageDetail (header + body + log combined)
FR21: Epic 3 — GetBusinessProcessSource (ObjectScript + BPL XML)
FR22: Epic 3 — ListBusinessProcessMethods (method enumeration)
FR23: Epic 3 — GetBusinessProcessInstance (runtime state)
FR24: Epic 1 — ExplainError (%Status decode + category)
FR25: Epic 1 — Pattern recognition (BP termination, timeouts, etc.)
FR26: Epic 1 — Event log severity filtering
FR27: Epic 4 — Chat tab in Visual Trace (4th tab)
FR28: Epic 4 — Chat is context-aware (selected session + message)
FR29: Epic 5 — Bookmark URL search-to-chat workflow
FR30: Epic 4 — Multi-turn state across HTTP requests
FR31: Epic 1 — HSCUSTOM + package mapping deployment
FR32: Epic 1 — LLM provider API key via IRIS Wallet
FR33: Epic 5 — Existing portal auth (no re-login)
FR34: Epic 1 — Read-only verifiable (no Ens mutations)
FR35: Epic 1 — Audit log per chat turn
FR36: Epic 1 — IRIS RBAC inheritance
FR37: Epic 1 — Non-PHI namespace scoping (v1)
```

## Epic List

### Epic 1: Diagnostic Foundation — Terminal Bot with Core Q&A
Operators can launch a terminal diagnostic session for any Ensemble session, ask "what happened?", get decoded errors, inspect message bodies, and review event logs — all read-only, all from a single command. This epic establishes the Phase 1 Gate.

**User value:** On-call engineer Alex resolves an 11pm incident by typing one command and asking natural-language questions instead of juggling four tabs.

**FRs covered:** FR1, FR2, FR3, FR4, FR5, FR8, FR9, FR10, FR11, FR13, FR17, FR18, FR19, FR24, FR25, FR26, FR31, FR32, FR34, FR35, FR36, FR37

---

### Epic 2: Routing & Message Intelligence
Operators can ask why a router made a decision and get the full message header landscape for a session — revealing rule logic and detailed per-message metadata.

**User value:** Alex extends her terminal diagnostic to routing questions: "Why did the router send this to EpicOperation?"

**FRs covered:** FR6, FR12, FR14

---

### Epic 3: Business Process Code Reasoning
Operators can ask what a Business Process actually did — reading source code, runtime state, and method catalog — to explain behavior that isn't visible in message headers alone.

**User value:** Engineering lead Morgan can ask "Was this a code issue or a configuration issue?" and get source-code-informed answers.

**FRs covered:** FR7, FR20, FR21, FR22, FR23

---

### Epic 4: Portal Chat Tab — Visual Trace Integration
Operators can access the diagnostic chat as a fourth tab in the Management Portal's Visual Trace page. The chat is context-aware: it knows which session is displayed and which message is selected. Multi-turn state persists across HTTP requests.

**User value:** Junior developer Jordan sees a Chat tab in the portal, asks "what happened?", and gets an answer without knowing how to read a Visual Trace.

**FRs covered:** FR27, FR28, FR30

---

### Epic 5: Complete Search-to-Chat Workflow
Operators can search for sessions from a single bookmarked URL in the Management Portal, select a session, and open it directly in the Chat-enabled Visual Trace — the complete end-to-end integrated workflow, no second login required.

**User value:** Any operator bookmarks one URL per namespace and goes from "find session" to "chat with session" in two clicks.

**FRs covered:** FR29, FR33

---

### Epic 6: Deep Discovery & Cross-Instance Correlation
Operators can find sessions by clinical identifiers (patient MRN, organization ID) from message body fields, and can correlate sessions across distributed IRIS instances via SuperSession.

**User value:** Senior engineers can search "show me all sessions involving patient MRN X" or trace a transaction across a gateway→hub→pipeline production topology.

**FRs covered:** FR15, FR16

---

## Epic 1: Diagnostic Foundation — Terminal Bot with Core Q&A

Operators can launch a terminal diagnostic session for any Ensemble session, ask "what happened?", get decoded errors, inspect message bodies, and review event logs — all read-only, all from a single command. This epic establishes the Phase 1 Gate.

### Story 1.1: Project Foundation Setup

As a developer,
I want to initialize the SAgent package structure in the repository,
So that all subsequent stories compile cleanly without conflicts from the placeholder Sample.* classes.

**Acceptance Criteria:**

**Given** the repository contains src/Sample/* and module.xml references Sample.PKG
**When** the developer deletes src/Sample/, creates src/SAgent/Main/, src/SAgent/Tools/, src/SAgent/Portal/, and src/SAgent/Test/ directories, and updates module.xml to reference SAgent.Main.PKG, SAgent.Tools.PKG, SAgent.Portal.PKG
**Then** the project compiles with zero errors in HSCUSTOM
**And** no Sample.* classes remain in the namespace

**Given** the module.xml has been updated
**When** a developer inspects module.xml
**Then** Resource entries list SAgent.Main.PKG, SAgent.Tools.PKG, and SAgent.Portal.PKG in that order
**And** the SAgent.Portal resource is commented with "Track B — compile after Phase 1 gate"

---

### Story 1.2: Audit Log and Read-Only Policy

As a system administrator,
I want an audit trail for every chat turn and structural read-only enforcement,
So that I can verify no Ens data is mutated and every diagnostic interaction is traceable.

**Acceptance Criteria:**

**Given** SAgent.Main.AuditLog is compiled in HSCUSTOM
**When** SAgent.Main.AuditLog.Write(chatSessionId, ensSessionId, userInput, agentResponse, stats) is called
**Then** a row is inserted into ^SAgent.Main.AuditLogD with Timestamp, IRISUser ($USERNAME), EnsSessionId, ChatSessionId, UserInput, AgentResponse, ToolCalls (count), TokensPrompt, TokensCompletion, DurationMs
**And** the Write() ClassMethod handles its own %Save() — callers must not %Save() separately

**Given** SAgent.Main.ReadOnlyPolicy is compiled and attached to the ToolManager
**When** any tool with mutates=1 metadata is invoked
**Then** SAgent.Main.ReadOnlyPolicy.%CanExecute() returns an error and the tool does not execute
**And** a test in SAgent.Test.AgentTest verifies no %Save, %DeleteId, or ResubmitMessage call exists in any SAgent.Tools.* class

---

### Story 1.3: Core Session Timeline Tools

As an integration engineer,
I want to ask "what happened in this session?" and receive a complete, chronological narrative,
So that I can understand a production incident without opening multiple Management Portal tabs.

**Acceptance Criteria:**

**Given** SAgent.Tools.Trace is compiled with GetSessionSummary, GetSessionTimeline, and GetEventLog queries
**When** GetSessionSummary is called with a valid SessionId
**Then** it returns a %DynamicObject with messageCount, errorCount, startTime, endTime, rootBodyClass, and componentList
**And** the query completes within 10 seconds

**Given** GetSessionTimeline is called with a valid SessionId
**When** the query executes
**Then** rows include EventKind, Ts, MessageId, Subtype (decoded via %EXTERNAL), Status (decoded), Component, Flow, Detail, PairId, ErrorText (decoded via %ODBCOUT)
**And** rows are ordered by Ts ASC, EventKind ASC
**And** HS.Util.Trace.Request bodies are filtered out
**And** the query completes within 15 seconds

**Given** GetEventLog is called with optional messageId and minSeverity filters
**When** filters are provided
**Then** only matching rows are returned
**And** null/empty filters return all log entries for the session

---

### Story 1.4: Message Body Inspection Tools

As an integration engineer,
I want to ask "what's in the body of this message?" and receive decoded contents,
So that I can understand message payloads without writing ObjectScript queries.

**Acceptance Criteria:**

**Given** GetMessageBody is called with a valid MessageId for a %JSON.Adaptor body
**Then** response contains variant="json" and body as a decoded %DynamicObject

**Given** GetMessageBody is called for an EnsLib.HL7.Message body
**Then** response contains variant="vdoc_hl7", messageType, sendingFacility, timestamp, and a segments array with raw segment text truncated to maxStreamBytes

**Given** GetMessageBody is called for a message whose body class no longer exists
**Then** response contains variant="missing_class", error, and hint
**And** the conversation does not terminate

**Given** GetMessageBody is called for a null body (MessageBodyClassName="" and MessageBodyId="")
**Then** response contains variant="null" and conversation continues normally

---

### Story 1.5: Error Decoding Tool

As an integration engineer,
I want to ask "what does this error mean?" and receive a plain-English explanation,
So that I can understand IRIS %Status error codes without documentation lookup.

**Acceptance Criteria:**

**Given** ExplainError is called with a messageId pointing to a response header with IsError=1
**When** the tool reads ErrorStatus via %ODBCOUT
**Then** the response contains source (decoded string), errorCode, category, and message (human-readable)
**And** the tool reads from the response header, not the request header

**Given** the error contains "\<Ens\>ErrBPTerm"
**Then** category="BusinessProcessTermination" and knownCause is populated

**Given** the error contains "\<PROTECT\>"
**Then** category="ProtectError" and knownCause describes a protected global/permission issue

**Given** ExplainError is called with a raw statusText string containing "ERROR #NNNN:"
**Then** errorCode is extracted and message is the human-readable portion

---

### Story 1.6: Agent Assembly and Terminal Shell

As an integration engineer,
I want to type one command with a namespace and session ID and begin a multi-turn diagnostic conversation,
So that I can diagnose production incidents from a terminal without any additional setup.

**Acceptance Criteria:**

**Given** SAgent.Tools.Tools XData Definition includes SAgent.Tools.Trace, SAgent.Tools.Body, and SAgent.Tools.Errors via \<Include\> elements with ReadOnlyPolicy and ConsoleAudit policies attached
**When** SAgent.Main.Agent.%Init() is called
**Then** all tools are loaded and agent initializes in under 5 seconds

**Given** SAgent.Main.Agent is configured with PROVIDER="openai", MODEL="gpt-5.4", PROVIDERCONFIG referencing @{wallet.AISecrets.OpenAIKey}
**When** the wallet entry AISecrets.OpenAIKey is populated
**Then** agent.%Init() succeeds and agent.Provider is not null

**Given** Do ##class(SAgent.Main.Shell).Run("CENGATEWAY", 42751) is executed
**When** the agent initializes
**Then** a welcome banner shows namespace and session ID
**And** multi-turn conversation works — follow-up questions reference prior context
**And** SAgent.Main.AuditLog.Write() is called automatically after each turn
**And** %AI.Agent.Session is saved after each turn

**Given** a tool call fails (class missing, SQL error, etc.)
**When** the agent receives the error %DynamicObject
**Then** it narrates the partial result and continues the conversation

---

### Story 1.7: Installation Guide (README)

As a system administrator,
I want a step-by-step installation guide,
So that I can deploy SAgent to HSCUSTOM and verify it works before any demo.

**Acceptance Criteria:**

**Given** a fresh IRIS 2026.2 instance with AI Hub EAP build 141
**When** an administrator follows the README exactly
**Then** all SAgent.Main.* and SAgent.Tools.* classes compile in HSCUSTOM
**And** package mapping from HSCUSTOM to the target namespace is configured
**And** the IRIS Wallet entry AISecrets.OpenAIKey is created
**And** Do ##class(SAgent.Main.Shell).Run("TARGETNS", testSessionId) produces a response

**Given** a developer reads the README
**Then** it covers: prerequisites (IRIS version, AI Hub EAP build), compilation steps, Wallet key setup, package mapping configuration, first-run validation command, known limitations (non-PHI v1, EAP API churn caveat)

---

### Story 1.8: Phase 1 Gate Validation

As the development team,
I want to verify all Phase 1 Gate criteria pass before beginning Track B portal work,
So that the portal integration starts from a stable, verified foundation.

**Acceptance Criteria:**

**Given** Stories 1.1–1.7 are complete
**When** the team runs SAgent.Test.GateTest
**Then** all 7 criteria pass:
1. SAgent.Main.Agent compiles and initializes in under 5 seconds
2. SAgent.Main.Shell.Run() launches an interactive REPL session
3. SAgent.Tools.Trace and SAgent.Tools.Errors compile without errors
4. Agent correctly answers "What happened in session X?" (non-empty narrative returned)
5. Agent correctly answers "What does this error mean?" (decoded error returned)
6. Multi-turn confirmed: follow-up question correctly references prior context
7. Read-only verified: no Ens.* rows added or modified after a full chat turn

**Given** all 7 gate criteria pass
**Then** Track B work (Epic 4 — Portal Chat Tab) may begin in parallel with Epic 2 and Epic 3

---

## Epic 2: Routing & Message Intelligence

Operators can ask why a router made a decision and get the full message header landscape for a session — revealing rule logic and detailed per-message metadata.

### Story 2.1: Routing Explanation Tool

As an integration engineer,
I want to ask "why did the router take this path?" and receive the rule logic that drove the decision,
So that I can distinguish routing misconfigurations from message content issues.

**Acceptance Criteria:**

**Given** SAgent.Tools.Trace.GetRuleLog is added and registered in SAgent.Tools.Tools XData
**When** GetRuleLog is called with a valid SessionId
**Then** rows include: RuleLogId, TimeExecuted, RuleName, RuleSet, ActivityName, Component, CurrentHeaderId, Reason, ReturnValue, IsError, ErrorMsg
**And** rows are ordered by TimeExecuted ASC
**And** the query completes within 10 seconds

**Given** a session where a routing rule fired and selected a target
**When** an operator asks "why did the router choose EpicOperation?"
**Then** the agent retrieves the rule that fired, its reason, and the ReturnValue
**And** narrates this in plain English without exposing raw integers

---

### Story 2.2: Full Message Header List Tool

As an integration engineer,
I want to retrieve all message headers in a session with decoded field values,
So that I can see the complete message choreography with human-readable status and invocation types.

**Acceptance Criteria:**

**Given** SAgent.Tools.Trace.GetMessageHeaders is added and registered in SAgent.Tools.Tools XData
**When** GetMessageHeaders is called with a valid SessionId
**Then** rows include: MessageId, TimeCreated, TimeProcessed, TypeLabel (%EXTERNAL(Type)), StatusLabel (%EXTERNAL(Status)), InvocationLabel (%EXTERNAL(Invocation)), SourceComponent, TargetComponent, BodyClassName, BodyId, PairId, IsError, ErrorText (%ODBCOUT(ErrorStatus)), BusinessProcessId
**And** rows are ordered by %ID ASC
**And** HS.Util.Trace.Request bodies are filtered out

**Given** a session with Request (Type=1) and Response (Type=2) messages
**When** GetMessageHeaders returns results
**Then** TypeLabel shows "Request"/"Response" (not raw integers)
**And** StatusLabel shows "Completed"/"Error" (not raw integers)

---

## Epic 3: Business Process Code Reasoning

Operators can ask what a Business Process actually did — reading source code, runtime state, and method catalog — to explain behavior that isn't visible in message headers alone.

### Story 3.1: Business Process Source Code Tool

As an engineering lead,
I want to ask "what did the Business Process code do?" and receive the actual ObjectScript source or BPL XML,
So that I can determine whether a failure was a code issue or a configuration issue without opening Studio.

**Acceptance Criteria:**

**Given** SAgent.Tools.Process.GetBusinessProcessSource is compiled and registered in SAgent.Tools.Tools XData
**When** called with classname and no methodName
**Then** returns classDescription, super, isAbstract, deployed flag, list of method names with signatures, and list of XData block names

**Given** called with classname and methodName
**Then** returns description, formalSpec, returnType, isClassMethod, and the full Implementation stream as a string

**Given** the class is a BPL process (XData block "BPL" exists)
**Then** the response includes bpl field containing the full BPL XML string

**Given** the class has Deployed=1 (Implementation is empty)
**Then** the response includes note="Method Implementation is empty (class is deployed/disassociated from source)"
**And** the conversation continues with available metadata

**Given** the classname does not exist
**Then** response contains error="Class not found or not compiled: {classname}"
**And** the conversation does not terminate

---

### Story 3.2: Business Process Method Enumeration Tool

As an integration engineer,
I want to list all methods defined on a Business Process class,
So that I can understand the full interface of a BP without reading the full source.

**Acceptance Criteria:**

**Given** SAgent.Tools.Process.ListBusinessProcessMethods is compiled as a Query tool
**When** called with a classname
**Then** rows include: Name, ClassMethod (boolean), FormalSpec, ReturnType, Description (first 500 chars)
**And** rows are ordered by Name ASC
**And** system methods (prefixed with %) are excluded from results

---

### Story 3.3: Business Process Runtime State Tool

As an engineering lead,
I want to inspect the runtime state of a Business Process instance that handled a specific session,
So that I can understand what context variables and thread state existed when a failure occurred.

**Acceptance Criteria:**

**Given** SAgent.Tools.Process.GetBusinessProcessInstance is compiled and registered
**When** called with sessionId and classname (bpInstanceId optional)
**Then** if bpInstanceId is not provided, the tool queries Ens.MessageHeader to find BusinessProcessId for that session and class
**And** opens the BP instance and returns properties serialized via %ZEN.Auxiliary.altJSONProvider.%ObjectToAET(bp)

**Given** the BP is a BPL process (Ens.BP.Context row exists)
**Then** bplContext is included with user-declared context properties
**And** threads array lists active thread states with _NextState and _Status

**Given** the BP instance cannot be found
**Then** response contains error="No BP instance found for session + class combination"
**And** the conversation does not terminate

---

### Story 3.4: Combined Message Detail Tool

As an integration engineer,
I want to retrieve a single message's header fields, body summary, and related log entries in one call,
So that I can get a complete picture of a specific message without making three separate tool calls.

**Acceptance Criteria:**

**Given** SAgent.Tools.Body.GetMessageDetail is compiled and registered
**When** called with a valid MessageId
**Then** the response combines: header fields (decoded TypeLabel, StatusLabel, component names), body summary (from GetMessageBody dispatch chain), and related log entries from Ens_Util.Log where MessageId matches
**And** the combined response is a single %DynamicObject
**And** if the body is purged, body section contains variant="purged" without failing the entire call

---

## Epic 4: Portal Chat Tab — Visual Trace Integration

Operators can access the diagnostic chat as a fourth tab in the Management Portal's Visual Trace page. The chat is context-aware and multi-turn state persists across HTTP requests.

### Story 4.1: Custom VisualTrace Subclass with Chat Tab

> **⚠️ TRACK B START — DO NOT BEGIN UNTIL PHASE 1 GATE PASSES**
> Before starting this story, verify all 7 Phase 1 Gate criteria from Story 1.8 are confirmed:
> 1. SAgent.Main.Agent compiles and initializes in <5 seconds
> 2. SAgent.Main.Shell.Run() launches interactive REPL
> 3. SAgent.Tools.Trace and SAgent.Tools.Errors compile
> 4. Agent answers "What happened in session X?" correctly
> 5. Agent answers "What does this error mean?" correctly
> 6. Multi-turn confirmed (follow-up references prior context)
> 7. Read-only verified (no Ens.* rows modified after a chat turn)
>
> **Git workflow:** Branch `track-b` from `dev` AFTER Track A merges the gate. Then compile Track A classes from `dev` into HSCUSTOM before testing portal code.

As a portal operator,
I want to see a "Chat" tab in the Visual Trace page alongside Header, Body, and Contents,
So that I can start a diagnostic conversation without leaving the Management Portal.

**Acceptance Criteria:**

**Given** SAgent.Portal.VisualTrace extends EnsPortal.VisualTrace and is compiled in HSCUSTOM
**When** a user navigates to `https://<host>/csp/healthshare/<NS>/SAgent.Portal.VisualTrace.zen?SESSIONID=<id>`
**Then** four tabs are displayed: Header, Body, Contents, and Chat
**And** the existing three tabs function identically to stock EnsPortal.VisualTrace — no regressions

**Given** the Chat tab is clicked
**When** the tab renders
**Then** a chat interface is displayed with: a scrolling message log div, a textarea input, a Send button, and a Reset conversation button
**And** an initial greeting message references the current session ID

**Given** XData allTabs overrides the parent class's tab group
**When** a developer inspects SAgent.Portal.VisualTrace
**Then** the class contains XData allTabs with all four tab elements
**And** the chatTab element uses OnDrawContent="DrawChatUI" pointing to Method DrawChatUI(pSeed As %String) As %Status

---

### Story 4.2: Chat ZenMethod and HTTP-Stateless Session Persistence

As a portal operator,
I want to type a question in the Chat tab and receive an answer from the agent,
So that I can diagnose a session interactively within the Management Portal without a separate terminal.

**Acceptance Criteria:**

**Given** Method SendChatMessage(userInput As %String, selectedMessageId As %String = "") As %String [ZenMethod] is implemented
**When** an operator types a question and clicks Send
**Then** the agent processes the input using the P1 tool catalog and a response string is returned
**And** the response is rendered as a new message bubble in the chat log
**And** the response arrives within 30 seconds

**Given** an operator sends multiple messages in sequence
**When** each ZenMethod call runs
**Then** each turn loads %AI.Agent.Session via Session.Load(id, provider), calls agent.Chat(), calls Session.%Save(), stores %Id() in %session.Data("SAgent","SessionId")
**And** follow-up context is preserved correctly across HTTP requests

**Given** the operator clicks Reset
**When** ClassMethod ResetChat() As %Boolean [ZenMethod] executes
**Then** the existing session is deleted and %session.Data("SAgent","SessionId") is cleared

**Given** an operator clicks a message in the Visual Trace SVG
**When** the next SendChatMessage call runs
**Then** selectedMessageId is passed and "[Context: inspecting Ens session X; user is focused on message Y]" is prepended to the LLM prompt

**Given** the LLM provider is unreachable
**When** SendChatMessage executes
**Then** the method returns a user-visible error string (not a raw exception)
**And** SAgent.Main.AuditLog.Write() is still called

---

## Epic 5: Complete Search-to-Chat Workflow

Operators can search for sessions from a single bookmarked URL and open them directly in the Chat-enabled Visual Trace — no second login required.

### Story 5.1: Custom MessageViewer with Chat-Enabled Handoff

As a portal operator,
I want to search for sessions from a bookmarked URL and open any session directly in the Chat-enabled Visual Trace,
So that I can go from "find the failing session" to "chat with it" in two clicks without re-authenticating.

**Acceptance Criteria:**

**Given** SAgent.Portal.MessageViewer extends EnsPortal.MessageViewer and is compiled in HSCUSTOM
**When** a user navigates to `https://<host>/csp/healthshare/<NS>/SAgent.Portal.MessageViewer.zen`
**Then** the page functions identically to stock EnsPortal.MessageViewer — all search, filter, pagination, and result display work unchanged

**Given** an operator clicks a Session ID link in the results table
**When** ClientMethod showTrace(sessionId, evt) executes
**Then** it opens SAgent.Portal.VisualTrace.zen?SESSIONID=\<sessionId\> in a new window
**And** the opened page displays the Visual Trace with the Chat tab present

**Given** an operator already logged into the Management Portal navigates to the bookmarked URL
**Then** no second login prompt appears
**And** the portal CSP session cookie is reused automatically

**Given** a developer inspects SAgent.Portal.MessageViewer
**Then** the only override from EnsPortal.MessageViewer is ClientMethod showTrace() — all other methods are inherited unchanged

---

## Epic 6: Deep Discovery & Cross-Instance Correlation

Operators can find sessions by clinical identifiers from message body fields, and can correlate sessions across distributed IRIS instances via SuperSession.

### Story 6.1: Cross-Instance Session Correlation Tool

As a senior integration engineer,
I want to find all sessions related to a SuperSession identifier,
So that I can trace a business transaction across a distributed gateway→hub→pipeline IRIS topology.

**Acceptance Criteria:**

**Given** SAgent.Tools.Meta is compiled with FindRelatedSessions registered in SAgent.Tools.Tools XData
**When** FindRelatedSessions is called with a superSessionId string
**Then** rows are returned from Ens.SuperSessionIndex joined to Ens.MessageHeader including: SessionId, StartTime, RootBodyClass, MessageCount, ErrorCount
**And** rows are grouped by SessionId and ordered by MIN(TimeCreated) ASC
**And** if no related sessions exist the result envelope returns row_count=0 with truncated=false

---

### Story 6.2: Session Search by Body Field Values

As a senior integration engineer,
I want to find sessions involving a specific patient, organization, or other indexed body field value,
So that I can locate all sessions related to a clinical entity without knowing session IDs in advance.

**Acceptance Criteria:**

**Given** SAgent.Tools.Meta.FindSessionsByBody is compiled as a Query tool
**When** called with bodyClass, filterField (element_key), filterValue, and optional sinceHours (default 24)
**Then** the query joins Ens.MessageHeader to {BodyClass}_AdditionalInfo on element_key=filterField and AdditionalInfo=filterValue
**And** returns distinct SessionIds with StartTime, MessageCount, and ErrorCount
**And** the sinceHours parameter limits the time window to prevent full-table scans

**Given** the bodyClass or filterField does not exist in the namespace
**Then** the tool returns error="Body class or field not found" and hint="Verify the class is compiled and the field is indexed"
**And** the conversation does not terminate
