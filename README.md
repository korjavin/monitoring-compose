# Monitoring Compose Stack

Docker Compose monitoring stack featuring **Grafana**, **VictoriaMetrics**, **VictoriaLogs**, **vmagent**, and **Vector** — designed for GitOps deployment with Portainer behind Traefik, authenticated via **Pocket-ID (OIDC)**.

## Components

- **Grafana** (`:3000` via Traefik HTTPS) — Central observability dashboard with pre-provisioned VictoriaMetrics and VictoriaLogs datasources, plus Pocket-ID SSO.
- **VictoriaMetrics** (`:8428` internal) — High-performance, resource-efficient Prometheus-compatible metrics time series database.
- **VictoriaLogs** (`:9428` internal) — High-throughput, low-resource log database queried with LogsQL.
- **vmagent** (`:8429` internal) — Lightweight metrics scraper collecting internal metrics (VictoriaMetrics, VictoriaLogs, vmagent, Grafana) and local Docker containers via `docker_sd_configs`.
- **Vector** (log collector) — Ships stdout/stderr logs from Portainer and all host Docker containers directly into VictoriaLogs via the Docker engine socket (`/var/run/docker.sock`).

---

## Architecture

```
[ Docker Containers & Portainer ]
       │
       │ (docker.sock logs)
       ▼
   [ Vector ] ──────(HTTP /insert/jsonline)─────► [ VictoriaLogs ]
                                                         ▲
[ vmagent (scraper) ] ───(HTTP /api/v1/write)─► [ VictoriaMetrics ]
       │                                                 ▲
       ├─ Scrapes VictoriaMetrics:8428                   │
       ├─ Scrapes VictoriaLogs:9428                      │
       ├─ Scrapes vmagent:8429                           │
       ├─ Scrapes Grafana:3000                           │
       └─ Discovers Docker containers                    │
          (with label prometheus.scrape=true)            │
                                                         │
[ Traefik Proxy ] ─── HTTPS ───► [ Grafana ] ────────────┴─ (Datasources provisioned)
                                      │
                                      ▼ (OIDC / SSO)
                                [ Pocket-ID ]
```

---

## Prerequisites

- Docker engine with access to `/var/run/docker.sock`
- Portainer (for GitOps deployment)
- Traefik reverse proxy running on an external Docker network (`traefik_default`) with a configured certresolver (e.g. `myresolver`)
- Pocket-ID instance for OIDC authentication (e.g. `id.example.com`)
- DNS A/CNAME record pointing your Grafana domain (`grafana.example.com`) to your server

---

## Pocket-ID (OIDC) Setup

1. Open your Pocket-ID admin panel (e.g. `https://id.example.com`).
2. Go to **Applications / OIDC Clients** → **Create Application**.
3. Set the following details:
   - **Name**: `Grafana`
   - **Redirect URI**: `https://<GRAFANA_HOST>/login/generic_oauth`  
     *(e.g., `https://grafana.example.com/login/generic_oauth`)*
4. Copy the generated **Client ID** and **Client Secret**.
5. Grant admin rights via a Pocket-ID group:
   - Create a group whose **Name** (not Friendly name) is exactly `admin` — that is the value sent in the `groups` claim — and add your user to it.
   - If the Grafana client has **Allowed user groups** set, include `admin` there.
   - Members of `admin` become **Grafana server admin** (super admin) plus org **Admin** (`role_attribute_path` → `GrafanaAdmin`, `allow_assign_grafana_admin=true`).
   - Other authenticated users get **Viewer**.
   - Roles sync from Pocket-ID on every login, so manual role changes in the Grafana UI are overwritten; log out and back in after changing groups.

---

## Deploying via Portainer

### 1. Create Stack in Portainer

