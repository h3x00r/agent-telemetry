# Examples

Test payloads and queries. Paste PromQL, LogQL and TraceQL into Grafana → Explore with the matching data source. `<UPPER_CASE>` values come from the server's `.env`, `<vm>` is the agent VM's `hostname -s`, and `telemetry.lair.dev` / `192.0.2.10` stand for your server.

## Smoke test

Run on an agent VM that has the CA at `/etc/otel/otel-ca.crt` ([README step 2](README.md#2-connect-an-agent-vm)). The script sends one metric, one log record and one span. Expect `{"partialSuccess":{}} 200` three times. `OTLP` can also be `https://192.0.2.10:4318`.

```sh
OTLP=https://telemetry.lair.dev:4318; TOKEN='<OTLP_BEARER_TOKEN>'; CA=/etc/otel/otel-ca.crt
NOW=$(date +%s%N); TRACE=$(openssl rand -hex 16); SPAN=$(openssl rand -hex 8)
RES='"resource":{"attributes":[{"key":"service.name","value":{"stringValue":"smoke-test"}},{"key":"host.name","value":{"stringValue":"'$(hostname -s)'"}}]}'
post() { curl -sS --cacert "$CA" -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' -w ' %{http_code}\n' "$OTLP/v1/$1" -d "$2"; }

post metrics '{"resourceMetrics":[{'"$RES"',"scopeMetrics":[{"metrics":[{"name":"smoke.test","sum":{"aggregationTemporality":1,"isMonotonic":true,"dataPoints":[{"asInt":"1","startTimeUnixNano":"'$((NOW-1000000000))'","timeUnixNano":"'$NOW'"}]}}]}]}]}'
post logs '{"resourceLogs":[{'"$RES"',"scopeLogs":[{"logRecords":[{"timeUnixNano":"'$NOW'","severityText":"INFO","body":{"stringValue":"smoke test"},"traceId":"'$TRACE'","spanId":"'$SPAN'"}]}]}]}'
post traces '{"resourceSpans":[{'"$RES"',"scopeSpans":[{"spans":[{"traceId":"'$TRACE'","spanId":"'$SPAN'","name":"smoke","kind":2,"startTimeUnixNano":"'$((NOW-5000000))'","endTimeUnixNano":"'$NOW'"}]}]}]}'
echo "trace id: $TRACE"
```

Then check in Grafana → Explore, last 15 minutes:
- Prometheus: `smoke_test_total{host_name="<vm>"}` returns `1`. Reruns within 30 minutes add up.
- Loki: `{service_name="smoke-test"}` shows "smoke test".
- Tempo: search for the printed trace id.

## Query from a shell

Prometheus, Loki and Tempo are not published, so go through Grafana's datasource proxy. This works from anywhere that has `otel-ca.crt` (on the server: `docker compose run --rm -T --no-deps certs cat /tls/ca.crt > otel-ca.crt`). To use a service-account token, replace `-u "$AUTH"` with `-H "Authorization: Bearer <sa-token>"`.

```sh
G=https://telemetry.lair.dev:3000; AUTH='<GRAFANA_ADMIN_USER>:<GRAFANA_ADMIN_PASSWORD>'; CA=./otel-ca.crt
q() { p=$1; shift; curl -sSG --cacert "$CA" -u "$AUTH" "$G/api/datasources/proxy/uid/$p" "$@"; }

q prometheus/api/v1/query --data-urlencode 'query=sum by (host_name) (increase(claude_code_session_count_total[1d] anchored))'
q loki/loki/api/v1/query_range --data-urlencode 'query={service_name="claude-code"} | event_name="api_request"' --data-urlencode since=1h --data-urlencode limit=5
q tempo/api/search --data-urlencode 'q={ resource.service.name = "claude-code" }' --data-urlencode start=$(($(date +%s)-3600)) --data-urlencode end=$(date +%s)
q tempo/api/v2/traces/<trace-id>      # {"trace":{}} = not found
```
For PromQL only, on the stack host in the repo directory: `docker compose exec prometheus promtool query instant http://localhost:9090 '<promql>'`.

## Other OTel SDKs

Standard OTel SDK environment (checked with the Python and Node.js SDKs):
```sh
export OTEL_SERVICE_NAME=my-app
export OTEL_EXPORTER_OTLP_ENDPOINT=https://telemetry.lair.dev:4318
export OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer <OTLP_BEARER_TOKEN>"
export OTEL_EXPORTER_OTLP_CERTIFICATE=/etc/otel/otel-ca.crt   # Claude Code ignores this; it needs NODE_EXTRA_CA_CERTS
export OTEL_RESOURCE_ATTRIBUTES="host.name=$(hostname -s),deployment.environment.name=prod"
```
Python, with zero-code instrumentation:
```sh
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install                              # optional: instrument installed libraries
export OTEL_PYTHON_LOGGING_AUTO_INSTRUMENTATION_ENABLED=true    # without it, no logs are exported
opentelemetry-instrument --metrics_exporter otlp --logs_exporter otlp --traces_exporter otlp python app.py
```

## PromQL

`$__range` is the dashboard time range; outside Grafana, use a fixed range such as `[1d]`. Keep `anchored` ([why](REFERENCE.md#querying)).
```promql
# Sessions, cost (USD) and tokens per VM / model / type
sum by (host_name) (increase(claude_code_session_count_total[$__range] anchored))
sum by (host_name, model) (increase(claude_code_cost_usage_USD_total[$__range] anchored))
sum by (type) (increase(claude_code_token_usage_tokens_total[$__range] anchored))

# Cost per user (Claude Code labels every series with user_email)
sum by (user_email) (increase(claude_code_cost_usage_USD_total[$__range] anchored))

# Cost over the trailing hour per VM (time series; set the panel's Min interval to 1h for hourly buckets)
sum by (host_name) (increase(claude_code_cost_usage_USD_total[1h] anchored))

# Lines added/removed, edit-tool accept/reject, active hours
sum by (type) (increase(claude_code_lines_of_code_count_total[$__range] anchored))
sum by (decision) (increase(claude_code_code_edit_tool_decision_total[$__range] anchored))
sum by (host_name) (increase(claude_code_active_time_seconds_total[$__range] anchored)) / 3600

# Claude Code versions seen per VM in the last day
count by (host_name, service_version) (last_over_time(claude_code_session_count_total[1d]))

# Grok Build: sessions per VM and persona, total tokens (input + output) and cost per persona
count by (host_name, persona) (count by (host_name, persona, session_id) (last_over_time(grok_code_token_usage_total[$__range])))
sum by (persona) (increase(grok_code_token_usage_total{type=~"input|output"}[$__range] anchored))
sum by (persona) (increase(grok_code_cost_usage_USD_total[$__range] anchored))
```

Collector health (`job="otel-collector"`):
```promql
sum by (exporter) (otelcol_exporter_queue_size)                                  # bytes queued per backend
sum by (exporter) (rate({__name__=~"otelcol_exporter_send_failed_.+"}[5m]))      # empty until the first failed send
sum by (exporter) (rate({__name__=~"otelcol_exporter_enqueue_failed_.+"}[5m]))   # > 0 = queue full, data dropped
sum by (receiver) (rate(otelcol_receiver_refused_metric_points[5m])) + sum by (receiver) (rate(otelcol_receiver_refused_log_records[5m])) + sum by (receiver) (rate(otelcol_receiver_refused_spans[5m]))
sum by (transport) (rate(otelcol_receiver_accepted_metric_points[5m]))
```

Span metrics generated by Tempo:
```promql
sum by (service, span_name) (rate(traces_spanmetrics_calls_total[5m]))
histogram_quantile(0.95, sum by (le, span_name) (rate(traces_spanmetrics_latency_bucket{service="claude-code"}[5m])))
```

## LogQL

Only these labels go inside `{...}`: `service_name`, `service_namespace`, `deployment_environment`, `deployment_environment_name`, `host_name`. Filter on everything else after `|`.
```logql
{service_name="claude-code", host_name="<vm>"} | event_name="user_prompt"
{service_name="claude-code"} | event_name="api_error"
{service_name="claude-code"} | event_name="tool_result" | success="false"
{service_name="claude-code"} | session_id="<session-id>"
{service_name="grok-cli", host_name="<vm>"} | persona="<persona>"
{service_name="grok-cli"} | stop_reason!=""      # Grok Build model API calls (its events have no event_name)

# Cost per VM and model, API p95 latency per model, tool calls per tool
sum by (host_name, model) (sum_over_time({service_name="claude-code"} | event_name="api_request" | unwrap cost_usd [$__range]))
quantile_over_time(0.95, {service_name="claude-code"} | event_name="api_request" | unwrap duration_ms [$__range]) by (model)
sum by (tool_name) (count_over_time({service_name="claude-code"} | event_name="tool_result" [$__range]))

# Claude Code workflow runs per VM and persona (distinct workflow_run_id)
count by (host_name, persona) (sum by (workflow_run_id, host_name, persona) (count_over_time({service_name="claude-code"} | workflow_run_id!="" [$__range])))
```

## TraceQL

```traceql
{ resource.service.name = "claude-code" && resource.host.name = "<vm>" }
{ name = "claude_code.interaction" && duration > 2m }
{ resource.service.name = "claude-code" && status = error }
{ resource.service.name = "claude-code" } | rate() by (name)
{ resource.service.name = "claude-code" && span.bash_argv0 != nil } | count_over_time() by (span.bash_argv0)
```
The last one counts the programs Claude Code ran through its Bash tool (a TraceQL metrics query).
