# Pressure Test Scenario

## Setup

A user says: `/sustained 给这个数据管道加错误重试和监控`

The 6-layer workflow should be followed from SETUP through COMPLETION.

## Execution Checks

### Layer 1 — Directory Structure
- `.tasks/` created with `active/`, `completed/`, `paused/` subdirectories
- Active task folder: `active/001_{slug}/` containing `MISSION.md`, `STATE.md`, `DECISIONS.md`, `AUDIT.md`, `BRIEF.md`
- `ACTIVE_POINTER.md` points to `active/001_{slug}/`
- `.claude/` cleanup considered (7-day rule applied if applicable)

### Layer 2 — Mission Anchor
- MISSION.md contains: goals, acceptance criteria, technical constraints, out-of-scope, trigger scenarios, available tools
- Mission folder is numbered (`001_`) — never overwrites
- STATE.md tracks current progress, active files, dispatch strategy

### Layer 3 — Coordinator Lite
- Coordinator reads MISSION.md + STATE.md before dispatching
- Does NOT write implementation code itself (unless lightweight ≤3 files)
- Sub-agent dispatch strategy recorded in STATE.md

### Layer 4 — Forced Escalation
- Every sub-agent prompt includes the return-format template (STATUS: DONE/CONCERN/BLOCKED/NEEDS_CONTEXT)
- Implementer-prompt.md contains escalation rules (no silent failures)
- Coordinator responds to STATUS correctly (CONCERN → log, BLOCKED → escalate)

### Layer 5 — Sentry Audit
- Checkpoint triggers sentry per the interval table (6-15 tasks → every 4)
- Audit covers: consistency, completeness, blind spots, debris
- Audit report written to AUDIT.md; coordinator responds

### Layer 6 — Tool Awareness
- Tool detection runs during SETUP (command -v / which)
- Available tools written to MISSION.md
- Sub-agent prompts include "report COORD if tool is missing"

### Cross-Session Continuity
- Compression recovery steps C1-C5 are followed on session restore
- STATE.md kept current at end of every turn
- RECENTER protocol executed at checkpoints

### COMPLETION
- Final sentry audit triggers
- Mission archived: `active/{NNN}_{slug}/` moved to `completed/{NNN}_{slug}/`
- STATE.md archived with folder (no separate collapse step)
- Email notification: sent if env vars configured, skipped otherwise

## Expected Pass Criteria

1. SKILL.md documents all 6 layers
2. implementer-prompt.md contains forced escalation return format (STATUS, core changes, verification)
3. sentry-prompt.md contains 4 audit dimensions (consistency, completeness, blind spots, debris)
4. All STATUS values (DONE/CONCERN/BLOCKED/NEEDS_CONTEXT) are handled in the skill
5. RECENTER protocol is documented for both session-internal and cross-session
6. Email notification is configurable via env vars (not hardcoded)
7. Tool detection is cross-platform (command -v, not Linux-only which)
