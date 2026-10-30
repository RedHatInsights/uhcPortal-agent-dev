# uhcportal-agent-dev — Additional Instructions

OpenCode-native profile instructions (the legacy `CLAUDE.md` was removed with the
Claude runtime; git history preserves it for rollback).

## Tool-call narration

Keep the runner's Ultra Caveman output policy. Apply these additional rules across
all subagents and skills:

- Call tools directly. After a result, make the next tool call without commentary.
  Do not emit text solely to announce, acknowledge, or describe routine work.
- No tool-call preambles, transition comments ("looks good — now on to this"),
  self-congratulation, repeated plans, or paraphrases of tool results.
- Emit task prose only for necessary clarification, actionable blockers, security
  warnings, irreversible-action confirmation, or a concise final outcome.
- Continue required status tool calls, progress checkpoints, and structured
  metadata. Do not duplicate those updates in narration.
- Keep human-facing deliverables in normal prose with all required formats and
  evidence: Jira comments, PR/MR descriptions and review replies, commits,
  documentation, and code comments.

## Target Repos
- **uhc-portal**: `https://github.com/RedHatInsights/uhc-portal.git`

## Detected Tech Stacks
- **uhc-portal**: envs=[browser, node, patternfly-mcp], personas=[frontend]

## Team Conventions

<!-- Fill in team-specific conventions after scaffolding -->
<!-- Examples: version managers, test commands, PR review norms -->

## OpenCode

This profile uses native subagents in `.opencode/agents/` instead of a persona
directory per role; the persona prompt bodies now live there. The shared workflow
guidance may still refer to `personas/<name>/prompt.md`; map those role names to
the matching native subagents and do not search for the deleted persona files.

OpenCode V2 does not expose the legacy LSP tool. Use each repository's
typecheck, lint, and test commands instead.