1. Navigate to **Stacks** → **Add stack**.
2. **Name**: `monitoring`
3. **Build method**: **Repository**
4. **Repository URL**: `https://github.com/korjavin/monitoring-compose`
5. **Repository reference**: `deploy` *(Important: watch `deploy`, not `master`)*
6. **Compose path**: `docker-compose.yml`
7. Enable **Automatic updates** and copy the **Webhook URL** (needed for GitHub Actions `PORTAINER_REDEPLOY_HOOK`).

### 2. Configure Environment Variables

Set the environment variables in Portainer stack settings (refer to [.env.example](file:///.env.example)):

| Variable | Description | Example / Default |
| :--- | :--- | :--- |
| `GRAFANA_HOST` | Domain for Grafana UI | `grafana.example.com` |
| `LOG_HOSTNAME` | `host` field on collected logs | `docker-host` |
| `TRAEFIK_NETWORK_NAME` | External Traefik network | `traefik_default` |
| `TRAEFIK_CERTRESOLVER` | Traefik ACME resolver | `myresolver` |
| `POCKET_ID_HOST` | Pocket-ID domain | `id.example.com` |
| `POCKET_ID_CLIENT_ID` | Pocket-ID OIDC client ID | `<client-id>` |
| `POCKET_ID_CLIENT_SECRET` | Pocket-ID OIDC client secret | `<client-secret>` |
| `GRAFANA_ADMIN_USER` | Fallback local admin user | `admin` |
| `GRAFANA_ADMIN_PASSWORD` | Fallback local admin password | `<secure-password>` |
| `GF_AUTH_DISABLE_LOGIN_FORM` | Hide the password login form (SSO only). Set `false` in Portainer to regain local admin login if Pocket-ID is down | `true` |
| `VM_RETENTION_PERIOD` | VictoriaMetrics retention | `1y` |
| `VLOGS_RETENTION_PERIOD` | VictoriaLogs retention | `30d` |

---

## Features & Usage

### 1. Pre-Provisioned Datasources
Grafana starts immediately with the following datasources provisioned:
- **VictoriaMetrics** (Default, Prometheus type) — `http://victoriametrics:8428`
- **VictoriaLogs** (Official `victoriametrics-logs-datasource` plugin) — `http://victorialogs:9428`

### 2. Viewing Container Logs in Grafana
1. Open Grafana → **Explore**.
2. Select **VictoriaLogs** datasource.
3. Query logs using **LogsQL**:
   - Show logs from Portainer:
     ```logsql
     container_name:portainer
     ```
   - Show errors across all containers:
     ```logsql
     _msg:"error" OR _msg:"exception"
     ```
   - Filter by stream (stderr) and container:
     ```logsql
     container_name:portainer AND stream:stderr
     ```

### 3. Automatic Metrics Scraping for Local Containers
`vmagent` is configured with Prometheus `docker_sd_configs`. Any container on the host can opt into metrics scraping simply by adding Docker labels:

```yaml
labels:
  - "prometheus.scrape=true"
  - "prometheus.port=8080"      # Optional: defaults to target port
  - "prometheus.path=/metrics"  # Optional: defaults to /metrics
```

### 4. Direct Metrics Ingestion
Applications or containers on the same network can push metrics directly to VictoriaMetrics using its standard entrypoints:
- Prometheus Remote Write: `http://victoriametrics:8428/api/v1/write`
- Influx line protocol: `http://victoriametrics:8428/write`
- OpenTelemetry: `http://victoriametrics:8428/opentelemetry/api/v1/push`

---

## CI/CD & Image Vendoring

- `.github/workflows/deploy.yml`: Automatically pushes changes from `master` to `deploy` and triggers the Portainer webhook.
- `.github/workflows/vendor-images.yml`: Weekly scheduled job that pulls upstream images from Docker Hub, tags and pushes them to GitHub Container Registry (`ghcr.io`), updates `deploy`, and notifies Portainer.

### Required GitHub Secrets
- `PORTAINER_REDEPLOY_HOOK`: The webhook URL provided by Portainer for this stack.

---

## License

[MIT](file:///LICENSE) &copy; 2026 Korzhavin Ivan
