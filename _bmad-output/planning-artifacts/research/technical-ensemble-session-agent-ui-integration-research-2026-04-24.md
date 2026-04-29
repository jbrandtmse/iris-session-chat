---
stepsCompleted: [1, 2, 3, 4, 5, 6]
inputDocuments:
  - irislib/EnsPortal/VisualTrace.cls
  - irislib/EnsPortal/MessageViewer.cls
  - irislib/EnsPortal/Dialog/standardDialog.cls
  - irislib/EnsPortal/Template/filteredViewer.cls
  - irislib/EnsPortal/Template/standardPage.cls
  - irislib/EnsPortal/Util/PageLinks.cls
  - irislib/%AI/Shell/Console.cls
  - irislib/%AI/Shell/ConsoleAgent.cls
  - irislib/%AI/Shell/StreamRenderer.cls
  - irislib/%AI/Agent.cls, Agent/Session.cls
  - irislib/%ZEN/Component/page.cls
  - irislib/%CSP/Session.cls
  - Prior research - technical-ensemble-session-inspection-agent-research-2026-04-24.md
  - Prior art - iris-view-agent + sources/diagramtool
  - Web sources via Perplexity
workflowType: 'research'
lastStep: 6
workflowComplete: true
research_type: 'technical'
research_topic: 'UI Integration - Phased Deployment of Ensemble Session Inspection Agent'
research_goals: 'Deliver hackathon-ready, decision-complete technical research so that on build day, the team codes from the document without research interruptions. Three phases: (1) terminal REPL bot via %AI.Shell.Console, (2) chat tab injected into custom EnsPortal.VisualTrace via subclassing, (3) custom EnsPortal.MessageViewer that hands off to custom VisualTrace via bookmark URL. All compiled in HSCUSTOM with namespace switching at runtime.'
user_name: 'Developer'
date: '2026-04-24'
web_research_enabled: true
source_verification: true
depends_on: technical-ensemble-session-inspection-agent-research-2026-04-24.md
---

# Research Report: UI Integration Architecture

**Date:** 2026-04-24
**Author:** Developer
**Research Type:** Technical — UI Integration
**Depends on:** [Prior research on the agent/tool side](technical-ensemble-session-inspection-agent-research-2026-04-24.md)

---

## Research Overview

This research extends the earlier agent/tool-side research with a focused study on **how to deploy the Ensemble Session Inspection Agent into the Management Portal experience** in three phases:

1. **Phase 1 — Terminal REPL bot**: operator launches from an IRIS terminal, enters namespace + session ID, chats with the agent.
2. **Phase 2 — Chat tab in custom VisualTrace**: a subclass of `EnsPortal.VisualTrace` adds a 4th tab alongside Header / Body / Contents.
3. **Phase 3 — Custom MessageViewer with handoff**: a subclass of `EnsPortal.MessageViewer` provides the search experience; clicking a session routes to the custom VisualTrace; operators reach the custom page via a saved bookmark.

All custom code is compiled in **HSCUSTOM** namespace and accessed under the existing portal CSP application (via bookmark). The agent switches `$NAMESPACE` at runtime to inspect each target interop namespace.

**Why this research is its own document:** the prior research (linked above) answered the data/model side — what tables, what correlation logic, what tool schemas. This research answers the UI/integration side — how to expose those tools in three progressively-more-integrated surfaces without fighting the InterSystems Management Portal. The two documents are complementary and should be read together.

**Design constraints from the user**:
- 4-hour build time (but with unlimited research time ahead of the hackathon)
- Compiled in HSCUSTOM with `$NAMESPACE` switching
- Bookmark-based deployment (no Portal menu integration)
- Blocking request/response is fine (no streaming required)
- Ideal: ride the Portal's CSP session (no second login)

**Key questions this document must answer definitively**:
- Is subclassing `EnsPortal.VisualTrace` / `EnsPortal.MessageViewer` viable, or must we build parallel pages?
- How does an `%AI.Agent.Session` persist across ZenMethod AJAX calls? (Each call is a new HTTP request — the agent object is destroyed.)
- What's the exact deployment topology (CSP app, web application, URL routing)?
- What breaks when we switch `$NAMESPACE` inside a CSP request?

---

## Technical Research Scope Confirmation

**Research Topic:** UI Integration — Phased Deployment of Ensemble Session Inspection Agent

**Research Goals:** Deliver hackathon-ready, decision-complete technical research for the three deployment phases.

**Research Methodology:**
- Full source reading of `EnsPortal.VisualTrace`, `MessageViewer`, their parent classes, `%AI.Shell.*`, relevant Zen/CSP base classes.
- Multiple Perplexity passes on Zen subclassing, CSP app deployment, agent state persistence.
- Prior-art cross-reference (HealthShare portal extensions in `irislib/`).
- Every non-trivial claim cross-validated.
- Explicit confidence levels where certainty not reached.

**Scope Confirmed:** 2026-04-24

---

## Section 1 — Foundations: Class Hierarchy & Key Discoveries

### 1.1 Class Hierarchy Maps (Verified from Source)

**Phase 1 — Terminal bot path:**

```
%RegisteredObject
  └── %AI.Agent                          (irislib/%AI/Agent.cls)
        └── %AI.Shell.ConsoleAgent       (irislib/%AI/Shell/ConsoleAgent.cls)
              └── Custom.EnsSession.ConsoleAgent  ← we optionally subclass

%AI.Shell.Console                        (abstract, driver class — DO NOT subclass)
  - ClassMethod Run(provider, config, model, toolsets...)
```

**Phase 2 — VisualTrace subclass path:**

```
%ZEN.Component.page                      (root Zen page class)
  └── %CSP.Portal.standardPage           (portal framework)
        └── %CSP.Portal.standardDialog   (dialog variant)
              └── EnsPortal.Template.base           (shared EnsPortal utils)
                    └── EnsPortal.Dialog.standardDialog  (irislib/EnsPortal/Dialog/standardDialog.cls)
                          └── EnsPortal.VisualTrace     (irislib/EnsPortal/VisualTrace.cls)
                                ├── Ens.Enterprise.Portal.VisualTrace ← PRIOR-ART SUBCLASS
                                └── Custom.EnsPortal.VisualTrace      ← we build this
```

**Phase 3 — MessageViewer subclass path:**

```
EnsPortal.Template.viewerPage
  └── EnsPortal.Template.filteredViewer  (irislib/EnsPortal/Template/filteredViewer.cls)
        └── EnsPortal.MessageViewer      (irislib/EnsPortal/MessageViewer.cls)
              └── Custom.EnsPortal.MessageViewer  ← we build this
```

### 1.2 The Decisive Finding — InterSystems Already Subclasses These Pages

**`Ens.Enterprise.Portal.VisualTrace`** (`irislib/Ens/Enterprise/Portal/VisualTrace.cls`, 247 lines) is **an InterSystems-provided subclass of `EnsPortal.VisualTrace`** that does exactly what we want to do for Phase 2: adds a new tab.

