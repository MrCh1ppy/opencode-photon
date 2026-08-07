# opencode-photon

A lightweight orchestration control layer for OpenCode that separates runtime state management from orchestration intelligence.

## v0: Config-Only Prototype

This version is **pure configuration** — no plugin code. It uses OpenCode's native agent system to validate the core hypothesis:

> Can a cheap Dispatcher + stateless Judge replace an expensive stateful Orchestrator?

## Architecture

```
User
  ↓
Dispatcher (primary) — receives input, always consults Judge first
  ↓ task
Judge (subagent) — analyzes state, outputs structured JSON decisions
  ↓ task (via Dispatcher)
Specialists — bounded execution
  ├── explorer      (codebase search — first when context unknown)
  ├── oracle        (architecture/ hard problems — before deep work)
  ├── low-fixer     (mechanical, low-risk, reversible)
  ├── medium-fixer  (bounded scope, known patterns)
  └── deep-fixer    (approved architecture, cross-system)
  ↓ JSON result
Dispatcher
  ↓ task
Judge — next decision
  ↓
... loop until COMPLETE
```

## Agents

| Agent | Mode | Role | Permissions |
|-------|------|------|-------------|
| `dispatcher` | primary | User-facing runtime, executes Judge decisions | task only |
| `judge` | subagent | Orchestration brain, outputs JSON decisions | none |
| `explorer` | subagent | Fast codebase search | read-only |
| `oracle` | subagent | Architecture decisions, hard problems | read-only |
| `low-fixer` | subagent | Mechanical, low-risk changes | edit only |
| `medium-fixer` | subagent | Bounded implementation, known patterns | edit, bash |
| `deep-fixer` | subagent | Approved architecture implementation | edit, bash |

## Routing Gates (Judge)

Specialists are selected by **risk, uncertainty, and blast radius** — never by file/line count:

1. **Location unknown?** → `explorer`
2. **Approach unclear?** (architecture, root cause, security/data) → `oracle`
3. **Risk assessment:**
   - Low risk, reversible, exact instructions → `low-fixer`
   - Bounded scope, known pattern → `medium-fixer`
   - High risk, cross-system, irreversible → `deep-fixer`

**Escalation after failure:**
- Transient/tool error → retry same tier (max 2 retries)
- Scope underestimated → escalate one tier
- Wrong approach → `oracle` for redesign
- Security, migrations, public APIs, data integrity → always high-risk

## Specialist Output Contract

All specialists return structured JSON:

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

## Judge Output Schema

```json
{
  "assessment": "Brief state analysis",
  "decision": "CONTINUE|DELEGATE|RETRY|REPLAN|ROLLBACK|WAIT|VERIFY|ASK_ORACLE|STOP|COMPLETE",
  "actions": [{"type": "delegate", "agent": "...", "prompt": "..."}],
  "state_patch": {
    "facts_add": [],
    "decisions_add": [],
    "obsolete_add": [],
    "failures_add": [],
    "retry_count": 0
  },
  "user_message": "optional"
}
```

## Usage

```bash
git clone git@github.com:MrCh1ppy/opencode-photon.git
cd opencode-photon
opencode
```

`default_agent` is `dispatcher` — describe your task and routing happens automatically.

## What v0 Validates

- Judge can reliably route via explicit gates
- Tiered fixer dispatch reduces cost vs. one-size-fits-all
- JSON output contracts are stable enough for orchestration
- Permission isolation works via config

## Known Limitations (v1+ plugin runtime)

- "Judge-first" sequencing is prompt-enforced, not permission-enforced — Dispatcher has task access to all specialists, discipline comes from its prompt
- No persistent canonical state (prompt/todo only)
- No state patch validation
- No cost tracking or event-gated Judge triggering
- `edit: allow` cannot enforce file-count scope; bash is unrestricted within each agent
- No model/cost differentiation between tiers yet (assign cheaper models per tier in `opencode.json`)

## License

MIT
