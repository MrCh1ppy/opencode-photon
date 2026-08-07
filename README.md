# opencode-photon

A lightweight orchestration control layer for OpenCode that separates runtime state management from orchestration intelligence.

## v0: Config-Only Prototype

This version is **pure configuration** — no plugin code. It uses OpenCode's native agent system to validate the core hypothesis:

> Can a cheap Dispatcher + stateless Judge replace an expensive stateful Orchestrator?

## Architecture

```
User
  ↓
Dispatcher (primary) — receives input, routes to Judge
  ↓ task
Judge (subagent) — analyzes state, outputs structured decisions
  ↓ task
Specialists — bounded execution
  ├── low-fixer     (simple, single-file)
  ├── medium-fixer  (multi-file, clear scope)
  ├── deep-fixer    (complex, architectural)
  ├── explorer      (codebase search)
  └── oracle        (hard problems)
  ↓ result
Dispatcher
  ↓ task
Judge — next decision
  ↓
... loop until COMPLETE
```

## Agents

| Agent | Mode | Role | Permissions |
|-------|------|------|-------------|
| `dispatcher` | primary | User-facing runtime, routes to Judge | task:judge only |
| `judge` | subagent | Orchestration brain, outputs JSON decisions | none |
| `low-fixer` | subagent | Simple, single-file changes | edit only |
| `medium-fixer` | subagent | Multi-file, standard patterns | edit, bash |
| `deep-fixer` | subagent | Complex, architectural changes | edit, bash |
| `explorer` | subagent | Fast codebase search | read-only |
| `oracle` | subagent | Hard problem solving | read-only |

## Fixer Tier Selection

Judge selects the **lowest sufficient tier** to minimize cost:

| Tier | Scope | Examples | Cost |
|------|-------|----------|------|
| `low-fixer` | 1 file, <20 lines | Typo, config value, simple rename | Lowest |
| `medium-fixer` | 2-5 files, clear pattern | Feature impl, test updates, moderate refactor | Medium |
| `deep-fixer` | Architectural, cross-system | Migration, API redesign, performance | Highest |

Escalation: if a fixer fails or reports complexity beyond its tier, Judge retries with next tier up.

## Usage

```bash
# Clone and enter directory
git clone git@github.com:MrCh1ppy/opencode-photon.git
cd opencode-photon

# Start opencode — default_agent is dispatcher
opencode
```

Then describe your task. The Dispatcher will route through Judge automatically.

## What v0 Validates

- Judge can reliably select next action and appropriate fixer tier
- Permission isolation works via config
- Delegation prompt quality is maintained
- Judge output protocol is stable
- Cost optimization through tiered fixer dispatch

## What v0 Does NOT Have

- Deterministic state storage (uses prompt/todo only)
- State patch validation
- Event-gated Judge triggering
- Cost tracking
- Compaction hooks

These require the plugin runtime in v1+.

## Judge Output Schema

```json
{
  "assessment": "Brief state analysis",
  "decision": "CONTINUE|DELEGATE|RETRY|REPLAN|ROLLBACK|WAIT|VERIFY|ASK_ORACLE|STOP|COMPLETE",
  "actions": [{"type": "delegate", "agent": "low-fixer|medium-fixer|deep-fixer|explorer|oracle", "prompt": "..."}],
  "state_patch": {
    "facts_add": [],
    "decisions_add": [],
    "obsolete_add": [],
    "failures_add": []
  },
  "user_message": "optional"
}
```

## License

MIT
