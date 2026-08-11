# opencode-photon

Photon v2 是 OpenCode 的纯配置编排层。当前架构中，Orchestrator 承担面向用户的决策角色，Dispatcher 承担有界执行角色，其下设有一组有界 Specialist 代理。

## 架构

```text
User <-> Orchestrator / K3
             |-- Oracle / GPT-5.6 Sol
             `-- Dispatcher / DS V4 Flash
                       |-- Explorer / DS V4 Flash
                       |-- Low Fixer / DS V4 Flash
                       |-- Medium Fixer / GPT-5.6 Terra
                       `-- Deep Fixer / GPT-5.6 Sol
```

总共有七个代理，不存在 `observer`。

- **Orchestrator** 是唯一的 primary agent，也是唯一直接与用户沟通的代理。它为 Dispatcher 定义有界执行节点，授权 Specialists，评估结果并报告最终结果。
- **Dispatcher** 是有界的应用层协调器。在获得授权的节点内，它可以组织多次 Specialist 调用、恢复已有的 Specialist 会话、收集证据、执行获授权的检查、重试可逆操作、跟进有前景的分支，以及从无成果的分支回退。只有在节点完成或继续执行将越过 Orchestrator 边界时，它才会返回。它绝不与用户沟通，也不修改源文件。Orchestrator 可以指定必须使用的确切 Specialist，此时 Dispatcher 必须使用该 Specialist，不得替换；也可以授权一个有界 Specialist 集合，由 Dispatcher 在其中进行战术选择。
- **Oracle** 仅由 Orchestrator 调用，用于提供困难架构问题和根因分析方面的建议。
- **Explorer 和三个 Fixers** 是由 Dispatcher 调用的 Specialists。升级路径始终是 `Specialist -> Dispatcher -> Orchestrator`。

`subagent_depth: 2` 允许调用链 `Orchestrator -> Dispatcher -> Specialist`，且不会更深。Dispatcher 可以在一个节点内多次调用或恢复已授权的 Specialists。只有在任务确实需要逐步控制时，Orchestrator 才会要求每次调用后都返回。

## 代理

| 代理 | 模式 | 模型 | 职责 |
| --- | --- | --- | --- |
| `orchestrator` | primary | `kimi-for-coding/k3` | 用户入口、决策、Specialist 选择、验收和长期记忆。`edit: deny`、`bash: deny`。 |
| `dispatcher` | subagent | `deepseek/deepseek-v4-flash` | 执行获授权的节点：调用或恢复已授权的 Specialists，压缩结果，并在检查点返回。`edit: deny`、`bash: allow`（绝不用于修改源文件）。 |
| `oracle` | subagent | `openai/gpt-5.6-sol` | 为不明确的架构、根因、安全性、兼容性或不可逆权衡提供高级建议。只读。 |
| `explorer` | subagent | `deepseek/deepseek-v4-flash` | 只读代码库侦察和证据收集。 |
| `low-fixer` | subagent | `deepseek/deepseek-v4-flash` | 简单、机械、低风险且可逆的变更。 |
| `medium-fixer` | subagent | `openai/gpt-5.6-terra` | 有界的多文件实现、标准重构和测试更新。 |
| `deep-fixer` | subagent | `openai/gpt-5.6-sol` | 对已批准架构进行复杂、高风险的实现，并提供 Rollback Plan。 |

## 路由循环

1. Orchestrator 解读用户请求并选择 Specialist 路径。
2. 它向 Dispatcher 下达一个节点指令，明确目标、范围、节点是否只读或允许变更、确切的 Specialist 或有界的授权集合、约束、验收标准和返回条件。
3. Dispatcher 通过调用或恢复已授权的 Specialists、收集证据，并在适当情况下重试或回退来执行节点；它在检查点或边界处返回。
4. Orchestrator 决定是继续、重试、切换 Specialist、恢复同一个 Dispatcher 会话、咨询 Oracle、询问用户还是停止。
5. 当 Orchestrator 向用户报告最终结果时，循环结束。

Dispatcher 会在获授权的节点内自主推进，不会仅因一次 Specialist 调用完成就返回。节点完成或到达 Orchestrator 边界时，它必须返回。一个根任务复用一个 Dispatcher 会话（`task_id`）。

### 必须返回的边界

出现以下任一情况时，Dispatcher 必须返回：

- 节点已完成；
- 调查已准备好转入实现；
- 需要变更但未获授权；
- 已批准的范围必须扩大；
- 实现方法必须发生实质性变化；
- 需要新的或未获授权的 Fixer；
- 需要作出涉及架构、安全性、数据完整性、兼容性、公共 API、迁移或不可逆操作的决策；
- 需要用户输入；
- 重要证据仍然相互冲突；
- 获授权的方法失败，需要采用其他方法；
- 执行或重试预算已耗尽；或
- Orchestrator 设置了明确边界。

## 委派粒度

委派给 `medium-fixer` 或 `deep-fixer` 时，Orchestrator 的节点指令向 Dispatcher 提供目标、边界、约束和验收标准，而不是逐步操作。这些 Fixers 使用的模型层级高于 Dispatcher，因此过于规定性的指令会不必要地限制其获批准的实现工作。只有 `low-fixer` 会收到详细的逐步指令。

变更授权必须明确。当某个 Fixer 层级或有界 Fixer 集合获授权后，Dispatcher 必须留在该授权范围内；不得升级到集合之外更强的 Fixer，如需其他层级，必须返回 Orchestrator。

## 权限矩阵

| 权限 | orchestrator | dispatcher | oracle | explorer | fixers |
| --- | --- | --- | --- | --- | --- |
| mode | primary | subagent | subagent | subagent | subagent |
| edit | deny | deny | deny | deny | allow |
| bash | deny | allow (never to modify source files) | deny | allow (read-only by rule) | allow |
| read/grep/glob/list/lsp | allow | allow | allow | allow | allow |
| external_directory | ask | ask | ask | ask | ask |
| task -> dispatcher | allow | deny | deny | deny | deny |
| task -> oracle | allow | deny | deny | deny | deny |
| task -> explorer | deny | allow | deny | deny | deny |
| task -> fixers | deny | allow | deny | deny | deny |

Fixers 是唯一可以修改源文件的代理。尽管 Dispatcher 和 Explorer 具有 `bash: allow`，但它们的提示词规则将 Bash 限制为只读或验证用途。它们绝不能使用 Bash 绕过 `edit: deny` 来修改源文件。

## 配置

`opencode.json` 将 `orchestrator` 设为默认代理，设置 `subagent_depth: 2`，并通过 `instructions` 字段注入 `~/.config/opencode/language.md`。语言指令要求面向用户的沟通使用简体中文，同时保持代码、标识符、路径和配置内容的原有形式。

`opencode.json` 中的提供商凭据通过 `{file:...}` 引用从本地密钥文件读取。例如：

```json
{
  "apiKey": "{file:~/.local/share/opencode/secrets/kimi.key}"
}
```

`opencode.json` 和 `tui.jsonc` 都加载 `@cortexkit/opencode-magic-context`（解析版本为 `0.36.0`）。该插件通过 Dreamer 提供跨会话记忆、上下文压缩和后台维护，并属于当前架构的一部分。

绝不要提交真实的密钥值。

## 使用方法

```bash
cd opencode-photon
opencode
```

向 Orchestrator 描述任务。它会通过 Dispatcher 将工作路由给适当的 Specialist。

## 适用范围

本仓库通过原生 OpenCode 配置验证编排工作流。运行时行为（包括提供商身份验证、会话继续和嵌套子代理调用）取决于正在运行的 OpenCode 版本和环境。

## 许可证

MIT
