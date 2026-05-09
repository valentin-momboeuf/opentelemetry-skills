# CLAUDE.md — opentelemetry-skills

Claude Code skills repo for OpenTelemetry. Each skill lives at `skills/<name>/SKILL.md`.

## Skill writing rules

- **Frontmatter required**: `name` and `description`, with explicit triggers
- **The `description` must say WHEN to trigger** the skill, not just what it does
- **Style**: imperative, concise, no filler. The skill is read by an LLM, not a human
- **Concrete examples** > theory. Include imports, versions, exact commands
- **Pitfalls flagged**: idempotence, permissions, secrets, metrics cardinality, sampling

### SKILL.md skeleton

```markdown
---
name: edot-collector-config
description: Use when configuring the Elastic/OTel Collector (receivers, processors, exporters) for traces/metrics/logs pipelines. Triggers on "configure the collector", "add an exporter", "OTLP pipeline".
---

# When to use it

...

# Steps

1. ...

# Pitfalls

- ...
```

## Repo conventions

- All documentation, code, examples, and commits in **English**
- Conventional commits: `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`
- One folder = one skill. No multi-file skills unless necessary (assets, scripts)
- Embedded scripts go under `skills/<name>/scripts/`, shebang `#!/usr/bin/env bash`, `set -euo pipefail`, must pass `shellcheck`

## Testing a skill locally

```bash
# install the repo as a local plugin
/plugin install /Users/valentin/projets/opentelemetry-skills

# reload after edits
/plugin reload opentelemetry-skills
```

## Scope

- ✅ Instrumentation (auto/manual SDKs, EDOT, OTLP)
- ✅ Collector (configuration, processors, exporters)
- ✅ Backends (Elastic, Jaeger, Prometheus, Grafana)
- ✅ Operational patterns (sampling, cardinality, cost)
- ❌ Generic non-OTel skills (belong elsewhere)
