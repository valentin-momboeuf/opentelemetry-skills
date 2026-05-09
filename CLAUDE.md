# CLAUDE.md — opentelemetry-skills

Repo de skills Claude Code autour d'OpenTelemetry. Chaque skill vit dans `skills/<nom>/SKILL.md`.

## Règles d'écriture des skills

- **Frontmatter obligatoire** : `name` et `description` (en français, avec déclencheurs explicites)
- **Le `description` doit dire QUAND déclencher** le skill, pas seulement ce qu'il fait
- **Style** : impératif, concis, pas de remplissage. Le skill est lu par un LLM, pas un humain
- **Exemples concrets** > théorie. Inclure les imports, versions, et commandes exactes
- **Pièges signalés** : idempotence, permissions, secrets, cardinalité métriques, sampling

### Squelette d'un SKILL.md

```markdown
---
name: edot-collector-config
description: Use when configuring the Elastic/OTel Collector (receivers, processors, exporters) for traces/metrics/logs pipelines. Triggers on "configure le collector", "ajoute un exporter", "pipeline OTLP".
---

# Quand l'utiliser

...

# Étapes

1. ...

# Pièges

- ...
```

## Conventions repo

- Documentation : **français** — Code, exemples, commits : **anglais**
- Commits conventionnels : `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`
- Un dossier = un skill. Pas de skill multi-fichier sauf si nécessaire (assets, scripts)
- Si un skill embarque des scripts : `skills/<nom>/scripts/`, shebang `#!/usr/bin/env bash`, `set -euo pipefail`, passe `shellcheck`

## Tester un skill localement

```bash
# installer le repo comme plugin local
/plugin install /Users/valentin/projets/opentelemetry-skills

# recharger après modif
/plugin reload opentelemetry-skills
```

## Périmètre

- ✅ Instrumentation (SDK auto/manuel, EDOT, OTLP)
- ✅ Collector (configuration, processors, exporters)
- ✅ Backends (Elastic, Jaeger, Prometheus, Grafana)
- ✅ Patterns d'exploitation (sampling, cardinalité, coût)
- ❌ Skills génériques non-OTel (à mettre ailleurs)
