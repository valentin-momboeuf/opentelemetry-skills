---
name: otel-collector-debug
description: Use when diagnosing a running OpenTelemetry Collector — symptoms like no data arriving at the backend, OOM/restart loops, dropped spans/metrics/logs, exporter retries, queue full, high CPU, or wrong data shape. Triggers on "my collector is dropping data", "no traces arriving", "collector OOMing", "debug otel collector", "pipeline diagnostic", "check otelcol logs", "zpages", "pprof collector", "telemetrygen test", "Mimir/Tempo/Jaeger isn't getting metrics/traces/logs". Walks through the standard diagnostic toolbox (collector logs, self-metrics on :8888, zpages on :55679, debug exporter, pprof, health_check, telemetrygen), maps symptoms to likely root causes, and produces an evidence-backed diagnostic report.
---

# When to use this skill

The user reports an OpenTelemetry Collector misbehaving in a running environment (dev, staging, prod) and you need to diagnose it. Typical symptoms:

- "No traces / metrics / logs are reaching the backend"
- "Collector is OOMing or restarting"
- "Data is being dropped or queued forever"
- "High CPU / slow / latency on this collector"
- "Some signals get through, others don't"
- "Backend cardinality explodes after I deployed this collector"
- "I deployed a config change and now nothing works"

If the question is purely about *writing* a config (no running collector to debug), use `otel-config-validator` instead. This skill assumes a collector is running and reachable, or at least its logs and self-metrics are.

# Mental model

Three places where data flows can break:

1. **Ingestion** (receiver) — clients can't reach the collector, or get rejected.
2. **Processing** (pipeline) — data is dropped, mutated incorrectly, or starves on memory pressure.
3. **Egress** (exporter) — backend unreachable, throttled, queue saturating, retries spinning.

Most symptoms map to one of these, sometimes two (e.g., backend slow → exporter queue full → upstream backpressure → memory_limiter starts dropping at the receiver). Always identify the **primary** failure point first; downstream symptoms are noise that will resolve once the root is fixed.

# Diagnostic toolbox

## 1. Collector logs

The first thing to check, every time. Look for:

- `error` / `warn` level entries with component names: `receiver`, `processor`, `exporter`, `extension`
- `dial tcp: connection refused` → backend or upstream unreachable
- `Exporting failed. Will retry the request after interval.` → exporter struggling, watch retry escalation
- `Sending queue is full` / `enqueue failed` → backend can't keep up, backpressure starts
- `data refused due to memory limit` → memory_limiter dropping at ingress (a symptom of upstream pressure)
- `unknown component` / `cannot find type` → wrong distribution (core vs contrib)

In Kubernetes:
```bash
kubectl logs -n <ns> <pod> -c otel-collector --tail=200 -f
kubectl logs -n <ns> <pod> --previous   # if it crashed
```

In Docker / systemd:
```bash
docker logs -f --tail 200 otel-collector
journalctl -u otelcol-contrib -n 200 -f
```

If logs are too quiet to see anything, temporarily raise the level (then revert):
```yaml
service:
  telemetry:
    logs:
      level: debug
```

## 2. Self-metrics (`:8888/metrics`)

Exposed when `service.telemetry.metrics` is configured. Scrape with `curl` or a Prometheus instance pointing at the pod.

Key metrics by signal-type pattern (`spans`, `metric_points`, `log_records`):

| Metric | What it tells you |
|---|---|
| `otelcol_receiver_accepted_<sig>` | Data the receiver took in |
| `otelcol_receiver_refused_<sig>` | Data the receiver rejected (auth, format, size) |
| `otelcol_processor_accepted_<sig>` | Data that made it through processors |
| `otelcol_processor_refused_<sig>` | Data dropped by a processor (`memory_limiter` is the usual culprit) |
| `otelcol_processor_dropped_<sig>` | Data explicitly dropped (filter, sampler, transform) |
| `otelcol_exporter_sent_<sig>` | Data the backend acknowledged |
| `otelcol_exporter_send_failed_<sig>` | Failed sends (transient or final) |
| `otelcol_exporter_enqueue_failed_<sig>` | Couldn't even queue → queue full |
| `otelcol_exporter_queue_size` / `_queue_capacity` | Backend keep-up indicator |
| `otelcol_processor_tail_sampling_*` | Tail-sampling decision counters |
| `otelcol_process_runtime_total_alloc_bytes` | Memory pressure signal |

