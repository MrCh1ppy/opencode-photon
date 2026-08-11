# opencode-photon

A configuration-only orchestration layer for OpenCode (Photon v2). The active architecture uses an Orchestrator as the user-facing decision authority and a Dispatcher as its single-step execution subagent, backed by bounded specialist agents.

## Architecture

```text
User <-> Orchestrator / K3
             |-- Oracle / GPT-5.6 Sol
             `-- Dispatcher / DS V4 Flash
                       |-- Explorer / DS V4 Flash
                       |-- Low Fixer / DS V4 Flash
                       |-- Medium Fixer / GPT-5.6 Terra
                       `-- Deep Fixer / GPT-5.6 Sol
```

Seven agents total; there is no `observer`.

- The **Orchestrator** is the only primary agent and the only agent that talks to the user. It decides the Specialist, specifies the exact Fixer, controls the Dispatcher one step at a time, accepts results, and reports back.
- The **Dispatcher** is an application-layer subagent. The Orchestrator pokes it one step at a time: it invokes exactly the specified Specialist, compresses the result, and returns. It never chooses Specialists, never talks to the user, and never edits files.
- **Oracle** is called only by the Orchestrator for hard architecture/root-cause advice.
- **Explorer and the three Fixers** are called only by the Dispatcher. The escalation path is always Specialist -> Dispatcher -> Orchestrator.

`subagent_depth: 2` allows Orchestrator -> Dispatcher -> Specialist; nothing deeper. Each Orchestrator instruction triggers at most one Specialist call ("one poke, one jump").

## Agents

| Agent | Mode | Model | Responsibility |
| --- | --- | --- | --- |
| `orchestrator` | primary | `kimi-for-coding/k3` | User entry, decisions, Specialist selection, acceptance, long-term memory. `edit: deny`, `bash: deny`. |
| `dispatcher` | subagent | `deepseek/deepseek-v4-flash` | Executes the Orchestrator-specified Specialist call, compresses results, returns. `edit: deny`, bash read/verify only. |
| `oracle` | subagent | `openai/gpt-5.6-sol` | Advanced advisor for unclear architecture, root causes, security, compatibility, or irreversible trade-offs. Read-only. |
| `explorer` | subagent | `deepseek/deepseek-v4-flash` | Read-only codebase reconnaissance and evidence collection. |
| `low-fixer` | subagent | `deepseek/deepseek-v4-flash` | Simple, mechanical, low-risk, reversible changes. |
| `medium-fixer` | subagent | `openai/gpt-5.6-terra` | Bounded multi-file implementation, standard refactoring, test updates. |
| `deep-fixer` | subagent | `openai/gpt-5.6-sol` | Complex, high-risk implementation of already-approved architecture (with Rollback Plan). |

## Routing Loop

1. The Orchestrator understands the user's request and selects the Specialist route.
2. It instructs the Dispatcher: goal, scope, the exact Specialist, constraints, and acceptance criteria.
3. The Dispatcher invokes that one Specialist, compresses the handoff, and returns.
4. The Orchestrator decides: continue, retry, switch Specialist, resume the same Dispatcher session, ask Oracle, ask the user, or stop.
5. The loop ends when the Orchestrator reports the final result to the user.

The Dispatcher never advances to the next phase on its own. When uncertain, it returns to the Orchestrator instead of deciding. One root task reuses one Dispatcher session (task_id).

## Permission Matrix

| Permission | orchestrator | dispatcher | oracle | explorer | fixers |
| --- | --- | --- | --- | --- | --- |
| mode | primary | subagent | subagent | subagent | subagent |
| edit | deny | deny | deny | deny | allow |
| bash | deny | read/verify only (unknown = ask) | deny | read-only whitelist | allow |
| read/grep/glob/list/lsp | allow | allow | allow | allow | allow |
| external_directory | ask | ask | ask | ask | ask |
| task -> dispatcher | allow | deny | deny | deny | deny |
| task -> oracle | allow | deny | deny | deny | deny |
| task -> explorer | deny | allow | deny | deny | deny |
| task -> fixers | deny | allow | deny | deny | deny |

Fixers are the only agents that modify source files. Dispatcher bash rules deliberately avoid `bash: allow` so Bash cannot bypass its `edit: deny`.

## Configuration

`opencode.json` sets `orchestrator` as the default agent, `subagent_depth: 2`, and injects `~/.config/opencode/language.md` through the `instructions` field. The language directive requires Simplified Chinese for user-facing communication while keeping code, identifiers, paths, and configuration content in their original form.

Provider credentials in `opencode.json` are read from local key files through `{file:...}` references, e.g.:

```json
{
  "apiKey": "{file:~/.local/share/opencode/secrets/kimi.key}"
}
```

Never commit real key values.

## Usage

```bash
cd opencode-photon
opencode
```

Describe the task to the Orchestrator. It routes the work directly to the appropriate specialist through the Dispatcher.

## Scope

This repository validates the orchestration workflow through native OpenCode configuration. Runtime behavior such as provider authentication, session continuation, and nested subagent calls depends on the running OpenCode version and environment.

## License

MIT
