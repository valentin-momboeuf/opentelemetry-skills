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
