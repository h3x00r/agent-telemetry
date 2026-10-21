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
- The **Claude Code** dashboard comes from `grafana/provisioning/dashboards/claude-code.json` and is the home page. The **Grok Build** dashboard comes from `grok-build.json` in the same folder. Neither can be saved from the UI: edit the file (Grafana reloads it within a minute), or open it from **Dashboards → Agent telemetry** and use **Save as copy**. Other `*.json` files in that folder load too.
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
- `persona=<name>` is optional; the flow timeline needs it ([Personas](#personas)).
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

## Agent configuration (Grok Build)

Grok Build (`grok`) sends metrics and events (OTLP logs), no traces. A nexus persona gets it from `nexus --config`
(its telemetry action). By hand, set the same in the environment grok starts in:
```sh
export GROK_EXTERNAL_OTEL=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_ENDPOINT=https://telemetry.lair.dev:4318
export OTEL_EXPORTER_OTLP_CERTIFICATE=/etc/otel/otel-ca.crt
H="Authorization=Bearer <OTLP_BEARER_TOKEN>,X-Nexus-Host=$(hostname -s),X-Nexus-Persona=<name>"
export OTEL_EXPORTER_OTLP_METRICS_HEADERS="$H" OTEL_EXPORTER_OTLP_LOGS_HEADERS="$H"
```

- grok ignores `OTEL_RESOURCE_ATTRIBUTES`. The collector takes the VM and persona from the `X-Nexus-Host` and
  `X-Nexus-Persona` headers instead (`resource/nexus` and `attributes/nexus` in `otel-collector/config.yaml`):
  `host.name` on the resource, so the label `host_name` in Prometheus and Loki; `persona` on each datapoint and
  event, so a Prometheus label, and in Loki a field to filter on (`| persona="<name>"`). Only where they are
  missing, so Claude Code's own values stand. Without the headers, grok's data has neither.
- The token goes in the per-signal headers, not `OTEL_EXPORTER_OTLP_HEADERS` (grok reads both), so docker compose
  and tracing in the same shell don't pick it up.
- grok trusts the CA through `OTEL_EXPORTER_OTLP_CERTIFICATE`, added to its usual ones. Not `SSL_CERT_FILE`: that
  replaces grok's trust store for its own API calls too.
- Its metrics are `grok_code_*`, e.g. `grok_code_token_usage_total` and `grok_code_cost_usage_USD_total` (no
  `_tokens` in the name, unlike Claude Code's). The **Grok Build** dashboard shows them; more queries:
  [EXAMPLES.md](EXAMPLES.md#promql).
- Token types overlap, unlike Claude Code's: `input` includes `cache_read`, and `output` includes `reasoning`
  (checked against the cost grok reports). Total tokens = `input` + `output`; cache hit ratio = `cache_read` / `input`.
- `grok_code_session_count_total` carries only `session_id` from grok (no user or model), and not every session sends
  it. To count sessions, count distinct `session_id` on `grok_code_token_usage_total`.
- grok's events have no `event_name` and an empty body. The dashboard tells them apart by the fields only one kind
  has: `stop_reason` for model API calls, `prompt_length` for prompts, `tool_call_count` with `duration_ms` for turn
  ends, `tool_name` with `duration_ms` for tool results.
- A failed export is silent. To see why: `RUST_LOG=debug GROK_LOG_FILE=/tmp/g.log grok -p hi`, then
  `grep "export health" /tmp/g.log` (`export_failures=0 export_successes=2` is good).

## Querying

Every Claude Code or Grok Build session is a new set of series, so use the `anchored` range modifier. Plain `increase()`/`rate()` extrapolate short series, e.g. 1.5 sessions for 1.

Loki index labels are `service_name`, `service_namespace`, `deployment_environment(_name)` and `host_name`. Everything else (`event_name`, `session_id`, `trace_id`, ...) is structured metadata. Grafana links between signals: trace → logs, log `trace_id` → trace, exemplars → trace, plus the service map from Tempo span metrics.

## Work logs and flow timelines

Besides its main dashboard, each agent has a **work log** (`claude-code-work.json`, `grok-build-work.json`) and a **flow timeline** (`claude-code-flow.json`, `grok-build-flow.json`).

- **Work log**: tables, top to bottom: **Sessions**; **Session work** (Claude Code: prompts, tool summary, shell commands, code edits, skills, subagents, compactions; Grok Build: turns, tool summary, file types); **Workflows** (Claude Code); **Tool calls**; **Session setup** (collapsed). Click a session id to set **Session** and narrow the time range to that session, a prompt id to set **Prompt**, a trace id to open the trace. Set **Prompt** back to `.*` to clear it; `.*` also matches events without a `prompt_id`.
- Tables built from several queries (Sessions, Prompts, Turns, Workflows, Grok Build's Tool calls) are joined by a Grafana SQL expression over hidden queries; only Sessions mixes Prometheus and Loki. Each hidden query ends in a placeholder row (`or label_replace(vector(0), ...)`), because an empty input breaks the SQL, and the SQL drops it.
- Limits: event tables show the newest 5000 rows (Loki `max_entries_limit_per_query`). Summary tables fail with a Loki series-limit error above about 1000 sessions, or about 500 prompts or turns with Session = All (Loki `max_query_series`). Narrow the time range or pick a session.
- Claude Code's Tokens, Cost and lines come from Prometheus and follow **Model**; commits, edits and active time come from Prometheus too but have no model label, and prompts, API calls and tool calls come from Loki: all of those ignore it. Prometheus also counts advisor-model tokens, which `api_request` events don't carry, so a session's Tokens exceed the sum of its prompts'.
- **Shell commands** (Claude Code) counts the Tempo span attributes `bash_argv0` (the program, e.g. `git`) and `bash_command_class` with TraceQL metrics. `tempo.yaml` allows those over 14 days (`query_frontend.metrics.max_duration`; Tempo's default is 24h). After pulling this change: `docker compose restart tempo`.
- **Workflows** (Claude Code) groups events by `workflow_run_id`, one per run of Claude Code's Workflow tool; user-defined workflows are named `custom`. Grok Build sends no workflow data.
- **Flow timeline**: an overview lane per persona (green = tools, blue = model only), then a row per persona with lanes for prompts, model calls, each tool, workflows, subagents and compactions (Claude Code) or turn ends (Grok Build), and failed tools, plus session lanes, workflow-run lanes (Claude Code) and its newest 500 events. A mark means at least one event in that time bucket; the bucket size depends on the time range. Session lanes and events link to the work log.

### Personas

The flow timelines repeat a row per persona; data without one shows only as `(none)` in the overview.
- Claude Code: add `persona=<name>` to `OTEL_RESOURCE_ATTRIBUTES`, e.g. `host.name=$(hostname -s),deployment.environment.name=prod,persona=<name>` in README step 2.
- Grok Build ignores `OTEL_RESOURCE_ATTRIBUTES`: send the `X-Nexus-Persona` header ([Agent configuration (Grok Build)](#agent-configuration-grok-build)).

### Optional detail: commands and file paths

Both agents report which tool ran, not what it did. These agent-side settings add more; all are off by default, and each stores more sensitive data in Loki or Tempo:

| Setting | Agent | Adds | Exposes |
|---|---|---|---|
| `OTEL_LOG_TOOL_DETAILS=1` | Claude Code | On tool events: `tool_parameters` (Bash command, MCP server and tool, skill and subagent names, commit id after `git commit`), `tool_input` (file paths, URLs, patterns), the full `error`; real skill names | Commands and paths, which can contain secrets |
| `OTEL_METRICS_INCLUDE_REPOSITORY=true` | Claude Code | Repository URL, owner and name on metrics and events | Repository names |
| `OTEL_LOG_TOOL_DETAILS=1` (or `[telemetry] otel_log_tool_details = true`) | Grok Build | Real tool and MCP names, `file_path`, and a preview of `tool_parameters` that includes shell commands | Commands and paths |
| `OTEL_LOG_TOOL_CONTENT=1` | Both | Claude Code: tool output (file contents read, Bash and web results) on trace spans; Edit/Write content also needs `OTEL_LOG_TOOL_DETAILS`. Grok Build: tool input, output and the full command on events | File contents |
| `OTEL_LOG_USER_PROMPTS=1` | Both | Prompt and response text | Everything typed |

Set them with the other telemetry settings (Claude Code: the managed-settings `env` block; Grok Build: its environment), then restart the agent sessions. The work logs don't show these fields yet: query them in Explore. Loki rejects a record whose structured metadata exceeds 256KB, and the record is lost ([runbook](runbook.md#troubleshooting)).

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
