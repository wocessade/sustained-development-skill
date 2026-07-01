---
name: sustained-development
description: 背景被动技能 — 在检测到复杂任务时自动触发（≥4文件或≥2子任务）。也支持 '/sustained' 或 '长任务' 手动触发。自动触发时告知用户后直接进入SETUP。
---

# Sustained Development — Attention Retention Workflow

## ⚡ Executive Pass (must-read —读完本节即可开始工作，其余用于查阅)

### Scope
```
Allowed:         Read files | Design prompts | Dispatch decisions | Audit
                 Lightweight direct execution (≤3 files + design + atomic, log to DECISIONS)
Not Allowed:     Write code (unless direct mode) | Skip verification
                 Modify MISSION.md (user only)
```
### Response Rules
```
DONE    → Update STATE.md, continue
CONCERN → Write to DECISIONS.md, continue (2 consecutive → force sentry)
BLOCKED → Has alternative → dispatch new task. No alternative → escalate to user
          【Do NOT ignore BLOCKED and investigate on your own】
```
### Crash Recovery (must execute before dispatch after compaction)
```
Step C1: Read STATE.md → extract current mission + progress
Step C2: Read DECISIONS.md (last 30 lines) → extract key decisions
Step C3: Read AUDIT.md (most recent) → extract unresolved items
Step C4: Output one-line recovery statement: "Session restored. Mission X, progress M/N..."
Step C5: Run drift self-check (see BRIEF.md protocol)
```
### Self-Refresh Anchor
```
After every 1 subtask → [RESET] phase, read this Executive Pass (~10 lines)
Purpose: Confirm you're still on track, not running on memory.
```

## Overview

**Agent attention isn't about intelligence — it's about having only one context window.** The solution isn't a bigger window; it's making the filesystem a reliable second brain — all decisions, state, feedback, and audits are written to files, and the agent must read files before making decisions, not rely on memory.

This workflow has six layers of defense:
1. **Directory Structure & Cleanup** — `.tasks/` organized by per-task folders, `.claude/` periodic cleanup
2. **Mission Anchor** — Goals/constraints/boundaries externalized to prevent memory drift
3. **Coordinator Lite** — Coordinator orchestrates and strategizes only, no deep work
4. **Forced Escalation** — Hard-constrained return format for sub-agents, forcing proactive feedback
5. **Sentry Audit** — Independent sub-agent periodically audits direction consistency
6. **Tool Awareness** — Proactively discover and use locally installed tools

## When to Use

### Trigger — 大任务自动触发模式（被动技能）

**本技能是背景被动技能，CLAUDE.md 已配置为每个用户请求时自动检测复杂度。**

**自动触发条件（满足任一即触发，无需用户确认）：**
- 任务涉及 ≥ 4 个文件的修改
- 任务包含 ≥ 2 个独立的子任务/步骤
- 用户使用多阶段描述（"先做A再做B然后C"）
- 用户使用审慎型措辞（"仔细做"、"一步步"、"这个比较复杂"）

触发时告知用户："检测到复杂任务，自动激活长任务模式（sustained-development）。"随后直接执行 SETUP 流程。

**注意：** 原来"需要用户输入 `/sustained`"的旧手动触发方式已废弃。复杂度检测全自动，判断不明确时偏保守触发（误触发无害，漏触发导致上下文丢失）。手动输入 `/sustained` 仍可触发（向后兼容）。

### Good Fit

- Task involves ≥ 4 file changes or ≥ 2 sub-tasks
- Task needs external tool validation (browser screenshots, test runs, API calls)
- Fixing a known issue where root cause is uncertain
- User uses careful-measure language ("仔细做", "一步步")

### Bad Fit

- Single-file edit (use executing-plans directly)
- Pure exploration (no "completion" concept)
- User explicitly says "quick change" or "快速改一下"

## Core Principle

**Externalize agent memory, institutionalize initiative.**

```
Don't trust the agent to "remember" — so read files before every decision.
Don't trust the agent to be "proactive" — so sub-agent return format forces feedback.
Don't trust the agent to "change course" — so the sentry independently audits direction.
```