```objectscript
Class Ens.Enterprise.Portal.VisualTrace Extends EnsPortal.VisualTrace [ System = 4 ]
{
    Parameter RESOURCE = "%Ens_MsgBank_MessageTrace:USE";
    Parameter PAGENAME = "Enterprise Visual Trace";

    // Override tabs — adds a 4th tab!
    XData allTabs [ XMLNamespace = "http://www.intersystems.com/zen" ]
    {
        <pane>
        <tabGroup id="contentTabs" showTabBar="true" remember="true" onshowTab="zenPage.updateTabs(true);">
        <tab id="headerDetails" caption="Details" ...>
        <tab id="bodyDetails" caption="Body" ...>
        <tab id="bodyContents" caption="Contents" ...>
        <tab id="resendTab" caption="Resends" title="Resend Data">
            <html id="resendData" OnDrawContent="DrawResendData" />
        </tab>
        </tabGroup>
        </pane>
    }

    Method DrawResendData(pSeed As %String) As %Status { ... }
}
```

**This is authoritative proof that**:
- Subclassing `EnsPortal.VisualTrace` works.
- Overriding `XData allTabs` to add a tab works.
- `OnDrawContent="..."` on a new `<html>` component calls a server-side method to render tab content.
- InterSystems themselves use this exact pattern in production code.

**Implication**: our subclassing risk is *dramatically lower* than initially assumed. Phase 2 is a well-trodden path.

### 1.3 Phase 1 Is Already Essentially Free

From `%AI.Shell.Console.Run()` source (lines 59-70 of `Console.cls`):

```objectscript
// Register any additional caller-supplied toolsets
For i=1:1:$GET(toolsets) {
    Set ts = toolsets(i)
    If $ISOBJECT(ts) {
        $$$ThrowOnError(agent.ToolManager.AddTool(ts))
    } ElseIf ts '= "" {
        $$$ThrowOnError(agent.UseToolSet(ts))
    }
}
```

`Run()` accepts a variadic list of toolset class names. So the absolute minimum path for Phase 1 is:

```objectscript
Do ##class(%AI.Shell.Console).Run("anthropic", apiKey, "claude-sonnet-4-5", "Custom.EnsSession.Tools")
```

That's one line. The default `ConsoleAgent` loads but its built-in `ShellTools` will coexist with ours — users can see both.

**Two polish options for Phase 1**:
1. **Subclass `ConsoleAgent`** to override `SetupDefaultTools()` and register only our tools (no filesystem/shell noise), and override `XData INSTRUCTIONS` with our Ensemble-inspector system prompt. But Console.Run hardcodes `%AI.Shell.ConsoleAgent` — so we can't use our subclass via `Run()`.
2. **Write a thin wrapper** — our own entry class that reimplements `Run()`'s logic using our custom `ConsoleAgent` subclass. More work, more control.

**Recommendation for Phase 1**: option 2 (thin wrapper). About 100 lines, gives us full control over the UX. Details in Section 7.

### 1.4 `%AI.Policy.ConsoleAuth` Behavior — Good News

From `irislib/%AI/Policy/ConsoleAuth.cls` lines 54-61:

```objectscript
Set authNeeded = toolObj.%Get("requires_auth", 0)
If authNeeded {
    If '$SYSTEM.Security.Check("%System_Callout","USE") {
        Return $$$ERROR(...)
    }
}
```

**`ConsoleAuth` ONLY prompts for tools that have `requires_auth=true` in metadata**, which maps to `Parameter REQUIRESAUTH = 1` on the tool class. Our `Custom.EnsSession.Tools.*` classes (per prior research) all have `REQUIRESAUTH = 0` — so **ConsoleAuth will NOT prompt for our tools**, even under the default `ConsoleAgent`. Zero UX noise.

---

## Section 2 — Agent State Persistence Across HTTP Requests

This was flagged as the hard problem. It turned out to be a well-solved problem built into the AI Hub SDK.

### 2.1 Source Reading: `%AI.Agent.Session` IS `%Persistent`

From `irislib/%AI/Agent/Session.cls`:

```objectscript
Class %AI.Agent.Session Extends %Persistent [ ClassType = persistent, System = 4 ]
{
    Property CreatedAt As %TimeStamp [ ... ];
    Property %agent As %Integer [ Internal ];           // Rust agent token
    Property Model As %String;
    Property SystemPrompt As %String(MAXLEN = "");
    Property ContextData As %Stream.GlobalCharacter [ Internal ];  // serialized session JSON

    // Auto-serializes on %Save()
    Method %OnBeforeSave(insert As %Boolean) As %Status [ Internal ]
    { ... // calls LLMSESSIONEXPORT into ContextData stream }

    // Restores live Rust session from ContextData
    Method Reconstitute(provider As %AI.Provider) As %Status { ... }

    // One-liner: open by ID + reconstitute
    ClassMethod Load(id As %String, provider As %AI.Provider) As %AI.Agent.Session { ... }

    // Alternative: JSON export/import
    Method Export() As %String { ... }
    Method Import(json As %String) As %Status { ... }
}
```

From the docstring (lines 10-12): *"Sessions can be saved to and loaded from the database. Use Export() / Import() to move sessions between processes or store them as raw JSON in your own persistence layer."*

**Storage**: `^AI.Agent.SessionD` global (standard `%Persistent` layout).

### 2.2 The Tool-Schema Question — Resolved

Initial concern: when `Reconstitute()` recreates the Rust agent, it passes `[]` for tools (line 84). Does that break tool calling?

**Answer — No.** Looking at `%AI.Agent.Chat()` (Agent.cls lines 224-248):

```objectscript
Set json = $ZF(-6, $$$IrisLLMLibrary, $$$LLMAGENTCHAT,
    session.%agent,          // session token
    ..Provider.%token,
    input,
    ..ToolManager.%token,    // ← tool manager token passed PER CALL
    ..Temperature,
    hasFeedback,
    fbObj)
```

The tool manager token is passed to every `Chat()` call separately from the session. The session stores only conversation messages; tools come from the agent's `ToolManager` each turn. When we reconstitute the session in a new HTTP request, we also recreate the agent (which has a fresh `ToolManager` with our toolset) — the two are loosely coupled by design.

### 2.3 Recommended Persistence Strategy

**Approach A — Persistent `%AI.Agent.Session` + CSP session pointer (RECOMMENDED)**

On first turn:
```objectscript
// Build agent
Set agent = ##class(Custom.EnsSession.Agent).%New()  // our declarative agent
$$$ThrowOnError(agent.%Init())

// Create chat session
Set session = agent.CreateSession({"temperature": 0.2, "cache": {"enabled": (1), ...}})

// Chat turn
Set response = agent.Chat(session, userInput)

// Persist
Do session.%Save()                                    // writes to ^AI.Agent.SessionD
Set %session.Data("EnsAgent","SessionId") = session.%Id()
Set %session.Data("EnsAgent","NS") = $NAMESPACE
```

On subsequent turns:
```objectscript
Set chatId = $G(%session.Data("EnsAgent","SessionId"))
Set agent = ##class(Custom.EnsSession.Agent).%New()
$$$ThrowOnError(agent.%Init())
Set session = ##class(%AI.Agent.Session).Load(chatId, agent.Provider)  // reconstitutes Rust state

Set response = agent.Chat(session, userInput)
Do session.%Save()
```

