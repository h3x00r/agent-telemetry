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

In a nexus persona, [README step 2](README.md#2-connect-an-agent-vm) is all it takes: `nexus --config` writes these settings into a managed-settings drop-in that it mounts into the persona, with `host.name` (the VM) and `persona` as resource attributes, and no `deployment.environment.name` (the **Environment** filter then offers only All).

Without nexus, as root on the VM, in the directory holding `otel-ca.crt`. This writes the settings to a managed-settings drop-in, which applies to every user on the VM and needs no shell env (including `NODE_EXTRA_CA_CERTS`). Managed-settings values are literal (no shell expansion), so `host.name` is rendered per VM when the file is written. Verified with Claude Code 2.1.302.
```sh
TOKEN='<OTLP_BEARER_TOKEN>'
install -D -m 0644 otel-ca.crt /etc/otel/otel-ca.crt
install -d /etc/claude-code/managed-settings.d
cat > /etc/claude-code/managed-settings.d/50-otel.json <<EOF
{
  "env": {
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_TRACES_EXPORTER": "otlp",
    "CLAUDE_CODE_ENHANCED_TELEMETRY_BETA": "1",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf",
    "OTEL_EXPORTER_OTLP_ENDPOINT": "https://telemetry.lair.dev:4318",
    "OTEL_EXPORTER_OTLP_HEADERS": "Authorization=Bearer $TOKEN",
    "NODE_EXTRA_CA_CERTS": "/etc/otel/otel-ca.crt",
    "OTEL_RESOURCE_ATTRIBUTES": "host.name=$(hostname -s),deployment.environment.name=prod"
  }
}
EOF
```
Restart Claude Code sessions that were already running. Check with `claude -p hi`, as any user.

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
([README step 2](README.md#2-connect-an-agent-vm)). By hand, set the same in the environment grok starts in:
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

Besides its main dashboard, each agent has a **work log** (`claude-code-work.json`, `grok-build-work.json`), a **flow timeline** (`claude-code-flow.json`, `grok-build-flow.json`) and a **conversations** dashboard ([Conversations](#conversations)).

- **Work log**: tables, top to bottom: **Sessions**; **Session work** (Claude Code: prompts, tool summary, shell commands, code edits, skills, subagents, compactions; Grok Build: turns, tool summary, file types); **Workflows** (Claude Code); **Tool calls**; **Session setup** (collapsed). Click a session id to set **Session** and narrow the time range to that session, a prompt id to set **Prompt**, a trace id to open the trace. Set **Prompt** back to `.*` to clear it; `.*` also matches events without a `prompt_id`.
- Tables built from several queries (Sessions, Prompts, Turns, Workflows, Grok Build's Tool calls) are joined by a Grafana SQL expression over hidden queries; only Sessions mixes Prometheus and Loki. Each hidden query ends in a placeholder row (`or label_replace(vector(0), ...)`), because an empty input breaks the SQL, and the SQL drops it.
- Limits: event tables show the newest 5000 rows (Loki `max_entries_limit_per_query`). Summary tables fail with a Loki series-limit error above about 1000 sessions, or about 500 prompts or turns with Session = All (Loki `max_query_series`). Narrow the time range or pick a session.
- Claude Code's Tokens, Cost and lines come from Prometheus and follow **Model**; commits, edits and active time come from Prometheus too but have no model label, and prompts, API calls and tool calls come from Loki: all of those ignore it. Prometheus also counts advisor-model tokens, which `api_request` events don't carry, so a session's Tokens exceed the sum of its prompts'.
- **Shell commands** (Claude Code) counts the Tempo span attributes `bash_argv0` (the program, e.g. `git`) and `bash_command_class` with TraceQL metrics, and `full_command` with tool details on ([Optional detail](#optional-detail-commands-and-file-paths)). `tempo.yaml` allows those over 14 days (`query_frontend.metrics.max_duration`; Tempo's default is 24h). After pulling this change: `docker compose restart tempo`.
- **Workflows** (Claude Code) groups events by `workflow_run_id`, one per run of Claude Code's Workflow tool; user-defined workflows are named `custom`. Grok Build sends no workflow data.
- **Flow timeline**: an overview lane per persona (green = tools, blue = model only), then a row per persona with lanes for prompts, model calls, each tool, workflows, subagents and compactions (Claude Code) or turn ends (Grok Build), and failed tools, plus session lanes, workflow-run lanes (Claude Code) and its newest 500 events. A mark means at least one event in that time bucket; the bucket size depends on the time range. Session lanes and events link to the work log.

### Personas

The flow timelines repeat a row per persona; data without one shows only as `(none)` in the overview. nexus sets the persona for both agents. By hand:
- Claude Code: add `persona=<name>` to `OTEL_RESOURCE_ATTRIBUTES`, e.g. `host.name=$(hostname -s),deployment.environment.name=prod,persona=<name>` ([Agent configuration (Claude Code)](#agent-configuration-claude-code)).
- Grok Build ignores `OTEL_RESOURCE_ATTRIBUTES`: send the `X-Nexus-Persona` header ([Agent configuration (Grok Build)](#agent-configuration-grok-build)).

### Optional detail: commands and file paths

Both agents report which tool ran, not what it did. These agent-side settings add more; all are off by default, and each stores more sensitive data in Loki or Tempo:

| Setting | Agent | Adds | Exposes |
|---|---|---|---|
| `OTEL_LOG_TOOL_DETAILS=1` | Claude Code | On tool events: `tool_parameters` (Bash command, MCP server and tool, skill and subagent names, commit id after `git commit`), `tool_input` (file paths, URLs, patterns), the full `error`; on tool spans: `full_command`, `file_path`, skill and subagent names; real skill names | Commands and paths, which can contain secrets |
| `OTEL_METRICS_INCLUDE_REPOSITORY=true` | Claude Code | Repository URL, owner and name on metrics and events | Repository names |
| `OTEL_LOG_TOOL_DETAILS=1` (or `[telemetry] otel_log_tool_details = true`) | Grok Build | Real tool, MCP and skill names, `file_path`, and a preview of `tool_parameters` (up to 4 KB) that includes shell commands | Commands and paths |
| `OTEL_LOG_TOOL_CONTENT=1` | Both | Claude Code: a `tool.output` event on its trace spans with what the tool returned: Read and Write content, the Edit patch, the Bash command and output, MCP and web results (Edit and Write also need `OTEL_LOG_TOOL_DETAILS`; tracing must be on). Grok Build: tool input, output, the full command and failure error messages on events | File contents, command output, fetched pages |
| `OTEL_LOG_USER_PROMPTS=1` | Both | Prompt and response text (responses can be set on their own with `OTEL_LOG_ASSISTANT_RESPONSES`) | Everything typed |

With nexus, turn on tool details and tool content with its **Tool details too** and **Tool content too** questions (`nexus --config` → `telemetry`), then `nexus <persona> restart`; nexus has no setting for prompts. By hand, set them with the other telemetry settings (Claude Code: the managed-settings `env` block; Grok Build: its environment), then restart the agent sessions. The collector masks common secret shapes in these fields before they are stored ([Secret masking](#secret-masking)). Loki rejects a record whose structured metadata exceeds 256KB, and the record is lost ([runbook](runbook.md#troubleshooting)). Claude Code sends each prompt twice (`prompt`, `prompt_text`) with no size cap, so the collector keeps one copy.

With tool details on, the work logs show these columns. Most appear once some row in the time range carries their field, so with details off they are absent, not empty. Grok Build's **Tool calls** always shows **File** and **Parameters**, empty until details are on.
- Claude Code **Tool calls**: **Command** (`full_command` from `tool_parameters`), **File** (`file_path` from `tool_input`) and **Input** (the raw `tool_input`: URLs, patterns, MCP arguments). Claude Code shortens long strings in `tool_input` and cuts it at about 4 KB. A cut value is not valid JSON, so **File** stays empty and **Input** shows the start.
- Claude Code **Shell commands**: **Full command** (the span's `full_command`). Each distinct command line is then its own row, so one program's calls spread over several rows. Tempo keeps the first 128 KB of an attribute (`distributor.max_attribute_bytes` in `tempo.yaml`; Tempo's default is 2 KB).
- Grok Build **Tool calls** and **Failures**: **File** (`file_path`) and **Parameters** (the `tool_parameters` preview; in Tool calls from the tool result or else from its decision, in Failures from the failed tool result). Grok Build cuts the preview at 4 KB, so it may not be valid JSON.

Long cells end in an ellipsis: hover one and click the eye icon for the full value. With details on, a full **Tool calls** table gets large (up to about 30 MB per refresh for 5000 Claude Code rows), so narrow the time range or pick a session. The conversations dashboards show the `OTEL_LOG_TOOL_CONTENT` bodies ([Conversations](#conversations)); the other fields (`error`, the Bash `description`) are in Explore. [EXAMPLES.md](EXAMPLES.md#logql) has a query that lists Bash commands.

#### Secret masking

Commands, paths, tool input and prompts can hold credentials. The collector's `transform/redact_secrets` processor (in `otel-collector/config.yaml`) replaces common secret shapes with `****` before Loki or Tempo stores them. It checks string log attributes, string log bodies, span attributes and span event attributes, and masks:
- private key blocks (the BEGIN and END lines stay);
- `Authorization`, `Proxy-Authorization`, `Cookie` and `Set-Cookie` headers, API-key headers such as `X-Api-Key` and `PRIVATE-TOKEN`, and `Bearer <token>`;
- the password in `scheme://user:password@host` and in `curl -u user:password`;
- the value of a key that ends in `password`, `passwd`, `passphrase`, `secret`, `token`, `api_key`, `access_key`, `secret_key`, `pwd`, `credentials`, `private_key`, `_PASS` and a few more (`GITHUB_TOKEN=…`, `DB_PASS=…`, `"password": "…"`, but not `max_tokens=100`, `TOKEN_FILE=/x` or fetch's `credentials: 'include'`), and of flags such as `--password`, `--token`, `--api-key` and `--github-token` (but not `--password-stdin`);
- a secret given to a command without a key name: `aws configure set aws_secret_access_key …`, `docker login -p …`, `mysql -p…`, `sshpass -p …` and `redis-cli -a …`;
- known token formats (GitHub, GitLab, AWS key ids, Slack, Anthropic, OpenAI, xAI, Google, npm, Hugging Face, Stripe, SendGrid, JWTs), and Slack and Discord webhook URLs.

The part that says what was there stays: `Bearer ****`, `GITHUB_TOKEN=****`, `ghp_****`, `deploy:****@host`. A `$VAR` or `$(...)` right after `=`, a flag, a header name, an auth scheme or `curl -u user:` stays too, since it is not the secret itself. In a URL or a quoted value it is masked: `https://deploy:$PW@host` becomes `https://deploy:****@host`, and `PGPASSWORD="$PW"` becomes `PGPASSWORD="****"`.

It is best effort. These get through: a format it doesn't know, an unquoted value after `:` or a space (YAML `password: value`, `.netrc` `password value`), a short flag such as `-p` on a command not listed above, a secret split across values, a token cut short by Claude Code's shortening of long strings, a secret in a file path, string values nested in maps or lists, resource attributes and metrics. A few harmless values are masked too: `docker build --secret id=x,src=y` becomes `--secret ****,src=y`, and in code `self.token = token` becomes `self.token = ****`. Grok Build already scrubs secret shapes and home paths when it sends, and drops a record that still holds one, so this step mainly protects Claude Code's data, which isn't known to be scrubbed.

To add or change a pattern, edit `transform/redact_secrets` in `otel-collector/config.yaml`. Each rule is in two lists: one for attributes (shared by logs, spans and span events) and one for log bodies. The comment above the processor explains the escaping. The collector reads its config only on start, so validate and run `docker compose restart otel-collector` on the server ([Change secret masking](runbook.md#change-secret-masking)). Masking applies to data received after that; data already stored stays until retention removes it.

## Conversations

Each agent has a **conversations** dashboard (`claude-code-conversations.json`, `grok-build-conversations.json`) that reads a session as a transcript, top to bottom:
- **Conversations**: one row per session: title (the first logged prompt), start, last activity, VM, persona, user, talks (prompts), model calls, subagents, Bash runs (Grok Build: terminal runs), files changed, lines added and removed (Claude Code), tokens, cache-read share, cost, failed tools. Click a conversation to select it and narrow the time range to it.
- **Talks**: one row per prompt: the prompt, the reply, and **What the agent did**, a collapsible list of its steps (Claude Code: the first 40 main-thread steps). Click a talk to narrow the panels below to it.
- **Flow**: lanes for prompts, model calls, each tool and failures; Claude Code adds the talk's trace.
- **Steps**: every tool call with its command and file. Click a step to inspect it.
- **Inspector**: **Step details** (every field of the tool event), **Evidence** (what the tool returned) and the step's change.
- **Changes**: every file edit in the conversation as a diff, newest first.

| Part | Needs |
|---|---|
| Title, prompt and reply text | `OTEL_LOG_USER_PROMPTS=1` |
| Commands, files, files changed, the parsed tool input | `OTEL_LOG_TOOL_DETAILS=1` |
| Evidence and changes | `OTEL_LOG_TOOL_CONTENT=1`; Claude Code's Edit and Write also need tool details |

With a setting off, its cells say "not logged" or show a short placeholder. Agent text only appears inside code blocks, and Grafana's HTML sanitizer stays on.

- **Claude Code**: Evidence and changes come from the `tool.output` event on its Tempo tool spans. A Grafana SQL expression turns the Edit patch (hunks) into a unified diff; a Write shows as all-added lines. A diff cut short is closed after its last complete line and marked "cut". Claude Code caps each value at 60 KB. `tempo.yaml` raises Tempo's limits to 128 KB per attribute (`distributor.max_attribute_bytes`, default 2 KB) and 20 MB per trace (`max_bytes_per_trace`, default 5 MB), because one trace holds a whole prompt and every Read or Edit adds to it. After pulling: `docker compose restart tempo`.
- **Grok Build**: everything comes from its Loki events. A change is rebuilt from the edit tool's input (`search_replace`, `edit`, `write`, `apply_patch`, `hashline_edit`); Grok Build reports no patch, and a step whose outcome isn't success wasn't applied. Without tool details it reports these tools as `custom_tool`, so their edits are missing. It cuts tool input and output at 60 KB.
- **Limits**: each event table reads at most 5000 Loki events. Changes covers up to 100 prompts with 100 edits each (Claude Code) or 200 edits (Grok Build). Claude Code's subagent and teammate sessions send no prompts, so they have no talks. Grok Build doesn't send a subagent's parent session, so its subagents appear as separate conversations.

## Behaviour and limits

- **Buffering**: the collector keeps a persistent queue (32 MiB per signal, fsync'd) and retries until a backend is back. Data is lost if the queue fills up, if a backend permanently rejects it (HTTP 400), or while the collector itself is down (`down`, upgrades, reboots): agents retry only briefly and do not spool. Watch `otelcol_exporter_queue_size`, `otelcol_exporter_send_failed_*` and `otelcol_receiver_refused_*` (job `otel-collector`).
- **Idle agents**: Claude Code sends a series only when it changes. Prometheus looks back 31 min (`--query.lookback-delta`), longer than the collector's 30 min delta → cumulative window, so per-bucket `increase()` still counts the first change after an idle gap.
- **Secret masking**: the collector masks common secret shapes in logs and traces before they are stored, best effort ([Secret masking](#secret-masking)).
- **Delta → cumulative** state lives in the collector's memory. A collector restart, or 30 min without data for a series, looks like a counter reset, which `increase()` handles. Run exactly one collector.
- **Experimental Prometheus features**: `created-timestamp-zero-ingestion` (counts each new series' first export) and `promql-extended-range-selectors` (`anchored`).
- **Retention**: Prometheus 15d or 3GB, Loki 30d, Tempo 14d. Loki and Tempo retention is time-only, so shorten it if ingest grows or the Docker disk is small (see the runbook's "Disk full" row).
- **Memory caps**: collector 1g, Prometheus 1g, Loki 1g, Tempo 1.5g, Grafana 1g.
- **Grafana users**: anyone who can log in to Grafana can reach Loki/Tempo HTTP APIs through the datasource proxy, including Loki's ingester shutdown endpoint. Only give logins to people you trust with the stack.
- **Firewall**: Docker-published ports bypass host `INPUT` rules (ufw). To limit who can reach 4317/4318/3000, use the `DOCKER-USER` chain (host step, optional). `<allowed-cidr>` must cover the agents and the admin workstations:
  `iptables -I DOCKER-USER -i <ext-if> ! -s <allowed-cidr> -p tcp -m multiport --dports 3000,4317,4318 -j DROP`
