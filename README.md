# Agent telemetry

Collects metrics, logs and traces from Claude Code and Grok Build (or any OpenTelemetry app) on many VMs into one server, and shows them in Grafana.

```
agent VMs ──OTLP──▶ otel-collector ──▶ Prometheus (metrics) / Loki (logs) / Tempo (traces) ──▶ Grafana
```

You need:
- **One server**: a Linux host with Docker (Compose v2). Agents must reach its ports 4317 and 4318, and your browser port 3000. The first start needs outbound HTTPS. Run the server commands as root or as a user in the `docker` group.
- **A server name** that the agent VMs and your browser resolve: a DNS `A` record, or `192.0.2.10 telemetry.lair.dev` in their `/etc/hosts`.
- **Agent VMs**: the machines running Claude Code or Grok Build in [nexus](https://github.com/h3x00r/agent) personas, set up as its runbook describes. Without nexus, see [Agent configuration](REFERENCE.md#agent-configuration-claude-code).

The commands below use the server name `telemetry.lair.dev` and IP `192.0.2.10`. Replace both with yours.

## Setup

### 1. Start the server

**On the server**, run this block once. Running it again replaces every secret.
```sh
git clone <repo-url> agent-telemetry && cd agent-telemetry

# Create .env with random secrets (Docker is the only tool needed)
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
`TLS_SANS` must list every name and IP agents use to reach the server. Keep `DNS:localhost` and `DNS:grafana`; the stack needs them.

Then start the stack and export its CA certificate:
```sh
docker compose up -d --wait     # the first start downloads images and takes a few minutes
docker compose ps -a            # 5 services "healthy"; init and certs "Exited (0)"
docker compose run --rm -T --no-deps certs cat /tls/ca.crt > otel-ca.crt   # CA for agents and browsers
```

Open `https://telemetry.lair.dev:3000` (or `https://192.0.2.10:3000`) and log in with `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD` from `.env`. The browser warns about the certificate: continue anyway, or copy `otel-ca.crt` to your computer and import it as a trusted CA.

### 2. Connect an agent VM

Once per VM. nexus writes all the agent settings itself; it needs only the server's address, the token and the CA.

**On the server**, in the repo directory: print the token, and copy the CA to the VM.
```sh
sed -n 's/^OTLP_BEARER_TOKEN=//p' .env
scp otel-ca.crt <vm>:
```

**On the VM**, as the user who runs nexus:
```sh
nexus --config
```
Pick `telemetry` in the menu, then answer:
1. the collector's address: `https://telemetry.lair.dev:4318` (or `https://192.0.2.10:4318`);
2. its bearer token: paste the one printed on the server. Nothing shows as you type;
3. the CA file: `~/otel-ca.crt`, where scp put it. nexus keeps a copy of its own;
4. `y` for each persona that should send;
5. `none` for tool details and tool content, unless you want them: see [Optional detail](REFERENCE.md#optional-detail-commands-and-file-paths).

nexus then sends a test export and prints `-- telemetry: https://telemetry.lair.dev:4318 took a test export with the token`, or what is wrong. Restart each persona it names (this ends its tmux sessions):
```sh
nexus <persona> restart
```
nexus labels the data with the VM (`host.name`) and the persona, so the VM and persona filters and the flow timelines work without more setup. To script it, use `nexus --config --yes` with the `--telemetry-*` options from the [nexus runbook](https://github.com/h3x00r/agent/blob/main/runbook.md#telemetry-opentelemetry).

### 3. Check that data arrives

**On the VM**, from a project directory:
```sh
nexus <persona> headless hi
```
Wait about a minute, then reload Grafana. The VM's `hostname -s` appears in the **VM** filter of the **Claude Code** dashboard (or **Grok Build**, for a grok persona), and **Sessions** counts the run.

Nothing? See [Troubleshooting](runbook.md#troubleshooting). To test the connection without an agent, run the [smoke test](EXAMPLES.md#smoke-test).

## View the data

After login, Grafana shows the **Claude Code** dashboard.
- Filter at the top with **VM**, **User**, **Environment** and **Model**. Pick the period at the top right; every total covers that period.
- **Traces** and **Collector health** start collapsed; click a row title to open them.
- Hover a panel's ⓘ to see what it counts.

Grok Build has its own **Grok Build** dashboard, with filters for **VM**, **Persona**, **User** and **Model**; its **Startup** row starts collapsed. Switch dashboards with the **Agent telemetry** menu at the top of any of them, or open them from **Dashboards → Agent telemetry**.

Each agent also has three more dashboards:
- **Work log** (**Claude Code work log**, **Grok Build work log**): tables of what each session did. Click a session to see its prompts (Grok Build: turns), tool calls and, for Claude Code, shell commands, edits and workflow runs; click a prompt to narrow the tool tables to it.
- **Flow timeline** (**Claude Code flow timeline**, **Grok Build flow timeline**): one row per persona, showing over time when it prompted, called the model, used each tool, and ran sessions and (Claude Code) workflows. It needs a persona on the data: see [Personas](REFERENCE.md#personas).
- **Conversations** (**Claude Code conversations**, **Grok Build conversations**): each session as a transcript. Click a conversation to see its prompts and replies, what the agent did, and every step; click a step to see what the tool returned and the file change it made. Text, commands and file contents appear only with the optional settings on: see [Conversations](REFERENCE.md#conversations).

Beyond the dashboards:
- **Drilldown → Logs / Metrics / Traces** in the left menu: browse every event, metric and trace without writing queries.
- **Explore**: run your own queries. Examples are in [EXAMPLES.md](EXAMPLES.md#promql).

To change a dashboard, edit its file in `grafana/provisioning/dashboards/` on the server (`claude-code.json`, `grok-build.json`, and `*-work.json` / `*-flow.json` / `*-conversations.json` for the work logs, timelines and conversations); Grafana loads it within a minute. For a personal copy, open it from **Dashboards → Agent telemetry** and use **Save as copy**.

## Next

| I want to… | Go to |
|---|---|
| See Claude Code's sessions, cost and tokens per VM | The **Claude Code** dashboard: [View the data](#view-the-data) |
| See Grok Build's cost, tokens and tools per VM and persona | The **Grok Build** dashboard: [View the data](#view-the-data) |
| See what a session or prompt did: tool calls, commands, workflow runs | The **work log** dashboards: [Work logs and flow timelines](REFERENCE.md#work-logs-and-flow-timelines) |
| See each persona's activity over time | The **flow timeline** dashboards: [Work logs and flow timelines](REFERENCE.md#work-logs-and-flow-timelines) |
| Read a session as a transcript: prompts, replies, tool output, diffs | The **conversations** dashboards: [Conversations](REFERENCE.md#conversations) |
| Record commands, file paths and tool output, not just tool names | [Optional detail](REFERENCE.md#optional-detail-commands-and-file-paths) |
| Write my own queries | [EXAMPLES.md](EXAMPLES.md) |
| Stop or start the stack | `docker compose down` (keeps data) / `docker compose up -d --wait`, on the server |
| Check the stack is healthy | [Health check](runbook.md#health-check) |
| Fix 401, certificate or connection errors | [Troubleshooting](runbook.md#troubleshooting) |
| Edit a config file | [Apply a change](runbook.md#apply-a-change) |
| Get a newer version of this repo | On the server: `git pull`, then `docker compose up -d --wait --force-recreate` |
| Rotate the token | [Rotate the OTLP token](runbook.md#rotate-the-otlp-token) |
| Add a server name or IP | [Change TLS names](runbook.md#change-tls-names-or-renew-the-server-cert) |
| Reset the Grafana password | [Reset the Grafana admin password](runbook.md#reset-the-grafana-admin-password) |
| Back up, upgrade, wipe | [Back up and restore](runbook.md#back-up-and-restore) |
| Send Claude Code's or Grok Build's data without nexus | [Agent configuration (Claude Code)](REFERENCE.md#agent-configuration-claude-code), [(Grok Build)](REFERENCE.md#agent-configuration-grok-build) |
| Send data from another app (Python, Node.js…) | [Other OTel SDKs](EXAMPLES.md#other-otel-sdks) |
| Ports, TLS, limits, how it works | [REFERENCE.md](REFERENCE.md) |
| Check for secrets before pushing | `tests/secrets.sh`: [Scan for secrets](runbook.md#scan-for-secrets) |

## License

The files in this repo are under the [MIT License](LICENSE). The images the stack runs (OpenTelemetry Collector, Prometheus, Loki, Tempo, Grafana and the plugins it installs, busybox, Alpine) and the ones the secret scan and CI use (gitleaks, the `actions/checkout` action) keep their own licenses. The agents it collects from, Claude Code (Anthropic) and Grok Build (xAI), keep their own terms of use. This project is not affiliated with or endorsed by Anthropic or xAI.
