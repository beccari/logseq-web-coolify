---
name: Deploy Logseq to Coolify
description: Deploy or update the Logseq DB stack (sync + web PWA) on any Coolify instance. Harness-agnostic — uses the Coolify MCP when available, otherwise the Coolify REST API. Asks the user for the instance parameters (base domain, sync admin token, Cognito values), validates the compose file locally, creates or updates the service, deploys, and verifies the URLs.
---

# Deploy Logseq (sync + web) to a Coolify instance

Drive a repeatable, parameterized deployment of this repository's stack
(`docker-compose.coolify.yml`). This skill is harness-agnostic: it only needs
the ability to ask the user questions, run shell commands, and make HTTP calls
(directly, or through a Coolify MCP server when one is connected). The compose
file has **nothing environment-specific baked in** — every value comes from
the parameters gathered here. Never assume a parameter: ask.

## Step 0 — Preflight

1. Locate `docker-compose.coolify.yml` in the repository root (the working
   directory). If missing, ask the user for its path and use it throughout.
2. Determine how to reach Coolify:
   - If this harness already has a working **Coolify MCP** connection (callable
     MCP tools for Coolify), use it — go to Step 2.
   - Otherwise ask the user: connect a Coolify MCP (Step 1) or proceed with the
     **REST API**, which needs `COOLIFY_HOST` (e.g. `app.example.com`) and
     `COOLIFY_API_TOKEN` in the environment. The REST path works in every
     harness with shell access.

## Step 1 — Connect the Coolify MCP (optional, only if the user wants it)

Ask the user for the Coolify instance URL (the MCP endpoint is
`<instance>/mcp`) and an API token. The token must be referenced through the
environment — never written literally into a config file.

Each harness configures remote MCP servers its own way; check the harness
docs. Shape to aim for: a remote/Streamable-HTTP server at `<instance>/mcp`,
OAuth disabled, header `Authorization: Bearer {env:COOLIFY_API_TOKEN}`
(env-substitution syntax varies per harness). After configuring, verify the
connection state before continuing. If connecting proves awkward, just use the
REST API instead — every later step supports both.

## Step 2 — Ask the user for the parameters

Ask for each missing parameter (use the harness's structured question/prompt
facility when it has one; batch independent questions into a single exchange).
Read `.env` in the repo first — if present, offer its values as prefilled
defaults, since they are the last deployment's answers.

| Parameter | Required | Notes |
|---|---|---|
| `BASE_DOMAIN` | yes | e.g. `example.com`. Hostnames are fixed: `sync.logseq.<BASE_DOMAIN>` and `web.logseq.<BASE_DOMAIN>`. |
| `DB_SYNC_ADMIN_TOKEN` | yes | Offer to generate with `openssl rand -hex 32`. |
| `COGNITO_ISSUER`, `COGNITO_CLIENT_ID`, `COGNITO_JWKS_URL` | yes | Logseq's official public Cognito pool. Prefill from `.env`; if unavailable, fetch the values from the upstream project docs (github.com/yshalsager/logseq-selfhost). |
| `SYNC_IMAGE` / `WEB_IMAGE` | no | Defaults: `ghcr.io/yshalsager/logseq-selfhost-{sync,web}:latest`. |
| Target | yes | Fresh deployment (new service) **or** update of an existing service (needs the Coolify service UUID). |
| Project / environment UUIDs | fresh only | Where to create the service; user picks them in Coolify or via the API. |

## Step 3 — Validate locally before touching Coolify

Render the compose file with the answers to catch missing variables early:

```sh
# Write the answers to a temporary env file OUTSIDE the repo, then:
docker compose --env-file <tmp.env> -f docker-compose.coolify.yml config
```

Expect `DB_SYNC_BASE_URL: https://sync.logseq.<BASE_DOMAIN>` and Traefik
`Host(...)` rules matching the chosen domain. Delete the temp file afterwards.
Any `:?required` error must be fixed before continuing.

## Step 4 — Create or update the service

Use MCP tools when connected; otherwise the Coolify REST API
(`https://$COOLIFY_HOST/api/v1/...`, header `Authorization: Bearer $COOLIFY_API_TOKEN`).
If a call is rejected, inspect the instance's own API reference
(`<host>/api/v1` OpenAPI) — shapes vary slightly across Coolify versions.

**Fresh deployment:**

1. Create a service in the chosen project/environment with `docker_compose_raw`
   = the full contents of `docker-compose.coolify.yml` (base64-encode the
   compose for updates via PATCH — a proven integration).
2. Create/update the service's environment variables: `BASE_DOMAIN`,
   `DB_SYNC_ADMIN_TOKEN`, `COGNITO_*`, plus any optional image pins (env-var
   endpoints under the service resource, or the Coolify UI's Environment
   Variables tab).
3. Set `connect_to_docker_network: true` on the service — the containers must
   join the proxy's external `coolify` network or Traefik cannot reach them.

**Update of an existing service (UUID known):**

1. PATCH the service with the current file contents as `docker_compose_raw`
   (base64-encoded).
2. Ensure the environment variables above exist and match the answers.
3. Ensure `connect_to_docker_network: true`.

## Step 5 — Deploy

Trigger the deployment (MCP deploy tool, or POST
`/api/v1/deploy?uuid=<service-uuid>`).

## Step 6 — Monitor and verify

1. Poll the service state via MCP or GET `/api/v1/services/<uuid>` until the
   service and its child applications report running; pull logs for anything
   failing. First deploys take several minutes (image pulls).
2. The `sync` container may show `running:unknown` — cosmetic (the distroless
   image has no healthcheck).
3. Verify from the outside:
   `curl -s https://sync.logseq.<BASE_DOMAIN>/health` → 200, and the web URL
   serves the PWA. A Traefik `503 no available server` usually means Step 4's
   network setting was missed or a stale router exists — never "fix" it by
   adding `priority=` or `PathPrefix()` to these routers.

## Step 7 — Report to the user

- Sync endpoint: `https://sync.logseq.<BASE_DOMAIN>` (port 8787)
- Web app: `https://web.logseq.<BASE_DOMAIN>` (port 8080)
- Volume: `<service-uuid>_sync-data` → `/app/data` (SQLite DB + assets)
- Client setup: Logseq **DB** client → sign in with a **free Logseq account**
  → Settings → Advanced → **Sync Server URL** → the sync endpoint → enable
  **Use Logseq Sync Beta** before creating or syncing a graph.
- TLS reminder: the proxy certificate must cover both hostnames
  (`*.logseq.<BASE_DOMAIN>` or per-host certs); with a private CA, it must be
  trusted on every client device.
