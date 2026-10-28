# Multica Deployment

Self-hosted deployment stack for [Multica](https://multica.ai) — the open-source collaboration platform designed to integrate AI coding agents as first-class team members alongside developers.

> **Official Documentation**: For complete setup guides, agent integrations, and architecture docs, visit [multica.ai/docs](https://multica.ai/docs).

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Configuration Reference](#configuration-reference)
  - [Database & Security](#database--security)
  - [Domains & URLs](#domains--urls)
  - [Authentication (Google OAuth)](#authentication-google-oauth)
  - [Transactional Email (Resend)](#transactional-email-resend)
  - [File Storage & CDN (AWS S3 + CloudFront)](#file-storage--cdn-aws-s3--cloudfront)
  - [Agent Runtime & LLM Providers](#agent-runtime--llm-providers)
- [Code-Server & Oh-My-Pi Runtime Setup](#code-server--oh-my-pi-runtime-setup)
- [Networking & Reverse Proxy](#networking--reverse-proxy)
- [Operations & Maintenance](#operations--maintenance)
- [Troubleshooting](#troubleshooting)
- [References & Documentation](#-references--documentation)

---

## Architecture Overview

This Docker Compose configuration deploys a complete Multica workspace:

```
                      ┌─────────────────────────────────────┐
                      │    Reverse Proxy (Caddy / Nginx)    │
                      │         (SSL / Port 443)            │
                      └───────┬──────────────┬──────────────┘
                              │              │
                    :3000     │              │     :8080
            ┌─────────────────▼──┐       ┌───▼────────────────┐
            │      Frontend      │       │      Backend       │
            │  (Next.js Web UI)  │◄─────►│   (API & Auth)     │
            └────────────────────┘       └───┬────────────┬───┘
                                             │            │
                                  ┌──────────▼──┐     ┌───▼──────────────┐
                                  │  PostgreSQL │     │   code-server    │
                                  │ (+ pgvector)│     │  (Agent Runtime) │
                                  └─────────────┘     └──────────────────┘
```

| Service | Image | Description |
| :--- | :--- | :--- |
| **`frontend`** | `ghcr.io/multica-ai/multica-web` | Next.js web application and dashboard |
| **`backend`** | `ghcr.io/multica-ai/multica-backend` | API server, task orchestrator, and authentication handler |
| **`postgres`** | `pgvector/pgvector:pg17` | PostgreSQL 17 database with vector extension for embeddings |
| **`code-server-init`** | `codercom/code-server:4.121.0` | One-shot init container that installs Oh-My-Pi into the shared home volume |
| **`code-server`** | `codercom/code-server:4.121.0` | In-browser VS Code runtime equipped for AI agent CLI execution |

---

## Prerequisites

- **Docker Engine** 24.0+ and **Docker Compose** v2+
- **OpenSSL** (for generating cryptographically secure secrets)
- A **Domain Name** pointing to your host with an SSL/TLS reverse proxy (e.g., Caddy, Traefik, or NGINX)

---

## Quick Start

### 1. Clone & Prepare Environment

```bash
# Copy example environment file
cp .env.example .env
```

### 2. Generate Secrets

Generate strong, random credentials for database and token signing:

```bash
# Generate 16-byte hex password for PostgreSQL
openssl rand -hex 16

# Generate 32-byte hex secret for JWT token signing
openssl rand -hex 32
```

Assign these outputs to `POSTGRES_PASSWORD` and `JWT_SECRET` in your `.env` file.

### 3. Configure `.env`

Open `.env` and fill in your domain URLs, email credentials, OAuth, and API keys:

```bash
nano .env   # or open with your preferred editor
```

### 4. Start Services

```bash
# Launch the stack in detached mode
docker compose up -d

# Check service status and health
docker compose ps
```

---

## Configuration Reference

### Database & Security

| Variable | Description | Example / Default |
| :--- | :--- | :--- |
| `POSTGRES_DB` | PostgreSQL database name | `multica` |
| `POSTGRES_USER` | PostgreSQL superuser | `multica` |
| `POSTGRES_PASSWORD` | PostgreSQL password | *Generate with `openssl rand -hex 16`* |
| `JWT_SECRET` | Secret key used to sign session JWTs | *Generate with `openssl rand -hex 32`* |

### Domains & URLs

| Variable | Description | Example / Default |
| :--- | :--- | :--- |
| `FRONTEND_ORIGIN` | Public origin of the frontend UI | `https://multica.example.com` |
| `ALLOWED_ORIGINS` | Permitted CORS origins for API requests | `https://multica.example.com` |
| `MULTICA_APP_URL` | Base public URL of your Multica instance | `https://multica.example.com` |
| `COOKIE_DOMAIN` | Optional domain scope for session cookies | `.example.com` |

### Authentication (Google OAuth)

| Variable | Description |
| :--- | :--- |
| `GOOGLE_CLIENT_ID` | OAuth 2.0 Client ID from Google Cloud Console |
| `GOOGLE_CLIENT_SECRET` | OAuth 2.0 Client Secret from Google Cloud Console |
| `GOOGLE_REDIRECT_URI` | Authorized redirect URI (e.g. `https://multica.example.com/api/auth/callback/google`) |

### Transactional Email (Resend)

Used for invitation emails, password resets, and notifications:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `RESEND_API_KEY` | API Key from [Resend](https://resend.com) | `re_123456789...` |
| `RESEND_FROM_EMAIL` | Sender address verified with Resend | `noreply@example.com` |

### File Storage & CDN (AWS S3 + CloudFront)

Optional configuration for cloud file attachments, avatar uploads, and media storage:

| Variable | Description | Example |
| :--- | :--- | :--- |
| `S3_BUCKET` | AWS S3 Bucket name | `my-multica-uploads` |
| `S3_REGION` | AWS Region where the bucket is located | `us-west-2` |
| `CLOUDFRONT_DOMAIN` | CloudFront distribution domain | `d111111abcdef8.cloudfront.net` |
| `CLOUDFRONT_KEY_PAIR_ID` | CloudFront Key Pair ID for signed URLs | `K2JC3ND02R65MT` |
| `CLOUDFRONT_PRIVATE_KEY` | RSA Private Key for CloudFront signed URLs | `-----BEGIN RSA PRIVATE KEY-----...` |

### Agent Runtime & LLM Providers

| Variable | Description | Example / Notes |
| :--- | :--- | :--- |
| `CODE_SERVER_PASSWORD` | Web access password for the `code-server` IDE | Set a strong password |
| `TZ` | Timezone inside containers | `UTC` or `Europe/Berlin` |
| `OPENROUTER_API_KEY` | API key for OpenRouter LLM gateway | `sk-or-v1-...` |
| `LM_STUDIO_API_KEY` | Optional API key for LM Studio | `lm-studio` |
| `LM_STUDIO_BASE_URL` | Host/endpoint for local LM Studio instance | `http://localhost:1234/v1` |
| `MULTICA_API_KEY` | Multica workspace API key for runtime daemons | `mul_...` |

---

## Code-Server & Oh-My-Pi Runtime Setup

The `code-server` container acts as an isolated sandbox for running coding agents (like Claude Code, OpenCode, or custom agent daemons) connected to your Multica workspace.

### Automated Setup via `code-server-init`

Oh-My-Pi is **automatically installed** before `code-server` launches:
- The **`code-server-init`** container checks `/home/coder/.local/bin` on startup. If not already present, it downloads and installs Oh-My-Pi (`curl -fsSL https://omp.sh/install | sh`) and creates the `/home/coder/.omp` directory.
- `code-server` waits for this init step (`condition: service_completed_successfully`), ensuring that `omp` is in the `PATH` as soon as code-server becomes accessible.
- LLM credentials from `.env` are automatically mounted to `/home/coder/.omp/.env`.

### Pairing with Multica Workspace:

1. **Add a Runtime in Multica UI:**
   - Log into your Multica web dashboard.
   - Navigate to **Runtimes**.
   - Click **Add Runtime** to connect your coding agent.

2. **Connect the agent daemon inside `code-server`:**
   Open the built-in terminal in `code-server` (or execute via Docker):
   - Install Multica Daemon: `curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash`
   - Login to Multica: `multica login --token mul_YOUR_TOKEN_HERE`

3. **Verify Oh-My-Pi agent:**
   ```bash
   omp --version
   ```

---

## Networking & Reverse Proxy

By default, ports are kept within the internal Docker network. To expose them, route traffic through a reverse proxy (e.g. Caddy or Nginx) to the appropriate containers:

- **Frontend**: port `3000`
- **Backend API**: port `8080`
- **Code-Server Web IDE**: port `8080`

### Example Caddyfile Configuration

```caddy
multica.example.com {
    # Route API and backend traffic
    handle /api/* {
        reverse_proxy backend:8080
    }

    # Route frontend web app
    handle {
        reverse_proxy frontend:3000
    }
}

code.example.com {
    # Route to code-server
    reverse_proxy code-server:8080
}
```

> **Note:** If exposing ports directly without a reverse proxy for local testing, add port mappings under `ports:` in `docker-compose.yml`:
> ```yaml
> frontend:
>   ports:
>     - "3000:3000"
> backend:
>   ports:
>     - "8080:8080"
> code-server:
>   ports:
>     - "8443:8080"
> ```

---

## Operations & Maintenance

### Viewing Logs

```bash
# Stream all logs
docker compose logs -f

# Follow logs for a specific service
docker compose logs -f backend
docker compose logs -f frontend
docker compose logs -f code-server
```

### Updating Images

```bash
docker compose pull
docker compose up -d --remove-orphans
```

### Persistent Volumes

Data is preserved across container restarts using Docker volumes:

- `pgdata`: PostgreSQL data directory (database records, vector embeddings).
- `backend_uploads`: Local fallback directory for file attachments (`/app/data/uploads`).
- `code-server-home`: User directory (`/home/coder`) preserving code-server extensions, settings, and agent workspaces.


## References & Documentation

- **Multica Documentation**: [https://multica.ai/docs](https://multica.ai/docs)
- **Multica Website**: [https://multica.ai](https://multica.ai)
- **GitHub Repository**: [https://github.com/multica-ai/multica](https://github.com/multica-ai/multica)
- **Oh-My-Pi (OMP)**: [https://omp.sh](https://omp.sh)