**Pros**:
- Clean. Built-in. Survives process restarts.
- Can inspect session history via the DB.
- Works across the entire CSP-session lifetime.

**Cons**:
- Needs cleanup — add a purge task or a `/reset` button that calls `session.%DeleteId()`.
- Stores in a global (`^AI.Agent.SessionD`) — minimal footprint per chat but accumulates over time.

**Approach B — Export/Import + CSP session storage**

```objectscript
// After chat turn
Set jsonStr = session.Export()
Set %session.Data("EnsAgent","Context") = jsonStr
```

Next turn:
```objectscript
Set jsonStr = $G(%session.Data("EnsAgent","Context"))
Set session = agent.CreateSession(...)
Do session.Import(jsonStr)
```

**Pros**: No persistent database rows. Auto-cleanup when CSP session ends.
**Cons**: `%session.Data()` storage can bloat; JSON may be large.

**Recommendation**: Approach A for the hackathon — simpler code path, built-in methods, matches documented usage. Add a `/reset` button that calls `session.%DeleteId()` to restart the conversation.

### 2.4 Cross-Process Caveat

From the `Reconstitute` docstring (line 77): *"Must be called after opening a persisted session because `%agent` tokens do not survive across process boundaries."*

This is handled automatically by `Session.Load(id, provider)` — it calls `%OpenId` then `Reconstitute` in one step. We just need to call `Load()`, not `%OpenId()` directly.

---

## Section 3 — ZenMethod AJAX Pattern (The Chat Plumbing)

### 3.1 How ZenMethod Works

A Zen "ZenMethod" is a server-side ObjectScript method that gets auto-exposed to client-side JavaScript as a synchronous blocking call. The framework generates an AJAX hyperevent under the hood.

**Server-side declaration:**
```objectscript
Method SendChatMessage(userInput As %String, selectedMessageId As %String = "") As %String [ ZenMethod ]
{
    // ... fetch session-db-id from %session, load session, call agent.Chat()
    // ... persist session via %Save()
    // Return formatted response as HTML/Markdown string
    Quit responseHtml
}
```

**Client-side call:**
```javascript
var response = zenPage.SendChatMessage(inputText, selectedMsgId);
// 'response' is the string returned from the ObjectScript method
```

**Key properties**:
- **Synchronous by default**: the browser blocks until the server returns.
- **String-based**: return types must be primitives (`%String`, `%Boolean`, `%Integer`) or `%ZEN.proxyObject` (which serializes as a JS object).
- **Security**: gated through `standardDialog.OnPreHyperEvent()` which checks `GetHyperEventResources()` callback — override to control access.

### 3.2 Return Type Options

| Return type | Client side receives | Use case |
|---|---|---|
| `%String` | JS string | Rendered HTML, markdown, error message |
| `%Boolean` | JS bool (as 1/0) | Success flag |
| `%Integer` | JS number | Count, ID |
| `%ZEN.proxyObject` | JS object (property bag) | Structured response with multiple fields |
| Nothing | undefined | Fire-and-forget action |

For our chat, **return `%String`** containing pre-rendered HTML (Markdown → HTML server-side via `##class(%AI.System).RenderMarkdown()`). Simpler than passing structured data back and rendering client-side.

### 3.3 Resource-Check Override

`EnsPortal.Dialog.standardDialog.OnPreHyperEvent()` (lines 190-214) calls `GetHyperEventResources(pMethod)` to determine required resources. Default returns `""` (no extra resource required beyond the page resource).

Our subclass can override to keep the chat method accessible to anyone who can view the page:

```objectscript
ClassMethod GetHyperEventResources(pMethod As %String = "") As %String
{
    // No additional restrictions beyond %Ens_MessageTrace:USE (inherited)
    Quit ""
}
```

Or require a specific resource:
```objectscript
ClassMethod GetHyperEventResources(pMethod As %String = "") As %String
{
    If (pMethod = "SendChatMessage") || (pMethod = "ResetChat") {
        Quit "%Ens_MessageTrace:USE"  // same as page resource — trivially allowed
    }
    Quit ""
}
```

### 3.4 Error Handling

ZenMethod exceptions bubble up as client-side JavaScript exceptions via `zenExceptionHandler()`. Best practice: wrap the method body in `Try/Catch` and return an error-tagged string instead of throwing:

```objectscript
Method SendChatMessage(input As %String) As %String [ ZenMethod ]
{
    Try {
        // ... chat logic
        Quit "<div class='chat-response'>" _ responseHtml _ "</div>"
    } Catch ex {
        Quit "<div class='chat-error'>Error: " _ $ZCVT(ex.DisplayString(), "O", "HTML") _ "</div>"
    }
}
```

---

## Section 4 — CSP Deployment Topology (The Critical Infrastructure Decision)

### 4.1 The Problem Statement

Our custom classes compile in HSCUSTOM. The portal CSP app serves pages from **whichever namespace the user is currently in** (e.g., `/csp/healthshare/cengateway` → IRISAPP). We need:

1. Classes compiled ONCE in HSCUSTOM to be **callable from every interop namespace**.
2. The CSP URL to hit the portal's existing CSP app (same session cookie, no re-auth).
3. `$NAMESPACE` to be the user's target namespace when our tool runs SQL against `Ens.MessageHeader`.

### 4.2 The Solution: Package Mapping

*Source: Perplexity confirmation against InterSystems docs (`irisforhealthlatest/csp/docbook/DocBook.UI.Page.cls?KEY=AFNS`) — standard HealthShare/HSCUSTOM pattern.*

**Package mapping** exposes classes from a source namespace's database to other namespaces WITHOUT copying. The classes live once (in HSCUSTOM), but `##class(Custom.Foo).Method()` works from any namespace that has the mapping.

**The `%ALL` shortcut**: Instead of mapping to each namespace individually, map to `%ALL` — a wildcard that applies to every namespace on the server.

### 4.3 Deployment Recipe

**Step 1: Install classes in HSCUSTOM**
- Compile `Custom.EnsSession.*`, `Custom.EnsPortal.VisualTrace`, `Custom.EnsPortal.MessageViewer` in HSCUSTOM.

**Step 2: Create package mapping in `%SYS`** (Management Portal → System Administration → Configuration → System Configuration → Namespaces → [select %ALL or each target namespace] → Package Mappings → New)
- Package Name: `Custom.EnsSession`
- Source Database: HSCUSTOMCODE (or whichever code DB HSCUSTOM uses)

- Package Name: `Custom.EnsPortal`
- Source Database: HSCUSTOMCODE

Apply to `%ALL` if we want universal availability; or apply per-namespace (IRISAPP, IRISAPP, etc.) if more controlled.

**Step 3: Confirm CSP app routing works**
- The portal URL `/csp/healthshare/cengateway/Custom.EnsPortal.VisualTrace.zen?SESSIONID=42751` resolves via the IRISAPP CSP app.
- The CSP app looks up `Custom.EnsPortal.VisualTrace` — thanks to package mapping, finds it in HSCUSTOMCODE.
- The page runs **with `$NAMESPACE = "IRISAPP"`** — no switching needed.
- Our tool queries `Ens.MessageHeader` in IRISAPP — correct target namespace automatically.

