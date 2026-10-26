# logseq-web-coolify — Logseq DB (web + sync) for Coolify

Self-hosted Logseq **DB version** stack: sync server (SQLite + assets) and the
web app / PWA, based on [yshalsager/logseq-selfhost](https://github.com/yshalsager/logseq-selfhost).

**Portable by design:** no hostnames, secrets, or Coolify UUIDs are baked in —
everything instance-specific is a variable set in one place.

## Configuration (per instance)

Required environment variables — set them in the Coolify service's
**Environment Variables** tab, or in `.env` locally (see `.env.example`):

| Variable | Meaning |
|---|---|
| `BASE_DOMAIN` | Base domain of the deployment. Hostnames are fixed as `sync.logseq.<BASE_DOMAIN>` and `web.logseq.<BASE_DOMAIN>` — only the base domain varies. |
| `DB_SYNC_ADMIN_TOKEN` | Sync admin token (`openssl rand -hex 32`). |
| `COGNITO_ISSUER` | Logseq's official Cognito issuer (see upstream docs). |
| `COGNITO_CLIENT_ID` | Logseq's official Cognito client id. |
| `COGNITO_JWKS_URL` | Logseq's official Cognito JWKS url. |

Optional: `SYNC_IMAGE`, `WEB_IMAGE` (image pins), `DB_SYNC_STORAGE_DRIVER`,
`DB_SYNC_ASSETS_DRIVER`, `DB_SYNC_LOG_LEVEL`.

Derived for you: `DB_SYNC_BASE_URL = https://sync.logseq.<BASE_DOMAIN>` — the
compose files compute it from `BASE_DOMAIN`, so it always matches the public
sync URL exactly.

Resulting URLs (Coolify prefixes the volume with the service UUID):

| | |
|---|---|
| Sync endpoint | `https://sync.logseq.<BASE_DOMAIN>` (port 8787) |
| Web app | `https://web.logseq.<BASE_DOMAIN>` (port 8080) |
| Volume | `<service-uuid>_sync-data` → `/app/data` (SQLite DB + assets) |

## Files

- `docker-compose.coolify.yml` — **the deploy file**: paste into Coolify
  "New Resource → Docker Compose", set the variables above, deploy.
- `docker-compose.yml` — local `docker compose up` variant (no Traefik labels,
  no `coolify` network).
- `.env` / `.env.example` — real values / template (`.env` is gitignored)
- `.agents/skills/deploy-logseq/` — portable **Agent Skill**: asks for the
  parameters and performs the deployment/update through the Coolify MCP or
  REST API (works in any harness)
- `AGENTS.md` — cross-harness entry point (Codex, Cursor, ...) routing agents
  to the skill; `.claude/skills` symlinks to it for Claude Code.

## Deploy on a Coolify instance

Or ask any agent harness to **run the `deploy-logseq` skill** — a portable
skill (`.agents/skills/deploy-logseq/SKILL.md`, routed via `AGENTS.md`) that
asks for the parameters and runs these steps interactively through the
Coolify MCP or REST API.

Manual steps:

1. Create a project/environment, then **New Resource → Docker Compose** and
   paste `docker-compose.coolify.yml`.
2. Add the required environment variables from the table above.
3. In the service settings enable **Connect to Docker Network → coolify**
   (`connect_to_docker_network: true` via UI or API PATCH).
4. Deploy.

## How routing works (hard-won lessons)

The Coolify **API cannot set the Domains field for compose services** (both
`domains` on create and update are rejected). The workaround that works:

1. Attach Traefik labels directly in the compose — plain `Host(...)` rule,
   `entrypoints=https`, `tls=true`, explicit `service=`, and a
   `loadbalancer.server.port` label. **Do not add `priority=` or `PathPrefix()`
   to these routers** — that combination made Traefik answer
   `503 no available server` for the router even though the identical backend
   served fine via a plain router.
2. Set `connect_to_docker_network: true` (PATCH via API or UI) and join the
   external `coolify` network (the default proxy network name on every Coolify
   install) — without this Traefik cannot reach the containers
   (`503 no available server`).
3. **Do not use `SERVICE_FQDN_*` env vars** for custom hostnames — Coolify
   overwrites their values with generated `sslip.io` URLs and creates dead
   routers that interfere.

Resulting label shape (see `docker-compose.coolify.yml`):

```yaml
labels:
  - traefik.enable=true
  - traefik.http.routers.lqsrv9f3.rule=Host(`sync.logseq.${BASE_DOMAIN}`)
  - traefik.http.routers.lqsrv9f3.entrypoints=https
  - traefik.http.routers.lqsrv9f3.tls=true
  - traefik.http.routers.lqsrv9f3.service=lqsrv9f3
  - traefik.http.services.lqsrv9f3.loadbalancer.server.port=8787
```

- TLS: the proxy must present a certificate valid for
  `sync.logseq.<BASE_DOMAIN>` and `web.logseq.<BASE_DOMAIN>` — e.g. a wildcard
  `*.logseq.<BASE_DOMAIN>` cert or per-host certificates. With a private
 /internal CA, install the CA cert in the proxy, trust it on every client
  device, and make sure the renewal flow keeps those SANs.

## Updating the deployment

Edit `docker-compose.coolify.yml`, then either use the Coolify UI (compose
file editor) or:

```bash
export COOLIFY_HOST=app.example.com           # your Coolify instance
export COOLIFY_SERVICE_UUID=xxxxxxxxxxxxxxxx  # UUID of this compose service
export COOLIFY_API_TOKEN=...                  # Coolify API token

# PATCH compose (base64) + trigger deploy
python3 - <<'EOF'
import json, urllib.request, base64, os
TOKEN = os.environ["COOLIFY_API_TOKEN"]
HOST = os.environ["COOLIFY_HOST"]
UUID = os.environ["COOLIFY_SERVICE_UUID"]
compose = open("docker-compose.coolify.yml").read()
req = urllib.request.Request(f"https://{HOST}/api/v1/services/{UUID}",
    data=json.dumps({"docker_compose_raw": base64.b64encode(compose.encode()).decode()}).encode(),
    headers={"Authorization": f"Bearer {TOKEN}", "Content-Type": "application/json"}, method="PATCH")
print("compose ->", urllib.request.urlopen(req, timeout=30).status)
req = urllib.request.Request(f"https://{HOST}/api/v1/deploy?uuid={UUID}",
    headers={"Authorization": f"Bearer {TOKEN}"}, method="POST")
print("deploy ->", urllib.request.urlopen(req, timeout=60).status)
EOF
```

Monitoring via the Coolify MCP (`https://<COOLIFY_HOST>/mcp`, Streamable HTTP,
Bearer token): `list_service_applications`,
`get_logs(resource=service_application, uuid=<child>, parent_uuid=<service>)`,
`control`, `deploy`.

## Connect Logseq clients

1. Install a Logseq **DB** client (desktop / iOS TestFlight / Android APK).
2. Sign in with a **free Logseq account** (auth uses Logseq's Cognito; only
   storage/sync is self-hosted — no subscription needed).
3. Settings → Advanced → **Sync Server URL** → `https://sync.logseq.<BASE_DOMAIN>`
4. Enable **Use Logseq Sync Beta** before creating or syncing a graph.
5. Success = cloud icon with green dot in the top bar.

## Data & backups

All state lives in the `sync_data` volume (`/app/data`): SQLite graph DB +
uploaded assets. The `web` service is stateless. On Coolify the volume is
named `<service-uuid>_sync-data` — find it with `docker volume ls | grep sync-data`.

```bash
docker run --rm -v <service-uuid>_sync-data:/data -v "$PWD":/backup \
  alpine tar czf /backup/sync-data-$(date +%F).tgz -C /data .
```

## Notes / gotchas

- `DB_SYNC_BASE_URL` must equal the public sync URL **exactly** (the compose
  files derive it from `BASE_DOMAIN`, which prevents drift).
- Do **not** swap `sync` for the `-sync-worker` image — incompatible storage.
- Both images track Logseq `master` weekly; pin a tag for stability.
- Web app wants a Chromium-based browser (File System Access API).
- Hardening: read-only rootfs, dropped capabilities, no-new-privileges.
- `sync` container shows "running:unknown" in Coolify — cosmetic (no
  healthcheck in the distroless image); the service answers `/health` fine.
- A stale Traefik router named `lqdiagnose` may linger until the next proxy
  reload — harmless (unused hostname).
