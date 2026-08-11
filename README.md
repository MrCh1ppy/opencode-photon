# opencode-photon

A configuration-only orchestration layer for OpenCode. The active architecture uses a primary Dispatcher, an advisory Judge, and bounded specialist agents.

## Architecture

```text
User
  |
  v
Dispatcher (primary)
  |
  +-- Judge (advisory coordinator)
  |
  +-- explorer -> Explorer
  +-- oracle -> Oracle
  +-- low-fixer
  +-- medium-fixer
  +-- deep-fixer
```

The Dispatcher is the only user-facing agent. It has `edit: allow` and `bash: allow`. Its `task` permission starts with the broad rule `"*": deny`, followed by specific allows for `judge`, `explorer`, `oracle`, `low-fixer`, `medium-fixer`, and `deep-fixer`. This ordering keeps delegation restricted to the approved workflow.

## Agents

| Agent | Mode | Model | Responsibility |
| --- | --- | --- | --- |
| `dispatcher` | primary | `deepseek/deepseek-v4-flash` | Receives requests, follows Judge routing, invokes specialists, and reports evidence. |
| `judge` | subagent | `openai/gpt-5.6-terra` | Advisory coordinator. It must not delegate, edit files, or run shell commands. |
| `explorer` | subagent | `deepseek/deepseek-v4-flash` | Read-only codebase search and pattern discovery. It is invoked through the `explorer` task route. |
| `oracle` | subagent | `openai/gpt-5.6-sol` | Read-only advisor for unclear architecture, root causes, security, or data-integrity concerns. |
| `low-fixer` | subagent | `deepseek/deepseek-v4-flash` | Executes simple, mechanical, low-risk changes. It has `bash: allow`. |
| `medium-fixer` | subagent | `openai/gpt-5.6-terra` | Executes bounded multi-file changes following approved patterns. |
| `deep-fixer` | subagent | `openai/gpt-5.6-sol` | Executes approved complex or cross-system implementation work. |

## Judge Loop

1. The Dispatcher calls Judge before specialist work and passes any existing `resume_state`.
2. Judge returns routing guidance, constraints, acceptance criteria, and stop conditions.
3. The Dispatcher calls one permitted specialist according to that guidance.
4. The Dispatcher summarizes affected files, changes, failures, and validation evidence to Judge.
5. Judge provides the next action. The Dispatcher preserves and passes `resume_state` on every loop turn.
6. The loop ends when Judge reports `complete`, `stopped`, or `blocked`, or when the user terminates it.

Judge is advisory and never delegates directly. The Dispatcher owns specialist invocation and result collection.

## Configuration

`photon.json` sets `dispatcher` as the default agent and injects `~/.config/opencode/language.md` through the `instructions` field. The language directive requires Simplified Chinese for user-facing communication while keeping code, identifiers, paths, and configuration content in their original form.

Provider credentials in `opencode.json` are read from environment variables through these references:

```json
{
  "apiKey": "{env:KIMI_FOR_CODING_API_KEY}"
}
```

```json
{
  "apiKey": "{env:OPENAI_API_KEY}"
}
```

Set the corresponding environment variables before using models from those providers. Never commit their values.

## Usage

```bash
cd opencode-photon
opencode
```

Describe the task to the Dispatcher. It routes the work through Judge and the appropriate risk-tiered specialist.

## Scope

This repository validates the orchestration workflow through native OpenCode configuration. Runtime behavior such as provider authentication and session continuation depends on the running OpenCode version and environment.

## License

MIT
