# Orchestration Contract: Readable Markdown Handoff

This is the authoritative transport contract for the global opencode orchestration layer. Agent ids are filenames. The custom reconnaissance role is `explorer`; there is no `explore` alias. Readable Markdown Handoff is intentionally lightweight and non-strict: AI consumers should understand the declared headings and labels without requiring a program parser.

## Common Specialist Skeleton

Every specialist response uses these headings in this order:

1. `## Status`
2. `## Summary`
3. Role-specific sections, when applicable
4. `## Evidence`
5. `## Files`
6. `## Validation`
7. `## Blocker`
8. `## Next Action`

The Status section contains `status: success|partial|failed|escalated`. Summary and Validation are readable Markdown strings. Evidence is a bullet list, using `- None` when empty. Files are bullets in the form ``- `path`: description`` or `- None` when empty. Blocker contains `blocker_kind` and a readable explanation. Next Action contains `recommended_next_action`, `escalation_target`, and a readable recommendation.

## Field Types And Enums

| Field | Type | Readable representation |
| --- | --- | --- |
| `changes_made` | string-array | `## Changes Made`, with `- path: change` bullets |
| `commands_run` | string-array | `## Commands Run`, with `- command: result` bullets or `- None` |
| `facts` | string-array | `## Facts`, with fact bullets |
| `evidence` | string-array | `## Evidence`, with evidence bullets |
| `files` | file-array | Path-and-description bullets such as `- path/to/file: description` |
| `options` | object-array | `### Option N`, followed by `Approach:` and `Trade-offs:` |
| `next_steps` | string-array | `## Next Steps`, with action bullets |
| `decisions` | object-array | Decision headings with `text` and `continuity` labels |
| `root_cause` / `recommendation` / `risks` | markdown-string | Readable Markdown scalar sections |
| `summary` / `validation` | markdown-string | Readable Markdown scalar sections |
| `agent` / `status` / `blocker_kind` / `recommended_next_action` / `escalation_target` | scalar-enum or scalar-string | One labeled line each |

Specialist `status` is `success|partial|failed|escalated`. `recommended_next_action` is `accept|retry|escalate`. `blocker_kind` is `none|user_input_required|environment|dependency|test_failure|data_integrity|security|unknown`. `escalation_target` is `judge|oracle|low-fixer|medium-fixer|deep-fixer|user|none`.

Semantic expectations are: success normally pairs with accept and blocker_kind none; failed requires a meaningful blocker and normally retry; escalated requires recommended_next_action escalate and a real escalation target. Missing or ambiguous critical fields are a handoff problem, not an invitation to invent values.

## Role-Specific Sections

- explorer adds `## Facts` between Summary and Evidence.
- oracle adds `## Root Cause`, `## Options`, `## Recommendation`, `## Next Steps`, and `## Risks`; each option uses `### Option N`, `Approach:`, and `Trade-offs:`.
- low-fixer and medium-fixer add `## Changes Made` and `## Commands Run`.
- deep-fixer adds `## Changes Made`, `## Approach`, `## Commands Run`, and `## Rollback Plan`.

Files use ``- `path`: description`` bullets. Options use a numbered heading plus Approach and Trade-offs labels. Decisions use a numbered Decision heading with text and continuity labels.

## Judge Handoff

Judge uses this response skeleton: `## Status` with `terminal_status: complete|partial|needs_input|blocked|stopped` and `status: success`; then `## Summary`, `## User Message`, `## Changes`, `## Validation`, `## Risks`, `## Execution Summary`, and `## Resume State`.

Execution Summary lists each specialist as `completed` or `not run`. Resume State requires Goal, Facts, Decisions, Files Touched, Open Questions, and Plan. Decisions use:

```markdown
#### Decision D1
- text: Decision text.
- continuity: Initial decision
```

New continuity is `Initial decision` or `Continued from: Dn`. A revision uses `Revised because: reason`. Never delete an old decision. If the predecessor cannot be confirmed, use `Unknown predecessor; requires review`.

## Dispatcher Consumption Rules

1. Treat the response as readable handoff text, not a strict serialization format.
2. Locate the declared headings and labeled enum values before interpreting prose.
3. Use the exact enum tokens when present; do not silently rewrite spelling or casing.
4. Read Status, Blocker, Next Action, Summary, Evidence, Files, and Validation first.
5. For Judge, additionally read User Message and Resume State before routing or terminating.
6. Missing or ambiguous critical fields become validation-failure or require clarification; do not invent terminal states.
7. Translate specialist outcomes as `success + accept -> complete`, `partial -> partial`, `failed + retry -> stopped` or `needs_input`, `escalated -> blocked`, and missing input -> `needs_input`.
8. Preserve facts, files, validation, risks, and resume state when handing off to Judge.
9. Keep the final user-facing report in Simplified Chinese with `结论`, `证据摘要`, `验证结果`, `风险`, `当前状态`, and `下一步`.
10. Reuse prior `task_id` for Judge and specialists when available; if reuse fails, retain resume state and report the new-session fallback.

## Compatibility

Legacy bare JSON, fenced JSON, and MD-Envelope v1 responses may remain understandable during migration. Consumers should make a best-effort reading, prefer Readable Markdown Handoff, and record ambiguity rather than claiming strict parser validation. This is model-level compatibility guidance, not a program-level three-phase parser or guarantee.

## Reporting

Internal prompts may be English, but the final user-facing report must be in Simplified Chinese using the fixed fields above.
