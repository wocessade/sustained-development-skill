# Sustained Development — Attention Retention Workflow

A [Claude Code](https://claude.ai/code) skill for maintaining focus and consistency during long, multi-step development tasks.

**Problem:** Agent attention isn't about intelligence — it's about a single context window. The solution isn't a bigger window; it's making the filesystem a reliable second brain.

## Features

- **6-layer defense system** against context decay, task boundary drift, and verification slack
- **Cross-session continuity** — survive context compaction without losing progress
- **Forced Escalation** — sub-agents must report STATUS clearly (DONE/CONCERN/BLOCKED)
- **Sentry Audit** — independent sub-agent reviews direction at checkpoints
- **Coordinator Lite** — the coordinator orchestrates, sub-agents implement
- **Multi-platform** — works on Linux, macOS, and Windows (Git Bash / WSL)

## When to Use

| Use | Don't Use |
|-----|-----------|
| Tasks involving 4+ file changes or 2+ sub-tasks (auto-detected) | Single-file edits |
| Multi-step analysis with verification | Pure exploration ("no finish line") |
| Bug fixes where root cause is unknown | Quick fixes the user explicitly requested |
| Explicit trigger: "/sustained" or "long task" | |

## Quick Start

In Claude Code:

```
/sustained Add error retry and monitoring to the data pipeline
```

## Architecture

```
               ┌─────────────────────────┐
               │   SETUP (Mission Anchor) │
               │   - Analyze plan         │
               │   - Detect tools         │
               │   - Write MISSION.md     │
               └─────────┬───────────────┘
                         │
               ┌─────────▼───────────────┐
         ┌─────│   EXECUTION LOOP         │─────┐
         │     │   [STRATEGY] → [DISPATCH] │     │
         │     │   [PARSE]   → [ACT]      │     │
         │     │   [LOG]     → [BRIEF]    │     │
         │     │   [DRIFT]   → [RESET]    │     │
         │     └─────────┬───────────────┘     │
         │               │                     │
         │     ┌─────────▼───────────────┐     │
         │     │   CHECKPOINT?           │     │
         │     │   ├─ Yes → RECENTER     │     │
         │     │   │        + SENTRY     │     │
         │     │   └─ No  → continue     │     │
         │     └─────────────────────────┘     │
         └─────────────────────────────────────┘
                         │
               ┌─────────▼───────────────┐
               │   COMPLETION            │
               │   - Final sentry audit  │
               │   - Archive mission     │
               │   - Notify (optional)   │
               └─────────────────────────┘
```

## 6-Layer Defense

| Layer | Mechanism |
|-------|-----------|
| 1. Directory & Cleanup | `.tasks/` with per-task folders in `active/completed/paused`, `.claude/` periodic cleanup |
| 2. Mission Anchor | Externalized goals/constraints in numbered MISSION.md |
| 3. Coordinator Lite | Orchestrate only — no deep implementation work |
| 4. Forced Escalation | Hard-constrained sub-agent return format |
| 5. Sentry Audit | Independent sub-agent at checkpoints |
| 6. Tool Awareness | Auto-detect local tools, never install blindly |

## Project Structure

| Path | Purpose |
|------|---------|
| `SKILL.md` | Full workflow definition (Chinese, primary working language) |
| `manifest.yaml` | Declarative loading manifest |
| `README.md` | This file — English documentation |
| `implementer-prompt.md` | Sub-agent prompt template with forced escalation format |
| `sentry-prompt.md` | Audit agent prompt with 4 review dimensions |
| `test-scenario.md` | Pressure test scenario and expected criteria |

## Email Notification (Optional)

The COMPLETION step can send a notification email if configured. Set these environment variables:

```bash
export SMTP_SERVER=smtp.example.com
export SMTP_PORT=587
export SMTP_USER=your@email.com
export SMTP_PASS=your-app-password
export NOTIFY_EMAIL=your@email.com
```

If unset, the email step is skipped — it never blocks completion.

## License

MIT