## Cross-Session Continuity Protocol

**Context compaction can happen at any time — not just at checkpoints, but at the start of any turn.** This is the most important protocol in sustained-development, with higher priority than all other layers.

### Compaction Detection

Every turn, the coordinator must determine whether it is in a post-compaction session:

| Signal | Meaning |
|--------|---------|
| `<conversation_history_summary>` tag in context | Compacted |
| Doesn't remember what was done last turn | Compacted |
| STATE.md progress doesn't match memory | Compacted |
| `SessionStart hook` system-reminder in context | Possibly new session |

**Rule: Better to falsely assume compaction (waste one read) than to miss it (continue on stale memory).**

### Compaction Recovery

When compaction is detected, execute these steps **before producing any output or making any decisions**:

```
Step C1: Read STATE.md
         → Get current mission, total progress, active files, pending items
Step C2: Read DECISIONS.md (last 30 lines only)
         → Get key decisions from last session
Step C3: Read AUDIT.md (most recent audit only)
         → Get unresolved audit findings
Step C4: Output one-line recovery statement:
         "Session restored. Mission: <name>, progress <M>/<N>, phase: <stage>, pending: <items>"
Step C5: If STATE.md shows in_progress tasks → continue execution
         If all tasks completed → enter COMPLETION flow
```

**Step C4 is mandatory** — it syncs the user and coordinator on current state instead of starting from zero.

### Cross-Session STATE Transfer

At the end of every turn (including pre-compaction), the coordinator must ensure STATE.md contains enough info for the recovery self to continue:

- The `## Key Context at Last Reset` field must be updated after every round
- Key context = what we're doing + why + what's next (≤ 3 sentences)
- Don't assume "I'll remember" — write it down

### Post-Compaction Pitfalls

| Pitfall | Avoidance |
|---------|-----------|
| Repeating completed work | Must read STATE.md first, compare task status |
| Forgetting validated decisions | Must read DECISIONS.md recent entries first |
| Reopening resolved issues | AUDIT.md shows closed findings → don't reopen |
| Mixing two missions' tasks | STATE.md `## Current Mission` field is the authority |

---

## BRIEF.md Protocol — Briefings & Drift Self-Check

### Purpose

BRIEF.md is a **structured briefing appended after each subtask**. It serves two roles:
1. **Executor's self-examination tool** — writing the brief checks whether you're deviating from acceptance criteria
2. **Third-party observer interface** — with a fixed format, a cron or observer agent can read the latest brief, detect drift, and send ALERT to the coordinator

### Brief Format

After each subtask (Direct or Subagent), after updating STATE.md but before entering RESET, append one entry to the current task folder's `BRIEF.md` (locate via `cat .tasks/ACTIVE_POINTER.md`):

```markdown
## Brief #N — YYYY-MM-DD HH:MM — <subtask-name>

STATUS: [DONE | CONCERN | BLOCKED]
Done: <1-2 sentences>
Progress: <completed M / total N>

drift self-check:
  - Still within MISSION.md acceptance criteria: yes/no
  - If no, is the deviation recorded in DECISIONS.md: yes/no【no→pause & escalate】
  - This subtask leads to completion or a branch: completion/branch
  - If branch, is the return-to-main-path marked: yes/no【no→must mark】

Next: <what to dispatch next>
```

### drift self-check Trigger Rules

| When | Who | Condition |
|------|-----|-----------|
| Writing BRIEF.md | Coordinator | **Every entry must include it**, attached in the brief |
| Before dispatching a new task | Coordinator | Read last 2-3 BRIEF.md entries, check if drift is flagged |
| Cross-session recovery | Coordinator | Read last 1 BRIEF.md + AUDIT.md, confirm no unresolved drift |
| Third-party observer (reserved) | cron/agent | Read latest BRIEF.md, if drift=no → write ALERT |

### drift Response Rules

