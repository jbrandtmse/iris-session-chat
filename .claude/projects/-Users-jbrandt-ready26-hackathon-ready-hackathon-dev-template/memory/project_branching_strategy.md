---
name: Hackathon Git Branching Strategy
description: Git branching and Track B start trigger for the Ensemble Session Inspection Agent hackathon build
type: project
---

# Hackathon Git Branching Strategy

**Why:** The project has two parallel build tracks (Track A: agent/tools, Track B: portal UI) that can run concurrently after the Phase 1 Gate passes. The branching strategy reflects this.

## Branch Structure

```
main
 └── dev              ← integration branch (both tracks merge here)
      ├── track-a     ← SAgent.Main.* + SAgent.Tools.* work
      └── track-b     ← SAgent.Portal.* work (starts AFTER gate)
```

## Rules

- **Story-level commits** on each branch: `git commit -m "Story 1.3: Core Session Timeline Tools"`
- **Track A merges to `dev`** when Phase 1 Gate passes (Story 1.8 all 7 criteria pass)
- **Track B branches from `dev`** ONLY AFTER Track A gate merge — never before
- **Both tracks merge to `main`** when demo-ready
- Track A and Track B files never conflict (different directories: `src/SAgent/Tools/` vs `src/SAgent/Portal/`)

## CRITICAL: Track B Start Trigger

**Track B MUST NOT start until ALL 7 Phase 1 Gate criteria pass (Story 1.8):**

1. SAgent.Main.Agent compiles and initializes in <5 seconds
2. SAgent.Main.Shell.Run() launches interactive REPL
3. SAgent.Tools.Trace and SAgent.Tools.Errors compile
4. Agent answers "What happened in session X?" correctly
5. Agent answers "What does this error mean?" correctly
6. Multi-turn confirmed (follow-up references prior context)
7. Read-only verified (no Ens.* rows added/modified after a chat turn)

**When gate passes:** Track A developer merges `track-a` → `dev`, then announces "Gate passed — Track B can start." Track B developer branches from `dev` at that point.

## IRIS-Specific Note

Git branches manage source files; HSCUSTOM manages compiled classes. Track B developer needs Track A classes compiled in the SHARED HSCUSTOM namespace before they can test portal code. If developers share one IRIS instance, Track A compilations are immediately visible to Track B. If they have separate IRIS instances, Track B developer pulls Track A source from `dev` and compiles it locally first.

## Simplified Alternative (Solo or Small Team on Shared IRIS)

If 1-2 developers share one IRIS server, a single `dev` branch with story-level commits works fine. The file-level directory separation (`SAgent/Tools/` vs `SAgent/Portal/`) prevents merge conflicts naturally. Skip the branch overhead; use story commits as checkpoints instead.

**How to apply:** When Story 1.8 is complete and the gate passes, explicitly call out: "Phase 1 Gate passed — time to start Track B."
