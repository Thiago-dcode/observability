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
