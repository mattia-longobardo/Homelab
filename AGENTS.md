# Docker Stack Conventions (authoritative)

These are the authoritative project conventions for the multi-stack Docker Compose environment at `/home/mattia/docker/`. They are the unified standard used across all AI assistants (**Claude**, **Codex**, and **Google Antigravity**) whenever touching Docker services, compose configurations, networks, or `.env` files in this homelab.

When creating, modifying, or refactoring any Docker service or Compose file under `/home/mattia/docker/`, follow this specification exactly. When modifying an existing service, change only what the user asked for plus what is strictly required to keep the service valid and consistent — never silently rewrite unrelated configuration.

---

## 1. Primary Workflow

1. **Read before writing.** Open the target stack's `docker-compose.yml` and its `.env` (and any included sub-composes) so you replicate live conventions. Pick a representative existing service in that file as your visual template.
2. **Decide placement** using the Stack Map (Section 2).
3. **Check shared infrastructure (`db/`)** before creating any backing services. Always reuse central PostgreSQL, Redis, ArangoDB, Silo (S3), and Stalwart (SMTP) instead of spinning up standalone redundant sidecars.
4. **Write the service** following the Canonical Service Key Order, Storage Standard, Permissions, Health & Healing, Networking, and Resource Limits.
5. **Self-verify** against the Quality Assurance checklist and report the results concisely.

---

## 2. The Stack Map (Canonical Placement)

Each top-level directory under `/home/mattia/docker/` is an independent stack with its own `docker-compose.yml` and `.env`. Place services strictly by purpose:

| Stack Directory | Purpose & Scope | Core Services & Examples |
| :--- | :--- | :--- |
| `network/` | Edge, ingress, DNS, mail gateway, self-healing | `traefik`, `cloudflare` (tunnel), `cloudflare-ddns`, `pihole`, `stalwart` (mail), `watchtower`, `autoheal`, `error-pages` |
| `db/` | Shared data persistence, cache, S3, core metrics | `postgres` (pgvector:pg18), `redis` (centralized DB 0-15), `arangodb:3.12`, `silo` (S3), `prometheus` |
| `dashboard/` | Management, alerting UIs & telemetry exporters | `portainer`, `grafana`, `gotify`, `cadvisor`, `node-exporter`, `blackbox-exporter` |
| `security/` | Identity (SSO), VPN mesh, password vault, NVR | `authentik` (server/worker), `vaultwarden`, `netbird` (server/dashboard/client), `frigate` |
| `storage/` | Document management, cloud files, no-code DB | `bytestash`, `paperless-ngx`, `seafile` (+ `seafile-mysql`), `teable` |
| `streaming/` | Media server (GPU), automation & acquisition | `plex` (NVIDIA runtime), `overseerr`, `sonarr`, `radarr`, `qbittorrent`, `jackett`, `tdarr`, `plex-syncer` |
| `travel/` | Travel booking, flight tracking, location history | `trek`, `airtrail`, `travel-app` (unified travel agent) |
| `projects/` | Custom full-stack applications (`include:`) | `horizon` (FastAPI + Next.js + Worker + MCP), `trading` (FastAPI + Next.js) |
| `website/` | Public websites and landing pages (`include:`) | `personal`, `giacur` |
| `ai/` | Automation & AI pipelines (reserved / inactive) | `n8n`, `runners` (currently commented out — do not touch without explicit request) |

---

## 3. Storage & Persistence Standard (Bind Mounts Only)

- **STRICTLY NO NAMED DOCKER VOLUMES**: Top-level `volumes:` definitions do not exist and are strictly prohibited.
- **NO ANONYMOUS VOLUMES**: If an upstream image defines an unwanted `VOLUME` in its Dockerfile (e.g. bundled DB data), neutralize it by providing an explicit bind mount or `tmpfs:`.
- **Bind Mount Paths**:
  1. **Stack-local state & configuration**: `./<service-name>:/data` (or `./<service-name>:/config` for `linuxserver.io` images).
  2. **Bulk / Shared Media Storage**: `../../storage/<folder>` or `/home/mattia/storage/<folder>` (e.g. `movies`, `tv`, `downloads`, `private`, `seafile`, `silo`).
  3. **Read-Only Mounts (`:ro`)**: Mandatory for configuration files, templates, sockets, and static certificates (e.g. `/var/run/docker.sock:/var/run/docker.sock:ro`, `./prometheus.yml:/etc/prometheus/prometheus.yml:ro`).
  4. **Volatile & Transcoding Paths**: Use `/dev/shm` or `tmpfs:` (e.g. Plex `/dev/shm:/transcode`, Seafile `/opt/seafile/pids`).