```
Any "no" in drift self-check → pause current work:
  1. Update DECISIONS.md: record the drift fact and reason
  2. Present two options: course correct / formally pivot
  3. After user choice: course correct → revert to correct path, pivot → update MISSION.md (user does)
  4. Update BRIEF.md: append a "course correction" entry
```

### Third-Party Observer Interface (Reserved)

Claude Code currently doesn't support persistent observer processes. The BRIEF.md format reserves this interface:

```
Observer responsibilities (when supported):
  1. Periodically read the latest BRIEF.md entry
  2. Check drift self-check result
  3. 2 consecutive drift=yes → write an ALERT to the current task folder's ALERT.md
  4. Coordinator checks ALERT.md before each dispatch

Note: This interface is currently empty. Coordinator self-checks via RESET phase instead.
```

---

## Layer 1: Directory Structure & Cleanup

### Directory Layout

Each task has its own folder; status is determined by which subdirectory it's in:

```
{project_root}/
  .tasks/
    ACTIVE_POINTER.md              ← Points to current active folder (one line)
    active/                        ← Currently executing (≤ 1 folder)
      {NNN}_{feature-slug}/
        MISSION.md                 ← Mission anchor (no NNN prefix — folder name is the identifier)
        STATE.md                   ← Coordinator state register (frequently updated)
        DECISIONS.md               ← Decision log (append-only)
        AUDIT.md                   ← Sentry audit record (append-only)
        BRIEF.md                   ← Subtask briefings (append-only)
        ALERT.md                   ← Third-party observer alert (optional)
    completed/                     ← Finished
      {NNN}_{feature-slug}/
        (same files)
    paused/                        ← User manually paused
      {NNN}_{feature-slug}/
        (same files)
```

**Numbering:** The leading `NNN` is zero-padded, starting from 001. Scan `active/` + `completed/` + `paused/` for the max number and increment by 1. The `feature-slug` uses English kebab-case (e.g., `add-error-retry`) for cross-platform compatibility.

**State transitions:**
- New task → create `active/{NNN}_{slug}/`
- Pause → `mv active/{NNN}_{slug}/ paused/{NNN}_{slug}/`
- Complete → `mv active/{NNN}_{slug}/ completed/{NNN}_{slug}/`

### ACTIVE_POINTER.md

`ACTIVE_POINTER.md` replaces the old `ACTIVE_MISSION.md`. It stores a single relative path:

```
active/001_add-error-retry/
```

For compaction recovery, RECENTER, etc., run `cat .tasks/ACTIVE_POINTER.md` to get the current active task folder path. No more full copy of MISSION.md.

### .claude/ Cleanup Protocol

Execute automatically on completion of each full task:

1. **Old plan cleanup:** Move current plan to `completed/{NNN}_{slug}/`, list other plans for user to confirm deletion
2. **Old spec cleanup:** Archive specs associated with archived plans
3. **Unassociated .md files:** List for user to confirm deletion
4. **Retention rule:** Don't delete files from the last 7 days; don't execute any deletion without user approval

## Layer 2: Mission Anchor

### MISSION.md (per-folder numbering)

Each new mission auto-increments the number (001, 002...), **never overwrites existing folders**. `ACTIVE_POINTER.md` points to the current active task folder.

**Numbering:**
- Scan `.tasks/active/`, `.tasks/completed/`, `.tasks/paused/` for all `{NNN}_*` folders
- Take max NNN + 1, zero-pad to three digits
- First mission creates folder `active/001_<slug>/`, with MISSION.md title `# 001: <feature-name>`
- Archival = move the entire folder (no per-file copy)

```markdown
# {NNN}: <feature-name>

## Goal
<1-3 sentences>

## Acceptance Criteria
- [ ] <verifiable criterion 1>
- [ ] <verifiable criterion 2>

## Technical Constraints
- <framework/API/environment limits>

## Out of Scope
- <explicitly what NOT to do, to prevent scope creep>

## Trigger Report Scenarios
Must pause and report (no bypass) when:
1. Plan is found infeasible
2. A step fails 2 consecutive times
3. Uncovered dependencies or side effects
4. Decision conflicts with existing constraints

## Available Tools
- code: VSCode editor
- Chrome: Web browser, for UI verification
- node/npm/npx: Node.js
- python3: Python 3.11
- winget: Windows package manager
- <project-specific tools>

## Decision Log
<!-- Appended to DECISIONS.md; this section only records the reference -->
```

