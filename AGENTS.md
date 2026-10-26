# Agent instructions

Logseq DB self-hosted stack (sync server + web PWA) for Coolify, fully
parameterized — no hostnames, secrets, or Coolify UUIDs are baked in.

## Deploying / updating

For any deployment, update, or Coolify-related task, load and follow
`.agents/skills/deploy-logseq/SKILL.md` — a portable Agent Skill. It asks the
user for the instance parameters and runs the deployment step by step, using
the Coolify MCP when the harness has one connected, and the Coolify REST API
otherwise. It works in any agent harness that can ask questions, run shell
commands, and make HTTP calls.

Skill-aware harnesses also discover it directly: OpenCode and others read
`.agents/skills/`; Claude Code via the `.claude/skills` symlink.

## Rules

- Keep the stack parameterized: environment-specific values (domains, admin
  token, Cognito pool values, Coolify UUIDs) live only in `.env` (gitignored)
  or in the Coolify UI — never in compose files or docs.
- Hostname pattern is fixed: `sync.logseq.<BASE_DOMAIN>` and
  `web.logseq.<BASE_DOMAIN>`; only `BASE_DOMAIN` varies per instance.
- Read `README.md` ("How routing works") before touching Traefik labels:
  plain `Host(...)` rules only — no `priority=` or `PathPrefix()`; never use
  `SERVICE_FQDN_*` env vars for custom hostnames.
