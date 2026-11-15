# Coolify Docker Compose Obot Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Obot as a repository-tracked Docker Compose application on Coolify in Project `Infrastructure` / Environment `ai gateway` accessible at `https://obot.ty-dev.site`.

**Architecture:** Single container running `ghcr.io/obot-platform/obot:latest` with embedded PostgreSQL 18 persisting to named volume `obot_data:/data`, host Docker socket mounted at `/var/run/docker.sock`, and Traefik reverse-proxy routing for `https://obot.ty-dev.site` over the `coolify` Docker network.

**Tech Stack:** Docker Compose v2, Coolify API/MCP, Traefik, Obot, Bash, Git.

## Global Constraints

- Project UUID: `plcmnur9rpv2l58kdsagb8bx` ("Infrastructure")
- Environment UUID: `vuojxlapuhnlbesv5qs3qc5y` ("ai gateway")
- Server UUID: `eg08k0kwsww48o004k88okgc` ("infrastructure Deployment Server")
- Destination UUID: `q00gocs8ccowsw4488kk8ww8` ("coolify")
- Git Repository: `TY-DEVS/obot` (branch: `main`)
- Public FQDN: `https://obot.ty-dev.site`
- Image: `ghcr.io/obot-platform/obot:latest`
- Internal service port: `8080`

---

### Task 1: Create `docker-compose.yaml` and `.env.example`

**Files:**
- Create: `docker-compose.yaml`
- Create: `.env.example`

**Interfaces:**
- Produces: Docker Compose configuration for Obot service with healthcheck, `/data` volume, docker socket mount, and `coolify` external network.

- [ ] **Step 1: Write `docker-compose.yaml`**

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

- [ ] **Step 2: Write `.env.example`**

```env
OBOT_SERVER_SERVER_URL=https://obot.ty-dev.site
OBOT_SERVER_ENABLE_AUTHENTICATION=true
OBOT_BOOTSTRAP_TOKEN=your-secure-bootstrap-token
PORT=8080
```

- [ ] **Step 3: Validate docker-compose syntax locally**

Run: `docker compose config`
Expected: Valid compose config with no syntax errors.

- [ ] **Step 4: Commit compose files**

Run:
```bash
git add docker-compose.yaml .env.example
git commit -m "feat(infra): add docker-compose.yaml and .env.example for Coolify deployment"
```

---

### Task 2: Push Changes to Remote Repository

**Files:**
- Repository: `origin main`

- [ ] **Step 1: Check remote branch status and push**

Run: `git push origin main`
Expected: Successfully pushed commits to `https://github.com/TY-DEVS/obot.git`.

---

### Task 3: Provision Application in Coolify

**Interfaces:**
- Consumes: `TY-DEVS/obot` main branch, Coolify Project `plcmnur9rpv2l58kdsagb8bx`, Environment `vuojxlapuhnlbesv5qs3qc5y`.
- Produces: Coolify Application UUID for `obot`.

- [ ] **Step 1: Call `create_application` via Coolify MCP**

Call `create_application` with:
- `action`: `github_app`
- `data`:
  - `project_uuid`: `plcmnur9rpv2l58kdsagb8bx`
  - `environment_uuid`: `vuojxlapuhnlbesv5qs3qc5y`
  - `server_uuid`: `eg08k0kwsww48o004k88okgc`
  - `destination_uuid`: `q00gocs8ccowsw4488kk8ww8`
  - `github_app_uuid`: `bkgsksk484o0skk0oos8ss8s`
  - `git_repository`: `TY-DEVS/obot`
  - `git_branch`: `main`
  - `build_pack`: `dockercompose`
  - `name`: `obot`
  - `description`: `Obot AI governance platform and MCP/LLM gateway`
  - `docker_compose_location`: `/docker-compose.yaml`

- [ ] **Step 2: Verify Application Created**

Check returned UUID and call `get_application` to confirm state.

---

### Task 4: Configure Domain & Environment Variables in Coolify

**Interfaces:**
- Consumes: Application UUID from Task 3.
- Produces: Configured FQDN `https://obot.ty-dev.site` and required runtime environment variables.

- [ ] **Step 1: Update Application Domain & Compose Routing**

Call `manage_application` (`action: update`) with:
- `uuid`: Application UUID
- `data`:
  - `docker_compose_domains`: `{"obot":{"domain":"https://obot.ty-dev.site","redirect":"both"}}`

- [ ] **Step 2: Generate Secure Bootstrap Token & Set Environment Variables**

Call `manage_env` (`action: bulk_update`) on application resource with:
- `OBOT_SERVER_SERVER_URL`: `https://obot.ty-dev.site`
- `OBOT_SERVER_ENABLE_AUTHENTICATION`: `true`
- `OBOT_BOOTSTRAP_TOKEN`: Secure 32-character generated token
- `PORT`: `8080`

- [ ] **Step 3: Verify Environment Variables Stored**

Call `list_env_keys` to verify all required keys exist.

---

### Task 5: Trigger Deployment & Validate Live Service

**Interfaces:**
- Consumes: Configured Application UUID.
- Produces: Live, healthy Obot service responding on `https://obot.ty-dev.site`.

- [ ] **Step 1: Trigger Application Deployment**

Call `deploy` with `uuid: <application_uuid>`.

- [ ] **Step 2: Monitor Deployment Status**

Call `list_deployments` and `get_deployment` to watch progress until finished successfully.

- [ ] **Step 3: Verify Container Health & Logs**

Call `get_application` to verify `status: "running:healthy"` and `get_logs` to ensure embedded Postgres and Obot server started cleanly.

- [ ] **Step 4: End-to-End HTTP Verification**

Perform HTTP request to `https://obot.ty-dev.site` to ensure 200 OK and that the Obot UI loads properly.
