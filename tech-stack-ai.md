# AI overview

## Config & Environment Files

| Config File | Location | Config Value | What it's for | How it's used |
|---|---|---|---|---|
| `.env` | root | `POSTGRES_DB` | Name of the Postgres database created inside the `database` container | Read by the Postgres image on first boot to create the initial database; also referenced in the `healthcheck` command in `docker-compose.yml` |
| `.env` | root | `POSTGRES_USER` | Username for the Postgres superuser created on first boot | Used by Postgres to create the role, by `postgres_exporter` to authenticate, and interpolated into `DATA_SOURCE_NAME` |
| `.env` | root | `POSTGRES_PASSWORD` | Password for `POSTGRES_USER` | Used for authentication by any service connecting to Postgres (API, `postgres_exporter`); interpolated into `DATA_SOURCE_NAME` |
| `.env` | root | `DATA_SOURCE_NAME` | Full Postgres connection URL built from the three values above | Consumed by `postgres_exporter` (its standard env var name) to know which database to connect to and scrape metrics from |
| `docker-compose.yml` | root | `database.ports` | Host↔container port mapping for Postgres (`5433:5432`) | Lets tools on the host (e.g. a local DB client) reach Postgres on `localhost:5433`, while other containers still use the internal port `5432` |
| `docker-compose.yml` | root | `api.env_file` | Points the `api` service at the sibling `learn-ops-api` repo's own `.env` | Docker Compose loads that file's variables into the `api` container's environment at startup |
| `docker-compose.yml` | root | `client.env_file` | Points the `client` service at the sibling `learn-ops-client` repo's own `.env` | Docker Compose loads that file's variables into the `client` container's environment at startup |
| `docker-compose.yml` | root | `prometheus.volumes` | Bind-mounts the local `prometheus.yml` into the Prometheus container | Prometheus reads this mounted file as its scrape configuration on startup |
| `docker-compose.yml` | root | `networks.learningplatform` | Declares one shared Docker network named `learningplatform` | Lets services address each other by service name (e.g. `database`, `api`) instead of IP, since they all join this network |
| `prometheus.yml` | root | `global.scrape_interval` | How often Prometheus polls every target for metrics (`15s`) | Applied as the default polling frequency across all `scrape_configs` unless a job overrides it |
| `prometheus.yml` | root | `scrape_configs[django].metrics_path` | Custom metrics endpoint path exposed by the Django API (`/metrics/metrics`) | Prometheus calls this path on the `api` target instead of the default `/metrics` |
| `prometheus.yml` | root | `scrape_configs[django].targets` | The scrape target for the Django job (`api:8000`) | Tells Prometheus which host:port to poll for Django metrics, using the Docker network service name |
| `prometheus.yml` | root | `scrape_configs[postgresql].targets` | The scrape target for the Postgres job (`postgres_exporter:9187`) | Tells Prometheus to poll `postgres_exporter`, which translates Postgres internals into Prometheus-readable metrics |
| `valkey/docker-compose.yml` | `valkey/` | `valkey.ports` | Host↔container port mapping for Valkey (`6379:6379`) | Exposes the Redis-compatible Valkey server on the host for local access/debugging |
| `valkey/docker-compose.yml` | `valkey/` | `valkey.command` (`--save 900 1`, `--loglevel notice`) | Persistence and logging flags passed to `valkey-server` | Tells Valkey to snapshot to disk if at least 1 key changed in 900s, and to log at `notice` verbosity |
| `valkey/docker-compose.yml` | `valkey/` | `valkey.volumes` (`valkey-data`) | Named volume for Valkey's data directory (`/data`) | Persists Valkey's snapshot files across container restarts/recreation |
| `valkey/docker-compose.yml` | `valkey/` | `valkey-monitor.entrypoint` | Runs `valkey-cli -h valkey monitor` as the container's process | Starts a standing debug container that streams every command Valkey processes in real time, for observability |

## Starting the System

All start-up is driven through the root `Makefile`, which wraps `docker compose` (and, for first-time setup, `scripts/setup.sh`). The targets relevant to getting the system running, in the order you'd typically reach for them:

| Target | Command it runs | What it starts | When to use it |
|---|---|---|---|
| `make setup` | `./scripts/setup.sh` | Not a service starter itself — the first-time wizard that clones sibling repos, collects secrets, writes `.env` files, and *optionally* offers to run `docker compose up` at the end | Once, on a brand-new machine, before any `up*` target will work |
| `make doctor` | `./scripts/setup.sh --doctor` | Nothing — runs the same wizard in check-only mode | To verify prerequisites (git/docker/python/make, `.env` files present) without changing anything |
| `make up` | `pull` (dependency), then `docker compose up --build -d`, then `docker compose logs -f` | **Every** service in `docker-compose.yml`: `database`, `api`, `client`, `prometheus`, `grafana`, `postgres_exporter` | Full local stack, including metrics/dashboards — the default "just start everything" command |
| `make up-api` | `docker compose up --build -d api`, then `docker compose logs -f api` | Only `api` — plus `database`, since `api` declares `depends_on: database` in `docker-compose.yml` | Backend-only work where you don't need the React client or the observability stack |
| `make up-client-api` | `docker compose up --build -d api client`, then `docker compose logs -f api client` | `api`, `client`, and (transitively) `database` | Full-stack app work (frontend + backend) without paying the cost of building/running Prometheus/Grafana/exporter |
| `make restart` | `pull` (dependency), `docker compose down`, then `docker compose up --build -d`, then `docker compose logs -f` | Stops everything, then starts the **full** stack (same scope as `make up`) | Recovering from a broken state or picking up new images/config without a manual `down` + `up` |

