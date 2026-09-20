# camelAI on Coolify

Self-hosted [camelAI](https://camelai.com/docs/self-hosting/overview) coding
agent deployed as a Coolify Docker Compose resource.

- Release `selfhost-v0.1.18` — every image digest is pinned from that
  release's `selfhost-release.json`.
- LLM provider: **CAMEL AI Stream** (`https://stream.camelai.com/v1`,
  OpenAI-compatible, model `auto`).
- Authentication: **Cloudflare Access** (`SELFHOST_AUTH_MODE=cloudflare-access`).
- `CONNECTIONS_BINDING_ENABLED=false` — deployed apps cannot read workspace
  connections.

## How it is wired

| Piece | Detail |
| --- | --- |
| Coolify app domain | `https://camelai.382972.xyz` (Traefik + Let's Encrypt) |
| App container | host network, listens `0.0.0.0:3001`, reaches Traefik via `host.docker.internal` |
| local-artifacts | bridge network, published on `127.0.0.1:7001` for the host-network app |
| Sandbox runtimes | `project-build`, `analysis`, `db-query`, `container-egress` images pre-pulled by one-shot services |
| State | Docker volumes `app-state` and `local-artifacts-repos` |

## Required environment variables (set in Coolify, not here)

Secrets (generate with `openssl rand -hex 32`):

- `TOKEN_SIGNING_SECRET`
- `INTEGRATION_SECRET_KEY`
- `ADMIN_API_KEY`
- `LOCAL_ARTIFACTS_SECRET`

Provider and public config:

- `SELFHOST_PUBLIC_BASE_URL=https://camelai.382972.xyz`
- `SELFHOST_AI_PROVIDER=custom`
- `SELFHOST_AI_API_KEY=<CAMEL AI Stream key>`
- `SELFHOST_AI_BASE_URL=https://stream.camelai.com/v1`
- `SELFHOST_AI_MODEL=auto`
- `SELFHOST_AI_NAME=CAMEL AI Stream`
- `SELFHOST_AI_AUTH_TYPE=bearer`
- `SELFHOST_AI_API=openai-completions`
- `SELFHOST_AUTH_MODE=cloudflare-access`
- `CLOUDFLARE_ACCESS_TEAM_DOMAIN=https://neonfalcon.cloudflareaccess.com`
- `CLOUDFLARE_ACCESS_AUD=<Access application audience tag>`
- `CLOUDFLARE_ACCESS_DEFAULT_ORG_NAME=Ouroboric`

Image pins (from `selfhost-release.json` for `selfhost-v0.1.18`):

- `SELFHOST_APP_IMAGE`
- `SELFHOST_LOCAL_ARTIFACTS_IMAGE`
- `SELFHOST_PROJECT_BUILD_IMAGE`
- `SELFHOST_ANALYSIS_IMAGE`
- `SELFHOST_DB_QUERY_IMAGE`
- `SELFHOST_CONTAINER_EGRESS_IMAGE`

Optional:

- `CONNECTIONS_BINDING_ENABLED=false`
- `SELFHOST_BIND_ADDRESS=0.0.0.0` (Coolify Traefik must reach the app; the
  standalone docs default this to `127.0.0.1` behind bundled Caddy)
- `SELFHOST_APP_PORT=3001`

## Verify

```bash
# Internal health (inside the container / on the VPS)
curl -fsS http://127.0.0.1:3001/api/selfhost/health

# Public URL — should present the Cloudflare Access sign-in page
curl -sI https://camelai.382972.xyz
```

Coolify app logs should show `workerd` starting, the D1 migrations applying,
and `Local bindings loopback on http://127.0.0.1:<port>`.

## Operational notes

- The app container has read-write access to `/var/run/docker.sock`. Treat it
  as root-equivalent VM access; do not expose the origin port publicly.
- Upgrades: pick a newer `selfhost-v*` release, update the six image pins in
  Coolify's environment variables to the new `selfhost-release.json` digests,
  and redeploy. Never mix releases.
- Backups: snapshot the `app-state` and `local-artifacts-repos` volumes.
- Outbound email is not supported; provision users through Cloudflare Access.
