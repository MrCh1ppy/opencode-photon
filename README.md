# opencode-photon

A lightweight orchestration control layer for OpenCode that separates runtime state management from orchestration intelligence.

## v0: Config-Only Prototype

This version is **pure configuration** — no plugin code. It uses OpenCode's native agent system to validate the core hypothesis:

> Can a cheap Dispatcher gateway + session-reused Judge replace an expensive general-purpose Orchestrator?

## Architecture

```
User
  ↓
Dispatcher (primary) — permission-enforced gateway, can ONLY call judge
  ↓ task (exactly once per user request)
Judge (subagent) — orchestration brain, owns the delegation loop
  ↓ task (directly, results return to Judge)
Specialists — bounded execution
  ├── explorer      (codebase search — first when location unknown)
  ├── oracle        (architecture/hard problems — before deep work)
  ├── low-fixer     (mechanical, low-risk, reversible)
  ├── medium-fixer  (bounded scope, known patterns)
  └── deep-fixer    (approved architecture, cross-system)
  ↓
Judge reconciles results → terminal JSON → Dispatcher relays user_message
```

**Key property**: Dispatcher has task permission only for `judge`. Judge-first sequencing is **permission-enforced**, not prompt discipline.

## Agents

| Agent | Mode | Role | Permissions |
|-------|------|------|-------------|
| `dispatcher` | primary | Gateway: receive input, call judge once, relay report | task:judge only |
| `judge` | subagent | Orchestration brain, owns delegation loop | task: all specialists |
| `explorer` | subagent | Fast codebase search | read-only |
| `oracle` | subagent | Architecture decisions, hard problems | read-only |
| `low-fixer` | subagent | Mechanical, low-risk changes | edit only |
| `medium-fixer` | subagent | Bounded implementation, known patterns | edit, bash |
| `deep-fixer` | subagent | Approved architecture implementation | edit, bash |

## Judge Session Model

Within one task lifecycle, **one Judge session is reused** across the whole delegation loop:

- Specialist results return directly to Judge — no re-serialization of state
- Judge maintains an internal **evidence ledger**: goal, facts (with provenance), hypotheses (separate from facts), decisions (with superseded IDs), files touched, validation evidence, failures, open questions, plan
- Every terminal response includes a compact `resume_state` — the fallback when native session continuation is unavailable across user turns

## Routing Gates (Judge)

Specialists selected by **risk, uncertainty, and blast radius** — never by file/line count:

1. **Location unknown?** → `explorer`
2. **Approach unclear?** (architecture, root cause, security/data, conflicting evidence) → `oracle`
3. **Risk assessment:**
   - Low risk, reversible, exact instructions → `low-fixer`
   - Bounded scope, known pattern → `medium-fixer`
   - High risk, cross-system, irreversible → `deep-fixer`

**Budget limits**: max 2 retries per delegation, max 8 total specialist calls, max 3 mutating calls without user confirmation. Mutating specialists are serialized.

## Oracle vs Judge

Judge coordinates; Oracle solves. Oracle is **mandatory** for irreversible decisions, security/data integrity, conflicting evidence, and unclear root cause — and **forbidden** for ordinary tier selection or mechanical failures. Default: one Oracle call per unresolved issue.

## Contracts

### Judge Terminal Response (to Dispatcher)

```json
{
  "status": "complete|partial|needs_input|blocked|stopped",
  "user_message": "final user-facing report or question",
  "changes": [],
  "validation": [],
  "risks": [],
  "execution_summary": {"delegations": 0, "retries": 0, "oracle_calls": 0},
  "resume_state": {}
}
```

### Specialist Result (to Judge)

```json
{
  "status": "success|partial|failed|escalate|too-complex|needs-oracle",
  "files_touched": [],
  "changes_made": "...",
  "scope_violation": false,
  "commands_run": [],
  "validation": "...",
  "blocker_kind": null,
  "recommended_next_action": "accept|retry|escalate|ask-oracle|rollback"
}
```

## Usage

```bash
git clone git@github.com:MrCh1ppy/opencode-photon.git
cd opencode-photon
opencode
```

`default_agent` is `dispatcher` — describe your task; Judge orchestrates automatically.

## What v0 Validates

- Permission-enforced judge-first sequencing works via config alone
- Judge can own the full delegation loop with session reuse
- Risk-based tier routing reduces cost vs. one-size-fits-all
- JSON contracts are stable enough for machine-checked orchestration

## Known Limitations (v1+ plugin runtime)

- Judge session reuse grows context monotonically within a task — no canonical state store yet
- `resume_state` handoff across user turns depends on platform subagent-resume semantics (unverified)
- No independent verification of specialist self-reported results (Judge has no read/bash)
- `edit: allow` / `bash: allow` cannot enforce file-count or command scope — prompt-level only
- No model/cost differentiation between tiers yet (assign per-tier models in `opencode.json`)
- No cost tracking or event-gated Judge triggering

## License

MIT
