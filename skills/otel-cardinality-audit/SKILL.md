---
name: otel-cardinality-audit
description: Use when investigating, sizing, or fixing metric cardinality explosion in an OpenTelemetry pipeline — backend rejecting series, Prometheus/Mimir hitting label limits, Elastic TSDB blowing up, runaway billing on a metrics SaaS, or a sudden spike in unique series after a deploy. Triggers on "metric cardinality explosion", "too many series", "Mimir 429 / max-series-per-user", "Prometheus out of memory", "label cardinality", "high-cardinality attribute", "drop labels", "metric_relabel_configs", "delete_matching_keys", "request_id on a metric", "trace_id leaked into metrics", "URL path as label", "pod name on long-lived metric", or any task where the user has counted (or wants to count) distinct label-value combinations and decide what to remove. Diagnoses where the cardinality is coming from, ranks offending labels, and produces a remediation YAML using `attributes` / `transform` / `filter` processors.
---

# When to use this skill

The user has either:

- **Observed a symptom** — backend errors (`max-series-per-user-per-metric exceeded`, `429`, `field_limit_exceeded`), gateway memory creeping up after a deploy, Prometheus eating RAM, billing spike on a metrics SaaS — and suspects label cardinality.
- **Wants to audit before it happens** — a config change, a new instrumentation library, a new service shipping metrics, and they want to know if any attribute is going to explode the series count.
- **Wants a remediation** — a list of suspect labels and the OTel processor block that drops or normalizes them.

If the question is just about writing OTTL syntax to drop a key, hand to `ottl-helper`. If it's a general config review, hand to `otel-config-validator`. This skill is the cardinality-specific workflow: **measure → rank → fix**.

# Mental model — what cardinality really is

Cardinality of a metric is the number of distinct **(metric name, label-set)** combinations that the backend stores as separate series. Each series is a row in the time-series database, with its own retention cost.

```
cardinality(metric) ≈ ∏ distinct_values(label_i)
```

It's **multiplicative**, not additive. A metric with three labels of size {10, 50, 5} produces up to **2,500** series. Adding one extra label with 100 distinct values lifts the ceiling to **250,000**. This is why the wrong attribute on a metric is catastrophic and the same attribute on a log or trace is fine — logs/traces are not aggregated by label set.

Two distinct dangers:

1. **Unbounded fields** — `request_id`, `trace_id`, `span_id`, `user_id`, `session_id`, raw URL with IDs, `instance` with ephemeral pod hash, `pod.name` (StatefulSet OK, Deployment churn). Each request creates a new series. Rate of growth ≈ request rate.
2. **Bounded but very large fields** — `customer_id` (100k tenants), full URL path enumeration, `k8s.namespace.name × k8s.deployment.name × k8s.container.name` cross-product on shared metrics. Stable but huge.

Acceptable cardinality is backend-specific:

| Backend | Soft limit per metric | Hard limit / behavior |
|---|---|---|
| Prometheus / Mimir | ~10k series/metric is loose; 100k is bad | `max-series-per-user-per-metric` (default 100k) → 429 on ingest |
| Cortex / GrafanaCloud Mimir | per-tenant configurable | `max_global_series_per_user` |
| Elastic TSDB | per-data-stream | `index.mapping.total_fields.limit` (default 1000) on dimensions |
| Datadog | per-metric custom-metric quota | charged per million; soft cap then sales call |
| New Relic | event-attribute cap | dropped at the agent if too high |

# Audit workflow

Five steps, in order. Don't jump to remediation without having ranked the offenders.

## Step 1 — Confirm the symptom and signal

