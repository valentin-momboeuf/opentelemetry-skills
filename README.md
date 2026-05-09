# opentelemetry-skills

Collection de skills Claude Code autour d'**OpenTelemetry** : instrumentation, collector, traces, métriques, logs, exporters et bonnes pratiques d'exploitation.

## Structure

```
opentelemetry-skills/
├── .claude-plugin/
│   └── plugin.json       # Manifeste du plugin Claude Code
├── skills/
│   └── <skill-name>/
│       └── SKILL.md      # Un skill par dossier (frontmatter YAML + contenu)
├── CLAUDE.md             # Conventions du repo
└── README.md
```

Chaque skill est un dossier sous `skills/` contenant au minimum un fichier `SKILL.md` avec en-tête YAML :

```markdown
---
name: nom-du-skill
description: Quand l'utiliser (déclencheurs explicites pour le routage)
---

Contenu du skill...
```

## Installation locale

Ajouter le repo comme plugin local dans Claude Code :

```bash
# depuis n'importe quel projet
/plugin install /Users/valentin/projets/opentelemetry-skills
```

Ou en tant que marketplace Git une fois publié :

```bash
/plugin marketplace add <git-url>
```

## Conventions

- Documentation en **français**, code et commits en **anglais**
- Commits conventionnels : `feat:`, `fix:`, `docs:`, `refactor:`, `chore:`
- Voir [CLAUDE.md](./CLAUDE.md) pour le détail
