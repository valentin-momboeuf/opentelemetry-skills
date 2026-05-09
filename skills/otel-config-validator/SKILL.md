---
name: otel-config-validator
description: Use when reviewing or validating an OpenTelemetry Collector configuration (otelcol / otelcol-contrib / EDOT YAML). Triggers on "valide ma config collector", "review otel config", "audit pipeline OTLP", "mon collector ne démarre pas", "le collector OOM", "pourquoi mes traces n'arrivent pas", or any review of a `config.yaml` for the OpenTelemetry Collector. Checks YAML schema, distribution coverage, pipeline references, processor order (memory_limiter first / batch last), and the classic pitfalls (TLS, OTLP ports 4317 vs 4318, tail_sampling load-balancing, batch sizing, high-cardinality attributes on metrics, debug exporter verbosity, k8sattributes RBAC).
---

# Quand utiliser ce skill

L'utilisateur soumet une configuration OpenTelemetry Collector (`config.yaml`, `otelcol-config.yaml`, ou un manifeste Helm/K8s contenant un bloc `config:`) et il faut :

- la valider avant déploiement,
- ou diagnostiquer un comportement bizarre (OOM, traces qui n'arrivent pas, sampling raté, logs noyés, métriques à cardinalité explosive).

Si ce qu'on examine n'est pas une config collector (par ex. config d'un SDK OTel côté appli, ou d'un backend type Jaeger/Prometheus), ne pas appliquer ce skill.

# Méthode

Trois phases dans cet ordre. Ne pas sauter de phase : un piège de phase 3 peut masquer un problème de phase 2, et vice-versa.

## Phase 1 — Validation syntaxique et distribution

1. **Identifier la distribution** depuis l'image Docker, le binaire ou les composants utilisés :
   - `otel/opentelemetry-collector` (core) — minimal
   - `otel/opentelemetry-collector-contrib` — la plupart des composants utiles
   - `docker.elastic.co/beats/elastic-agent` ou EDOT collector
   - `quay.io/signalfx/splunk-otel-collector`, `public.ecr.aws/aws-observability/aws-otel-collector`, etc.

   Composants qui ne sont **pas** dans `core` (donc nécessitent contrib ou un dérivé) :
   `tail_sampling`, `transform`, `filter`, `k8sattributes`, `resourcedetection`, `filelog`, `journald`, `loadbalancing` (exporter), `routing` (connector).

2. **Lancer la validation native** :
   ```bash
   otelcol-contrib validate --config=config.yaml
   ```
   Ou via Docker pour matcher la version cible :
   ```bash
   docker run --rm -v "$(pwd)/config.yaml:/etc/otelcol/config.yaml" \
     otel/opentelemetry-collector-contrib:0.110.0 \
     validate --config=/etc/otelcol/config.yaml
   ```

3. **Si le binaire n'est pas accessible**, vérifier le YAML à la main :
   - Indentation cohérente (espaces, jamais tabs)
   - Clés top-level autorisées : `extensions`, `receivers`, `processors`, `exporters`, `connectors`, `service`
   - Chaque pipeline déclare au minimum `receivers` (≥1) et `exporters` (≥1) ; `processors` est optionnel mais recommandé
   - Pas de clés inconnues (typo type `processers`, `exportor`)

## Phase 2 — Validation structurelle

Cross-référencer `service.pipelines` avec les composants définis :

- ✅ Tout composant cité dans une pipeline est défini dans la section correspondante
- ✅ Tout composant défini est utilisé dans au moins une pipeline (sinon : warning, code mort qui dérive)
- ✅ Pas de doublon de nom dans la même section
- ✅ Le nom (`otlp`) ou nom alternatif (`otlp/jaeger`, `otlp/internal`) est cohérent entre définition et référence
- ✅ Type de signal cohérent : un processor `tail_sampling` n'a de sens qu'en pipeline `traces` ; un receiver `filelog` n'alimente pas une pipeline `traces` ou `metrics`
- ✅ Extensions citées dans `service.extensions` sont définies en haut (et inversement, sinon non chargées)
- ✅ Connectors : si un composant est utilisé à la fois comme exporter d'une pipeline et receiver d'une autre, il doit être un `connector`, pas un exporter classique

## Phase 3 — Audit qualitatif

### Ordre des processors

Convention OpenTelemetry, du premier au dernier dans la pipeline :

1. **`memory_limiter`** — toujours premier, sinon OOM en cas de surcharge.
2. **`k8sattributes` / `resourcedetection`** — enrichissement contexte (avant les transformations qui pourraient s'en servir).
3. **`attributes` / `resource` / `transform`** — modifications de champs.
4. **`filter` / `probabilistic_sampler`** — réduction de volume tôt = économie sur la suite du pipeline.
5. **`tail_sampling`** — décision finale par trace, après que tous les spans sont arrivés.
6. **`batch`** — toujours en dernier avant les exporters. Optimise les RPC.

Drapeaux rouges à signaler immédiatement :

- `batch` placé avant un sampler ou un filter → le batching empêche les décisions par-trace ou casse l'efficacité du filtre.
- `memory_limiter` absent ou pas en première position → OOM imminent sous charge.
- `tail_sampling` après `batch` → comportement incorrect.

### Catalogue de pièges

**`memory_limiter`**
- Absent dans une pipeline qui reçoit du trafic externe → critique.
- `limit_mib + spike_limit_mib` doivent rester `<` mémoire allouée au container/process, sinon le limiter déclenche après l'OOM kill.
- `check_interval: 1s` recommandé.

**`batch`**
- `send_batch_size` (défaut 8192) : cible normale.
- `send_batch_max_size` : hard cap ; si défini, doit être ≥ `send_batch_size`.
- `timeout` : force flush, ex `200ms` pour les traces, plus long pour des logs basse fréquence.
- Toujours en **dernière position** avant les exporters.

**`tail_sampling`**
- Toutes les spans d'une trace doivent atteindre la même instance de collector.
- En multi-instance : tier amont avec `loadbalancing` exporter routant par `traceID`.
- `decision_wait` : durée d'attente d'une trace complète (typiquement 30s).
- `num_traces` : capacité mémoire, surveiller.
- Combiné avec `batch` en amont = casser le bon fonctionnement.

**TLS / sécurité**
- `tls.insecure: true` ou bloc `tls` absent en prod = trafic clair. Acceptable seulement si TLS termine ailleurs (sidecar, service mesh) — à expliciter dans un commentaire.
- `tls.insecure_skip_verify: true` : presque jamais justifié hors dev/PoC.
- mTLS recommandé entre collectors internes (gateway ↔ agent).

**Ports OTLP**
- `4317` = gRPC
- `4318` = HTTP (JSON ou protobuf)
- Bind à `0.0.0.0:4317` requis en container (pas `localhost` ni `127.0.0.1`, sinon non joignable depuis l'extérieur du namespace réseau).

**Exporters**
- `logging` exporter → renommé `debug` depuis v0.86 (déprécié, retiré ensuite). À mettre à jour.
- `debug` avec `verbosity: detailed` en prod = noyade des logs et coût CPU.
- Pour tout exporter réseau : configurer `sending_queue` + `retry_on_failure` (résilience aux pannes transitoires).
- `sending_queue.storage:` pour persistance de la queue (extension `file_storage`).

**Receivers**
- `filelog` :
  - `start_at: beginning` rejoue tout l'historique au démarrage (drama si gros fichiers).
  - Préférer `start_at: end` + `storage:` (extension `file_storage`) pour persister la position.
  - `include_file_path: true` utile, mais attention cardinalité côté backend.
- `prometheus` : utiliser `metric_relabel_configs` pour drop les labels qui explosent la cardinalité.
- `otlp` : déclarer `protocols.grpc` ET/OU `protocols.http` selon les clients ; un client HTTP qui frappe un endpoint gRPC-only échoue silencieusement côté collector.
- `hostmetrics` : la liste de scrapers active (`cpu`, `memory`, `disk`, `filesystem`, `network`, …) impacte fortement le volume — ne pas tout activer par défaut.

**Cardinalité métriques**
- `attributes` / `resource` / `transform` qui injectent des champs très variables (UUID, IP, `user_id`, `request_id`, `trace_id`, `span_id`) sur des **métriques** → explosion de cardinalité côté backend (Prometheus, Mimir, TSDB Elastic).
- Acceptable sur traces/logs, jamais sur metrics. Signaler systématiquement.

**Self-observability du collector**
- `service.telemetry.metrics.level: detailed` et `address: 0.0.0.0:8888` pour exposer les métriques internes (`otelcol_*`).
- `service.telemetry.logs.level: warn` en prod (`info` génère beaucoup, `debug` énormément).
- Sans cette télémétrie interne, le collector est aveugle sur lui-même → pas de diag possible.

**Kubernetes — `k8sattributes`**
- Nécessite un `ClusterRole` avec `get,list,watch` sur `pods`, `namespaces`, et selon configuration `replicasets`, `nodes`.
- Déploiement : `DaemonSet` pour enrichissement local par nœud, ou `Deployment` avec `passthrough: true` pour mode passe-plat.
- Sans le RBAC : les pods démarrent mais les attributs k8s manquent silencieusement.

**Versioning**
- Composants en `alpha` / `beta` : changements breaking entre versions mineures, à pinner.
- Vérifier que la version du binaire match les composants utilisés (un champ retiré dans v0.110 ne sera pas accepté).

# Format de sortie

Rendre un rapport structuré, en trois sections :

```markdown
## ❌ Erreurs (bloquantes)
- [`processors.batch`] manquant en fin de pipeline `traces`, ligne 42
- [`service.pipelines.traces.processors`] référence `tail_sampling/v2` non défini

## ⚠️ Warnings (à corriger)
- [`exporters.otlp.tls.insecure`] = true en environnement déclaré prod, ligne 88
- [`processors.attributes/enrich`] ajoute `user_id` sur la pipeline `metrics` → cardinalité

## 💡 Suggestions (optimisations)
- [`receivers.filelog.start_at`] passer à `end` + extension `file_storage` pour persistance
- [`service.telemetry.metrics`] non exposé, le collector est aveugle sur lui-même
```

Toujours citer le chemin YAML (`processors.batch.timeout`) ou le numéro de ligne pour que l'utilisateur sache où corriger.

Si tout est OK : un bloc unique `✅ Configuration valide` avec la liste des composants détectés, leur rôle, et la distribution requise.

# Pièges du skill lui-même

- Ne pas inventer de composants. Si un composant n'est pas connu, le signaler comme tel et suggérer `otelcol validate` plutôt que d'affirmer.
- Ne pas réécrire la config en entier sans demander. Le rapport est le livrable principal ; les patchs ne sont produits que sur demande explicite.
- Les versions du collector évoluent vite (changements breaking en mineure). Si un doute existe sur un champ, dire que la doc de la version cible doit être consultée.
