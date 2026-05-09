# opentelemetry-skills

Claude Code skills for **OpenTelemetry**: instrumentation, collector, traces, metrics, logs, exporters, and operational best practices.

## Layout

```
opentelemetry-skills/
├── .claude-plugin/
│   └── plugin.json       # Claude Code plugin manifest
├── skills/
│   └── <skill-name>/
│       └── SKILL.md      # one skill per directory (YAML frontmatter + body)
├── CLAUDE.md             # repo conventions
└── README.md
```

Each skill lives in its own folder under `skills/` and contains at minimum a `SKILL.md` file with a YAML frontmatter:

```markdown
---
name: skill-name
description: When to use it (explicit triggers for routing)
---

Skill body...
```

## Skills

### `otel-config-validator`

Validate an OpenTelemetry Collector configuration (`config.yaml`, `otelcol-config.yaml`, or a Helm/K8s manifest containing a `config:` block).

Runs a 3-phase audit:

1. **Syntax & distribution** — identifies the required distribution (core, contrib, EDOT, vendor flavors), runs `otelcol validate` when available, falls back to manual YAML review.
2. **Structural** — cross-references `service.pipelines` against defined components, signal-type coherence, connectors, dead components.
3. **Qualitative** — processor ordering (`memory_limiter` first, `batch` last, `tail_sampling` placement) and the classic pitfalls catalog: TLS, OTLP ports `4317` vs `4318`, `tail_sampling` load-balancing requirement, batch sizing, high-cardinality attributes on metrics, deprecated `logging` exporter, `filelog` replay, missing self-telemetry, k8sattributes RBAC, and more.

Output is a structured report with three buckets: ❌ blocking errors, ⚠️ warnings, 💡 optimization suggestions, each with a YAML path or line number to the offending location.

Triggers on prompts like *"validate my collector config"*, *"audit OTLP pipeline"*.

### `otel-pipeline-designer`

Generate a deployable OpenTelemetry Collector configuration *from scratch* given a natural-language intent like *"K8s logs + Java traces to Tempo + Loki"* or *"VM fleet to Mimir + Tempo"* or *"two-tier with tail sampling to Datadog"*.

Three steps:

1. **Gather intent** — signals (traces/metrics/logs), sources (k8s, VMs, SDKs, prometheus, kafka), environment, backends, and constraints (TLS, multi-tenancy, sampling).
2. **Design topology** — pick the distribution (core vs contrib), the deployment topology (single agent / agent + gateway / sidecar), the receivers, processors (in canonical order: `memory_limiter` first, `k8sattributes`, `resourcedetection`, transforms, filters, `tail_sampling`, `batch` last), and the exporters (with retry+queue, modern `otlphttp` to Loki rather than the deprecated `loki` exporter, `prometheusremotewrite` with `X-Scope-OrgID` for Mimir, `loadbalancing` by traceID before any tail-sampling gateway).
3. **Emit YAML** — a deployable config plus a Topology/Distribution/Configuration/RBAC/Rationale/TODO report. Multi-tier setups get one config per tier.

Output structure: 🗺️ Topology / 📦 Distribution required / ⚙️ Configuration / 🔐 RBAC (if K8s) / 💬 Rationale / 📋 TODO before deploying.

Triggers on prompts like *"design a pipeline for X to Y"*, *"build me an otelcol config"*, *"k8s logs + traces to Tempo+Loki"*, *"two-tier tail sampling to Datadog"*.

### `otel-sampling-strategy`

Design and size an OpenTelemetry sampling strategy from volumetry, error rate, latency targets, and backend cost. Picks between head sampling (SDK), tail sampling (collector), or a combined strategy; sizes the ratios; configures `tail_sampling` policies (errors, latency, per-service preservation, probabilistic baseline, rate limiting); produces a deployable YAML plus a back-of-envelope volume / memory / cost calculation.

Three-budget sizing methodology:

1. **Input volume** — incoming traces/sec × spans/trace.
2. **Output target** (cost-driven) — derived from the backend's per-million-spans price or storage budget.
3. **Gateway memory** — `num_traces × spans/trace × span_size / replicas`, sized at `traces/sec × decision_wait × 1.5` headroom.

