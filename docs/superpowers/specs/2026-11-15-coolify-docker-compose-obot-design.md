# Obot Deployment on Coolify via Docker Compose Design Specification

- **Date:** 2026-11-15
- **Author:** Antigravity Agent
- **Target Environment:** Coolify Project `Infrastructure` (`plcmnur9rpv2l58kdsagb8bx`), Environment `ai gateway` (`vuojxlapuhnlbesv5qs3qc5y`)
- **Target Server:** `infrastructure Deployment Server` (`eg08k0kwsww48o004k88okgc`), Network Destination `coolify` (`q00gocs8ccowsw4488kk8ww8`)
- **Domain:** `https://obot.ty-dev.site`

---

## 1. Overview & Objective

Deploy **Obot** (AI governance platform, MCP gateway, and LLM gateway) into Coolify using a repository-tracked Docker Compose configuration (`docker-compose.yaml`). The deployment runs on the existing infrastructure deployment server with Traefik routing and automated SSL certificate management for `https://obot.ty-dev.site`.

---

## 2. Architecture & Container Layout

- **Container Image:** `ghcr.io/obot-platform/obot:latest`
- **Database Engine:** Embedded PostgreSQL 18 running within the container (managed via `run.sh` entrypoint when `OBOT_SERVER_DSN` is empty).
- **Service Name:** `obot`
- **Exposed Port:** Container port `8080`, mapped via Traefik.
- **Docker Socket:** `/var/run/docker.sock` mounted into the container to enable Obot to orchestrate hosted MCP servers and tool containers dynamically.
- **Data Persistence:** Dedicated named volume `obot_data` mounted at `/data`, persisting both the embedded PostgreSQL database (`/data/postgresql`) and application cache (`/data/cache`).
- **Network:** Connected to the external `coolify` network for reverse-proxy integration with Traefik.

---

## 3. Configuration & Environment Variables

The following environment variables will be declared in `.env.example` and configured in Coolify:

| Variable | Default / Value | Description |
| :--- | :--- | :--- |
| `OBOT_SERVER_SERVER_URL` | `https://obot.ty-dev.site` | Fully qualified external URL for redirects and gateway endpoints |
| `OBOT_SERVER_ENABLE_AUTHENTICATION` | `true` | Enforces authentication on server access |
| `OBOT_BOOTSTRAP_TOKEN` | *(Secret Alphanumeric)* | Master bootstrap admin token for initial platform setup |
| `OBOT_SERVER_PORT` | `8080` | Internal listening port |
| `PORT` | `8080` | Port variable standard |

---

## 4. Manifests

### 4.1 `docker-compose.yaml`
```yaml
services:
  obot:
    image: ghcr.io/obot-platform/obot:latest
    container_name: obot
    restart: unless-stopped
    ports:
      - "${PORT:-8080}:8080"
    environment:
      - OBOT_SERVER_SERVER_URL=${OBOT_SERVER_SERVER_URL:-https://obot.ty-dev.site}
      - OBOT_SERVER_ENABLE_AUTHENTICATION=${OBOT_SERVER_ENABLE_AUTHENTICATION:-true}
      - OBOT_BOOTSTRAP_TOKEN=${OBOT_BOOTSTRAP_TOKEN}
      - OBOT_SERVER_PORT=8080
    volumes:
      - obot_data:/data
      - /var/run/docker.sock:/var/run/docker.sock
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/readyz || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    networks:
      - coolify

volumes:
  obot_data:
    name: obot_data

networks:
  coolify:
    external: true
```

### 4.2 `.env.example`
```env
OBOT_SERVER_SERVER_URL=https://obot.ty-dev.site
OBOT_SERVER_ENABLE_AUTHENTICATION=true
OBOT_BOOTSTRAP_TOKEN=generate-a-secure-token-here
PORT=8080
```

---

## 5. Coolify Provisioning & Deployment Workflow

1. **Commit & Push**: Add `docker-compose.yaml` and `.env.example` to the git repository and push to `origin main`.
2. **Resource Creation**: Create an application in Coolify using the `coolify` MCP server:
   - Project: `plcmnur9rpv2l58kdsagb8bx`
   - Environment: `vuojxlapuhnlbesv5qs3qc5y`
   - Destination: `q00gocs8ccowsw4488kk8ww8`
   - Build pack: `dockercompose`
   - Repository: `TY-DEVS/obot`
   - Branch: `main`
   - Compose location: `/docker-compose.yaml`
3. **Environment Setup**: Set environment variables (`OBOT_BOOTSTRAP_TOKEN`, `OBOT_SERVER_SERVER_URL`, `OBOT_SERVER_ENABLE_AUTHENTICATION`).
4. **Domain Configuration**: Configure FQDN `https://obot.ty-dev.site` and compose domain mapping for service `obot`.
5. **Deployment & Health Check**: Trigger deployment and monitor build/deploy logs until the healthcheck passes (`running:healthy`).
6. **E2E Verification**: Verify HTTP response from `https://obot.ty-dev.site` and validate that the bootstrap login screen responds.
