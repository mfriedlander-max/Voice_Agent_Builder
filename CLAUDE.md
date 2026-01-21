# CLAUDE.md

## Critical: Workflow Compliance

**Failure to follow the workflow defined in AI_PLAN.md is a Project Failure.**

---

## Mandatory Behaviors

### On Every Session Start, Resume, or Context Compaction
1. Read `AI_PLAN.md` completely before any work
2. Read `AI_SCRATCHPAD.md` to restore full state
3. Check Branch Map for `in-progress` branches
4. Resume from first unchecked task

### After Every Implementation
Update these documents immediately to reflect changes:
- [ ] `AI_PLAN.md` — mark tasks complete, update branch status
- [ ] `README.md` — update if features/setup/usage changed
- [ ] `AI_SCRATCHPAD.md` — log decisions, blockers, next steps

**If any document is out of date, update it before continuing other work.**

---

## Workflow Violations (Project Failure)
- Starting work without reading AI_PLAN.md
- Skipping tasks or working out of order
- Leaving documentation stale after implementation
- Ignoring branch map status
