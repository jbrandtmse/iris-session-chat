---
date: 2026-04-29
status: approved
scope: minor
---

# Sprint Change Proposal — 2026-04-29

## Section 1: Issue Summary

**Problem statement:** Epic 1 tool stories (1.3 Message Body Inspection, 1.4 Core Session Timeline, 1.5 Error Decoding) and the Phase 1 Gate validation all require realistic Ensemble session data to execute against, but no story in Epic 1 currently creates that data. The Phase 1 Gate story also hardcoded a proprietary production namespace name (`CENGATEWAY`) that must not appear in shared or open-source artifacts.

**Discovery context:** Identified during post-hackathon planning review, before any implementation work has begun. No code exists yet, so no rollback is needed.

**Evidence:**
- Story 1.8 (Gate Validation) calls `SAgent.Main.Shell.Run("CENGATEWAY", 42751)` — no story creates session 42751 or the CENGATEWAY namespace
- Stories 1.3–1.5 acceptance criteria validate tool behavior against "a valid SessionId" with no mechanism to produce one on a fresh environment
- `prd.md` user journeys and `epics.md` contain `CENGATEWAY`, `CQGATEWAY` — proprietary production namespace names from a specific customer environment

---

## Section 2: Impact Analysis

**Epic impact:** Epic 1 only. One story added (1.2), existing stories 1.2–1.8 renumber to 1.3–1.9. No other epics affected.

**Story impact:**
- New Story 1.2 added: Sample Interoperability Production and Test Session Seeder
- Story 1.9 (was 1.8): Phase 1 Gate — acceptance criteria updated to reference seeder output and correct story count dependency

**Artifact conflicts:**
- `epics.md`: Story insertion, renumbering, gate story update, FR coverage map addition, namespace name sanitization
- `prd.md`: FR38 addition, namespace name sanitization (3 occurrences)
- `architecture.md`: Namespace name sanitization (1 occurrence)
- Research files: Namespace name sanitization (informational, lower priority)

**Technical impact:** None — pre-implementation. The sample production lives in `SAgent.Test.*`, already an established package in the architecture (src/SAgent/Test/ is created in Story 1.1).

---

## Section 3: Recommended Approach

**Selected:** Option 1 — Direct Adjustment

**Rationale:** The change is purely additive (one new story) plus editorial sanitization. No existing story is invalidated, no epic structure changes, no MVP scope change. The sample production is a natural complement to Story 1.1's directory setup and is independently valuable for all future epic work.

**Effort:** Low — one new story, one updated story, editorial find-and-replace across three files.  
**Risk:** Low — pre-implementation, no code to break.  
**Timeline impact:** None — Story 1.2 is infrastructure that enables Stories 1.3–1.5 and was implicitly required before; making it explicit adds one story's worth of work that was already going to be done informally.

---

## Section 4: Detailed Change Proposals

### Change A — New Story 1.2 (insert after current Story 1.1)

Existing Stories 1.2–1.8 renumber to 1.3–1.9.

```
Story 1.2: Sample Interoperability Production and Test Session Seeder

As a developer building and validating diagnostic tools,
I want a realistic sample Interoperability production in the IRISAPP namespace
with a seeder that generates sessions covering all diagnostic scenarios,
So that every tool story has concrete, repeatable session data to validate against.

Acceptance Criteria:

Given SAgent.Test.SampleProduction.Setup.CreateAndRun() is called in IRISAPP
When it completes
Then a production named "SAgent.Sample" is configured and running in IRISAPP
And the production has: a Business Service, a routing rule with two branches,
  a Business Process, and two Business Operations

Given SAgent.Test.SampleProduction.Seeder.GenerateSessions() is called
When it completes
Then it returns a %DynamicObject with session IDs for each scenario:
  - sessionIdSimple   — successful passthrough (BS → Router → BO1)
  - sessionIdRouted   — routing decision (Router fires a rule and selects BO2)
  - sessionIdFailed   — session with a known error (BP throws a caught exception,
                        ErrorStatus populated on the response header)
  - sessionIdHL7      — EnsLib.HL7.Message body variant
  - sessionIdJsonBody — %JSON.Adaptor body variant (custom SAgent.Test.SampleMsg)
And all 5 sessions are accessible in the target namespace's Visual Trace

Given a developer follows the README
Then they can call GenerateSessions() and use the returned session IDs
  in all subsequent tool validation steps and in SAgent.Test.GateTest
```

### Change B — Update Story 1.9 (was 1.8): Phase 1 Gate Validation

```
Section: Dependencies line

OLD: Given Stories 1.1–1.7 are complete
NEW: Given Stories 1.1–1.8 are complete
     And SAgent.Test.SampleProduction.Seeder.GenerateSessions() has been called
       and its output stored as: Set sessions = ##class(SAgent.Test.SampleProduction.Seeder).GenerateSessions()

Section: Criteria 4 and 5

OLD:
  4. Agent correctly answers "What happened in session X?" (non-empty narrative returned)
  5. Agent correctly answers "What does this error mean?" (decoded error returned)
  ...
  Given Do ##class(SAgent.Main.Shell).Run("CENGATEWAY", 42751) is executed

NEW:
  4. Agent correctly answers "What happened in session X?" using sessions.sessionIdSimple
     (non-empty narrative returned)
  5. Agent correctly answers "What does this error mean?" using sessions.sessionIdFailed
     (decoded error returned)
  ...
  Given Do ##class(SAgent.Main.Shell).Run("IRISAPP", sessions.sessionIdSimple) is executed
```

### Change C — Add FR38 to PRD and FR Coverage Map

```
PRD — Functional Requirements, after FR37:

FR38: A sample Interoperability production and session seeder are provided in the
SAgent.Test package so that developers and validators can generate repeatable,
realistic session data covering all diagnostic tool scenarios (simple passthrough,
routing decision, Business Process error, HL7 body, JSON body) without requiring
access to a production namespace.

epics.md FR Coverage Map, after FR37 line:

FR38: Epic 1, Story 1.2 — Sample production and test session seeder
```

### Change D — Namespace name sanitization

Replace all occurrences of proprietary namespace names across planning artifacts:

| Find | Replace with | Files |
|---|---|---|
| `CENGATEWAY` | `IRISAPP` | prd.md (4×), architecture.md (1×), epics.md (1×), research files |
| `CQGATEWAY` | `IRISAPP` | prd.md (2×), research files |
| `FHIRGATEWAY` | `IRISAPP` | research files |
| `(CENGATEWAY, CQGATEWAY)` | `(e.g. IRISAPP)` | prd.md Journey 3 |

---

## Section 5: Implementation Handoff

**Scope classification:** Minor — all changes are story/document edits with no architectural implications.

**Handoff:** Developer agent — direct implementation of artifact edits.

**Responsibilities:**
- Update `epics.md`: insert Story 1.2, renumber 1.2–1.8 → 1.3–1.9, update Story 1.9 gate criteria, add FR38 to coverage map, sanitize namespace names
- Update `prd.md`: add FR38 to functional requirements section, sanitize namespace names
- Update `architecture.md`: sanitize namespace name (1 occurrence)
- Update research files: sanitize namespace names (informational)

**Success criteria:**
- `grep -r "CENGATEWAY\|CQGATEWAY\|FHIRGATEWAY" _bmad-output/` returns no results
- Epic 1 in `epics.md` contains Stories 1.1–1.9, Story 1.2 is the sample production
- Story 1.9 gate criteria reference `sessions.sessionIdSimple` and `sessions.sessionIdFailed`
- FR38 appears in both `prd.md` and the FR Coverage Map in `epics.md`
