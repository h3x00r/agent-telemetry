# Runbook

Day-to-day procedures. First-time setup: [README.md](README.md). Ports, TLS and limits: [REFERENCE.md](REFERENCE.md). Queries: [EXAMPLES.md](EXAMPLES.md).

The examples use `telemetry.lair.dev` / `192.0.2.10` for the stack host. Run every command **on the stack host, in the repo directory (the one with `compose.yaml`)**, unless the step says it runs on an agent VM.

## Health check

```sh
docker compose ps -a     # 5 services "healthy"; init and certs "Exited (0)"
docker compose exec prometheus promtool query instant http://localhost:9090 'up == 0'                         # blank = all targets up
docker compose exec prometheus promtool query instant http://localhost:9090 'otelcol_exporter_queue_size > 0'    # blank = queues drained
docker compose run --rm -T --no-deps certs openssl x509 -noout -enddate -in /tls/server.crt                   # cert expiry
docker system df -v | grep -E '^otel_'                    # volume sizes
df -h "$(docker info -f '{{.DockerRootDir}}')"            # free space on the Docker disk
docker compose logs --since 1h otel-collector | grep -iE 'error|warn'   # expect nothing
```
A service not healthy or missing: `docker compose logs --tail 50 <svc>`, fix the cause, then `docker compose up -d --wait`.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Agent gets 401 / `Unauthenticated` | Missing or wrong token | The header must be exactly `Authorization=Bearer <OTLP_BEARER_TOKEN>`, with a capital B. |
| `unable to verify the first certificate` (Claude Code) | CA not trusted | Set `NODE_EXTRA_CA_CERTS=/etc/otel/otel-ca.crt`. Claude Code ignores `OTEL_EXPORTER_OTLP_CERTIFICATE`. |
| `Setting the TLS ServerName to an IP address is not permitted` | gRPC to an IP | Use a name from `TLS_SANS`, or `http/protobuf`. |
| `Hostname/IP does not match certificate's altnames` (Go: `certificate is valid for …, not …`) | Name not in `TLS_SANS` | Add it, then [re-issue](#change-tls-names-or-renew-the-server-cert). |
| Connection refused or timeout | Stack down, DNS or firewall | Server: `docker compose ps -a`, `sudo iptables -nvL DOCKER-USER`. Agent VM: `getent hosts telemetry.lair.dev`. |
| HTTP 400 `Client sent an HTTP request to an HTTPS server.` | `http://` endpoint | Use `https://`. |
| Data has no `host_name` label | `host.name` not set | Add it to `OTEL_RESOURCE_ATTRIBUTES`. |
| No traces from Claude Code | Beta tracing off | Set `OTEL_TRACES_EXPORTER=otlp` and `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`. |
| `claude_code.interaction` root span missing in `-p` runs | Client-side race | Known issue; nothing to fix on the server. |
| `otelcol_exporter_queue_size` keeps growing | A backend is down or slow | `docker compose ps -a`; `docker compose logs <backend>`. The queue drains once the backend is back. |
| `otelcol_exporter_enqueue_failed_*` > 0 | Queue full, data dropped | Fix the backend (row above). |
| `otelcol_receiver_refused_*` > 0 | Collector memory limit reached, or queue full | If `enqueue_failed` is also > 0, see the row above. Otherwise check `docker stats --no-stream`, raise `mem_limit` and `GOMEMLIMIT` (about 64% of it) in `compose.yaml`, then run `docker compose up -d --wait`. |
| Collector logs show 400 from Loki | Record rejected (e.g. structured metadata > 256KB) | The record is dropped for good. Raise `max_structured_metadata_size` in `loki/loki.yaml`, then `docker compose restart loki`. |
| Disk full | Retention too long | `docker system df -v`. Lower `storage.tsdb.retention` in `prometheus/prometheus.yml`, `retention_period` in `loki/loki.yaml`, or both `block_retention` values in `tempo/tempo.yaml` (the `overrides` one wins), then [apply](#apply-a-change). |
| `docker compose start` fails | The exited `init` container was pruned | Use `docker compose up -d`. |
| Grafana login fails after editing `.env` | The password only applies on first start | [Reset it](#reset-the-grafana-admin-password). |
| Drilldown apps missing in Grafana | No internet on first start | Allow outbound HTTPS, then `docker compose restart grafana`. |

## Apply a change

Validate first. These commands use the image tags in `compose.yaml`; a failure prints an error.
```sh
docker compose config -q
docker compose run --rm -T --no-deps otel-collector validate --config=/etc/otelcol-contrib/config.yaml
docker compose run --rm -T --no-deps --entrypoint promtool prometheus check config /etc/prometheus/prometheus.yml
docker compose run --rm -T --no-deps loki -config.file=/etc/loki/loki.yaml -verify-config
docker compose run --rm -T --no-deps tempo -target=all -config.file=/etc/tempo/tempo.yaml -config.verify=true
```

Then apply the change. After applying, `docker compose ps -a` shows the service healthy.

| Changed | Run | Note |
|---|---|---|
| `otel-collector/config.yaml` | `docker compose restart otel-collector` | Resets delta→cumulative state; `increase()` treats it as a counter reset. |
| `prometheus/prometheus.yml` | `docker compose kill -s SIGHUP prometheus` | `docker compose logs --since 1m prometheus` shows "Completed loading of configuration file". |
| `loki/loki.yaml`, `tempo/tempo.yaml` | `docker compose restart loki` (or `tempo`) | The collector queues data while it is down. |
| `grafana/provisioning/datasources/` | `docker compose restart grafana` | Only `datasources/` and `dashboards/` are mounted; a new provisioning folder also needs a volume line in `compose.yaml`. |
| `grafana/provisioning/dashboards/*.json` | Nothing | Grafana reloads it within a minute. |
| `compose.yaml` (including command-line flags), `.env` | `docker compose up -d --wait` | Recreates changed services. `restart` keeps the old flags and `.env`. |
| Several files, e.g. after `git pull` | `docker compose up -d --wait --force-recreate` | Recreates every service; data stays in the volumes. |

## Add an agent VM

Follow [README step 2](README.md#2-connect-an-agent-vm), then [step 3](README.md#3-check-that-data-arrives).

## Rotate the OTLP token

1. Generate a token: `docker compose run --rm -T --no-deps certs openssl rand -hex 32`
2. Set `OTLP_BEARER_TOKEN` in `.env`, then run `docker compose up -d --wait otel-collector`.
3. **On every agent VM, as root**, then restart running Claude Code sessions. Until then, those agents get 401 and their data is dropped.
```sh
NEW='<NEW_TOKEN>'
sed -i "s/Bearer [0-9a-f]*/Bearer $NEW/" /etc/claude-code/managed-settings.d/50-otel.json
```

## Change TLS names or renew the server cert

The CA stays the same, so agents need nothing.
1. Edit `TLS_SANS` in `.env`. It takes comma-separated `IP:`/`DNS:` entries, e.g. `IP:192.0.2.10,DNS:telemetry.lair.dev,DNS:localhost,DNS:grafana`. Keep `DNS:localhost` (Grafana healthcheck) and `DNS:grafana` (Prometheus scrape).
   For a renewal, skip this step: `up -d` re-issues the cert within 30 days of expiry. To force a re-issue earlier, run `docker compose run --rm -T --no-deps certs rm -f /ca/sans`.
2. `docker compose up -d --wait`. Then `docker compose logs --tail 1 certs` must say "issued server cert".
3. `docker compose restart otel-collector grafana`. Otherwise the collector picks up the new cert within 24h, and Grafana only on its next restart.

## Reset the Grafana admin password

```sh
docker compose exec grafana grafana cli admin reset-admin-password '<new>'
```
Also set `GRAFANA_ADMIN_PASSWORD` in `.env` to the new value. It only applies to an empty `grafana-data`, so after a wipe a stale value would bring the old password back.

## Back up and restore

Back up. The collector keeps accepting data and queues it while the backends are stopped. The backup holds `.env` (Grafana encryption keys) and the CA private key (`tls-ca.tgz`), so keep it private and off the Docker disk.
```sh
B=~/otel-backup/$(date +%F); (umask 077; mkdir -p "$B"); install -m 0600 .env "$B/.env"
docker compose stop prometheus loki tempo grafana
for v in prometheus-data loki-data tempo-data grafana-data tls-ca tls; do
  docker run --rm -v otel_$v:/v:ro -v "$B":/b busybox:1.36.1-musl tar czf /b/$v.tgz -C /v .
done
docker compose up -d --wait
```

Restore. On a new host, copy `$B/.env` into the repo directory first. Pick the backup (edit the date):
```sh
B=~/otel-backup/YYYY-MM-DD
ls "$B"/*.tgz            # must list 6 files
```
Continue only if it lists 6 files. In the same shell, run the block below. **It deletes all current data, including the collector queue.**
```sh
docker compose down -v
docker compose create
for v in prometheus-data loki-data tempo-data grafana-data tls-ca tls; do
  docker run --rm -v otel_$v:/v -v "$B":/b:ro busybox:1.36.1-musl tar xzf /b/$v.tgz -C /v
done
docker compose up -d --wait
```

## Upgrade an image

1. [Back up](#back-up-and-restore). Read the release notes for config changes.
2. Change the tag in `compose.yaml`. For Grafana, also check the plugin pins in `GF_PLUGINS_PREINSTALL`.
3. [Validate](#apply-a-change). The validate commands use the new tag.
4. `docker compose pull <svc> && docker compose up -d --wait <svc>`, then run the [health check](#health-check).

To roll back, restore the old tag and run `docker compose up -d --wait`. If the new version migrated the data, also restore the backup.

## Wipe everything

**Destructive.** [Back up](#back-up-and-restore) first. `docker compose down -v` deletes all data, the collector queue **and the CA**. After `docker compose up -d --wait`, export the new CA with `docker compose run --rm -T --no-deps certs cat /tls/ca.crt > otel-ca.crt` and install it on every agent ([README step 2](README.md#2-connect-an-agent-vm)) and browser. The Grafana admin login reverts to the `.env` values.
