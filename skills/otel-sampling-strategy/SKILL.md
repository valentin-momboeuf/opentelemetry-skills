---
name: otel-sampling-strategy
description: Use when designing or sizing a sampling strategy for OpenTelemetry traces — choosing between head (SDK) and tail (collector) sampling, picking ratios, and configuring `tail_sampling` policies based on traffic volume, error rate, latency requirements, and backend cost. Triggers on "design a sampling strategy", "head vs tail sampling", "configure tail_sampling", "what sampling ratio should I use", "size tail_sampling memory", "Datadog/Tempo/Honeycomb cost too high", "keep all errors", "drop healthchecks at the gateway", "OTEL_TRACES_SAMPLER", "parentbased_traceidratio", "probabilistic sampler", or any task that combines volumetry, sampling, and a cost or memory constraint. Picks head, tail, or combined; sizes ratios from volume + budget; produces a YAML config plus a back-of-envelope volume / memory / cost calculation.
---

# When to use this skill

The user wants to:

- **Choose between head and tail sampling** for a given setup, with rationale.
- **Size sampling ratios** to hit a target volume or backend cost.
- **Configure `tail_sampling`** with the right policies (errors, latency, probabilistic, per-service, rate-limiting) and the right `decision_wait` / `num_traces`.
- **Verify** an existing sampling configuration against stated objectives (no missed errors, hits cost target, doesn't OOM the gateway).

If the user just wants the YAML for a sampling config without any sizing decisions → `otel-pipeline-designer` is enough. This skill is for the strategy + sizing call: when the answer depends on volume, cost, or memory budget.

# Mental model — head vs tail

| Dimension | Head sampling (SDK) | Tail sampling (collector) |
|---|---|---|
| Where decided | At trace start, in the SDK | At trace end, after spans are aggregated |
| Cost | ~free; cuts volume right at the source | Memory-bound at the gateway tier |
| Visibility | Cannot see trace duration, status, or attributes when deciding | Sees the complete trace |
| Topology | Pure SDK config (`OTEL_TRACES_SAMPLER`) | Requires a gateway tier with `loadbalancing` exporter routing by traceID upstream |
| Error preservation | Errors get dropped at the same rate as normals | Can keep 100% of errors while dropping 95% of normals |
| Determinism | Deterministic per traceID (parent-based) | Probabilistic over traces seen in `decision_wait` |
| Best for | High-volume, cost-sensitive, errors not critical | Need errors / slow traces preserved; cost still matters |

The two combine: **head sample at the SDK to cut raw egress** (e.g. 10%), then **tail sample at the gateway to preserve errors and slow traces** within that 10%. Combined ratio multiplies (head 10% × tail 50% = 5% overall).

⚠️ The 90 % of traces head-sampled out are gone forever — including their errors. **If error preservation matters, do not head-sample below 100 %.** Use 100 % head + tail-sampling at the gateway.

# Sizing methodology

Three back-of-envelope budgets to match: **input volume**, **gateway memory**, **backend cost**.

## Step 1 — Input volume

```
incoming_traces_per_sec   = <stated by user, or estimated from RPS × traces_per_request>
avg_spans_per_trace       = <stated, or estimate from instrumentation depth — typically 4–10>
incoming_spans_per_sec    = traces × spans_per_trace
```

## Step 2 — Output target (cost-driven)

Most managed backends price per million spans (Datadog, New Relic, Honeycomb) or per ingested GB (Tempo + S3, Grafana Cloud). Compute the target output rate:

```
target_output_spans_per_sec = monthly_budget / (cost_per_million_spans × seconds_per_month / 1_000_000)
required_keep_ratio         = target_output_spans_per_sec / incoming_spans_per_sec
```

If `required_keep_ratio < 1.0`, sampling is needed. The smaller the ratio, the more aggressive.

## Step 3 — Tail sampling memory

The gateway tier buffers up to `num_traces` traces in memory for `decision_wait` seconds.

Standard starting points:
- `decision_wait: 30s` — covers most synchronous trace durations. Set above your p99 latency.
- `num_traces` ≈ `incoming_traces_per_sec × decision_wait × 1.5` (50 % headroom).
- `expected_new_traces_per_sec` = `incoming_traces_per_sec` (sizes the policy engine's internal maps).

Memory math (rough, ignoring overhead):

```
memory_per_replica = (num_traces × avg_spans_per_trace × avg_span_size_bytes) / num_gateway_replicas
```

`avg_span_size_bytes` is typically 1–2 KB after attributes. Use **1.5 KB** for sizing.

If memory budget is tight, lower `decision_wait` (and lose tail accuracy on slow traces) or scale gateway replicas horizontally.

# Policy catalog

`tail_sampling` policies are evaluated per trace; if **any** policy says `Sampled`, the trace is kept. (OR semantics — the opposite of `filter`.)

| Policy `type` | Use case | Notes |
|---|---|---|
| `status_code` (`status_codes: [ERROR]`) | Keep all errors | Cost ≈ error_rate × incoming volume |
| `latency` (`threshold_ms`) | Keep slow traces | Cost ≈ tail of latency distribution × volume |
| `numeric_attribute` | Specific failure modes (e.g. `http.status_code` ≥ 500) | Often overlaps with `status_code` — pick one |
| `string_attribute` | Per-service preservation (`service.name in [list]`) | The standard quota tool |
| `boolean_attribute` | Custom flag (e.g. `app.is_high_value == true`) | App must set the flag |
| `probabilistic` (`sampling_percentage`) | Baseline of normal traces | The volume-cutting workhorse |
| `rate_limiting` (`spans_per_second`) | Hard cap on noisy services | Useful as a safety net |
| `and` / `or` (composite) | Combine policies | e.g. `service=proxy AND latency>2s` |
| `composite` (with rates per sub-policy) | Quota allocation across categories | Rarely needed; powerful when it is |
| `trace_state` (regex on W3C tracestate) | Keep based on header propagation | Niche |

A typical four-policy stack for a mature deployment: **errors + slow + per-service preservation + probabilistic baseline**.

# Strategy patterns

## Pattern 1 — Cost-driven, errors must survive (most common)

- Head sample: **100 %** (no SDK sampling). Tail needs the full picture.
- Tail sample at gateway:
  - `status_code: [ERROR]` (keep all errors)
  - `latency: threshold_ms: <slo_p99>` (keep slow)
  - `probabilistic: <ratio_to_hit_budget>%` (baseline)

## Pattern 2 — Volume cut, errors not critical

- Head sample at SDK: `parentbased_traceidratio` with the target ratio.
- No tail sampling needed.

```bash
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
```

`parentbased_traceidratio` follows the trace context — if a parent service decided to sample, child services keep that decision. Pure `traceidratio` does not, and produces broken traces in distributed systems.

## Pattern 3 — Two-stage with budget cap

- Head sample at SDK at e.g. 10 % to cut SDK→Collector egress (cheap).
- Tail sample at gateway with errors + slow + probabilistic on the 10 % remainder.
- Combined ratio multiplies: 10 % head × 50 % tail = 5 % overall.
- ⚠️ 90 % of errors are lost at head sampling. Only viable if error preservation is best-effort (e.g. errors are also tracked via dedicated metrics/logs pipelines).

## Pattern 4 — Per-service preservation

- 100 % head sample.
- Tail sample with multiple `string_attribute` policies:
  - keep all from `[payments, checkout, billing]` (low volume, high importance)
  - keep slow + errors everywhere
  - probabilistic 5 % baseline
  - `rate_limiting` on the chatty services as a safety net

# Configuration template

```yaml
processors:
  tail_sampling:
    decision_wait: 30s
    num_traces: 100000               # incoming_traces_per_sec × decision_wait × 1.5
    expected_new_traces_per_sec: 5000

    policies:
      # 1. Always keep errors. Cost ≈ error_rate × volume.
      - name: keep-errors
        type: status_code
        status_code:
          status_codes: [ERROR]

      # 2. Always keep slow traces. Threshold = your SLO p99.
      - name: keep-slow
        type: latency
        latency:
          threshold_ms: 1000

      # 3. Always keep critical low-volume services.
      - name: keep-critical-services
        type: string_attribute
        string_attribute:
          key: service.name
          values: [payments, checkout]

      # 4. Probabilistic baseline of everything else.
      - name: baseline-probabilistic
        type: probabilistic
        probabilistic:
          sampling_percentage: 5
```

This must run on a gateway tier; the agent tier upstream **must** use a `loadbalancing` exporter routing by `traceID` so all spans of a trace land on the same gateway replica.

# Output format

When designing or sizing a strategy, produce this report:

```markdown
## Inputs
- Incoming traces/sec: <value>
- Avg spans/trace: <value or assumption>
- Error rate: <value>
- Latency target (p95 or p99): <value>
- Backend cost or volume budget: <value>
- Topology: <gateway replicas, memory per replica, etc.>

## Strategy
<head only | tail only | combined> — one paragraph rationale tied to the constraints.

## Sizing
| Metric | Value | How |
|---|---|---|
| `decision_wait` | <Xs> | <reasoning vs p99> |
| `num_traces` | <value> | traces/sec × decision_wait × 1.5 |
| Estimated gateway memory per replica | <MiB or GiB> | num_traces × spans/trace × 1.5KB / replicas |
| Probabilistic baseline ratio | <%> | derived from budget |
| Output spans/sec | <value> | sum(kept policies) |
| Estimated monthly cost | <$ or volume> | output × pricing |

## Configuration
```yaml
<deployable config>
```

## Trade-offs
- <what gets dropped that the user might miss>
- <memory/CPU vs accuracy trade-offs>
- <head-sampling caveats if combined>

## TODO before deploying
- <verify error rate / latency assumption against real traffic>
- <ensure loadbalancing tier in place>
- <set memory limits aligned with the per-replica calc>
- <pin contrib version>
```

# Skill self-pitfalls

- **Tail sampling cannot recover errors that head sampling already dropped.** If error preservation matters, head sample at 100 % (no SDK sampling) and only tail sample.
- **Tail sampling needs a load-balancing tier.** Agent → `loadbalancing` exporter routing by `traceID` → gateway replicas. Without this, decisions are inconsistent across replicas and traces are silently broken.
- **`decision_wait` shorter than your slowest synchronous trace** silently drops those traces from the sampling decision (they get evicted before all spans arrive). Size to p99 latency at minimum.
- **`num_traces` too low** → traces evicted before decision → silent loss. Use 50 % headroom over `traces_per_sec × decision_wait`.
- **Memory math is per-replica, not cluster-wide.** Three replicas with `num_traces: 100000` each costs ~3× the memory of one replica with the same setting. Always divide by replica count.
- **`parentbased_traceidratio` is the canonical head sampler.** Don't recommend plain `traceidratio` unless the user explicitly needs to break parent context (rare).
- **Backend pricing is usually per span, not per trace.** Report cost in spans/sec × pricing. A trace with 6 spans costs 6× a single-span trace.
- **"Keep all errors" sounds free but isn't.** Cost = error_rate × incoming volume. State this number explicitly so the user isn't surprised.
- **Report numbers, don't hand-wave.** A sizing answer without an output-spans-per-sec or estimated-monthly-cost line is incomplete — that's what makes the strategy decision concrete.