Ask (or infer from the user's message):

- **Backend error or alert?** — exact message, the metric name(s) involved, the timestamp the issue started.
- **Sudden vs gradual?** — sudden after a deploy points to a new attribute or a new instrumentation library; gradual points to organic growth (more pods, more tenants).
- **Per-metric or pipeline-wide?** — one metric exploding looks different from every metric growing together (probably a global resource attribute issue).
- **Where is the count taken?** — backend (after collector), collector (`otelcol_exporter_sent_metric_points_total`), or SDK (instrumentation count).

## Step 2 — Measure cardinality at the source

Pick the cheapest data source that answers the question.

### Self-telemetry (collector)

Enable the collector's own metrics (`service.telemetry.metrics.address: 0.0.0.0:8888`) and look at:

- `otelcol_processor_accepted_metric_points` / `_refused_metric_points` — flow rate.
- `otelcol_exporter_sent_metric_points{exporter="..."}` — what's leaving.
- `otelcol_processor_batch_batch_send_size{processor="batch"}` — average batch size; a sudden drop hints at a flood of unique series too small to batch.

### Prometheus-style introspection (the backend)

If the backend is Prometheus-compatible (Prometheus, Mimir, Cortex, Thanos, GrafanaCloud, VictoriaMetrics, Elastic in promQL mode):

```promql
# Top metrics by series count (most useful single query)
topk(20, count by (__name__)({__name__=~".+"}))

# Within a single metric, rank labels by distinct values
count by (label_name) (count by (metric_name, label_name) (metric_name))

# Distinct values of one suspect label across a metric
count(count by (suspect_label)(metric_name))

# Mimir/Cortex: per-tenant cardinality endpoint (no PromQL)
GET /api/v1/cardinality/label_names
GET /api/v1/cardinality/label_values?label_names[]=suspect_label
```

### debug exporter (collector)

Wire a temporary `debug` exporter at the end of the metrics pipeline and tail a sample:

```yaml
exporters:
  debug:
    verbosity: detailed       # in dev only; floods logs
    sampling_initial: 5
    sampling_thereafter: 200
```

Then `kubectl logs … | grep <metric_name>` and inspect the attributes block for each datapoint. The eyeball test catches `request_id` / UUIDs immediately.

### Elastic / OpenSearch

```
GET <metrics-data-stream>/_field_caps?fields=*
GET <metrics-data-stream>/_search
{ "size": 0, "aggs": { "card_user": { "cardinality": { "field": "attributes.user_id" } } } }
```

## Step 3 — Rank the offenders

For each suspect label, capture: distinct value count, growth rate (per hour or per deploy), business need, source of the value (SDK / k8sattributes / resourcedetection / scrape relabeling).

Score the label as one of:

| Verdict | Meaning | Action |
|---|---|---|
| 🔴 **Drop unconditionally** | unbounded id, no aggregation use | `delete_key` / `delete_matching_keys` |
| 🟠 **Normalize** | bounded after a regex (e.g. URL path with IDs) | `replace_pattern` to a low-card form |
| 🟡 **Cap or bucket** | bounded but very large; aggregation is acceptable | move to histogram bucket, `truncate_all`, or aggregate by a parent dimension |
| 🟢 **Keep** | bounded, small, dashboard-relevant | no action |

The rule of thumb: if you wouldn't put it on a Grafana dropdown, it doesn't belong on a metric.

## Step 4 — Pick the right tool

| Tool | Use when |
|---|---|
| **`attributes` processor** (`actions: [{action: delete, key: ...}]`) | Drop or hash a known attribute by name. Simple, declarative, predates OTTL |
| **`transform` (OTTL)** at `context: datapoint` | Bulk delete by regex (`delete_matching_keys`), normalize a value (`replace_pattern`), cap length (`truncate_all`) |
| **`filter` (OTTL)** | Drop entire datapoints whose attribute combinations are noise (debug metrics, healthcheck spans bleeding into metrics) |
| **`metricstransform` processor** | Aggregate away a label entirely (sum across `pod.name` → keep only `service.name`) |
| **Prometheus receiver `metric_relabel_configs`** | Pre-collector, only for prometheus-scrape sources. Drops at scrape time |
| **SDK Views API** (Java/.NET/Go SDK) | Best fix when feasible: don't emit the high-card attribute in the first place |

The hierarchy: **fix at the source (SDK Views) > fix at the prometheus receiver (relabel) > fix at the collector (transform/attributes) > tell the backend to absorb it**. The further upstream, the cheaper.

## Step 5 — Produce the remediation YAML

Give the user a deployable processor block with comments explaining each rule, and a verification step.

# Remediation recipes

Each recipe is a self-contained YAML block that can drop into a metrics pipeline. **All examples assume `error_mode: ignore`** so a single bad datapoint doesn't drop the whole batch.

## Recipe 1 — Drop unbounded id-like keys (the workhorse)

```yaml
processors:
  transform/cardinality:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          # Drop common unbounded ids that should never be on metrics.
          - delete_matching_keys(attributes, "^(request_id|trace_id|span_id|session_id|correlation_id|user_id|order_id|tenant_request_id)$")
```

Use a single `delete_matching_keys` rather than a chain of `delete_key`s — same cost, easier to maintain.

## Recipe 2 — Targeted single-key drop (when you know the exact attribute)

```yaml
processors:
  attributes/cardinality:
    actions:
      - key: customer_uuid
        action: delete
      - key: http.user_agent     # always wide, almost never used as a metric dimension
        action: delete
```

The `attributes` processor is the right tool when the rule is *just* "drop these named keys". No OTTL needed.

## Recipe 3 — Normalize URL paths (the second-most-common offender)

```yaml
processors:
  transform/normalize_url:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          # Replace UUIDs in path with ":id" so /users/4f3e... → /users/:id
          - replace_pattern(attributes["http.route"], "[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}", ":id") where attributes["http.route"] != nil
          # Replace any numeric path segment with ":id"
          - replace_pattern(attributes["http.route"], "/[0-9]+(/|$)", "/:id$$1") where attributes["http.route"] != nil
          # Drop the full url, keep only the route
          - delete_key(attributes, "http.url")
```

If `http.route` is missing (some libraries set only `http.target` or `http.url`), normalize that field instead — but prefer the route if it exists, since the SDK has already templated it.

## Recipe 4 — Cap pathologically long string values

```yaml
processors:
  transform/cap_lengths:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          # Truncate every string attribute to 64 chars; protects against accidentally putting
          # a serialized payload or stack trace into a label.
          - truncate_all(attributes, 64)
```

Pair with monitoring on `otelcol_processor_*_dropped_log_records` — if it spikes after this lands, an attribute was load-bearing somewhere.

## Recipe 5 — Aggregate a high-card label out of existence

```yaml
processors:
  metricstransform/drop_pod:
    transforms:
      # Sum http_server_requests across pod.name so we keep service-level series only.
      - include: http_server_requests_total
        match_type: strict
        action: update
        operations:
          - action: aggregate_labels
            label_set: [service.name, http.method, http.status_code]    # the labels to KEEP
            aggregation_type: sum
```

`metricstransform` aggregates the datapoint values across the dropped labels, which is the right semantic for sums and counts. For gauges and histograms, prefer dropping the label at the SDK or accept the imprecise aggregation.

## Recipe 6 — Filter out a whole metric family that's noise

```yaml
processors:
  filter/drop_debug_metrics:
    error_mode: ignore
    metrics:
      metric:
        # Drop metrics whose name matches a known debug/internal pattern.
        - 'IsMatch(name, "^(debug_|test_|internal_temp_).*")'
```

Use sparingly. A blanket filter is a maintenance burden. Better to fix the source.

## Recipe 7 — Prometheus receiver relabel (cardinality fix at scrape time)

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: my-service
          static_configs:
            - targets: ['service:9090']
          metric_relabel_configs:
            # Drop the request_id label from any metric that exposes it.
            - action: labeldrop
              regex: request_id|trace_id|user_id
            # Drop the entire histogram if it carries unbounded path label
            - source_labels: [__name__, path]
              regex: 'http_request_duration_seconds;.*\\?.*'
              action: drop
```

Cheaper than the collector-side fix because the high-card series never enters the pipeline.

# Pipeline placement

In a multi-processor pipeline, cardinality processors go **after** enrichment (`k8sattributes`, `resourcedetection`) and **before** `batch`:

```
memory_limiter → k8sattributes → resourcedetection → transform/cardinality → batch → exporter
```

Reasoning:

- After enrichment so any high-card resource attribute (`k8s.pod.name`, `host.id`) added by the enricher is in scope of the drop rule.
- Before `batch` so the cardinality reduction shrinks the batch.

# Output format

When auditing or fixing, produce this report:

```markdown
## 🩺 Symptom
<what the user observed; the backend message; the metric(s) involved>

## 🔍 Suspects
| Attribute | Distinct values | Verdict | Source |
|---|---|---|---|
| `request_id` | unbounded (~RPS) | 🔴 Drop | SDK auto-instrumentation |
| `http.route` | ~60 (after normalization) | 🟠 Normalize | SDK |
| `k8s.pod.name` | ~30 churning daily | 🟠 Aggregate | k8sattributes |
| `service.name` | 12 | 🟢 Keep | resourcedetection |

## ⚙️ Configuration
\`\`\`yaml
<the deployable processor block(s), ordered for the pipeline>
\`\`\`

## ✅ Verification
- After deploy, run `topk(20, count by (__name__)({__name__=~".+"}))` and confirm the affected metric drops by the expected factor.
- Watch `otelcol_exporter_sent_metric_points` for the metrics pipeline; expect a step-down proportional to series reduction.
- Confirm the dashboards / alerts using the dropped labels still work (or have been updated).

## 💬 Rationale
<one paragraph: why this set of fixes, what was traded off, anything deferred>

## 📋 TODO before deploying
- <e.g., upstream SDK View change in the Java service to stop emitting `request_id`>
- <e.g., update the Mimir tenant `max_global_series_per_user` if reduction not enough>
- <e.g., delete the existing high-card series from the backend (Prom: tombstones; Mimir: forget)>
```

If the user asked only for an audit (no fix), stop after `🔍 Suspects` and `💬 Rationale`. Don't write a YAML they didn't ask for.

# Skill self-pitfalls

- **Cardinality is multiplicative.** When ranking, multiply the distinct-value counts across the labels on a metric to estimate the worst case. Two labels of size 50 and 200 are not "250 series", they are "10 000".
- **`pod.name` on long-lived counters is a slow-burn explosion.** Each rolling deploy churns the label and the old series stays for the retention window. Aggregate it out for cumulative metrics; keep it on short-retention debug metrics.
- **Don't drop `service.name` or `service.namespace`.** They look high-card in big environments but they are the primary axis of every dashboard. Filter or aggregate other labels first.
- **`http.url` is the worst offender by default.** It contains query strings, fragments, and IDs. `http.route` (templated) is the right field; if the SDK only sets `http.url`, normalize it.
- **Existing series don't disappear.** Dropping a label in the collector stops new series, but the backend keeps the old ones until retention expires (Prom: forever unless tombstoned; Mimir: per-tenant retention; Elastic: per-data-stream retention). Always tell the user this and surface a cleanup step if relevant.
- **The fix at the SDK is always better.** OTel SDKs (Java, .NET, Go, Python) have a Views API to drop attributes before they're emitted. If the user controls the application, recommend that as the canonical fix and the collector-side rule as the defense-in-depth.
- **Don't mix `transform` for cardinality with `transform` for general field normalization.** Keep one processor per concern (`transform/cardinality`, `transform/redact_pii`) so an audit log change to one doesn't risk regressing the other.
- **Test the regex.** A wrong `replace_pattern` can match too much (eat real path segments) or too little (miss a UUID variant). Run the rule against a sample dump from `debug` exporter before deploying.
- **`metricstransform` is unmaintained-but-stable.** It still works as of contrib v0.110, but new functionality lives in OTTL. Use it for the aggregate-labels case where OTTL has no equivalent yet, and revisit each minor release.
