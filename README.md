# OTel ingestion stack

```
agents (OTel SDK, other VMs) ──OTLP──▶ otel-collector ──▶ Prometheus (metrics) / Loki (logs) / Tempo (traces) ──▶ Grafana
```

Everything runs in Docker. The host needs nothing besides Docker. Data lives in named volumes (`otel_*`), and the configs in this directory are mounted read-only.

| Service | Image | Published |
|---|---|---|
| otel-collector | `otel/opentelemetry-collector-contrib:0.162.0` | `0.0.0.0:4317` (OTLP gRPC), `0.0.0.0:4318` (OTLP HTTP), both TLS + bearer token |
| prometheus | `prom/prometheus:v3.15.0` | internal only |
| loki | `grafana/loki:3.7.8` | internal only |
| tempo | `grafana/tempo:3.1.0` | internal only |
| grafana | `grafana/grafana:13.2.3` | `0.0.0.0:3000` (HTTPS) |
| init | `busybox:1.36.1-musl` | one-shot: fixes the collector queue volume and provides a probe binary for healthchecks |
| certs | built locally from `alpine:3.24.2` + openssl | one-shot: private CA and server cert (volumes `tls-ca`, `tls`) |

## Boot up

The examples use the host name `telemetry.lair.dev` and the IPv4 address `192.0.2.10`. Replace both with your own everywhere, including runbook.md and EXAMPLES.md.

1. Point the name at the host: a DNS `A` record, or `192.0.2.10 telemetry.lair.dev` in each agent's `/etc/hosts`.
2. In this directory, create `.env`. Docker is the only tool needed; secrets come from a throwaway container.
```sh
rnd() { docker run --rm busybox:1.36.1-musl sh -c 'head -c 32 /dev/urandom | od -An -tx1 | tr -d " \n"'; }
cat > .env <<EOF
TLS_SANS=IP:192.0.2.10,DNS:telemetry.lair.dev,DNS:localhost,DNS:grafana
OTLP_BEARER_TOKEN=$(rnd)
GRAFANA_ROOT_URL=https://telemetry.lair.dev:3000/
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=$(rnd)
GRAFANA_SECRET_KEY=$(rnd)
GRAFANA_SECRETS_KEK=$(rnd)
EOF
chmod 600 .env
```
`TLS_SANS` must list every name and IP that agents and browsers use. Keep `DNS:localhost` (Grafana healthcheck) and `DNS:grafana` (Prometheus scrape).

