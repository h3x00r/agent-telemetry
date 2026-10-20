# Reference

How the stack is built, and the details behind the setup. Setup: [README.md](README.md). Procedures: [runbook.md](runbook.md). Queries: [EXAMPLES.md](EXAMPLES.md).

## Services and ports

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

## Compose commands

Run on the server, in the repo directory.
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

## .env and Grafana

`.env` holds the OTLP bearer token, the TLS SANs, the Grafana admin login and Grafana's encryption keys. Back it up, and never change the Grafana keys once Grafana stores secrets.

- The admin credentials apply only on the first start with an empty `grafana-data` volume. After that, see [Reset the Grafana admin password](runbook.md#reset-the-grafana-admin-password).
- The **Claude Code** dashboard comes from `grafana/provisioning/dashboards/claude-code.json` and is the home page. It can't be saved from the UI: edit the file (Grafana reloads it within a minute), or open it from **Dashboards → Agent telemetry** and use **Save as copy**. Other `*.json` files in that folder load too.
- Grafana is HTTPS only; plain http returns 400. To avoid the browser warning, import `otel-ca.crt` as a trusted CA.
- On first start, Grafana downloads the pinned Drilldown apps (Logs, Traces, Metrics) from grafana.com. That needs outbound HTTPS once; after that they live in the `grafana-data` volume.

## TLS and auth

- The `certs` service creates a private CA (10 years, stored only in volume `tls-ca`) and a server cert (825 days) for the names and IPs in `TLS_SANS`. The same cert serves 4317, 4318 and 3000.
- `TLS_SANS` must list every name and IP that agents and browsers use. Keep `DNS:localhost` (Grafana healthcheck) and `DNS:grafana` (Prometheus scrape).
- The server cert is re-issued on `docker compose up -d` when `TLS_SANS` changes or it is within 30 days of expiry. The CA stays the same, so agents keep working. The collector reloads the cert within 24h; Grafana needs `docker compose restart grafana`.
- Agents must send `Authorization: Bearer <OTLP_BEARER_TOKEN>`. Requests without it get 401 (HTTP) or Unauthenticated (gRPC). Plaintext connections are refused. To change it, see [Rotate the OTLP token](runbook.md#rotate-the-otlp-token).

## Agent configuration (Claude Code)

[README step 2](README.md#2-connect-an-agent-vm) writes these settings to `/etc/claude-code/managed-settings.d/50-otel.json`, which applies to every user on the VM and needs no shell env (including `NODE_EXTRA_CA_CERTS`). Managed-settings values are literal (no shell expansion), so `host.name` is rendered per VM when the file is written. Verified with Claude Code 2.1.302.

Without root, set the same values in the user's shell environment instead:
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

- `host.name` is **required**: Claude Code does not set it, and it is what tells VMs apart in Prometheus and Loki labels.
- Trust the CA with `NODE_EXTRA_CA_CERTS`. Claude Code's HTTP exporter ignores `OTEL_EXPORTER_OTLP_CERTIFICATE`: verification fails with "unable to verify the first certificate".
- The header must be exactly `Authorization=Bearer <token>`, with a capital B.
- gRPC must use a **hostname** from `TLS_SANS`, not the IP. Claude Code refuses an IP as the TLS server name ("Setting the TLS ServerName to an IP address is not permitted"). The VM must resolve that name, via DNS or `/etc/hosts`:
  ```sh
  export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
  export OTEL_EXPORTER_OTLP_ENDPOINT=https://telemetry.lair.dev:4317   # e.g. /etc/hosts: 192.0.2.10 telemetry.lair.dev
  ```
- Repo-level `.claude/settings.json` cannot enable exporters. Use managed settings, user env, or a wrapper.
- Prompt, response and tool content are redacted by default (`<REDACTED>`). Leave `OTEL_LOG_USER_PROMPTS` unset unless you want them stored.
- No temporality setting is needed. Claude Code sends delta metrics, and the collector converts them to cumulative.
- Keep `session_id` on metrics (the default). It keeps concurrent sessions apart.
- Defaults: metrics are exported every 60s and logs every 5s. Metrics also flush on exit, which matters for short `claude -p` runs.
- In `-p` (headless) mode, beta tracing sometimes loses the `claude_code.interaction` root span because of a client-side startup race.

Any other OTel SDK works too: see [EXAMPLES.md](EXAMPLES.md#other-otel-sdks).

## Querying

Every Claude Code session is a new set of series, so use the `anchored` range modifier. Plain `increase()`/`rate()` extrapolate short series, e.g. 1.5 sessions for 1.

Loki index labels are `service_name`, `service_namespace`, `deployment_environment(_name)` and `host_name`. Everything else (`event_name`, `session_id`, `trace_id`, ...) is structured metadata. Grafana links between signals: trace → logs, log `trace_id` → trace, exemplars → trace, plus the service map from Tempo span metrics.

## Behaviour and limits

- **Buffering**: the collector keeps a persistent queue (32 MiB per signal, fsync'd) and retries until a backend is back. Data is lost if the queue fills up, if a backend permanently rejects it (HTTP 400), or while the collector itself is down (`down`, upgrades, reboots): agents retry only briefly and do not spool. Watch `otelcol_exporter_queue_size`, `otelcol_exporter_send_failed_*` and `otelcol_receiver_refused_*` (job `otel-collector`).
- **Idle agents**: Claude Code sends a series only when it changes. Prometheus looks back 31 min (`--query.lookback-delta`), longer than the collector's 30 min delta → cumulative window, so per-bucket `increase()` still counts the first change after an idle gap.
- **Delta → cumulative** state lives in the collector's memory. A collector restart, or 30 min without data for a series, looks like a counter reset, which `increase()` handles. Run exactly one collector.
- **Experimental Prometheus features**: `created-timestamp-zero-ingestion` (counts each new series' first export) and `promql-extended-range-selectors` (`anchored`).
- **Retention**: Prometheus 15d or 3GB, Loki 30d, Tempo 14d. Loki and Tempo retention is time-only, so shorten it if ingest grows or the Docker disk is small (see the runbook's "Disk full" row).
- **Memory caps**: collector 1g, Prometheus 1g, Loki 1g, Tempo 1.5g, Grafana 1g.
- **Grafana users**: anyone who can log in to Grafana can reach Loki/Tempo HTTP APIs through the datasource proxy, including Loki's ingester shutdown endpoint. Only give logins to people you trust with the stack.
- **Firewall**: Docker-published ports bypass host `INPUT` rules (ufw). To limit who can reach 4317/4318/3000, use the `DOCKER-USER` chain (host step, optional). `<allowed-cidr>` must cover the agents and the admin workstations:
  `iptables -I DOCKER-USER -i <ext-if> ! -s <allowed-cidr> -p tcp -m multiport --dports 3000,4317,4318 -j DROP`