**This is the simplest possible deployment.** No namespace switching in code, no separate login, same CSP session cookie.

### 4.4 The Bookmark URL Pattern

For the user's saved link, the URL shape is:

```
https://<portal-host>/csp/healthshare/<TARGET_NS>/Custom.EnsPortal.MessageViewer.zen
```

- `<portal-host>`: the IRIS host (e.g., `cen-hsgw-02.val.medallies.cloud`).
- `<TARGET_NS>`: the interop namespace the user wants to inspect (IRISAPP, IRISAPP, IRISAPP, etc.). The user can keep multiple bookmarks, one per namespace.
- No query parameters needed for the viewer; the user searches within the page.

**From the viewer, clicking a session opens**:
```
https://<portal-host>/csp/healthshare/<TARGET_NS>/Custom.EnsPortal.VisualTrace.zen?SESSIONID=<id>
```

### 4.5 Alternative — Why Not Namespace Switching?

You initially mentioned switching via `$NAMESPACE = <ns>` in code. Two reasons package mapping is better:

1. **CSP session fragility**: Changing `$NAMESPACE` mid-request can invalidate open objects, cursor state, and `%session` bindings. Not worth the risk.
2. **Class-cache re-resolution**: After `ZN HSCUSTOM`, your class references need to resolve in the new namespace too. Package mapping eliminates this entire concern.

**Package mapping is the standard HealthShare approach and carries zero runtime cost**.

### 4.6 Minimal Required Resources (SQL Privileges)

The agent's tools query `Ens.MessageHeader` etc. via SQL. These queries need:

- `SELECT` on `Ens.MessageHeader`, `Ens_Util.Log`, `Ens_Rule.Log`, `Ens.SuperSessionIndex` in the target namespace.
- The page resources (`%Ens_MessageTrace:USE`, `%Ens_MessageHeader:USE`) for the portal pages.
- NO mutation privileges required (our read-only design).

For hackathon purposes, running under an existing portal-admin user is fine. In production, provision a dedicated `EnsAgentUser` IRIS user with the minimum SQL SELECT grants.

---

## Section 5 — Namespace Switching Safety in CSP Context

### 5.1 What `ZN <ns>` Actually Does

`ZN` changes the process's current namespace, which affects:
- Class caching (new class references resolve in the new namespace)
- Global references (naked references follow the new default)
- `%Library` mappings

### 5.2 Dangers Inside a CSP/Zen Hyperevent

Within a `%OnAfterCreatePage`, a ZenMethod, or an `OnDrawContent` callback:

- **DON'T** change `$NAMESPACE` — it persists for the entire HTTP request. Any later code (including framework code after the hyperevent returns) may malfunction.
- **DO** use **temporary namespace switching with `new $NAMESPACE`** if absolutely necessary:
  ```objectscript
  New $NAMESPACE
  Set $NAMESPACE = "IRISAPP"
  // scoped work
  // on Quit, $NAMESPACE automatically restores
  ```
- **Or** use **`ZUtil` cross-namespace calls**:
  ```objectscript
  Set result = ##class(Custom.EnsSession.Tools.Trace).GetSessionSummary@"IRISAPP"(sessionId)
  // @namespace syntax runs the method in the target namespace without changing $NAMESPACE
  ```

### 5.3 Recommendation

**With package mapping in place, we DON'T NEED to switch namespaces at all**. The user's bookmark URL selects the namespace via the CSP app prefix. Our page runs with `$NAMESPACE` already correct. Our SQL queries hit the right tables.

If we ever DO need cross-namespace inspection (e.g., "show me related sessions in sister namespaces"), use the `methodname@"ns"` syntax rather than `ZN` to keep it scoped.

---

## Section 6 — Chat UI Component Strategy

### 6.1 The Constraint

The chat tab content is a `<html id="chatUI" OnDrawContent="DrawChatUI" />` — server-rendered HTML at page-load time, then static. But a chat interface needs:

- A scrolling message log (receives messages from user + agent)
- An input textarea
- A Send button
- AJAX communication with the agent via `zenPage.SendChatMessage()`

**Approach**: Server-render the initial skeleton (empty message log + input + send button), then use a small amount of client-side JavaScript to append messages and drive the chat.

### 6.2 Proposed DrawChatUI Output

```objectscript
Method DrawChatUI(pSeed As %String) As %Status
{
    &html<
    <div id="chatContainer" style="display: flex; flex-direction: column; height: 100%; min-height: 500px;">
        <div id="chatMessages" style="flex: 1; overflow-y: auto; padding: 10px; border: 1px solid #ccc; background: #fafafa;">
            <div class="chatMessage chatSystem">
                <b>Agent:</b> Hello! I can help you understand what happened in session #(..sessionId)#.
                Ask me anything about the messages, errors, or business processes involved.
            </div>
        </div>

        <div id="chatInputArea" style="padding: 10px; border-top: 1px solid #ccc;">
            <textarea id="chatInput" rows="3" style="width: 100%; box-sizing: border-box;"
                      onkeydown="zenPage.chatHandleKey(event);"
                      placeholder="Type your question and press Enter (Shift+Enter for newline)"></textarea>
            <div style="margin-top: 5px; display: flex; justify-content: space-between;">
                <button id="chatSendBtn" onclick="zenPage.chatSend();">Send</button>
                <button id="chatResetBtn" onclick="zenPage.chatReset();">Reset conversation</button>
            </div>
        </div>
    </div>

    <style>
    .chatMessage { margin: 6px 0; padding: 8px; border-radius: 4px; }
    .chatUser    { background: #e3f2fd; }
    .chatAgent   { background: #fff; border: 1px solid #eee; }
    .chatSystem  { background: #fff9c4; font-style: italic; }
    .chatError   { background: #ffebee; color: #c00; }
    .chatThinking { color: #888; font-style: italic; }
    </style>
    >
    Quit $$$OK
}
```

### 6.3 Client-Side Chat Wiring

Added as a `ClientMethod` on our subclass:

```objectscript
ClientMethod chatSend() [ Language = javascript ]
{
    var input = document.getElementById('chatInput');
    var text = input.value.trim();
    if (!text) return;

    var log = document.getElementById('chatMessages');

    // Append user bubble
    var userDiv = document.createElement('div');
    userDiv.className = 'chatMessage chatUser';
    userDiv.innerHTML = '<b>You:</b> ' + this.htmlEscape(text);
    log.appendChild(userDiv);
    input.value = '';

    // Thinking indicator
    var thinkDiv = document.createElement('div');
    thinkDiv.className = 'chatMessage chatThinking';
    thinkDiv.innerHTML = 'Thinking...';
    log.appendChild(thinkDiv);
    log.scrollTop = log.scrollHeight;

    // Call ZenMethod (blocking — UI freezes briefly, acceptable for demo)
    var responseHtml = zenPage.SendChatMessage(text, zenPage.currentId || '');

    // Replace thinking with response
    log.removeChild(thinkDiv);
    var agentDiv = document.createElement('div');
    agentDiv.className = 'chatMessage chatAgent';
    agentDiv.innerHTML = '<b>Agent:</b> ' + responseHtml;
    log.appendChild(agentDiv);
    log.scrollTop = log.scrollHeight;
}

ClientMethod chatHandleKey(evt) [ Language = javascript ]
{
    if (evt.keyCode === 13 && !evt.shiftKey) {
        evt.preventDefault();
        this.chatSend();
    }
}

ClientMethod chatReset() [ Language = javascript ]
{
    if (!confirm('Reset the conversation? The agent will forget context.')) return;
    zenPage.ResetChat();
    document.getElementById('chatMessages').innerHTML = '';
    document.getElementById('chatInput').value = '';
}

ClientMethod htmlEscape(s) [ Language = javascript ]
{
    return s.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
}
```