**How they differ:**
- **Scope** is the main axis: `up` starts everything; `up-api` and `up-client-api` start progressively larger subsets, relying on `depends_on` in `docker-compose.yml` to pull in `database` automatically. Neither `up-api` nor `up-client-api` starts `prometheus`, `grafana`, or `postgres_exporter`.
- **`pull`** isn't a starting target on its own — it only fetches images (`docker compose pull --ignore-buildable`) and is listed as a prerequisite of `up` and `restart`, so those two always fetch the latest pullable images first; `up-api` and `up-client-api` skip that step and just build/start directly.
- **`restart` vs `up`**: functionally `restart` starts the same full stack as `up`, but it forces a `docker compose down` first, so it's the right choice when containers are already running in a bad state, whereas `up` just brings things up (and will no-op/reuse containers that are already healthy).
- **`setup`/`doctor`** aren't part of the day-to-day start cycle — they're onboarding/diagnostic steps that exist to make sure the `.env` files and sibling repos that `up*` targets depend on are actually in place first.

Note: none of these targets start `valkey/docker-compose.yml` — that compose file lives in its own directory and isn't referenced anywhere in the root `Makefile` or root `docker-compose.yml`, so the Valkey broker has to be started separately (e.g. `docker compose -f valkey/docker-compose.yml up -d`).

## Service Ports & URLs

| Service | Port | URL |
|---|---|---|
| `database` (Postgres) | `5433` (host) → `5432` (container) | `postgresql://localhost:5433` |
| `api` (Django) | `8000` | http://localhost:8000 |
| `api` debugger (debugpy) | `5678` | N/A — remote-debugger attach port, not an HTTP endpoint |
| `client` (React) | `3000` | http://localhost:3000 |
| `prometheus` | `9090` | http://localhost:9090 |
| `grafana` | `3001` (host) → `3000` (container) | http://localhost:3001 |
| `postgres_exporter` | `9187` | http://localhost:9187/metrics |
| `valkey` | `6379` | `redis://localhost:6379` |
| `valkey-monitor` | none exposed | N/A — no `ports` mapping in `valkey/docker-compose.yml`; it's a debug sidecar reachable only inside the `learningplatform` network |

Ports/URLs sourced from the `ports:` entries in `docker-compose.yml` (root) and `valkey/docker-compose.yml`. `database` and `valkey` are non-HTTP protocols (Postgres wire protocol and Redis protocol respectively), so their "URL" is a connection string, not something you'd open in a browser.

## Service Dependencies

These are the `depends_on` relationships declared in `docker-compose.yml` (root) and `valkey/docker-compose.yml`:

| Service | Depends On | Why |
|---|---|---|
| `api` | `database` (`condition: service_healthy`) | Django's ORM opens a real connection to Postgres via `DATA_SOURCE_NAME`/DB settings on every request that touches the database (auth, courses, cohorts, etc.), and runs migrations against it on startup. The `service_healthy` condition means Compose waits for `database`'s `pg_isready` healthcheck to pass before starting `api` at all — otherwise the first connection attempt would fail. |
| `prometheus` | `api` | Per `prometheus.yml`, the `django` scrape job polls `api:8000/metrics/metrics` every 15s to pull application metrics. `depends_on` here has no health condition, so it only guarantees the `api` container has *started* (not that Django is actually serving yet) before `prometheus` starts. |
| `grafana` | `prometheus` | Grafana is configured to query Prometheus as its metrics data source, issuing PromQL queries against it (internally at `prometheus:9090`) to render dashboard panels. Without Prometheus up, Grafana has no backend to query, though again `depends_on` here just waits for container start, not for Prometheus to be ready to answer queries. |
| `postgres_exporter` | `database` | `postgres_exporter` reads `DATA_SOURCE_NAME` from the shared `.env` and opens its own connection to Postgres to read internal stats (e.g. `pg_stat_activity`), translating them into Prometheus-format metrics it exposes on `:9187` for `prometheus` to later scrape. |
| `valkey-monitor` | `valkey` | Its entrypoint runs `valkey-cli -h valkey monitor`, connecting directly to the `valkey` hostname to stream every command Valkey processes in real time. There's nothing to attach to — and the container would just fail to connect — if `valkey` isn't already up. |

