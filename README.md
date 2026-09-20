# camelAI Coolify Deployment Guide

This guide will help you deploy [camelAI](https://camelai.com) to [Coolify](https://coolify.io) using the CAMEL AI Stream API as your LLM provider.

## Overview

camelAI is an AI coding assistant platform. This deployment:
- Runs camelAI as a self-hosted instance on Coolify
- Uses **CAMEL AI Stream** (`https://stream.camelai.com/v1`) as the LLM backend
- Disables authentication for easy access (enable in production)
- Disables deployed-app connections for security

## Prerequisites

1. **Coolify** installed and running on a Linux server
2. **Docker** access enabled in Coolify
3. A **domain name** pointed to your server (for production)
4. Your **CAMEL AI API key**: `qaml_live_4hSxqSQP-lw_cI1NIO23rpJlZ81jPPGnagbJ-amx7Qk`

## Deployment Steps

### Step 1: Clone or Upload Files

Upload the following files to your server or git repository:
- `docker-compose.yml`
- `.env`

### Step 2: Configure Environment Variables

Edit the `.env` file and update:

```bash
# Set your public URL (required for production)
SELFHOST_PUBLIC_BASE_URL=https://camel.yourdomain.com

# Your CAMEL AI API key (already configured)
SELFHOST_AI_API_KEY=qaml_live_4hSxqSQP-lw_cI1NIO23rpJlZ81jPPGnagbJ-amx7Qk

# Choose your default model (optional, can be changed in UI)
SELFHOST_AI_MODEL=claude-sonnet-4-20250514
```

### Step 3: Deploy in Coolify

1. Log into your Coolify dashboard
2. Go to **Projects** → **New Project**
3. Select **Docker Compose** as the deployment method
4. Paste the contents of `docker-compose.yml` or connect your git repository
5. Add all environment variables from `.env` file
6. Click **Deploy**

### Step 4: Configure Networking

In Coolify, ensure:
- Port **3001** is exposed for the main application
- Port **7001** is exposed for local artifacts service
- If using a domain, configure the Coolify reverse proxy

### Step 5: Access camelAI

Once deployed:
- Open `http://your-server-ip:3001` or your configured domain
- You should see the camelAI login/interface
- Start using the AI coding assistant!

## Configuration Options

### Available Models via CAMEL AI Stream

You can change the model in the camelAI settings UI. Supported models include:
- `claude-sonnet-4-20250514` (default)
- `claude-opus-4-20250514`
- `gpt-4o`
- `gpt-4o-mini`
- And more - see [CAMEL AI Fleet](https://camelai.com/docs/stream/fleet)

### Environment Variables Reference

| Variable | Description | Default |
|----------|-------------|---------|
| `SELFHOST_PUBLIC_BASE_URL` | Public URL for camelAI | `http://localhost:3001` |
| `SELFHOST_AI_API_KEY` | Your CAMEL AI API key | - |
| `SELFHOST_AI_MODEL` | Default LLM model | `claude-sonnet-4-20250514` |
| `LOCAL_AUTH_BYPASS` | Disable auth (dev only!) | `true` |
| `CONNECTIONS_BINDING_ENABLED` | Enable workspace connections | `false` |

## Production Hardening

For production use, make these changes:

### 1. Enable Authentication

Replace the auth bypass with proper OIDC/Pomerium:

```yaml
# Remove these from docker-compose.yml:
# LOCAL_AUTH_BYPASS: "true"
# LOCAL_AUTH_BYPASS_HOSTS: "*"
# LOCAL_AUTH_USER_EMAIL: admin@example.com
# LOCAL_AUTH_USER_NAME: Admin

# Add Pomerium configuration (see camelAI docs)
```

### 2. Enable Connections (Optional)

If you want workspace connections:
```yaml
CONNECTIONS_BINDING_ENABLED: "true"
```

### 3. Add TLS

Use Coolify's built-in SSL/TLS termination or add Caddy:

```yaml
# Add Caddy service for TLS
caddy:
  image: ghcr.io/qaml-ai/camelai-selfhost-caddy:selfhost-v0.1.18@sha256:8ebfc39bba585a34fc67495b21a739ee6d958f8e79324c703a5e0e42e2e1cfc6
  ports:
    - "443:443"
    - "80:80"
  volumes:
    - ./Caddyfile:/etc/caddy/Caddyfile:ro
```

### 4. Secure Your Secrets

Generate new secrets for production:
```bash
openssl rand -hex 32  # Repeat for each secret
```

## Troubleshooting

### Container Won't Start

Check logs:
```bash
docker compose logs app
docker compose logs local-artifacts
```

### Health Check Fails

The app depends on `local-artifacts` being healthy. Wait 30-60 seconds after deployment.

### Cannot Connect to Docker Socket

Ensure the Docker socket is mounted and accessible:
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:rw
```

### API Key Issues

Verify your CAMEL AI API key is valid:
```bash
curl https://stream.camelai.com/v1/models \
  -H "Authorization: Bearer qaml_live_4hSxqSQP-lw_cI1NIO23rpJlZ81jPPGnagbJ-amx7Qk"
```

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Coolify Server                        │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────────┐     ┌─────────────────┐           │
│  │   camelAI App   │────▶│ Local Artifacts │           │
│  │   (Port 3001)   │     │   (Port 7001)   │           │
│  └────────┬────────┘     └─────────────────┘           │
│           │                                              │
│           │ LLM API Calls                               │
│           ▼                                              │
│  ┌─────────────────┐                                    │
│  │ CAMEL AI Stream │                                    │
│  │ stream.camelai  │                                    │
│  └─────────────────┘                                    │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

## Resources

- [camelAI Documentation](https://camelai.com/docs/self-hosting/overview)
- [CAMEL AI Stream API](https://camelai.com/docs/stream/endpoints)
- [camelAI GitHub](https://github.com/qaml-ai/camelAI)
- [Coolify Documentation](https://coolify.io/docs)

## License

camelAI is MIT licensed. See [LICENSE](https://github.com/qaml-ai/camelAI/blob/main/LICENSE).