Quick triage:
```bash
curl -s http://<collector>:8888/metrics | \
  grep -E 'otelcol_(receiver|processor|exporter)_(accepted|refused|dropped|sent|send_failed|enqueue_failed)'
```

A non-zero `refused`, `dropped`, or `send_failed` is your starting point.

## 3. zpages (`:55679/debug/...`)

Requires the `zpages` extension. Browser-friendly; in headless contexts, `curl`.

| Endpoint | Use |
|---|---|
| `/debug/tracez` | Recent in-process spans the collector itself sampled — confirms data is flowing through the pipeline |
| `/debug/pipelinez` | Live state of every pipeline (receivers / processors / exporters status, errors) |
| `/debug/extensionz` | Loaded extensions and their state |
| `/debug/servicez` | Build version, config commit, uptime |
| `/debug/featurez` | Feature gates currently on |

When stuck on "is the data even reaching this collector?", `tracez` answers fast: if you see spans for your pipeline, the receiver is OK; if not, the issue is upstream of the receiver (network, client misconfig, wrong port).

## 4. debug exporter (verify what's flowing)

The fastest way to confirm the *content* of what the collector is processing. Add a debug exporter to a copy of the pipeline:

```yaml
exporters:
  debug:
    verbosity: detailed
    sampling_initial: 5
    sampling_thereafter: 200

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug, otlp/backend]
```

Then `tail -f` the collector logs and inspect the rendered spans/metrics/logs. Use this to:

- Confirm attribute names and values (especially when a `transform` processor is suspected)
- Validate cardinality concerns by inspecting actual label sets
- Verify that filtering / sampling drops what you expect

`verbosity: detailed` is expensive — use it for short windows then revert to `basic` or remove from prod pipelines. The exporter was renamed from `logging` to `debug` at v0.86.0; older configs that still say `logging` need updating.

## 5. pprof (`:1777/debug/pprof/...`)

Requires the `pprof` extension. Indispensable for memory or CPU issues.

```bash
# Heap profile (right after observing OOM symptoms)
go tool pprof -http=:9000 http://<collector>:1777/debug/pprof/heap

# 30s CPU profile
go tool pprof -http=:9000 http://<collector>:1777/debug/pprof/profile?seconds=30

# Goroutine state (look for leaks, e.g. tail_sampling stuck waiting)
curl -s http://<collector>:1777/debug/pprof/goroutine?debug=2 | head -200
```

OOM checklist:
1. Heap top-K — which processor / receiver dominates?
2. `tail_sampling`: is the `num_traces` cache the offender?
3. `batch`: any unusually large queued payloads?

## 6. health_check (`:13133`)

Basic liveness:
```bash
curl -s http://<collector>:13133/
# {"status":"Server available"}
```

`health_check_v2` (newer, contrib) adds per-pipeline status — useful to distinguish "process alive" from "pipeline X broken":
```bash
curl -s http://<collector>:13133/health/status
```

## 7. telemetrygen (synthetic clients)

When the application/SDK is suspected, take it out of the loop:

```bash
# 10 traces over OTLP gRPC
telemetrygen traces --otlp-endpoint=<collector>:4317 --otlp-insecure --traces=10

# Logs
telemetrygen logs --otlp-endpoint=<collector>:4317 --otlp-insecure --logs=100

# Metrics
telemetrygen metrics --otlp-endpoint=<collector>:4317 --otlp-insecure --metrics=20
```

If `telemetrygen` succeeds but real clients don't: the issue is on the client side (SDK config, network, auth). If both fail: collector or network.

# Symptom → diagnosis decision tree

## "No data is arriving at the backend"