3. Start the stack and export the CA:
```sh
docker compose up -d --wait
docker compose ps -a        # 5 services "healthy"; init and certs "Exited (0)"
docker compose run --rm -T --no-deps certs cat /tls/ca.crt > otel-ca.crt   # CA for agents and browsers
```
4. Open `https://telemetry.lair.dev:3000` (or `https://192.0.2.10:3000`) and log in with the admin credentials from `.env`. Then [wire agents](#wiring-agents-claude-code) and run the [smoke test](EXAMPLES.md#smoke-test).

## Operate

Procedures (health check, rotation, backup, upgrade, troubleshooting) are in [runbook.md](runbook.md). Test payloads and queries are in [EXAMPLES.md](EXAMPLES.md).

```sh
docker compose up -d --wait          # start / apply compose changes
docker compose ps -a
docker compose logs -f otel-collector
docker compose up -d <svc>           # (re)start one service (not "start": it fails if the exited init container was pruned)
docker compose restart <svc>         # pick up an edited config file (mounted dirs are read on start)
docker compose kill -s SIGHUP prometheus   # reload prometheus.yml without a restart
docker compose down                  # stop, keep data
docker compose down -v               # stop and wipe ALL data (including the CA: agents then need the new ca.crt)
docker compose run --rm -T --no-deps certs cat /tls/ca.crt > otel-ca.crt   # export the CA cert for agents/browsers
```

Grafana: `https://telemetry.lair.dev:3000` (or `https://192.0.2.10:3000`). Plain http returns 400. To avoid the browser warning, import `otel-ca.crt` as a trusted CA. The admin credentials are in `.env`. They apply only on the first start with an empty `grafana-data` volume; after that, reset with `docker compose exec grafana grafana cli admin reset-admin-password <new>`.
`.env` also holds the OTLP bearer token, the TLS SANs and Grafana's encryption keys. Back it up, and never change the Grafana keys once Grafana stores secrets.

## TLS and auth

- The `certs` service creates a private CA (10 years, stored only in volume `tls-ca`) and a server cert (825 days) for the names and IPs in `TLS_SANS`. The same cert serves 4317, 4318 and 3000.
- The server cert is re-issued on `docker compose up -d` when `TLS_SANS` changes or it is within 30 days of expiry. The CA stays the same, so agents keep working. The collector reloads the cert within 24h; Grafana needs `docker compose restart grafana`.
- Agents must send `Authorization: Bearer <OTLP_BEARER_TOKEN>`. Requests without it get 401 (HTTP) or Unauthenticated (gRPC). Plaintext connections are refused. To rotate the token: edit `.env`, run `docker compose up -d`, then update the agents.

On first start, Grafana downloads the pinned Drilldown apps (Logs, Traces, Metrics) from grafana.com. That needs outbound HTTPS once; after that they live in the `grafana-data` volume.

## Wiring agents (Claude Code)

Set these in each remote VM's environment, or in its managed settings. Verified with Claude Code 2.1.302.
1. Copy `otel-ca.crt` (see Operate) to each VM, e.g. `/etc/otel/otel-ca.crt`.
2. Get the token: `OTLP_BEARER_TOKEN` in `.env`.
3. `host.name` is **required**: Claude Code does not set it, and it is what tells VMs apart in Prometheus and Loki labels.

HTTP/protobuf (recommended; works with the name or the IP):
```sh
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_TRACES_EXPORTER=otlp                 # traces (beta) need this AND the next line
export CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_ENDPOINT=https://telemetry.lair.dev:4318   # or https://192.0.2.10:4318
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <OTLP_BEARER_TOKEN>"
export NODE_EXTRA_CA_CERTS=/etc/otel/otel-ca.crt
export OTEL_RESOURCE_ATTRIBUTES="host.name=$(hostname -s),deployment.environment.name=prod"
```
Trust the CA with `NODE_EXTRA_CA_CERTS`. Claude Code's HTTP exporter ignores `OTEL_EXPORTER_OTLP_CERTIFICATE`: verification fails with "unable to verify the first certificate". The header must be exactly `Authorization=Bearer <token>`, with a capital B.

gRPC: same as above, except it must use a **hostname** from `TLS_SANS`, not the IP. Claude Code refuses an IP as the TLS server name ("Setting the TLS ServerName to an IP address is not permitted"). The VM must resolve that name, via DNS or `/etc/hosts`:
```sh
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=https://telemetry.lair.dev:4317   # e.g. /etc/hosts: 192.0.2.10 telemetry.lair.dev
```

Managed settings: `/etc/claude-code/managed-settings.json`, or a drop-in such as `/etc/claude-code/managed-settings.d/50-otel.json`. Values are literal (no shell expansion), so render `host.name` per VM. Verified: this block works on its own, including `NODE_EXTRA_CA_CERTS`, with no shell env:
```json
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_TRACES_EXPORTER": "otlp",
    "CLAUDE_CODE_ENHANCED_TELEMETRY_BETA": "1",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "https://telemetry.lair.dev:4318",
    "OTEL_EXPORTER_OTLP_HEADERS": "Authorization=Bearer <OTLP_BEARER_TOKEN>",
    "NODE_EXTRA_CA_CERTS": "/etc/otel/otel-ca.crt",
    "OTEL_RESOURCE_ATTRIBUTES": "host.name=<THIS-VM-HOSTNAME>,deployment.environment.name=prod"
  }
}
```

Notes:
- Repo-level `.claude/settings.json` cannot enable exporters. Use user env, managed settings, or a wrapper.
- Prompt, response and tool content are redacted by default (`<REDACTED>`). Leave `OTEL_LOG_USER_PROMPTS` unset unless you want them stored.
- No temporality setting is needed. Claude Code sends delta metrics, and the collector converts them to cumulative.
- Keep `session_id` on metrics (the default). It keeps concurrent sessions apart.
- Defaults: metrics are exported every 60s and logs every 5s. Metrics also flush on exit, which matters for short `claude -p` runs.
- In `-p` (headless) mode, beta tracing sometimes loses the `claude_code.interaction` root span because of a client-side startup race.

Any other OTel SDK works too. Use `https://...:4317` (grpc) or `https://...:4318` (http/protobuf), the same `OTEL_EXPORTER_OTLP_HEADERS`, and trust the CA (`OTEL_EXPORTER_OTLP_CERTIFICATE` for most SDKs). Also set `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES=host.name=...`.

## Querying

Every Claude Code session is a new set of series, so use the `anchored` range modifier. Plain `increase()`/`rate()` extrapolate short series, e.g. 1.5 sessions for 1.
```promql
sum by (host_name) (increase(claude_code_session_count_total[$__range] anchored))
sum by (host_name, model) (increase(claude_code_cost_usage_USD_total[$__range] anchored))
sum by (type) (increase(claude_code_token_usage_tokens_total[$__range] anchored))
```
```logql
{service_name="claude-code"} | event_name="user_prompt"
sum by (host_name, model) (sum_over_time({service_name="claude-code"} | event_name="api_request" | unwrap cost_usd [$__range]))
```
Loki index labels are `service_name`, `service_namespace`, `deployment_environment(_name)` and `host_name`. Everything else (`event_name`, `session_id`, `trace_id`, ...) is structured metadata. Grafana links between signals: trace → logs, log `trace_id` → trace, exemplars → trace, plus the service map from Tempo span metrics.

## Behaviour and limits

- **Buffering**: the collector keeps a persistent queue (32 MiB per signal, fsync'd) and retries until a backend is back. Data is lost only if the queue fills up, or if a backend permanently rejects it (HTTP 400). Watch `otelcol_exporter_queue_size`, `otelcol_exporter_send_failed_*` and `otelcol_receiver_refused_*` (job `otel-collector`).
- **Delta → cumulative** state lives in the collector's memory. A collector restart, or 30 min without data for a series, looks like a counter reset, which `increase()` handles. Run exactly one collector.
- **Experimental Prometheus features**: `created-timestamp-zero-ingestion` (counts each new series' first export) and `promql-extended-range-selectors` (`anchored`).
- **Retention**: Prometheus 15d or 3GB, Loki 30d, Tempo 14d. Loki and Tempo retention is time-only, so shorten it if ingest grows or the Docker disk is small (see the runbook's "Disk full" row).
- **Memory caps**: collector 1g, Prometheus 1g, Loki 1g, Tempo 1.5g, Grafana 1g.
- **Grafana users**: anyone who can log in to Grafana can reach Loki/Tempo HTTP APIs through the datasource proxy, including Loki's ingester shutdown endpoint. Only give logins to people you trust with the stack.
- **Firewall**: Docker-published ports bypass host `INPUT` rules (ufw). To limit who can reach 4317/4318/3000, use the `DOCKER-USER` chain (host step, optional):
  `iptables -I DOCKER-USER -i <ext-if> ! -s <agent-cidr> -p tcp -m multiport --dports 3000,4317,4318 -j DROP`
