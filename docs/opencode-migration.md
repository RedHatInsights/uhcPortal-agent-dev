# OpenCode V2 Migration — uhcportal-agent-dev

Migration and rollback record for the `uhcportal-agent-dev` dev-bot instance
(REHOR-188). For the full migration contract — runtime architecture,
coordinator boundary, provider allowlist policy, and image acceptance gates —
see `dev-bot/docs/migrations/opencode-migration.md` in the `dev-bot` submodule.

## Prior state

- `dev-bot` gitlink: `e556ee4` (pre-OpenCode coordinator).
- `BOT_EXECUTION_ENGINE` defaulted to `python` (Claude SDK runner).
- Runtime `claude`, provider `vertex`, default Claude model via
  `VERTEX_ALLOWED_MODELS`.
- Profile at `instance/ocmui-config` (`BOT_CONFIG_PATH`): `CLAUDE.md`
  instructions, one persona directory (`frontend`), and `.claude/skills`-
  compatible `skills/` (`.gitkeep` only).

## New state

- `dev-bot` gitlink: `f196c4a` (OpenShift-Fleet/rehor master; includes REHOR-200
  OpenCode native V2, REHOR-211 per-cycle USD cost budget, and REHOR-212 terra
  exposure/pricing registry).
- `BOT_EXECUTION_ENGINE: coordinator` — set via GitOps in app-interface
  (`data/services/insights/platform-frontend-ai-dev/uhcportal-agent-dev-deploy.yml`);
  the template default remains `python` for rollback.
- Runtime `opencode`, provider `rehor-openai`, basic model `gpt-6-luna`
  (worker subagents, high reasoning effort), Sol `gpt-6.1-sol` for the
  coordinator primary agent (medium effort).
- Model access through the shared proxy's OpenAI-compatible gateway:
  `REHOR_MODEL_PROXY_URL: http://devbot-proxy:8450/v1`.
  `OPENAI_ALLOWED_MODELS` on the shared proxy already includes both
  `gpt-6.1-sol` and `gpt-6-luna`.
- Managed egress (SandboxTemplate networkPolicy, see below) allows TCP/8450 to
  `devbot-proxy`.
- Workflow stays `jira-sprint` (OCM UI Board, sprint-prefix/backlog pickup).

## Deployment shape

The workload **keeps the classic `Deployment` shape** — unlike
`hcc-orchestrator-agent-dev` (REHOR-187), this instance does not migrate to
OpenShell SandboxTemplate/SandboxWarmPool. The template changes are
runtime-only:

- `deploy/template.yaml` stays a classic `Deployment` + `${BOT_NAME}-egress`
  NetworkPolicy + KEDA cron `ScaledObject` targeting the Deployment
  (America/New_York, 09:00–18:00 weekdays), image referenced by
  `${BOT_IMAGE}:${IMAGE_TAG}` as before.
- The egress NetworkPolicy adds **TCP/8450** to `devbot-proxy` — the shared
  proxy's OpenAI-compatible model gateway used by the OpenCode coordinator
  runtime (`REHOR_MODEL_PROXY_URL: http://devbot-proxy:8450/v1`).
- New template parameters: `BOT_EXECUTION_ENGINE` (default `python` for
  rollback), `REHOR_OPENCODE_DEPLOYMENT_CONFIG`,
  `REHOR_OPENCODE_MAX_TOOL_CALLS`, `REHOR_MODEL_PROXY_URL`,
  `REHOR_MODEL_PROXY_TOKEN`, `REHOR_WORKSPACE_OWNER_UID`.
- New container env: the parameters above plus `CYCLE_RUNS_API_URL` (cycle
  record verification). The jira-sprint `BOT_SPRINT_PREFIX` /
  `BOT_INCLUDE_BACKLOG` env vars are unchanged.
- The Claude/Vertex runtime env (`CLAUDE_CODE_SKIP_VERTEX_AUTH`,
  `ANTHROPIC_VERTEX_*`, `CLOUD_ML_REGION`) is removed; the workload no longer
  consumes `GCP_*`/`VERTEX_ALLOWED_MODELS` on the OpenCode path.

## File conversion map

| Prior (Claude) | New (OpenCode) | Notes |
|---|---|---|
| `agent/personas/frontend/prompt.md` | `agent/.opencode/agents/frontend.md` | Full prompt body carried over; the persona `mcp.json` was empty and is dropped. |
| — | `agent/.opencode/agents/orchestrator-coordinator.md` | New primary agent; delegates implementation to the `frontend` subagent. |
| `agent/CLAUDE.md` | `agent/AGENTS.md` | All sections carried over (tool-call narration, Target Repos, Detected Tech Stacks, team conventions) plus an OpenCode section. `CLAUDE.md` removed — superseded by the OpenCode config; git history preserves it. |
| — | `agent/opencode.json` | `default_agent: orchestrator-coordinator`, `experimental.subagent_depth: 1`. |
| — | `opencode-deployment.json` | Sibling of `agent/`; `rehor-openai` provider (OpenAI Responses package) with `gpt-6.1-sol` (`reasoningEffort: medium`) and `gpt-6-luna` (`reasoningEffort: high` — bridges the gap to Sol for basic work). Wired via `REHOR_OPENCODE_DEPLOYMENT_CONFIG`. |
| `agent/skills/` | unchanged | `.gitkeep` placeholder only; OpenCode keeps `.claude/skills` compatibility. |
| `agent/mcp.json` | unchanged | `mcp-atlassian` via `${JIRA_MCP_URL}`. |
| `agent/project-repos.json` | unchanged | Target repo list. |

`instance.yaml` selectors added (`runtime: opencode`, `provider: rehor-openai`,
`model: gpt-6-luna`). Model policy: **Luna is the basic model** (worker
subagents, high reasoning effort to bridge the gap); **Sol is for the more
challenging coordination lane** (orchestrator-coordinator primary agent,
medium effort). `claude_md.strategy` and the `CLAUDE.md`/Vertex runtime env
(`CLAUDE_CODE_SKIP_VERTEX_AUTH`, `ANTHROPIC_*`, `CLOUD_ML_REGION`) were removed
as Claude leftovers superseded by the OpenCode configuration.

## Rollback procedure

1. Redeploy the previous image tag and config revision.
2. Set `BOT_EXECUTION_ENGINE=python` (GitOps parameter).
3. Revert this migration commit in the instance repository: it restores the
   `instance.yaml` Claude defaults (`runtime: claude`, `provider: vertex`,
   `claude_md` strategy), `agent/CLAUDE.md`, the `frontend` persona directory,
   the Vertex runtime env, the previous egress port list (without 8450), and
   the previous `dev-bot` gitlink (`e556ee4`) — the full pre-OpenCode profile.
   The Deployment shape itself is unchanged by this migration.

Note: changing the provider alone is **not** a runtime rollback. The Claude
runtime requires `BOT_EXECUTION_ENGINE=python` plus `runtime: claude` /
`provider: vertex`. `GCP_*`/`VERTEX_ALLOWED_MODELS` remain as template
parameters only because the app-interface caller still supplies them
(deprecated contract); the workload no longer consumes them on the OpenCode
path.
