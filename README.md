# Agent telemetry

Collects metrics, logs and traces from Claude Code (or any OpenTelemetry app) on many VMs into one server, and shows them in Grafana.

```
agent VMs ──OTLP──▶ otel-collector ──▶ Prometheus (metrics) / Loki (logs) / Tempo (traces) ──▶ Grafana
```

You need:
- **One server**: a Linux host with Docker (Compose v2). Agents must reach its ports 4317 and 4318, and your browser port 3000. The first start needs outbound HTTPS. Run the server commands as root or as a user in the `docker` group.
- **A server name** that the agent VMs and your browser resolve: a DNS `A` record, or `192.0.2.10 telemetry.lair.dev` in their `/etc/hosts`.
- **Agent VMs**: the machines running Claude Code, with root access.

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

Once per VM. **On the server**, in the repo directory: print the token, and copy the CA to the VM.
```sh
sed -n 's/^OTLP_BEARER_TOKEN=//p' .env
scp otel-ca.crt <vm>:
```

**On the VM**, in the directory holding `otel-ca.crt` (scp put it in your home directory), run `sudo -s`, then the block below. Paste the token printed on the server. The endpoint can also be `https://192.0.2.10:4318`.
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
This enables telemetry for every Claude Code user on the VM. Restart Claude Code sessions that were already running.

### 3. Check that data arrives

**On the VM**, as any user:
```sh
claude -p hi
```
Wait about a minute. In Grafana, open **Explore**, pick the **Prometheus** data source, switch the query editor from Builder to **Code**, enter `claude_code_session_count_total{host_name="<vm>"}` and click **Run query**. `<vm>` is the VM's `hostname -s`. At least one result means it works.

Nothing? See [Troubleshooting](runbook.md#troubleshooting). To test the connection without Claude Code, run the [smoke test](EXAMPLES.md#smoke-test).

## Next

| I want to… | Go to |
|---|---|
| See sessions, cost and tokens per VM | [EXAMPLES.md](EXAMPLES.md#promql): paste into Grafana → Explore |
| Stop or start the stack | `docker compose down` (keeps data) / `docker compose up -d --wait`, on the server |
| Check the stack is healthy | [Health check](runbook.md#health-check) |
| Fix 401, certificate or connection errors | [Troubleshooting](runbook.md#troubleshooting) |
| Edit a config file | [Apply a change](runbook.md#apply-a-change) |
| Rotate the token | [Rotate the OTLP token](runbook.md#rotate-the-otlp-token) |
| Add a server name or IP | [Change TLS names](runbook.md#change-tls-names-or-renew-the-server-cert) |
| Reset the Grafana password | [Reset the Grafana admin password](runbook.md#reset-the-grafana-admin-password) |
| Back up, upgrade, wipe | [Back up and restore](runbook.md#back-up-and-restore) |
| Send data from another app (Python, Node.js…) | [Other OTel SDKs](EXAMPLES.md#other-otel-sdks) |
| Ports, TLS, limits, how it works | [REFERENCE.md](REFERENCE.md) |
