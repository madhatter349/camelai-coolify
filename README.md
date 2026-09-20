# camelAI Coolify Deployment

This guide covers deploying [camelAI](https://camelai.com) to [Coolify](https://coolify.io) using the CAMEL AI Stream API as the LLM backend.

## Current Status

**Deployed** at: `https://xholxxtqrlkmvmf85edisqyu.382972.xyz`

⚠️ **Known Issue**: The Traefik loadbalancer port is set to 80 instead of 3001. This can only be fixed through the Coolify dashboard at `https://382972.xyz`. 

**Fix in dashboard**: 
1. Go to Projects → camelAI → Application Settings
2. Change the exposed port from 80 to 3001
3. Redeploy

**App UUID**: `xholxxtqrlkmvmf85edisqyu`
**Project UUID**: `akyfcybododt7jcqlmhtyxbh`

## Architecture

The deployment uses a single `dockerimage` deployment on Coolify with the pre-built image `ghcr.io/qaml-ai/camelai-selfhost-app:selfhost-v0.1.18`. CAMEL AI Stream (`https://stream.camelai.com/v1`) is used as the LLM backend.

Note: The official Docker Compose deployment requires two services (app + local-artifacts), but Coolify's API doesn't support Docker Compose deployments via API. The single image deployment works but without the local-artifacts sidecar.

## Environment Variables

All env vars are configured via the Coolify API:

| Variable | Value |
|----------|-------|
| `SELFHOST_AI_PROVIDER` | `custom` |
| `SELFHOST_AI_API_KEY` | `qaml_live_4hSxqSQP-lw_cI1NIO23rpJlZ81jPPGnagbJ-amx7Qk` |
| `SELFHOST_AI_BASE_URL` | `https://stream.camelai.com/v1` |
| `SELFHOST_AI_MODEL` | `claude-sonnet-4-20250514` |
| `LOCAL_AUTH_BYPASS` | `true` (no auth required) |
| `CONNECTIONS_BINDING_ENABLED` | `false` |

## Deployment via Coolify API

### Step 1: Create Project
```bash
curl -X POST "$COOLIFY_URL/api/v1/projects" \
  -H "Authorization: Bearer $COOLIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name": "camelAI"}'
```

### Step 2: Create Application
```bash
# Dockerfile must include EXPOSE 3001
DOCKERFILE_B64=$(echo -n "FROM ghcr.io/qaml-ai/camelai-selfhost-app:selfhost-v0.1.18
EXPOSE 3001" | base64)

curl -X POST "$COOLIFY_URL/api/v1/applications/dockerfile" \
  -H "Authorization: Bearer $COOLIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d "{
    \"name\": \"camelai\",
    \"project_uuid\": \"$PROJECT_UUID\",
    \"server_uuid\": \"$SERVER_UUID\",
    \"environment_uuid\": \"$ENV_UUID\",
    \"git_repository\": \"coollabsio/coolify\",
    \"git_branch\": \"main\",
    \"dockerfile\": \"$DOCKERFILE_B64\"
  }"
```

### Step 3: Configure Application
```bash
curl -X PATCH "$COOLIFY_URL/api/v1/applications/$APP_UUID" \
  -H "Authorization: Bearer $COOLIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "camelai",
    "ports_exposes": "3001",
    "health_check_path": "/api/selfhost/health",
    "health_check_port": "3001",
    "limits_memory": "2g",
    "limits_cpus": "1.0"
  }'
```

### Step 4: Set Environment Variables
```bash
curl -X POST "$COOLIFY_URL/api/v1/applications/$APP_UUID/envs" \
  -H "Authorization: Bearer $COOLIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key": "SELFHOST_AI_API_KEY", "value": "qaml_live_..."}'
```

### Step 5: Deploy
```bash
curl -X POST "$COOLIFY_URL/api/v1/applications/$APP_UUID/start" \
  -H "Authorization: Bearer $COOLIFY_API_TOKEN"
```

## Coolify API Gotchas

### Critical: Traefik Loadbalancer Port Bug
Coolify always generates `traefik.http.services.*.loadbalancer.server.port=80` regardless of `ports_exposes`. This means services on non-80 ports won't work via the API.

**Fix**: Must be done through the Coolify dashboard → Application Settings → change the port.

### API Endpoint Details
- `POST /api/v1/applications/dockerfile` - Create dockerfile app (returns 404 for dockercompose)
- `POST /api/v1/applications/dockerimage` - Create dockerimage app
- `PATCH /api/v1/applications/{uuid}` - Update app (cannot change `fqdn` or `docker_compose_raw`)
- `POST /api/v1/applications/{uuid}/envs` - Set env var (individual, not batch)
- `POST /api/v1/applications/{uuid}/start` - Deploy
- `POST /api/v1/applications/{uuid}/restart` - Restart
- `DELETE /api/v1/applications/{uuid}` - Delete

### Project/Environment UUIDs
```bash
# List projects
curl -H "Authorization: Bearer $TOKEN" "$URL/api/v1/projects"

# Get project details (includes environment UUIDs)
curl -H "Authorization: Bearer $TOKEN" "$URL/api/v1/projects/$PROJECT_UUID"

# Get server UUID
curl -H "Authorization: Bearer $TOKEN" "$URL/api/v1/servers"
```

### Creating Repos
```bash
# Use SSH key at /data/keys/id_ed25519_github
export GIT_SSH_COMMAND="ssh -i /data/keys/id_ed25519_github -o IdentitiesOnly=yes -o StrictHostKeyChecking=accept-new"

# Create repo via GitHub API (needs GITHUB_TOKEN)
gh repo create madhatter349/repo-name --private --source=. --push
```

## Troubleshooting

### "no available server" on HTTPS
- Let's Encrypt cert may not be issued yet. Wait 5 minutes.
- Or the loadbalancer port is wrong (see Critical Bug above).

### "404 page not found" on HTTP
- Traefik is routing correctly, but the backend port is wrong.
- The camelAI app serves its web UI on port 80 and workerd on port 3001.
- Health check endpoint is on port 3001: `/api/selfhost/health`

### Container not starting
- Check logs: `curl -H "Authorization: Bearer $TOKEN" "$URL/api/v1/applications/$UUID/logs"`
- Verify image exists: `ghcr.io/qaml-ai/camelai-selfhost-app:selfhost-v0.1.18`

### Coolify Dashboard Login
- Admin credentials in `/data/keys/cheapestinference-gateway-admin.env`
- Username: `admin`
- Password: `UeuuPTlsqjmWeZ68EJnvgUFStHhRYGJ5`