**Real dependencies not captured by `depends_on`** — worth knowing about since they can bite you even when Compose reports everything "started":

- **`prometheus` → `postgres_exporter`**: the `postgresql` scrape job in `prometheus.yml` polls `postgres_exporter:9187`, but `prometheus`'s `depends_on` in `docker-compose.yml` only lists `api`, not `postgres_exporter`. If `postgres_exporter` is slow to start, Prometheus just logs failed scrapes for that job rather than waiting.
- **`client` → `api`**: the React app calls the Django API for all its data (per the system diagram in `README.md`: `UI <--> API`), but `client` has no `depends_on` entry in `docker-compose.yml` at all. It will start regardless of whether `api` is ready, so the UI can come up before there's anything to talk to.

## Entry Points

`api` and `client` are the only services in this system with custom application source — their code lives in the sibling repos `learn-ops-api` and `learn-ops-client` (pulled in via `build.context: ../learn-ops-api` / `../learn-ops-client` in `docker-compose.yml`), not in `learn-ops-infrastructure` itself. Paths below are relative to each service's own repo.

| Service | Startup File | Routes / URL Config File |
|---|---|---|
| `api` | `manage.py` — Django's CLI entry point; the container's `CMD` runs `python3 manage.py runserver 0.0.0.0:8000` (wrapped by `entrypoint.sh`, which runs migrations/fixtures first, then hands off via `exec "$@"`) | `LearningPlatform/urls.py` — the Django `ROOT_URLCONF` (set in `LearningPlatform/settings.py`) that Django loads to resolve every incoming request to a view |
| `client` | `src/index.js` — the React entry module that `react-scripts start` (the `npm start` script run by the container's `CMD`) mounts into `public/index.html`'s root DOM node | `src/components/ApplicationViews.js` — defines the app's `<Route>` tree (via `react-router-dom`), mapping URL paths like `/`, `/cohorts`, `/courses` to their view components |
| `database` | N/A | N/A |
| `prometheus` | N/A | N/A |
| `grafana` | N/A | N/A |
| `postgres_exporter` | N/A | N/A |
| `valkey` | N/A | N/A |
| `valkey-monitor` | N/A | N/A |

The last six run unmodified off-the-shelf images (`postgres:16`, `prom/prometheus`, `grafana/grafana`, `postgres-exporter`, `valkey/valkey`) — there's no application source for them anywhere in this system, so there's no startup file or routes file to point to. Their behavior comes entirely from image defaults plus the config files already covered above (e.g. `prometheus.yml`), not from code.

## Services

| Service Name | Tech Stack (including version) | Purpose |
|---|---|---|
| `database` | PostgreSQL 16 (`postgres:16` image) | Stores all persistent application data — users, courses, cohorts, assessments, etc. |
| `api` | Python 3.11.11, Django 5.2.17, Django REST Framework 3.18.0 (from `learn-ops-api`'s `Pipfile.lock`; also `django-allauth` 0.54.0 / `dj-rest-auth` 4.0.1 for GitHub OAuth) | Backend REST API — business logic, GitHub OAuth authentication, and all reads/writes to `database` |
| `client` | Node.js 22.13.0, React 16.13.1, React Router 5.2.0 (from `learn-ops-client`'s `package.json`) | Frontend web app instructors and students use to track progress, manage cohorts/courses, etc., talking to `api` over HTTP |
| `prometheus` | Prometheus (`prom/prometheus:latest` — no version pinned in this repo) | Scrapes and stores time-series metrics from `api` (`/metrics/metrics`) and `postgres_exporter`, per `prometheus.yml` |
| `grafana` | Grafana (`grafana/grafana:latest` — no version pinned in this repo) | Queries Prometheus and renders metrics as dashboards |
| `postgres_exporter` | Prometheus Postgres Exporter (`quay.io/prometheuscommunity/postgres-exporter:latest` — no version pinned in this repo) | Connects to `database` and translates its internal stats into a Prometheus-scrapable `/metrics` endpoint |
| `valkey` | Valkey (`valkey/valkey:latest` — no version pinned in this repo) | Redis-compatible in-memory data store / message broker (per `README.md`'s system diagram, used as the broker between `api` and other services like Monarch/Hashtagger) |
| `valkey-monitor` | Valkey CLI (`valkey/valkey:latest` image, running `valkey-cli monitor`) | Debug sidecar that streams every command processed by `valkey` in real time — not part of the application, just observability tooling |

Versions for `api` and `client` come from their own repos' lockfiles (`Pipfile.lock`, `package.json`), since that's where their dependencies are actually pinned. The four monitoring/data-store images (`prometheus`, `grafana`, `postgres_exporter`, `valkey`) are all pulled with the `latest` tag in the compose files, so their exact version isn't reproducible from this repo alone — worth pinning if version stability ever matters here.