**Reference convention:** When other protocol docs (compaction recovery, RECENTER, drift check) say "read MISSION.md", they mean the current active task's `MISSION.md`. Get the folder path via `cat .tasks/ACTIVE_POINTER.md`, then read the file inside.
   **Note:** `.tasks/MISSION.md` (not inside a numbered folder) is an illegal orphan. If found during SETUP, must archive before proceeding.

### STATE.md

STATE.md is the first entry point for session recovery — it must let a compacted coordinator understand current state within 5 seconds.

```markdown
# State

## Current Mission
<mission-name> — started <YYYY-MM-DD>

## Progress
total=N  completed=M  ongoing=K  checkpoint_interval=I

## Active Files
- <path>: <purpose>

## Dispatch Strategy Log
- Subtask N: <Direct|Subagent|Parallel, agent type, tools>

## Pending Returns
- [ ] <items flagged BLOCKED/CONCERN pending resolution>

## Key Context at Last Reset
<summary for the coordinator — ≤ 3 sentences: what + why + next>
```

**Multi-mission rules:**
- STATE.md always keeps only the current active mission's detailed state
- When a mission completes, append its STATE summary as a DECISIONS.md mission-completion entry, then create fresh STATE.md for the new mission
- `## Mission History` collapsed area records completed mission summaries (name + completion date + result, one per line)
- Never maintain two missions' detailed state in one STATE.md — it causes confusion on recovery

### RECENTER Protocol

**In-session checkpoint (after every N subtasks):**

```
1. Read current MISSION.md (not from memory. Locate: `cat .tasks/ACTIVE_POINTER.md` for folder, then read MISSION.md inside)
2. Answer in one sentence: "How far are we from acceptance criteria?"
3. Check if any trigger report scenario is happening
4. Update STATE.md
```

**Cross-session recovery (each turn start, if compaction detected):**

```
1. Execute Cross-Session Continuity Protocol steps C1-C5
2. Output recovery statement to user
3. Check key context from last reset — this explains "why we're doing this"
4. If key context is blank or stale → pause, do a mini-RECENTER, then continue
```

### DECISIONS.md Minimal Format

Each decision must include these fields (one-line header + optional details):

```markdown
## YYYY-MM-DD — [type] title

Type: decision | audit-response | mission-completion | sentry-response
Conclusion: <one sentence>
Reason: <why this decision>
```
**Rules:**
- Append-only, newest at top
- Title must include date and type — for quick scanning on compaction recovery
- Audit responses and mission completions must also be recorded as decision entries

---

## Layer 3: Coordinator Lite

### Role Boundaries

| Allowed | Not Allowed |
|---------|-------------|
| Read MISSION.md / STATE.md | Write implementation code (see lightweight exception below) |
| Design subagent prompts | Deep-analyze code implementation |
| Decide dispatch strategy (parallel/serial/tool choice) | Replace subagent for verification |
| Parse subagent STATUS responses | Keep subagent code details in local memory |
| Update STATE.md / DECISIONS.md | Modify MISSION.md goals |
| Trigger sentry audits | Skip verification steps |
| **Lightweight direct execution:** When a single step is atomically verifiable (≤ 3 files, no new design work, each edit is a targeted replacement), the coordinator may execute it directly. Must record in DECISIONS.md: what was changed, why direct execution was chosen, and verification result. | Launch subagents for operations that are cheaper to do directly (single-file targeted edits, simple file creation, deletion of confirmed orphans). The subagent dispatch + context-load overhead is warranted only when the subagent needs independent judgment or handles ≥ 4 files. |

### Execution Mode Selection

The coordinator must choose an execution mode before each subtask and record it in STATE.md's Dispatch Strategy Log:

