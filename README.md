# Docker LGTM Stack

A self-hosted observability stack (Loki, Grafana, Tempo, Prometheus) running entirely with Docker Compose. All backends are exposed through a single nginx reverse-proxy that enforces HTTP Basic Auth, plus an optional [Grafana Alloy](https://grafana.com/docs/alloy/latest/) agent for collecting host metrics, logs, and traces.

## Components

| Service    | Image                                     | Host port(s)                  | Purpose                                          |
| ---------- | ----------------------------------------- | ----------------------------- | ------------------------------------------------ |
| tempo      | `grafana/tempo`                           | `3200`, `4317`, `4318`        | Trace storage + OTLP receiver, behind nginx auth |
| prometheus | `prom/prometheus`                         | `9090`                        | Metrics + remote-write receiver, behind nginx auth |
| loki       | `grafana/loki`                            | `3100`                        | Log storage, behind nginx auth                   |
| grafana    | `grafana/grafana`                         | `3000`                        | Dashboards/UI, auth via `GF_SECURITY_ADMIN_*`    |
| nginx      | `nginxinc/nginx-unprivileged`             | (shares ports above)          | Reverse proxy with Basic Auth gateway            |
| alloy      | `grafana/alloy`                           | `12345`, `4319`, `4320`       | Optional telemetry collector (see `docker-alloy/`) |

## Prerequisites

- Docker Engine with the Compose plugin (e.g. Docker Desktop)
- Ports `3000`, `3200`, `3100`, `9090`, `4317`, `4318` free on the host

## Quick start

```sh
# 1. Configure environment variables (see below)
cp .env.example .env

# 2. Create the htpasswd files (see below)
cp htpasswd/.tempo.htpasswd.example htpasswd/.tempo.htpasswd
cp htpasswd/.prometheus.htpasswd.example htpasswd/.prometheus.htpasswd
cp htpasswd/.loki.htpasswd.example htpasswd/.loki.htpasswd

# 3. Start the stack
docker compose up -d
```

Grafana is then available at <http://localhost:3000> (login with the admin user from `.env`).

> The `htpasswd/*.htpasswd` and `.env` files are **gitignored** — you must create
> them locally from the `.example` files. This keeps credentials out of version control.

## Environment setup

### Grafana (`.env`)

The root `.env` is loaded by the `grafana` service (`required: true`, so the stack
will not start without it). It maps directly to Grafana's env vars:

```sh
# .env (copy from .env.example)
GF_SECURITY_ADMIN_USER=grafana
GF_SECURITY_ADMIN_PASSWORD=change-me
GF_SECURITY_ADMIN_EMAIL=you@example.com
GF_FEATURE_TOGGLES_ENABLE="traceqlEditor metricsSummary"
```

| Variable                    | Meaning                                       |
| --------------------------- | --------------------------------------------- |
| `GF_SECURITY_ADMIN_USER`    | Grafana admin login                           |
| `GF_SECURITY_ADMIN_PASSWORD`| Grafana admin password                        |
| `GF_SECURITY_ADMIN_EMAIL`   | Admin email (for recovery/notifications)      |
| `GF_FEATURE_TOGGLES_ENABLE` | Enables TraceQL editor + metrics summary UI   |

### Alloy (`docker-alloy/.env`) — optional

Alloy only needs a `.env` if you use it. Copy the template and point it at your
stack (via `host.docker.internal`) or at Grafana Cloud:

```sh
cp docker-alloy/.env.example docker-alloy/.env
```

Local OSS targets (the defaults in `.env.example`):

| Variable              | Value / meaning                                          |
| --------------------- | -------------------------------------------------------- |
| `OTELHTTP_ENDPOINT`   | `http://host.docker.internal:4318` (Tempo OTLP HTTP)     |
| `OTEL_ENDPOINT`       | `host.docker.internal:4317` (Tempo OTLP gRPC)            |
| `PROMETHEUS_ENDPOINT` | `http://host.docker.internal:9090/api/v1/write`          |
| `LOKI_ENDPOINT`       | `http://host.docker.internal:3100/loki/api/v1/push`      |
| `*_USERNAME` / `*_PASSWORD` | Basic-auth credentials matching the htpasswd files below |
| `HOSTNAME`            | Label used for the host                                    |

To send to **Grafana Cloud** instead, use the hosted endpoints/credentials. The Alloy service publishes its UI on
`:12345` and its own OTLP receivers on `4319` (gRPC) and `4320` (HTTP), so it can
be used as an ingestion point for apps.

## Creating the htpasswd files

nginx serves Tempo, Prometheus, and Loki behind HTTP Basic Auth. Each service reads
its credentials from a file in `htpasswd/`, mounted into the nginx container at
`/etc/htpasswd/`:

| Service    | htpasswd file                    | Used by the nginx server block |
| ---------- | -------------------------------- | ------------------------------ |
| Tempo      | `htpasswd/.tempo.htpasswd`       | ports `3200`, `4317`, `4318`   |
| Prometheus | `htpasswd/.prometheus.htpasswd`  | port `9090`                    |
| Loki       | `htpasswd/.loki.htpasswd`        | port `3100`                    |

Each file is a single `username:$apr1$hash` line (Apache `apr1`/MD5 format). The
repo ships `.example` templates that document the command:

```sh
# Use openssl passwd -apr1 -table to generate password hash
username:$apr1$9QdPhmBM$EEWGz5vWWEBuAerqzdo///
```

### Generate a hash with `openssl`

```sh
echo -n "your-password" | openssl passwd -apr1 -stdin
# → $apr1$...
```



```sh
echo 'tempo_admin:$apr1$9QdPhmBM$EEWGz5vWWEBuAerqzdo///' > htpasswd/.tempo.htpasswd
```

Repeat for the username/password you want, writing each into the matching
`htpasswd/.tempo.htpasswd`, `htpasswd/.prometheus.htpasswd`, and
`htpasswd/.loki.htpasswd`.

> If you run the Alloy agent, set `*_USERNAME` / `*_PASSWORD` in
> `docker-alloy/.env` to the same credentials used in these files, so its OTLP,
> remote-write, and Loki pushes authenticate successfully.

## Accessing the stack

| Service    | URL                        | Auth                                        |
| ---------- | -------------------------- | ------------------------------------------- |
| Grafana    | http://localhost:3000      | Admin user from `.env`                      |
| Tempo      | http://localhost:3200      | Basic auth (`htpasswd/.tempo.htpasswd`)     |
| Prometheus | http://localhost:9090      | Basic auth (`htpasswd/.prometheus.htpasswd` |
| Loki       | http://localhost:3100      | Basic auth (`htpasswd/.loki.htpasswd`)      |
| Alloy UI   | http://localhost:12345     | None (if Alloy is running)                  |

Grafana is pre-provisioned with datasources for Prometheus, Tempo (streaming
enabled, service-map backed by Prometheus), and Loki, and loads dashboards from
`dashboards/`.

## Sending data to the stack

The nginx gateway authenticates every write endpoint, so clients must pass the
relevant `username:password`:

- **Traces (OTLP)** — `http://localhost:4318` or
  `localhost:4317` (gRPC) with Tempo basic-auth credentials.
- **Metrics (remote write)** — `http://localhost:9090/api/v1/write` with
  Prometheus basic-auth credentials.
- **Logs (Loki push)** — `http://localhost:3100/loki/api/v1/push` with Loki
  basic-auth credentials.

For apps running on the host, use `localhost`; from inside another container, use
`host.docker.internal` (as the Alloy config does).

## Data persistence

Docker volumes keep data across restarts:

| Volume             | Used by  | Holds                             |
| ------------------ | -------- | --------------------------------- |
| `tempo-data`       | tempo    | Trace WAL and blocks              |
| `prometheus-data`  | prometheus | Scraped metrics + TSDB            |
| `loki-data`        | loki     | Chunks, WAL, rules, retention     |

Loki is configured with 45 days retention and 7-day rejection of old samples.
`docker compose down` stops the stack without deleting volumes; add `-v` to
wipe them.

## Useful commands

```sh
docker compose up -d        # start the stack
docker compose logs -f      # follow logs
docker compose ps           # container status
docker compose down         # stop (keep volumes)
docker compose down -v      # stop and delete all data
```