---

## 4. User & Permissions Standard (Least Privilege with Pragmatism)

- **Default Principle**: Enforce non-root execution with `user: "1000:1000"` (or environment variables `PUID=1000`, `PGID=1000` for linuxserver/tdarr images).
- **Pragmatic Rule for Functional Stability**: **Service stability always overrides cosmetic uniformity.** If an image strictly requires root or its own internal UID/GID (e.g. internal privilege dropping, system access, or strict multi-user daemon bootstrap), preserve what the image requires and document the deviation with a comment.
  - *Documented exceptions*:
    - **Internal privilege drop**: `trek` (starts root, drops to bundled `node` user), `stalwart` (manages internal `stalwart` user).
    - **Host hardware & system monitoring**: `node-exporter` (`pid: host`, `privileged: true`), `cadvisor` (system cgroups), `frigate` (TPU/GPU/RTSP).
    - **Dedicated DB engines**: `postgres` uses `1000:1000`; `mariadb`, `arangodb`, `redis` manage their own internal system UIDs if unoverridden.

---

## 5. Health System, Self-Healing & Auto-Update

### A. Healthcheck Standard
Every exposed service or service with dependencies must declare an explicit `healthcheck` block:
```yaml
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8000/api/health"]  # or CMD-SHELL / redis-cli / python
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s    # Extend to 60s-180s for heavy bootstrap services (e.g. Seafile, Paperless)
```

### B. Autoheal Integration
For background workers, long-running web APIs, and services prone to stale heartbeats or network blips, attach the `autoheal` label:
```yaml
    labels:
      - "autoheal=true"
```
The central `autoheal` container in `network/` automatically monitors container health via the Docker socket and restarts any container marked `unhealthy`.

### C. Watchtower & Image Tagging Strategy
- **Default to `latest`**: The homelab runs `watchtower` in `network/` (`WATCHTOWER_CLEANUP=true`, poll interval 6h) to keep the fleet updated automatically.
- **Pinning Policy**: Pin image tags (e.g. `mariadb:10.11`, `arangodb:3.12`, `cloudflared:2026.8.1`, `server:2026.8.0`) **only** when upstream breaking changes, major DB migrations, or known regression bugs require version locking. Document the reason in a comment.

### D. Public Availability Monitoring (Blackbox & Grafana)
- **Mandatory Registration for Public Services**: Every service exposed to the public internet (any public domain/subdomain, excluding local-only `.local` domains and work stacks under `work/`) **MUST** be added to the Prometheus scrape configuration at `db/prometheus.yml` under the `blackbox-http` job targets (`https://<service-domain>`).
- **Dashboard & Alerting**: The blackbox exporter regularly probes all listed endpoints (`http_2xx`). Probe status feeds directly into the Grafana **Availability / Blackbox** dashboard (`dashboard/grafana-dashboards/availability.json`) and triggers the `ProbeFailed` alert rule in `dashboard/grafana-alerting/rules.yml` if any public endpoint goes down.

---

## 6. Shared Data, Cache & Infrastructure Tier (`db/`)

Shared databases and caches live centrally in `db/` on the `db_internal` network. **Do not create standalone sidecar containers for services that can use shared infrastructure.**