| Mode | When to Use | Overhead |
|------|-------------|----------|
| **Direct** | ≤ 3 files, pure edit/create/delete, no design decisions | 0 extra context |
| **Subagent (single)** | ≥ 4 files, or needs independent judgment (e.g., audit) | ~15K token startup cost |
| **Subagent (parallel)** | Multiple independent subtasks, no dependencies | ~15K × N but concurrent |

### Execution Loop

```
Each subtask:
  1. [STRATEGY] Examine task requirements:
     - Dependencies? → can they run in parallel?
     - What tools needed? → check MISSION.md + which
     - What context needed? → prepare prompt
     - What mode?
       · Simple independent → parallel subagents
       · Needs multi-step verification → subagent-driven-development
       · Lightweight single step → executing-plans
  2. [DISPATCH] Compose prompt with Forced Escalation template → dispatch
  3. [ACT]     Respond based on STATUS
  4. [LOG]     Update STATE.md + DECISIONS.md
  5. [BRIEF]   Write BRIEF.md (including drift self-check)
               → Read entire step before deciding next action
               → Any "no" in drift check → pause, execute drift response rules
  6. [DRIFT]   Before dispatching a new task, read last 2-3 BRIEF.md entries
               → Last entry all "yes" → continue
               → Any "no" → pause, execute drift response rules
  7. [RESET]   Prune local memory (keep only steps/progress/pointers)
               → Read Executive Pass self-refresh anchor section (confirm boundaries)
               → One-line check: "Is what I just did in the Allowed list?"
```

### Strategy Thinking Constraints

Prevent the coordinator from "overthinking":

- Each dispatch strategy time ≤ 1 file read + 1 strategy log entry
- Strategy result written to STATE.md "Dispatch Strategy" field (≤ 100 chars)
- Same subtask dispatched ≥ 3 times without success → wrong granularity, pause and report
- 80% of a prompt is good enough — sub-agents will ask

## Layer 4: Forced Escalation

### Sub-agent Return Format Constraint

All sub-agents (implementer, sentry, fixer) must start their return with this structure:

```
STATUS: [DONE | CONCERN | BLOCKED | NEEDS_CONTEXT]

Core Changes:
<2-3 sentences>

Verification:
<command and output>

If STATUS=CONCERN:
  Issue: <description>
  Impact: <scope>
  Option A: <approach + rationale>
  Option B: <approach + rationale>

If STATUS=BLOCKED:
  Blocked by: <reason>
  Attempted: <method + result>
  Suggestion: <approach>
```

### Coordinator Response Rules

```
STATUS=DONE       → Update STATE.md, continue
STATUS=CONCERN    → Must record in DECISIONS.md, assess, continue
STATUS=BLOCKED
  → Has alternative → evaluate → dispatch new task
  → No alternative → escalate to user
STATUS=NEEDS_CONTEXT → escalate (plan has gap)

Additional rules:
- 2 consecutive CONCERN → force sentry
- Do not ignore BLOCKED and research on your own
```

### Return Template (Embedded in Sub-agent Prompt)

Every sub-agent dispatch **must include** the following block at the end of the prompt:

```
---
## Return Format

Please end your reply with the following structure:

STATUS: [DONE | CONCERN | BLOCKED | NEEDS_CONTEXT]

Core Changes:
<2-3 sentences>

Verification:
<command and output>

If STATUS=CONCERN:
  Issue: ...
  Option A: ...
  Option B: ...

If STATUS=BLOCKED:
  Blocked by: ...
  Suggestion: ...
```

Also embed these rules:

```
When executing a subtask, you MUST flag CONCERN or BLOCKED in these situations:
  1. Current approach is infeasible (don't push through)
  2. Required tool is not in the environment (report COORD, don't install yourself)
  3. Discovered dependencies not covered in the plan
  4. An alternative approach is clearly better

Note: Flagging BLOCKED without a suggested alternative is insufficient.
Flagging CONCERN without providing options is incomplete.
```

## Layer 5: Sentry Audit

