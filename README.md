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

Triggers on prompts like *"validate my collector config"*, *"audit OTLP pipeline"*, *"my collector OOMs"*, *"my traces aren't arriving"*.

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
