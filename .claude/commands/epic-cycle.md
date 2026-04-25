# /epic-cycle

Execute the BMAD Method development implementation cycle using Agent Teams for all stories in an epic or a range of epics.

## Usage

```
/epic-cycle                    — run all epics from the beginning
/epic-cycle 1                  — run Epic 1 only
/epic-cycle 1-3                — run Epics 1 through 3
/epic-cycle 1.3                — run a single story (Epic 1, Story 3)
```

## Instructions

Read and follow the complete instructions in `docs/epic-cycle-teams.md`. That document is the authoritative source for:

- Pipeline flow (per-epic and per-story sequencing)
- Parallel execution rules (safe parallel windows)
- Spawn-on-demand agent coordination pattern
- Context handoff between pipeline stages
- Sprint planning gate (mandatory before each epic)
- Retrospective review and Story X.0 creation (mandatory before each epic)
- Shutdown-before-respawn sequencing
- Submodule commit order
- Completion logging
- Anti-patterns to avoid

## Key Constraints

- All agents must be spawned with `mode: "bypassPermissions"` (YOLO mode)
- All BMAD skills must be invoked via the `Skill` tool — never interpreted inline
- Lead creates story files directly via `Skill` tool — never via an agent
- Pre-draft Story N+1 while code-reviewer(N) is running (safe parallel window)
- Do NOT spawn developer(N+1) until code-reviewer(N) is complete and committed
- Use unique agent names per story: `dev-{epic}-{story}`, `cr-{epic}-{story}`
- Shut down each agent and wait for shutdown confirmation before spawning the next