### Trigger Strategy

| Total Subtasks | Checkpoint Interval | Sentry Trigger |
|---------------|-------------------|----------------|
| ≤ 5 | After all | 1 final audit |
| 6-15 | Every 4 subtasks | Mid + final |
| ≥ 16 | Every 3 subtasks | Multiple mid + final |

Additional triggers: 2 consecutive CONCERN / 3 failed dispatches for the same subtask / user manual request

### Audit Dimensions

1. **Consistency** — Does completed work deviate from MISSION.md?
2. **Completeness** — Are there half-done or skip-verification parts?
3. **Blind Spots** — Are there better alternative approaches?
4. **Debris** — Dead code, abandoned files, uncleaned plans?

### Coordinator Handling of Audit Report

```
PASS    → LOW issues → log to DECISIONS.md, MED → defer to next checkpoint
CONCERN → Assess HIGH/MED, dispatch fix tasks if possible, escalate if not
BLOCKED → Pause, assess alternatives, escalate to user if can't decide
         (Only the user can say "continue" to override a sentry BLOCKED)
```

### AUDIT.md Format

All sentry reports append to the current task folder's `AUDIT.md` (locate via `cat .tasks/ACTIVE_POINTER.md`):

```markdown
# Audit Log

## Audit #1 — 2026-06-15 14:30 — Triggered after subtask 3/8
STATUS: CONCERN

Issues:
- [MED] Caching used LRU but access pattern is sequential scan, LRU miss rate high
  → Adopted, switched to FIFO
- [LOW] Test coverage 67%, missed error paths
  → Logged, defer to next checkpoint

Coordinator response: Adopted LRU→FIFO, LOW deferred to next checkpoint
```

## Layer 6: Tool Awareness

### Mechanism

During dispatch strategy, the coordinator actively evaluates locally available tools:

```
"What tools does this task need? Which available tool fits?"
"Does this need browser UI verification? → Consider Chrome"
"Need to look at code structure? → codegraph or code"
"Is this tool in PATH? → command -v to confirm"
"Does the sub-agent need this tool? → specify permissions in prompt"
```

### Rules

- Prefer existing tools — don't use something "just because"
- If installing a new tool with winget, must log in DECISIONS.md
- Same tool fails ≥ 2 times → switch to alternative
- Browsers only for UI/functional verification, not intrusive operations
- When a sub-agent needs a specific tool, the coordinator must specify "you have X tool available" in the prompt
- Sub-agent prompts must include: "if the required tool isn't in the list, report COORD instead of trying to install it"

### Getting Available Tools

During SETUP, detect available tools and write them to MISSION.md's Available Tools section. Uses `command -v` (POSIX-compatible, cross-platform — Linux/macOS/Windows Git Bash/WSL):

```bash
# Core tools — cross-platform (Linux/macOS/Windows Git Bash/WSL)
for tool in node npm python3 python pip gh docker git; do
  if command -v "$tool" 2>/dev/null; then echo "  [found] $tool"; else echo "  [missing] $tool"; fi
done
# IDE tools — optional, may not be in PATH
for tool in code cursor; do
  command -v "$tool" 2>/dev/null && echo "  [found] $tool (IDE)" || true
done
```

## Complete Execution Flow

```
=== SETUP ===
1. Complex task auto-detected (passive trigger) — tell user "检测到复杂任务，自动激活长任务模式（sustained-development）。" then proceed directly. User can also manually invoke via "/sustained" or "开始长任务"
2. Coordinator confirms plan is ready
   - Plan source: EnterPlanMode, writing-plans, brainstorming, or user verbal description
   - If no plan yet → go through design phase first
3. Coordinator analyzes plan, estimates subtask count
4. Run tool detection (command -v)
5. Check for orphan: `.tasks/MISSION.md` in root (not inside any numbered folder). If found → archive to `.tasks/paused/orphan-<timestamp>/`, delete original. **Do not proceed until archived.**
6. Create task folder:
   - Scan `.tasks/active/`, `.tasks/completed/`, `.tasks/paused/` for all `{NNN}_*` folders, take max NNN +1, zero-pad
   - Extract feature-slug from plan or user description (English kebab-case)
   - Create folder: `.tasks/active/{NNN+1}_{slug}/`
   - **Never overwrite** any existing folder
7. Write `MISSION.md` inside the new folder (title: `# {NNN}: <feature-name>`)
8. User approval → freeze
9. Write `.tasks/ACTIVE_POINTER.md`:
   - Content: `active/{NNN+1}_{slug}/` (single relative path line)
   - No more full copy of MISSION.md