Strategy patterns covered: cost-driven with errors-must-survive, volume-cut-no-error-preservation, two-stage head + tail, per-service preservation with quota-style policies. Each pattern names the right tool and the canonical caveat (e.g. "head sampling cannot recover errors that the SDK already dropped").

Output is a structured report: 📋 Inputs / 🎯 Strategy / 📐 Sizing (table with computed numbers) / ⚙️ Configuration / ⚖️ Trade-offs / ✅ TODO before deploying.

Triggers on prompts like *"design a sampling strategy"*, *"head vs tail"*, *"what sampling ratio should I use"*, *"size tail_sampling memory"*, *"Datadog cost too high"*, *"keep all errors"*, *"OTEL_TRACES_SAMPLER"*, *"parentbased_traceidratio"*.

### `ottl-helper`

Write, explain, or validate OTTL (OpenTelemetry Transformation Language) statements for the contrib `transform`, `filter`, and `routing` (connector) processors.

Three modes:

1. **Write** — given an intent (*"redact emails in log bodies"*, *"drop healthcheck spans"*, *"route checkout traces to Datadog"*), produce the OTTL statement(s), the surrounding YAML block, and notes on `error_mode`, regex anchoring, and type coercion.
2. **Explain** — given an existing OTTL block, walk through it: which processor (mutate / drop / select), which context (resource/scope/span/spanevent/metric/datapoint/log), what each statement does, what gets dropped vs. mutated vs. routed.
3. **Validate** — find filter inversions ("DROP" not "keep" — the easy-to-flip semantic), missing `route()` in routing connector entries, unanchored regex over-matches, type mismatches in comparisons (`Int(...)` vs string `"200"`), trace-level operations attempted in per-span `filter`, and missing `error_mode: ignore` in production.

Includes a function reference (mutators, type/parsing, string, boolean predicates, time) and a recipe book covering the common patterns: PII redaction, healthcheck dropping, env tagging, high-cardinality key removal, and routing by `service.name`.

Triggers on prompts like *"write an OTTL statement"*, *"explain this transform processor"*, *"validate my filter"*, *"redact this attribute"*, *"drop these spans"*, *"route metrics by ..."*, *"transformprocessor"*, *"filterprocessor"*, *"routing connector"*.

### `otel-collector-debug`

Diagnose a *running* OpenTelemetry Collector when symptoms hit: no data reaching the backend, OOM/restart loops, dropped spans/metrics/logs, exporter retries, queue full, high CPU, or wrong data shape at the backend.

Walks the standard diagnostic toolbox:

1. **Collector logs** — error patterns (`Sending queue is full`, `data refused due to memory limit`, `dial tcp ... refused`, `unknown component`).
2. **Self-metrics** (`:8888/metrics`) — `otelcol_receiver_accepted_*`, `_processor_refused_*`, `_exporter_send_failed_*`, `_queue_size`/`_queue_capacity`.
3. **zpages** (`:55679/debug/{tracez,pipelinez,extensionz,servicez}`).
4. **debug exporter** to inspect what's actually flowing through the pipeline.
5. **pprof** (`:1777/debug/pprof/{heap,profile,goroutine}`) for memory and CPU issues.
6. **health_check** / **health_check_v2** for liveness vs per-pipeline health.
7. **telemetrygen** to take real clients out of the loop.

A symptom→diagnosis decision tree maps "no data arriving", "OOM", "dropped signals", "wrong attributes/cardinality", "broken-after-deploy", and "high CPU" to their primary root causes. Crucially, the skill enforces the distinction between *symptoms* (e.g. `memory_limiter` firing) and *root cause* (e.g. backend egress saturation) — the most common mis-diagnosis in collector incidents.

Output is a structured diagnostic report: 🔎 Symptom / 🎯 Likely root cause / 🧪 Evidence / 🔧 Action items / ❓ Open questions.

Triggers on prompts like *"my collector is dropping data"*, *"no traces arriving in Tempo"*, *"collector OOMing"*, *"Mimir is hitting cardinality limits since the rollout"*, *"debug otel collector"*.

## Local install

Add the repo as a local plugin in Claude Code:

```bash
# from any project
/plugin install /Users/valentin/projets/opentelemetry-skills
```

Or as a Git marketplace once published:

```bash
/plugin marketplace add <git-url>
```

## Conventions

- All documentation, code, and commits in **English**
- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`
- See [CLAUDE.md](./CLAUDE.md) for the full guide
