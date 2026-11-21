# observability

Loki + Grafana in Docker Compose. Logs are shipped to Loki by Grafana Alloy (configured in the
`thstudiov2` repo, `infrastructure/alloy/config.alloy`).

Everything goes through one nginx container, `gateway`. Loki and Grafana publish no ports.

| | Local PC | VPS |
|---|---|---|
| `INGEST_BIND` | `127.0.0.1` | `10.8.0.4` (WireGuard address) |
| `UI_BIND` | `127.0.0.1` | `127.0.0.1` (keep it: never on the tunnel) |
| Ingest listener (`:3100` -> Loki) | `localhost:3100` | `10.8.0.4:3100`, reachable only through the tunnel |
| Grafana UI listener (`:3000` -> Grafana) | `localhost:3001` | `127.0.0.1:3001` on the VPS, via `ssh -L 3001:127.0.0.1:3001 obs-vps` |

## Run

```bash
cp .env.example .env        # set GRAFANA_ADMIN_PASSWORD (and INGEST_BIND on the VPS)
docker compose up -d
curl localhost:3100/ready   # "ready" after ~15 s
```

Data lives in `./data` (git-ignored). Loki runs as uid `10001` and Grafana as uid `472`, so on a
fresh Linux host:

```bash
mkdir -p data/loki data/grafana
sudo chown -R 10001:10001 data/loki
sudo chown -R 472:472 data/grafana
```

## Gateway

`gateway` (nginx, `nginx/default.conf`) is the only container that publishes ports, one listener per
purpose:

- **Ingest `:3100`** -> Loki, published on `INGEST_BIND`. Allows just `POST /loki/api/v1/push` (what Alloy
  sends) and `GET /ready`; everything else returns 403, so a compromised peer can add log lines but
  cannot query, delete or read config.
- **UI `:3000`** -> Grafana, published on `UI_BIND:3001` (default `127.0.0.1`). Never set it to the
  tunnel address, so a compromised peer cannot even reach the login page. On the VPS you reach it
  with `ssh -L`.

Grafana queries Loki directly over the compose network, not through the gateway. To add a service,
give it a listener in `nginx/default.conf` and a `ports:` entry on `gateway`, never on the service.

Consequence: `curl localhost:3100/loki/api/v1/query...` no longer works from the host. Query in
Grafana (Explore), or from inside the network:

```bash
docker run --rm --network observability_loki curlimages/curl -s http://loki:3100/ready
```

## Settings (`.env`)

Compose reads `.env` for `compose.yaml` only. Loki settings reach `loki/config.yaml` because Loki
runs with `-config.expand-env=true` and compose forwards the variables into the container. The
defaults live in `loki/config.yaml` (`${VAR:-default}`); an empty or missing `.env` line uses them.

| Variable | Default | What it does |
|---|---|---|
| `GRAFANA_ADMIN_PASSWORD` | none, required | First-run Grafana admin password; Compose refuses to start without it |
| `INGEST_BIND` | `127.0.0.1` | Host address of the ingest port. VPS: `10.8.0.4` |
| `UI_BIND` | `127.0.0.1` | Host address of the Grafana UI. Keep it |
| `LOKI_RETENTION` | `720h` (30 d) | How long logs are kept; size it to the disk |
| `LOKI_REJECT_OLD_SAMPLES_MAX_AGE` | `168h` (7 d) | Refuse lines older than this at write time |
| `LOKI_INGESTION_RATE_MB` | `4` | MB/s accepted |
| `LOKI_INGESTION_BURST_MB` | `6` | Allowed burst, MB |
| `LOKI_PER_STREAM_RATE_LIMIT` | `3MB` | Per-stream rate per second |

Apply a change with `docker compose up -d` (recreates Loki). Image versions stay pinned in
`compose.yaml` on purpose. A literal `$` in `loki/config.yaml` must be written `$$`.

## Configuration

- `loki/config.yaml`: the image's stock config plus 30-day retention (`compactor.retention_enabled`),
  which the stock config lacks, so the disk is not filled indefinitely.
- `grafana/provisioning/datasources/loki.yaml`: creates the Loki data source automatically.
- The compose network is named `loki` on purpose: the a11studio local stack joins
  `observability_loki`. Do not rename it.

## VPS update

```bash
ssh -t obs-vps 'cd /opt/observability && git pull --ff-only && sudo docker compose up -d'
```