### 6.4 Context-Aware Chat

You specified *"chat about the whole session, with context of the selected message."* When the user clicks a message in the SVG trace, `zenPage.currentId` updates (see `VisualTrace.cls` `updateTabs()`). Passing it to `SendChatMessage` lets the agent know which message is currently focused:

```javascript
var responseHtml = zenPage.SendChatMessage(text, zenPage.currentId || '');
```

The server-side `SendChatMessage` can decorate the prompt with the selected-message context:

```objectscript
Method SendChatMessage(userInput As %String, selectedMessageId As %String = "") As %String [ ZenMethod ]
{
    // Enrich the user input with selection context
    Set contextualInput = userInput
    If selectedMessageId '= "" {
        Set contextualInput = "[The user is currently looking at message " _ selectedMessageId _ " in session " _ ..sessionId _ ".] " _ userInput
    }

    // ... load session, call agent.Chat(session, contextualInput) ...
}
```

---

## Section 7 — Phase 1 Build Recipe: Terminal REPL Bot

### 7.1 Target

An operator runs `Do ##class(Custom.EnsSession.Shell).Run()` from an IRIS terminal. The bot asks for namespace and session ID (or accepts them as arguments), switches namespace, and starts a chat REPL with the session in context.

### 7.2 Time Budget

**30-60 minutes** — simple, well-understood path.

### 7.3 Code

```objectscript
/// Terminal entry point for the Ensemble Session Inspection Agent.
/// Usage:
///   Do ##class(Custom.EnsSession.Shell).Run()
///   Do ##class(Custom.EnsSession.Shell).Run("IRISAPP", 42751)
Class Custom.EnsSession.Shell [ Abstract, System = 4 ]
{

/// Provider to use — reads from wallet at runtime.
Parameter PROVIDER = "anthropic";
Parameter MODEL = "claude-sonnet-4-5@20250929";

/// Launch the chat REPL. Both arguments are optional; prompt if missing.
ClassMethod Run(namespace As %String = "", sessionId As %Integer = 0) As %Status
{
    Set gray  = $C(27)_"[90m"
    Set cyan  = $C(27)_"[36m"
    Set bold  = $C(27)_"[1m"
    Set clr   = $C(27)_"[0m"

    // Welcome banner
    Write !, cyan, bold, "Ensemble Session Inspector", clr, !
    Write gray, "============================================================", clr, !!

    // Prompt for namespace
    If namespace = "" {
        Write "Namespace [", $NAMESPACE, "]: "
        Read ns
        If ns = "" Set ns = $NAMESPACE
    } Else {
        Set ns = namespace
    }

    // Prompt for session ID
    If sessionId = 0 {
        Write "Session ID: "
        Read sid
        If sid = "" {
            Write "No session ID — exiting.", !
            Quit $$$OK
        }
    } Else {
        Set sid = sessionId
    }

    // Switch namespace (safe here because we're a terminal, not a CSP request)
    If ns '= $NAMESPACE {
        Try {
            ZN ns
        } Catch ex {
            Write "Failed to enter namespace '", ns, "': ", ex.DisplayString(), !
            Quit ex.AsStatus()
        }
    }
    Write !, gray, "Inspecting session ", cyan, sid, clr, gray, " in namespace ", cyan, ns, clr, !!

    // Retrieve API key from wallet
    Set apiKey = ##class(%AI.Utils.SettingStore).Expand("@{wallet.AISecrets.AnthropicKey}")
    If apiKey = "" || (apiKey [ "@{") {
        Write "ERROR: Anthropic API key not set in wallet (AISecrets.AnthropicKey).", !
        Quit $$$ERROR($$$GeneralError, "Missing API key")
    }

    // Pre-seed conversation with the session context by writing a synthetic first user message.
    // The agent will see this as the opening turn and respond with a summary.
    Set openingMessage = "Please give me a summary of session " _ sid _ "."

    // Launch the shell with our toolset.
    // NOTE: Console.Run() hardcodes %AI.Shell.ConsoleAgent, so ShellTools will be loaded too.
    // For the hackathon this is acceptable; a cleaner wrapper can come later.
    Return ##class(%AI.Shell.Console).Run(
        ..#PROVIDER,
        apiKey,
        ..#MODEL,
        "Custom.EnsSession.Tools"
    )
}

}
```

### 7.4 Polish Path — Clean Custom Wrapper (Optional)

If the default ShellTools (filesystem/shell) cluttering the toolset bothers you, replace `Console.Run()` with a thin wrapper that uses a custom agent. ~100 lines, reuses patterns from `Console.Run()`. Not needed for a functional demo.

### 7.5 Verification Checklist

- [ ] HSCUSTOM has the `Custom.EnsSession.Shell` class compiled
- [ ] HSCUSTOM has the `Custom.EnsSession.Tools*` classes compiled (from prior research)
- [ ] IRIS Wallet entry `AISecrets.AnthropicKey` is set
- [ ] Running `Do ##class(Custom.EnsSession.Shell).Run("IRISAPP", 42751)` produces the shell
- [ ] Asking "what happened in this session" calls `GetSessionSummary` and returns a narrative

---

## Section 8 — Phase 2 Build Recipe: Chat Tab in Custom VisualTrace

### 8.1 Target

`Custom.EnsPortal.VisualTrace` extends `EnsPortal.VisualTrace` and adds a 4th tab labeled "Chat". The tab contains a chat UI that talks to the agent via ZenMethods.

### 8.2 Time Budget

**2-2.5 hours** — the bulk of the hackathon. Biggest risk areas:
- Getting the chat UI styled (~30 min)
- Wiring agent-state persistence across turns (~30 min — but we have the recipe)
- Handling edge cases (message selection context, markdown rendering) (~30 min)
- Deployment + first-run debugging (~30-60 min)

### 8.3 Class Skeleton

