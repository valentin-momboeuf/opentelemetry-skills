---
name: otel-config-validator
description: Use when reviewing or validating an OpenTelemetry Collector configuration (otelcol / otelcol-contrib / EDOT YAML). Triggers on "validate my collector config", "review otel config", "audit OTLP pipeline", "my collector won't start", "the collector is OOMing", "why aren't my traces arriving", or any review of a `config.yaml` for the OpenTelemetry Collector. Checks YAML schema, distribution coverage, pipeline references, processor order (memory_limiter first / batch last), and the classic pitfalls (TLS, OTLP ports 4317 vs 4318, tail_sampling load-balancing, batch sizing, high-cardinality attributes on metrics, debug exporter verbosity, k8sattributes RBAC).
---

# When to use this skill

The user submits an OpenTelemetry Collector configuration (`config.yaml`, `otelcol-config.yaml`, or a Helm/K8s manifest containing a `config:` block) and you need to:

- validate it before deployment, or
- diagnose unexpected behavior (OOM, traces not arriving, broken sampling, log flooding, runaway metric cardinality).

If what you're looking at is not a collector config (e.g. an OTel SDK config inside an application, or a backend config like Jaeger/Prometheus), do not apply this skill.

# Method

Three phases, in order. Do not skip a phase: a phase 3 issue can mask a phase 2 issue, and vice-versa.

## Phase 1 — Syntax and distribution

1. **Identify the distribution** from the Docker image, binary, or components used:
   - `otel/opentelemetry-collector` (core) — minimal
   - `otel/opentelemetry-collector-contrib` — most useful components
   - `docker.elastic.co/beats/elastic-agent` or EDOT collector
   - `quay.io/signalfx/splunk-otel-collector`, `public.ecr.aws/aws-observability/aws-otel-collector`, etc.

   Components **not** in `core` (so they require contrib or a derivative):
   `tail_sampling`, `transform`, `filter`, `k8sattributes`, `resourcedetection`, `filelog`, `journald`, `loadbalancing` (exporter), `routing` (connector).

2. **Run native validation**:
   ```bash
   otelcol-contrib validate --config=config.yaml
   ```
   Or via Docker to match the target version:
   ```bash
   docker run --rm -v "$(pwd)/config.yaml:/etc/otelcol/config.yaml" \
     otel/opentelemetry-collector-contrib:0.110.0 \
     validate --config=/etc/otelcol/config.yaml
   ```

3. **If the binary isn't available**, review the YAML manually:
   - Consistent indentation (spaces, never tabs)
   - Allowed top-level keys: `extensions`, `receivers`, `processors`, `exporters`, `connectors`, `service`
   - Each pipeline declares at least `receivers` (≥1) and `exporters` (≥1); `processors` is optional but recommended
   - No unknown keys (typos like `processers`, `exportor`)

## Phase 2 — Structural validation

Cross-reference `service.pipelines` with defined components:

- ✅ Every component cited in a pipeline is defined in the matching section
- ✅ Every defined component is used in at least one pipeline (otherwise: warning, drifting dead code)
- ✅ No duplicate names within the same section
- ✅ Names (`otlp`) and aliases (`otlp/jaeger`, `otlp/internal`) are consistent between definition and reference
- ✅ Signal type coherence: a `tail_sampling` processor only makes sense in a `traces` pipeline; a `filelog` receiver doesn't feed `traces` or `metrics` pipelines
- ✅ Extensions referenced under `service.extensions` are defined at the top (and vice-versa, otherwise not loaded)
- ✅ Connectors: if a component is used both as an exporter on one pipeline and a receiver on another, it must be a `connector`, not a regular exporter

## Phase 3 — Qualitative audit

### Processor ordering

OpenTelemetry convention, first to last in the pipeline:

1. **`memory_limiter`** — always first, otherwise OOM under load.
2. **`k8sattributes` / `resourcedetection`** — context enrichment (before any transform that might use it).
3. **`attributes` / `resource` / `transform`** — field modifications.
4. **`filter` / `probabilistic_sampler`** — early volume reduction = savings downstream.
5. **`tail_sampling`** — final per-trace decision, after all spans have arrived.
6. **`batch`** — always last before exporters. Optimizes RPCs.

Red flags to surface immediately:

- `batch` placed before a sampler or filter → batching prevents per-trace decisions or breaks filter efficiency.
- `memory_limiter` missing or not in first position → imminent OOM under load.
- `tail_sampling` after `batch` → incorrect behavior.

### Pitfalls catalog

**`memory_limiter`**
- Missing on a pipeline that receives external traffic → critical.
- `limit_mib + spike_limit_mib` must stay `<` memory allocated to the container/process, otherwise the limiter triggers after the OOM kill.
- `check_interval: 1s` recommended.