### A. Centralized Redis (`redis:latest` in `db/`)
- Host: `redis:6379` on network `db_internal`.
- Authentication: `${REDIS_PASSWORD}` from `.env`.
- **Logical Database Isolation (Redis DB Index `0-15`)**:
  - `DB 0`: **Authentik** (SSO, Celery task queue, session storage)
  - `DB 1`: **Paperless-ngx** (Celery broker, document indexing)
  - `DB 2`: **Seafile** (Seahub cache, Seafevents channel)
  - `DB 3`: **Teable** (BullMQ queues, backend cache)
  - `DB 4`: **Travel Agent** (Session store, property search cache)
  - `DB 5+`: Next new consumer services.
- **Policy**: `maxmemory-policy noeviction` with persistent RDB snapshots (`--save 60 1`) so Celery/BullMQ queues are never dropped.

### B. Centralized PostgreSQL (`postgres` in `db/`)
- Image: `pgvector/pgvector:pg18` on `db_internal:5432`.
- Standard logical DB naming: one DB per app (`authentik`, `paperless`, `teable`, `airtrail`, `horizon`, `trading`, `n8n`), with dedicated users and passwords (`DB_<APP>_PASSWORD`).

### C. Graph & Object Storage
- **ArangoDB 3.12**: Multi-model graph DB (`arangodb:8529` on `db_internal`).
- **Silo (S3)**: MinIO-compatible S3 object storage (`silo:9000` S3 API, `:9001` console on `db_internal` & `proxy_public`).

---

## 7. Networking Model & Reverse Proxy (Traefik) Standard

### The 4 Shared Global Networks
The homelab uses 4 well-defined shared external networks:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DOCKER NETWORK TOPOLOGY                           │
├───────────────────┬───────────────────┬───────────────────┬─────────────────┤
│ proxy_public      │ db_internal       │ metrics_internal  │ mail_internal   │
│ (Created by net)  │ (Created by db)   │ (Created by net)  │ (Created by net)│
├───────────────────┼───────────────────┼───────────────────┼─────────────────┤
│ Traefik Ingress   │ Postgres, Redis,  │ Prometheus,       │ Stalwart SMTP   │
│ & Public Web UIs  │ ArangoDB, Silo    │ Grafana, CAD, Node│ mx.longobardo.me│
└───────────────────┴───────────────────┴───────────────────┴─────────────────┘
```

1. **`proxy_public`**: **The ONLY network exposed to Traefik.** Never attach pure backing services (Redis, MySQL, internal workers) to `proxy_public`.
2. **`db_internal`**: Connects consuming apps to PostgreSQL, Redis, ArangoDB, Silo, and Prometheus.
3. **`metrics_internal`**: Scrapes telemetry from exporters and Traefik to Prometheus and Grafana.
4. **`mail_internal`**: Connects apps to the Stalwart mail server via hostname `mx.longobardo.me` (or `stalwart`).
5. **Private Stack Networks**: Use `<app>_internal` only for isolated sidecars that must never be shared cross-stack (e.g. `seafile_internal` for `seafile-mysql`).

### Host Ports Policy (`ports:`)
**Never publish host ports for anything reachable via HTTP.** All web traffic must route through Traefik on `proxy_public`. Host ports are strictly reserved for Layer 4 protocols:
- Traefik: `80:80`, `443:443`
- Mail (Stalwart): `25:25`, `465:465`, `587:587`, `993:993`, `4190:4190`
- WireGuard / NetBird: UDP ports
- Media / Streaming: Plex `32400:32400`, Tdarr `8266:8266`
- Surveillance: Frigate `8554:8554`, `8555:8555`
- Database Admin: Postgres `5432:5432`

### Traefik Reverse-Proxy Labels Standard
Traefik runs with `exposedbydefault=false` and entrypoint `web` (TLS is terminated upstream by Cloudflare Tunnel):
```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=proxy_public"
      - "traefik.http.routers.<service>-router.rule=Host(`${APP_HOST}`)"
      - "traefik.http.routers.<service>-router.entrypoints=web"
      - "traefik.http.services.<service>-service.loadbalancer.server.port=<container_port>"
      # When forwarded HTTPS protocol is required:
      - "traefik.http.middlewares.<service>-https.headers.customrequestheaders.X-Forwarded-Proto=https"
      - "traefik.http.routers.<service>-router.middlewares=<service>-https"