```objectscript
/// Custom VisualTrace with an agent chat tab alongside Header/Body/Contents.
/// Subclasses EnsPortal.VisualTrace using the same pattern InterSystems uses
/// in Ens.Enterprise.Portal.VisualTrace.
Class Custom.EnsPortal.VisualTrace Extends EnsPortal.VisualTrace [ System = 4 ]
{

Parameter PAGENAME = "Visual Trace with Agent";

/// Override tabs to add a 4th "Chat" tab.
XData allTabs [ XMLNamespace = "http://www.intersystems.com/zen" ]
{
<pane>
<tabGroup id="contentTabs" showTabBar="true" remember="true" onshowTab="zenPage.updateTabs(true);">
<tab id="headerDetails" caption="Header" title="Message Header Properties">
<html id="detailsContent" OnDrawContent="DrawDetailsContent" />
</tab>
<tab id="bodyDetails" caption="Body" title="Message Body Properties">
<html id="bodyInfo" OnDrawContent="DrawBodyInfo" />
</tab>
<tab id="bodyContents" caption="Contents" title="Message Body Contents">
<html id="fullContent" containerStyle="padding-top: 5px; padding-bottom: 5px;" OnDrawContent="DrawFullContentLinks" />
<iframe id="contentFrame" frameBorder="false" width="100%"/>
</tab>
<tab id="chatTab" caption="Chat" title="Chat about this session">
<html id="chatUI" OnDrawContent="DrawChatUI" />
</tab>
</tabGroup>
</pane>
}

/// Allow the chat ZenMethods without extra resource checks.
ClassMethod GetHyperEventResources(pMethod As %String = "") As %String
{
    Quit ""
}

/// Server-render the chat UI skeleton (empty log + input + send button).
Method DrawChatUI(pSeed As %String) As %Status
{
    // ... (see Section 6.2 content block) ...
    Quit $$$OK
}

/// ZenMethod: send a user message to the agent, return response HTML.
Method SendChatMessage(userInput As %String, selectedMessageId As %String = "") As %String [ ZenMethod ]
{
    Try {
        // 1. Load or create session
        Set sessDbId = $G(%session.Data("EnsAgent","SessionId"))
        Set sessNS   = $G(%session.Data("EnsAgent","NS"))

        // Reset if the user switched sessions / namespaces since the last chat turn
        If (sessNS '= $NAMESPACE) || '..SessionIdMatches(sessDbId) {
            Set sessDbId = ""
        }

        Set agent = ##class(Custom.EnsSession.Agent).%New()
        $$$ThrowOnError(agent.%Init())

        If sessDbId = "" {
            // New chat session — pre-seed with trace context
            Set chatSession = agent.CreateSession({
                "temperature": 0.2,
                "max_iterations": 12,
                "cache": {"enabled": (1), "cache_system_prompt": (1), "cache_tool_definitions": (1)}
            })
            // Record which ens session this chat is scoped to
            Set %session.Data("EnsAgent","EnsSessionId") = ..sessionId
        } Else {
            Set chatSession = ##class(%AI.Agent.Session).Load(sessDbId, agent.Provider)
        }

        // 2. Enrich input with selection context + ensemble session ID
        Set enrichedInput = userInput
        Set enrichedInput = "[Context: inspecting Ens session " _ ..sessionId _
            $S(selectedMessageId '= "":"; user is focused on message "_selectedMessageId, 1:"") _
            "]" _ $C(10) _ userInput

        // 3. Call the agent
        Set response = agent.Chat(chatSession, enrichedInput)

        // 4. Persist session
        Do chatSession.%Save()
        Set %session.Data("EnsAgent","SessionId") = chatSession.%Id()
        Set %session.Data("EnsAgent","NS") = $NAMESPACE

        // 5. Render response
        Set responseMd = response.Content
        // Convert Markdown to HTML via the AI Hub utility
        Set responseHtml = ..MarkdownToHtml(responseMd)
        Quit responseHtml

    } Catch ex {
        Quit "<span style='color: #c00;'>Error: " _ $ZCVT(ex.DisplayString(), "O", "HTML") _ "</span>"
    }
}

/// ZenMethod: reset the chat session.
ClassMethod ResetChat() As %Boolean [ ZenMethod ]
{
    Set sessDbId = $G(%session.Data("EnsAgent","SessionId"))
    If sessDbId '= "" {
        Do ##class(%AI.Agent.Session).%DeleteId(sessDbId)
    }
    Kill %session.Data("EnsAgent")
    Quit 1
}

/// Client-side: check if the cached session matches the current page's Ens sessionId.
ClassMethod SessionIdMatches(chatSessionId As %String) As %Boolean
{
    // Simple check — are we in the same Ens session as the chat was scoped to?
    Set cachedEnsId = $G(%session.Data("EnsAgent","EnsSessionId"))
    Quit (cachedEnsId = %page.sessionId)
}

/// Naive Markdown-to-HTML — delegate to AI Hub's renderer if available
ClassMethod MarkdownToHtml(md As %String) As %String
{
    // TODO: replace with %AI.System.RenderMarkdown output captured to string
    // Minimal HTML-escape fallback:
    Set html = $ZCVT(md, "O", "HTML")
    Set html = $REPLACE(html, $C(10), "<br/>")
    Quit html
}

/// Handle Enter key in input textarea.
ClientMethod chatHandleKey(evt) [ Language = javascript ]
{
    if (evt.keyCode === 13 && !evt.shiftKey) {
        evt.preventDefault();
        this.chatSend();
    }
}

/// Client-side chat send.
ClientMethod chatSend() [ Language = javascript ]
{
    // ... (see Section 6.3) ...
}

ClientMethod chatReset() [ Language = javascript ]
{
    // ... (see Section 6.3) ...
}

ClientMethod htmlEscape(s) [ Language = javascript ]
{
    return s.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');
}

}
```

### 8.4 The Custom Agent Class

`Custom.EnsSession.Agent` was already designed in the prior research (Step 4-5). Brief reminder:

```objectscript
Class Custom.EnsSession.Agent Extends %AI.Agent [ System = 4 ]
{
    Parameter PROVIDER = "anthropic";
    Parameter MODEL = "claude-sonnet-4-5@20250929";
    Parameter PROVIDERCONFIG = "{""api_key"": ""@{wallet.AISecrets.AnthropicKey}""}";
    Parameter TOOLSETS = "Custom.EnsSession.Tools";

    XData INSTRUCTIONS [ MimeType = text/markdown ]
    { ... Ensemble Session Inspector system prompt ... }
}
```

### 8.5 Verification Checklist

- [ ] Bookmark URL `/csp/healthshare/IRISAPP/Custom.EnsPortal.VisualTrace.zen?SESSIONID=42751` loads the custom page
- [ ] 4 tabs visible: Header / Body / Contents / Chat
- [ ] Chat tab shows a skeleton with message area + input
- [ ] Sending "what happened in this session" produces an agent narrative in under 15 seconds
- [ ] Clicking a message in the trace updates the "current message" context for next chat turn
- [ ] Reset button clears the chat and allows a fresh conversation
- [ ] Switching to a different session (via the Visual Trace nav) starts a new chat session automatically

### 8.6 Known Traps (and mitigations)