10. Check `.claude/plans/` for plan files older than 7 days. If found → list for user to confirm deletion. If none or all within 7 days → skip.

=== EXECUTION LOOP ===
Each subtask dispatch:
  [STRATEGY] Examine dependencies, tools, dispatch mode
  [DISPATCH] Compose prompt with Forced Escalation template → dispatch
  [PARSE]   Parse STATUS
  [ACT]     Respond per STATUS rules
  [LOG]     Update STATE.md + DECISIONS.md
  [BRIEF]   Write BRIEF.md (with drift self-check: acceptance criteria? ← deviation recorded? ← completion or branch? ← return path marked?)
            → Any "no" → pause, execute drift response rules
  [DRIFT]   Read last 2-3 BRIEF.md entries → if unresolved drift → pause
  [RESET]   Prune local memory (keep only steps/progress/pointers)
            → Read Executive Pass self-refresh anchor section (~10 lines)
            → One-line check: "Is what I just did in the Allowed list?"

Reached checkpoint?
  ├─ Yes → [RECENTER] Read MISSION.md + update STATE.md
  │        [SENTRY] Dispatch sentry
  │        [REACT] Process audit report → update AUDIT.md
  │        → Continue
  └─ No  → Continue

=== COMPLETION ===
1. Final sentry audit
2. Process all HIGH/MED items from audit (fix or log)
3. Archive mission: **move entire folder** `active/{NNN}_{slug}/` → `completed/{NNN}_{slug}/`. Write mission-completion entry in DECISIONS.md (with acceptance criteria status). No per-file copy.
4. Update `ACTIVE_POINTER.md`: If there's a pending mission in `paused/` → point to it. If none → write `all missions completed`.
5. STATE.md is archived with the folder. New mission starts with a fresh STATE.md.
6. **[Optional] Email notification (only if env vars configured):**
   - Check `SMTP_SERVER`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `NOTIFY_EMAIL` all set
   - If all set → send standard SMTP notification
     - Subject: `[Mission Complete] <mission-name>`
     - Body: mission name, completion time, acceptance criteria status, key decisions summary
     - **Retry up to 3 times on failure**, if still failed, inform user of the reason.
   - If not configured → **skip email step**, does not block COMPLETION
7. Check `.claude/plans/` for plans older than 7 days → list for user to confirm deletion
```

## Relationship to Other Superpower Skills

This skill doesn't replace existing skills — it's a **top-level orchestrator** that coordinates and combines them:

| Existing Skill | Role in This Flow |
|---------------|-------------------|
| EnterPlanMode | Upstream design phase. Produce spec and plan first, then execute via this flow |
| writing-plans | Plan source (alternative to EnterPlanMode) |
| subagent-driven-development | Core execution layer, dispatches subtasks |
| dispatching-parallel-agents | Used when tasks can run in parallel |
| executing-plans | Very lightweight single-step tasks |
| verification-before-completion | Internalized as the "Verification" field in sub-agent returns |

**Note:** Plan sources are unlimited. EnterPlanMode, writing-plans, brainstorming, or user-provided plans all work. This flow only cares about "now that we have a plan, how to execute without losing focus."

## Boundary Conditions

- User instructions > MISSION.md > this skill's rules
- Task ≤ 1 step → skip this flow, do it directly
- Sentry and coordinator repeatedly disagree (2 consecutive CONCERN) → escalate to user for judgment
- Sub-agents must not attempt to install tools not in the list, must report COORD