**`batch`**
- `send_batch_size` (default 8192): normal target.
- `send_batch_max_size`: hard cap; if defined, must be ≥ `send_batch_size`.
- `timeout`: force flush, e.g. `200ms` for traces, longer for low-frequency logs.
- Always in **last position** before exporters.

**`tail_sampling`**
- All spans of a single trace must reach the same collector instance.
- In multi-instance setups: an upstream tier with the `loadbalancing` exporter routing by `traceID`.
- `decision_wait`: how long to wait for a complete trace (typically 30s).
- `num_traces`: memory capacity, watch this.
- Combined with an upstream `batch` = breaks proper operation.

**TLS / security**
- `tls.insecure: true` or no `tls` block in production = clear-text traffic. Acceptable only if TLS terminates elsewhere (sidecar, service mesh) — should be made explicit in a comment.
- `tls.insecure_skip_verify: true`: rarely justified outside dev/PoC.
- mTLS recommended between internal collectors (gateway ↔ agent).

**OTLP ports**
- `4317` = gRPC
- `4318` = HTTP (JSON or protobuf)
- Bind to `0.0.0.0:4317` required in containers (not `localhost` or `127.0.0.1`, otherwise unreachable from outside the network namespace).

**Exporters**
- `logging` exporter → renamed to `debug` since v0.86 (deprecated, then removed). Update accordingly.
- `debug` with `verbosity: detailed` in production = log flooding and CPU cost.
- For any network exporter: configure `sending_queue` + `retry_on_failure` (resilience to transient failures).
- `sending_queue.storage:` for queue persistence (extension `file_storage`).

**Receivers**
- `filelog`:
  - `start_at: beginning` replays the entire history at startup (drama on large files).
  - Prefer `start_at: end` + `storage:` (extension `file_storage`) to persist position.
  - `include_file_path: true` is useful, but watch backend cardinality.
- `prometheus`: use `metric_relabel_configs` to drop labels that would explode cardinality.
- `otlp`: declare `protocols.grpc` AND/OR `protocols.http` depending on clients; an HTTP client hitting a gRPC-only endpoint fails silently on the collector side.
- `hostmetrics`: the active scrapers list (`cpu`, `memory`, `disk`, `filesystem`, `network`, …) heavily impacts volume — don't enable everything by default.

**Metrics cardinality**
- `attributes` / `resource` / `transform` injecting highly variable fields (UUID, IP, `user_id`, `request_id`, `trace_id`, `span_id`) on **metrics** → cardinality explosion in the backend (Prometheus, Mimir, Elastic TSDB).
- Acceptable on traces/logs, never on metrics. Always flag this.

**Collector self-observability**
- `service.telemetry.metrics.level: detailed` and `address: 0.0.0.0:8888` to expose internal metrics (`otelcol_*`).
- `service.telemetry.logs.level: warn` in production (`info` is verbose, `debug` is huge).
- Without this internal telemetry, the collector is blind to itself → no diagnostics possible.

**Kubernetes — `k8sattributes`**
- Requires a `ClusterRole` with `get,list,watch` on `pods`, `namespaces`, and depending on configuration `replicasets`, `nodes`.
- Deployment: `DaemonSet` for local node-scoped enrichment, or `Deployment` with `passthrough: true` for pass-through mode.
- Without the RBAC: pods start fine but k8s attributes silently miss.

**Versioning**
- `alpha` / `beta` components: breaking changes between minor versions, must be pinned.
- Make sure the binary version matches the components used (a field removed in v0.110 will be rejected).

# Output format

Produce a structured report with three sections:

```markdown
## ❌ Errors (blocking)
- [`processors.batch`] missing at the end of the `traces` pipeline, line 42
- [`service.pipelines.traces.processors`] references undefined `tail_sampling/v2`

## ⚠️ Warnings (should fix)
- [`exporters.otlp.tls.insecure`] = true in declared prod environment, line 88
- [`processors.attributes/enrich`] adds `user_id` on the `metrics` pipeline → cardinality

## 💡 Suggestions (optimizations)
- [`receivers.filelog.start_at`] switch to `end` + `file_storage` extension for persistence
- [`service.telemetry.metrics`] not exposed, the collector is blind to itself
```

Always cite the YAML path (`processors.batch.timeout`) or line number so the user knows where to fix.

If everything is fine: a single `✅ Configuration valid` block with the list of detected components, their roles, and the required distribution.

# Skill self-pitfalls

- Don't invent components. If a component isn't known, flag it as such and suggest `otelcol validate` rather than asserting.
- Don't rewrite the entire config without being asked. The report is the primary deliverable; patches are produced only on explicit request.
- Collector versions evolve quickly (breaking changes in minor releases). If unsure about a field, say the docs for the target version should be consulted.