```

---

## 8. Ecosystem Integrations (SSO & Mail)

### A. Single Sign-On (Authentik OIDC)
Integrate Authentik on all web applications where supported:
```yaml
      - OIDC_ISSUER=https://${AUTHENTIK_HOST}/application/o/<app>/
      - OIDC_CLIENT_ID=${<APP>_OIDC_CLIENT_ID}
      - OIDC_CLIENT_SECRET=${<APP>_OIDC_CLIENT_SECRET}
```

### B. SMTP Mail Delivery (Stalwart)
Route outgoing system emails through `mail_internal`:
```yaml
      - SMTP_HOST=${SMTP_HOST}          # Resolves to mx.longobardo.me
      - SMTP_PORT=465                   # 465 (Implicit SSL) or 587 (STARTTLS)
      - SMTP_USER=${SMTP_NOREPLY_USER}  # no-reply@longobardo.me
      - SMTP_PASS=${SMTP_NOREPLY_PASSWORD}
```

---

## 9. Resource Limits, Logging & Custom App Architecture

### A. Resource Limits (Mandatory on Every Service)
```yaml
    mem_limit: 1g          # Values: 64m, 128m, 512m, 1g, 2g, 4g, 6g
    cpus: "1.0"            # Always a quoted fractional string: "0.25", "0.5", "1.0", "2.0"
```

### B. Log Rotation
Prevent disk exhaustion from chatty services:
```yaml
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### C. Unified Custom Full-Stack Apps (`projects/` & `travel/`)
Custom apps (Horizon, Trading, Travel Agent) use a unified multi-stage container running Python 3.12 backend (port 8000) + Next.js 22 standalone frontend (port 3000) under a coordinated entrypoint with user `1000:1000`.

---

## 10. Canonical Service Key Order & Formatting

- **No `version:` key**.
- **2-space indentation**.
- Every service preceded by its banner comment with the exact long dashed rule:
  ```yaml
  # ServiceName # --------------------------------------------------------------------------------------------------------------
  ```

### Key Order in Service Block:
```yaml
  <service>:
    image: <repo>:latest
    container_name: <service>
    user: "1000:1000"
    depends_on:
      - <dependency>
    command: ...
    environment:
      - KEY=VALUE
    volumes:
      - ./<service>:/data
    restart: unless-stopped
    mem_limit: 1g
    cpus: "1.0"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s
    labels:
      - "autoheal=true"
      - "traefik.enable=true"
      - "traefik.docker.network=proxy_public"
      - "traefik.http.routers.<service>-router.rule=Host(`${HOST_VAR}`)"
      - "traefik.http.routers.<service>-router.entrypoints=web"
      - "traefik.http.services.<service>-service.loadbalancer.server.port=8080"
    networks:
      - proxy_public
      - db_internal
```

---

## 11. Quality Assurance Checklist

Before finalizing any Compose change or new service, verify:
- [ ] **Stack placement**: Placed in the right directory matching its purpose.
- [ ] **Shared DB/Cache**: Uses central Postgres (`db_internal:5432`) and central Redis (`redis:6379/<db_index>`); no unnecessary sidecars.
- [ ] **Storage**: Zero named volumes; zero anonymous volumes; bind mounts only; config/sockets `:ro`.
- [ ] **Permissions**: `user: "1000:1000"` (or justified, documented exception).
- [ ] **Health & Healing**: Declarative `healthcheck` configured; `- "autoheal=true"` added if appropriate.
- [ ] **Updates**: `latest` tag used (or pinned with documented reason).
- [ ] **Networking**: `proxy_public` used only if Traefik exposed; no host `ports:` for HTTP; global networks declared `external: true`.
- [ ] **Traefik labels**: Router/service names standard; `entrypoints=web`; `traefik.docker.network=proxy_public` present.
- [ ] **Limits**: Both `mem_limit` and quoted `cpus:` defined.
- [ ] **Secrets**: Passed via `.env` (`${VAR}`); new variables documented.
- [ ] **Availability Monitoring**: If exposed to the public internet (public domain, non-`.local`), target added to `db/prometheus.yml` under `blackbox-http` for the Grafana availability dashboard and alerting.
