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
Specialists (fixer / explorer / oracle) — bounded execution
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
| `fixer` | subagent | Bounded code changes | edit, bash |
| `explorer` | subagent | Fast codebase search | read-only |
| `oracle` | subagent | Hard problem solving | read-only |

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

- Judge can reliably select next action
- Permission isolation works via config
- Delegation prompt quality is maintained
- Judge output protocol is stable

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
  "actions": [{"type": "delegate", "agent": "fixer|explorer|oracle", "prompt": "..."}],
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