| Trap | Cause | Mitigation |
|---|---|---|
| `%session.Data()` state mismatches across sessions | CSP session persists across VisualTrace page loads; chat session is Ens-session-scoped | Compare cached Ens session ID to current `..sessionId` on first turn; reset if mismatch |
| User clicks through sessions quickly and sees stale chat | Same as above | Always check + reset |
| Long agent responses freeze the UI | ZenMethod is synchronous | Keep `max_iterations` ≤ 12; show "Thinking..." indicator; accept blocking UX for demo |
| Browser reload loses conversation | CSP session persists across reload | Works in our favor — `%session.Data` survives; the same chat session loads on reload |
| Multiple users share a portal user | `%session` is per-browser-session, not per-IRIS-user | Fine for demo; note for productionization |
| Markdown rendering looks rough | Placeholder `MarkdownToHtml` is minimal | Replace with a real renderer or use `%AI.System.RenderMarkdown` (need to capture to string) |

---

## Section 9 — Phase 3 Build Recipe: Custom MessageViewer with Handoff

### 9.1 Target

`Custom.EnsPortal.MessageViewer` extends `EnsPortal.MessageViewer`. The only meaningful change: clicking the "Trace" link on a session row opens **our custom VisualTrace** instead of the stock one.

### 9.2 Time Budget

**15-30 minutes** — trivial. The only real change is one JavaScript method.

### 9.3 Code

```objectscript
/// Message Viewer that hands off to Custom.EnsPortal.VisualTrace for chat-enabled tracing.
Class Custom.EnsPortal.MessageViewer Extends EnsPortal.MessageViewer [ System = 4 ]
{

Parameter PAGENAME = "Message Viewer (with Agent)";

/// Override to route to our custom visual-trace page.
ClientMethod showTrace(sessionId, evt) [ Language = javascript ]
{
    if (evt) {
        evt.cancelBubble = true;
        if (evt.stopPropagation) evt.stopPropagation();
    }
    if (sessionId != -1) {
        var URI = zenLink('Custom.EnsPortal.VisualTrace.zen?SESSIONID=' + sessionId);
        window.open(URI);
    }
}

}
```

That's the whole thing. Everything else (search, pagination, column rendering, side-panel tabs) inherits unchanged.

### 9.4 Verification Checklist

- [ ] Bookmark URL `/csp/healthshare/IRISAPP/Custom.EnsPortal.MessageViewer.zen` loads the custom page
- [ ] Search works identically to the stock MessageViewer
- [ ] Clicking a session ID in the results table opens `Custom.EnsPortal.VisualTrace.zen?SESSIONID=...` in a new window
- [ ] The opened page has the chat tab

---

## Section 10 — Cross-Cutting: Audit, Logging, Error UX

### 10.1 Audit: Record Every Chat Turn

Recommend adding a custom persistent class to log chat interactions:

```objectscript
Class Custom.EnsSession.AuditLog Extends %Persistent
{
    Property Timestamp As %TimeStamp [ InitialExpression = {$ZDT($H, 3, 1)} ];
    Property Namespace As %String;
    Property EnsSessionId As %Integer;
    Property ChatSessionId As %String;
    Property User As %String [ InitialExpression = {$USERNAME} ];
    Property UserInput As %String(MAXLEN = "");
    Property AgentResponse As %String(MAXLEN = "");
    Property TokensPrompt As %Integer;
    Property TokensCompletion As %Integer;
    Property DurationMs As %Integer;
    Property ToolCalls As %Integer;
    Index TimeCreated On Timestamp;
    Index EnsSession On EnsSessionId;
}
```

Call `Do log.%Save()` inside `SendChatMessage` after each turn. Zero demo cost, huge value for post-hoc analysis.

### 10.2 Error UX

The chat UI should visibly surface errors:
- If the ZenMethod throws → `Try/Catch` wrapper returns HTML-tagged error string. The error div renders red.
- If the LLM provider is down → show a specific "Provider unreachable — check wallet key or try again" message.
- If the session's tools return `{"error": "not_found"}` → the LLM narrates it; no special handling needed on the UI side.

### 10.3 Keeping the Chat Alive

The portal has a `keepAlive` mechanism (`startKeepAlive()` called in `onloadHandler`). This pings the server periodically to keep the CSP session from timing out. **Our chat relies on `%session.Data()` — so the keepAlive matters**. It's on by default; don't disable it.

---

## Section 11 — Deployment Master Plan

### 11.1 One-Time Setup (day before hackathon)

1. **Compile prior-research classes** in HSCUSTOM:
   - `Custom.EnsSession.Tools` (root toolset)
   - `Custom.EnsSession.Tools.Trace`, `.Body`, `.Process`, `.Errors`, `.Meta`
   - `Custom.EnsSession.Agent`
   - `Custom.EnsSession.ReadOnlyPolicy`