1. Logs for `error`/`warn` from the exporter — backend reachable? auth?
2. `otelcol_exporter_sent_<sig>` = 0 confirms egress is broken; `otelcol_receiver_accepted_<sig>` > 0 confirms data reaches the receiver.
3. If receiver counter is also 0: clients can't reach the collector. Try `telemetrygen` from inside the cluster/host. Check the bind address (`0.0.0.0` vs `localhost`/`127.0.0.1`), port (4317 vs 4318), and any service/ingress in front.
4. If receiver > 0 and `exporter_sent` = 0: walk the pipeline with `pipelinez` zpages, then add a debug exporter to confirm the data shape mid-pipeline.

## "Collector keeps OOMing / restarting"

1. Logs for `data refused due to memory limit` confirms `memory_limiter` is firing — that's the symptom, not the cause.
2. Check `otelcol_exporter_queue_size` near `_queue_capacity` and `otelcol_exporter_send_failed_*` rising → backend is the bottleneck, fix that or shrink the queue.
3. If memory grows even at idle: `pprof` heap. Common offenders: `tail_sampling` `num_traces` cache, `filelog` with too many files, memory leaks in alpha components.
4. Check `limit_mib + spike_limit_mib` < container memory limit. Otherwise the kubelet OOM-kills before the limiter can throttle.

## "Some signals are dropped"

1. `otelcol_processor_dropped_<sig>` — explicit drops by a `filter` or sampler. Verify the rule is intentional (a debug exporter on a copy of the pre-filter pipeline can confirm what's being dropped).
2. `otelcol_processor_refused_<sig>` — `memory_limiter` at work; see OOM section.
3. `otelcol_exporter_enqueue_failed_<sig>` — backend not keeping up; tune `sending_queue.queue_size` and `num_consumers`, or persist with `storage:`.

## "Wrong attribute values / cardinality at backend"

1. Add a debug exporter to print the *post-processing* output. The values you see here are exactly what hits the backend.
2. Compare against the input by adding a debug exporter on a copy of the pipeline *before* the suspect processor.
3. Common culprits: `transform` with a wrong OTTL expression, `resourcedetection` with `override: true` clobbering app-side resources, `attributes` injecting per-request fields on the metrics pipeline (cardinality bomb).

## "Pipeline broken right after deploy"

1. Logs at startup will mention the failing component (`error decoding 'service.pipelines'`, `unknown type ...`).
2. `otelcol validate --config` (or use the `otel-config-validator` skill) on the new file.
3. If using contrib-only components on a `core` distribution, the binary fails to load that component — switch image.

## "High CPU"

1. `pprof` CPU profile, 30s sample.
2. Common cost centers: `transform` heavy OTTL, `filter` regex-heavy, `tail_sampling` with many policies, `prometheus` receiver scraping at high frequency, `debug` exporter at `detailed` verbosity left in prod.

# Output format

Always produce a diagnostic report with these sections:

```markdown
## 🔎 Symptom
<what the user reported, in their words, plus what you confirmed observed>

## 🎯 Likely root cause
<single primary cause, with the evidence — log line / metric value / zpages observation that supports it>

## 🧪 Evidence
- <observation 1, with source>
- <observation 2, with source>
- ...

## 🔧 Action items
1. <immediate mitigation, command or config patch>
2. <follow-up, e.g. add monitoring, tune parameter>
3. <prevention, e.g. add an alert, adjust resource limits>

## ❓ Open questions / missing data
- <what data was missing that would have helped narrow further>
```

If the user only provided partial information (e.g., logs but no metrics), say so and list the missing data needed for a confident diagnosis. Don't pretend certainty.

# Skill self-pitfalls

- Don't recommend `verbosity: detailed` permanently — it's a short-window diagnostic tool.
- Don't escalate `service.telemetry.logs.level` to `debug` and forget to revert.
- Don't ask for config changes that require a restart in the middle of an incident without flagging the cost.
- If the answer involves running commands the user hasn't approved (kubectl, curl into the pod), say what you'd run rather than assuming you can.
- Distinguish *symptoms* from *root cause* in the report. `memory_limiter` firing is almost always a symptom of egress pressure, not the root cause.
