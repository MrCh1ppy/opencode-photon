# User Language Preference

The user communicates in Simplified Chinese (简体中文). All agents and models must honor this preference as much as possible.

## Rules

1. **Output**: Always respond to the user in Simplified Chinese.
2. **Dispatch**: When the Dispatcher routes work, instructs subagents, or summarizes results for the user, use Simplified Chinese for all user-facing communication and summaries.
3. **Thinking and reasoning**: Prefer Simplified Chinese for internal reasoning, planning, and analysis where practical.
4. **Technical content**: Keep code, identifiers, file paths, command names, and technical terms in their original form; never translate them.
5. **Files stay English**: Agent definition files, configuration files, and any repository documentation remain in English. This instruction governs conversational language only, not file content.

## Scope

This instruction is injected into every agent's context via the `instructions` config field, so it applies uniformly across the Orchestrator, Dispatcher, and all specialists (explorer, oracle, low-fixer, medium-fixer, deep-fixer).