2. **Wallet entry**: `AISecrets.AnthropicKey` (or your provider's key) via `##class(%SYSTEM.Encryption).WalletCreate()` etc.
3. **Test Phase 1** manually: `Do ##class(%AI.Shell.Console).Run("anthropic", apiKey, "claude-sonnet-4-5", "Custom.EnsSession.Tools")`. Verify the tools are listed and chatting works.
4. **Package mapping**: Map `Custom.EnsSession` and `Custom.EnsPortal` from HSCUSTOMCODE to `%ALL` (or to each target interop namespace).
5. **Confirm permissions**: the portal-user resource set includes `%Ens_MessageTrace:USE` and `%Ens_MessageHeader:USE`. No new resources needed.

### 11.2 Hackathon Day Code Order

Morning (4 hours total — prioritized so that each milestone is a shippable demo):

**Hour 1** — Phase 1 shell (`Custom.EnsSession.Shell`). Compile. Demo: terminal chat with a session. 🎯 **Milestone 1: terminal demo ready.**

**Hour 2** — Phase 3 (`Custom.EnsPortal.MessageViewer`). Compile. Deploy via existing HSCUSTOM. Demo: search a session, click the link, see the stock VisualTrace open. 🎯 **Milestone 2: custom messageviewer ready (but still routes to stock trace).**

**Hours 2.5-3.5** — Phase 2 skeleton (`Custom.EnsPortal.VisualTrace`). Compile. Add `XData allTabs` override with 4th tab. Add `DrawChatUI`. Deploy. Demo: custom VisualTrace with 4 tabs, chat tab shows empty skeleton. 🎯 **Milestone 3: chat tab renders.**

**Hour 4** — Wire `SendChatMessage` ZenMethod + `chatSend()` client method + state persistence. Demo: end-to-end chat. 🎯 **Milestone 4: full integration.**

**Hours 1.5 / 0.5 polish** — Markdown rendering improvement, audit log, CSS tweaks, any recovery.

**Fallback**: if we miss Milestone 4, we still have Milestones 1 + 2 + 3 as the demo. Milestone 3 alone is visually impressive.

### 11.3 What to Drop if Time Runs Out

In priority order of cuttable features:
1. Audit log (cut entirely — not demo-critical)
2. Markdown HTML rendering (fall back to `<pre>`-wrapped plaintext)
3. Selected-message context enrichment (chat works without it)
4. Chat reset button (fall back to page reload)
5. `/stats`, `/tools` equivalent UI (not needed in Phase 2 anyway)

---

## Section 12 — Morning-of-Hackathon 1-Page Quick Start

*Print this page. Keep it next to the keyboard.*

**Goal**: 4-hour build. Phase 1 (30 min) → Phase 3 (15 min) → Phase 2 (3+ hours).

**Key class names to compile**:
- `Custom.EnsSession.Shell` — terminal entry
- `Custom.EnsSession.Agent` — AI Hub agent (declarative)
- `Custom.EnsSession.Tools` + 5 sub-classes — the ToolSet (from prior research)
- `Custom.EnsPortal.MessageViewer` — Phase 3
- `Custom.EnsPortal.VisualTrace` — Phase 2

**Bookmark URL** (user keeps per-namespace):
```
https://<host>/csp/healthshare/<TARGET_NS>/Custom.EnsPortal.MessageViewer.zen
```

**One-line Phase 1 test**:
```objectscript
Do ##class(Custom.EnsSession.Shell).Run("IRISAPP", 42751)
```

**Phase 2 key patterns**:
- Override `XData allTabs` to add `<tab id="chatTab" ... OnDrawContent="DrawChatUI" />`
- `Method DrawChatUI(pSeed)` writes HTML chat skeleton
- `Method SendChatMessage(input, selectedMsgId) [ZenMethod]` — loads persistent session from `%session.Data`, calls `agent.Chat`, saves, returns HTML
- `ClientMethod chatSend()` — client-side JS; calls `zenPage.SendChatMessage(...)` and renders the response

**Phase 3 key pattern** (trivial):
```objectscript
ClientMethod showTrace(sessionId, evt) [ Language = javascript ]
{
    if (sessionId != -1) {
        var URI = zenLink('Custom.EnsPortal.VisualTrace.zen?SESSIONID=' + sessionId);
        window.open(URI);
    }
}
```

**Agent-state persistence one-liner**:
```objectscript
Set chatSession = ##class(%AI.Agent.Session).Load(sessionDbId, agent.Provider)
// ... agent.Chat(chatSession, input) ...
Do chatSession.%Save()
Set %session.Data("EnsAgent","SessionId") = chatSession.%Id()
```

**Gotchas in order of "how likely to bite you"**:
1. Package mapping missing — page 404s. Check map exists in each target NS BEFORE hackathon starts.
2. Wallet key not set / expired — agent throws on init. Verify before coding.
3. Markdown response renders as plaintext (no newlines) — ship with `<pre>` wrapper, fix later.
4. Chat state sticks from a prior session — always compare cached Ens session ID to current in ZenMethod, reset if different.
5. Forgetting `[ ZenMethod ]` decorator on server method — client call throws "method not found."
6. `%session.Data` is a $LIST, values are strings — don't try to stash OREFs there.

**Decision tree for "chat isn't working"**:
- Error on page load → check compile, check package mapping
- 4th tab missing → check `XData allTabs` override compiled
- Tab empty/broken → `DrawChatUI` issue — check browser console
- Send button does nothing → `chatSend` client method name mismatch
- Send calls but no response → `SendChatMessage` ZenMethod issue — check IRIS error log
- Response comes back but renders wrong → `MarkdownToHtml` — fall back to `<pre>` wrapper

---

## Section 13 — Research Synthesis

### 13.1 Key Decisions Locked Down

| Decision | Choice | Rationale |
|---|---|---|
| Subclassing portal classes | **YES, subclass** | Proven via `Ens.Enterprise.Portal.VisualTrace` prior art |
| Agent state across HTTP requests | **Persistent `%AI.Agent.Session` + CSP session pointer** | Built-in; one-liner `Session.Load()` |
| Tool-schema state | **Non-issue** — schemas come from ToolManager per `Chat()` call | Verified in Agent.cls source |
| Deployment topology | **HSCUSTOM + package map to %ALL** | Standard HealthShare pattern; no NS switching |
| CSP session inheritance | **Ride the portal CSP app via bookmark URL** | Same domain, same session cookie |
| Chat UI tech | **Server-render skeleton + client JS + ZenMethod** | Proven Zen pattern |
| Chat UX | **Blocking synchronous per turn** | Acceptable for demo; simpler |
| Context-awareness | **Pass `selectedMessageId` to ZenMethod, enrich prompt server-side** | Simple, works |
| Phase 1 terminal | **`%AI.Shell.Console.Run()` with toolset arg** | Essentially free |
| Phase 3 handoff | **Override `showTrace()` ClientMethod (1 line change)** | Trivially small |
| Read-only enforcement | **Already in toolset policies from prior research** | No new work |
| Audit logging | **Optional — `Custom.EnsSession.AuditLog` persistent class** | Cut if time short |

### 13.2 Closed Research Gaps

All 6 clarifying questions (from the scope phase) are answered:

1. ✅ Subclassing vs parallel — subclassing, proven by prior art.
2. ✅ Auth/session — ride the portal via bookmark, no second login.
3. ✅ Chat UX — replaces right-hand panel, session-scoped with message context, blocking.
4. ✅ Phase 1 terminal — either direct `%AI.Shell.Console.Run()` or a thin wrapper.
5. ✅ Deployment — HSCUSTOM + package map to %ALL, no namespace switching.
6. ✅ Saved link — plain bookmark URL; no portal menu integration needed.

### 13.3 Confidence Levels

| Area | Confidence | Notes |
|---|---|---|
| Phase 1 code path | **Very High** | One-line entry, proven SDK APIs |
| Phase 3 code path | **Very High** | One-line override, trivial |
| Phase 2 subclassing | **High** | Direct prior art in IRIS codebase |
| Phase 2 ZenMethod pattern | **High** | Standard Zen framework, used throughout portal |
| Agent state persistence | **High** | Built-in SDK feature, docstring-confirmed |
| CSP deployment (package map) | **High** | Standard HealthShare pattern |
| Markdown rendering | **Medium** | Placeholder in recipe; may need real renderer |
| Namespace context correctness | **High** | Portal CSP app sets it; our code inherits |

### 13.4 Residual Risks

1. **EAP API churn**: `%AI.*` may shift between IRIS builds. Low impact — our touches are small.
2. **`%AI.Agent.Session` purge policy**: we save but don't prune. Accept for demo; add a daily purge task post-hackathon.
3. **Concurrent users on same session**: if two operators chat about the same session simultaneously, they have separate CSP sessions so separate chat states — correct by default.
4. **Namespace with unusual package-mapping**: HealthShare base namespaces may already have `HS.*` maps conflicting with `Custom.*`. Should be fine (different package name) but verify one target NS before hackathon.

### 13.5 Document Handoff

The combined research across the two documents now covers:

- **Prior research** (`technical-ensemble-session-inspection-agent-research-2026-04-24.md`): Agent + ToolSet design, SQL patterns, body dispatch, error decoding. Produces the agent's ObjectScript classes.
- **This document** (`technical-ensemble-session-agent-ui-integration-research-2026-04-24.md`): How to deploy that agent into three UI surfaces. Produces the terminal shell, the custom VisualTrace, and the custom MessageViewer.

Together they're a complete build spec for the hackathon.

---

**Research completion date**: 2026-04-24
**Total research effort**: 2 phases (data/tool side + UI integration side)
**Target build day**: hackathon (4-hour window)
**Expected outcome**: all three phases functional; possible polish trims if time runs short
