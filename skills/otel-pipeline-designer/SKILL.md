---
name: otel-pipeline-designer
description: Use when the user describes an observability requirement in natural language and needs an OpenTelemetry Collector configuration generated from scratch. Triggers on "generate an otel collector config for...", "design a pipeline that...", "what config do I write for X to Y", "build me an otelcol config", "k8s logs + java traces to Loki+Tempo", "VM fleet to Mimir", "two-tier tail sampling to Datadog", or any request that combines a source (k8s, VMs, java/python/go apps, prometheus, kafka, journald) with a backend (Tempo, Loki, Mimir, Jaeger, Elastic, Datadog, vendor OTLP). Picks the right distribution (core vs contrib), wires receivers/processors/exporters in the canonical order (memory_limiter first, batch last), applies sane defaults for memory_limiter, batch, resourcedetection, sending_queue+retry, emits a deployable YAML plus a rationale, RBAC if K8s, and a TODO list of things to confirm before deploying.
---

# When to use this skill

The user describes a stack and a goal in natural language and wants a working OpenTelemetry Collector config — not a review of an existing config (that's `otel-config-validator`), and not a runtime debug (that's `otel-collector-debug`).

Typical phrasings:
- "Generate me an otel collector config to ship Java traces to Tempo and pod logs to Loki."
- "We have a VM fleet — Prom scrape on each box, ship to Mimir, plus OTLP traces to Tempo."
- "I need a two-tier setup: agents at the edge + gateway with tail sampling, output to Datadog."
- "What does the config look like to receive OTLP and export to Elastic Cloud?"

If the user already has a config and wants it reviewed → `otel-config-validator`.
If the user has a running collector misbehaving → `otel-collector-debug`.

# Method

Three steps, in order: gather intent, design topology, emit YAML.

## Step 1 — Gather intent

If the user hasn't already specified all of these, ask in **one** round (don't drip-feed):

| Dimension | What to find out |
|---|---|
| **Signals** | traces, metrics, logs (any subset; pipelines differ per signal) |
| **Sources** | what produces the data — Java/Python/Go/.NET SDKs (OTLP), filelog (k8s pods, syslog, app log files), prometheus scrape targets, hostmetrics, kafka, journald |
| **Environment** | Kubernetes (DaemonSet vs Deployment vs sidecar), VMs, bare metal, Lambda, ECS — drives k8sattributes, resourcedetection, deployment topology |
| **Backends** | Tempo / Jaeger / Datadog / Elastic / vendor-OTLP for traces; Loki / Elastic / Splunk for logs; Mimir / Prometheus / Datadog for metrics |
| **Constraints** | TLS / mTLS, multi-tenant headers (`X-Scope-OrgID`), sampling strategy, expected throughput, persistence requirements |

If a critical dimension is missing, ask. If it's not critical, default sensibly and flag your assumption in the rationale.

## Step 2 — Design topology

### Distribution selection

Default to **`otel/opentelemetry-collector-contrib`** for almost any non-trivial pipeline. Only stick to core if the answer literally uses just `otlp` receiver + `batch`/`memory_limiter` + simple `otlp` exporter.

Components that force contrib:
`tail_sampling`, `transform`, `filter`, `k8sattributes`, `resourcedetection`, `filelog`, `journald`, `kafka` (receiver/exporter), `prometheus` (receiver), `hostmetrics`, `loadbalancing` (exporter), `prometheusremotewrite`, `loki`, `datadog`, `elasticsearch`, vendor-specific exporters.

### Topology selection

- **Single agent** (DaemonSet on K8s, daemon on VM) when: low/medium volume, no tail sampling, all signals from local sources only.
- **Agent + Gateway** when: tail sampling is required (must aggregate by traceID at the gateway), multi-tenancy / per-tenant routing, or the gateway acts as a TLS termination + auth layer.
- **Sidecar** when: very high cardinality per pod, or strong network-locality requirements.

### Receiver selection from the intent

| Source | Receiver |
|---|---|
| Java/Python/Go/.NET app via OTLP SDK | `otlp` (gRPC 4317 + HTTP 4318) |
| Browser / mobile via OTLP HTTP | `otlp` (HTTP only, with CORS allowed origins) |
| K8s pod logs | `filelog` reading `/var/log/pods/*/*/*.log` (DaemonSet) |
| K8s cluster events | `k8s_events` |
| Prometheus targets | `prometheus` |
| Host metrics | `hostmetrics` (enable only the scrapers needed) |
| Kafka topic | `kafka` |
| journald (systemd) | `journald` |

### Processor pipeline (canonical order)

Apply in this order (drop the ones you don't need — don't add ones the user didn't ask for):

1. **`memory_limiter`** — always first. Sized at ~75 % of container memory, `spike_limit_mib` ~15 %, sum strictly less than container memory limit.
2. **`k8sattributes`** — only on K8s; needs the right `ClusterRole`. Run on DaemonSet for local pod attribution, or `Deployment` with `passthrough: true` for a gateway.
3. **`resourcedetection`** — `env` + `system` baseline; add `k8snode` for DaemonSet, `eks`/`gke`/`aks` for managed K8s.
4. **`attributes` / `resource` / `transform`** — for redaction, normalization, OTTL transforms.
5. **`filter` / `probabilistic_sampler`** — early volume cuts (cheap, large impact).
6. **`tail_sampling`** — only on the gateway tier, after spans for a trace are aggregated.
7. **`batch`** — always last before exporters.

### Exporter selection from the backend

| Backend | Exporter |
|---|---|
| Tempo / Jaeger / OTLP-native | `otlp` (gRPC) |
| Loki (3.x+) | `otlphttp` to Loki's `/otlp` endpoint (modern Loki is OTLP-native; `loki` exporter is deprecated) |
| Mimir / Prometheus | `prometheusremotewrite` (with `headers.X-Scope-OrgID` for multi-tenant) |
| Datadog | `datadog` (api.key from env) |
| Elastic Cloud / Elasticsearch | `elasticsearch` exporter |
| Generic OTLP vendor | `otlphttp` or `otlp` |
| Kafka downstream | `kafka` exporter |

Always wrap network exporters with:

```yaml
sending_queue:
  enabled: true
  queue_size: 1000
  num_consumers: 10
  storage: file_storage   # only if persistence needed AND file_storage extension is added
retry_on_failure:
  enabled: true
  initial_interval: 5s
  max_interval: 30s
  max_elapsed_time: 300s
timeout: 10s
```

For tail sampling at the gateway tier, the **agent tier must use the `loadbalancing` exporter** routing by `traceID` to the gateway. Without this, tail sampling sees only a fraction of each trace's spans on each replica and silently makes inconsistent decisions.

### Defaults catalog

Apply these unless the user constraints push elsewhere:

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 1024              # ~75% of container/pod memory limit
    spike_limit_mib: 256         # ~15%; sum < container memory limit
  batch:
    send_batch_size: 8192        # 1024 for logs at low rate
    send_batch_max_size: 10000
    timeout: 200ms               # 5s for low-volume signals
  resourcedetection:
    detectors: [env, system]     # add k8snode for DaemonSet, eks/gke/aks for managed
    timeout: 5s
    override: false              # don't clobber app-side resource attrs

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  file_storage:                  # only if exporters use storage: or filelog needs it
    directory: /var/lib/otelcol
  pprof:                         # optional but recommended for incident response
    endpoint: 0.0.0.0:1777

service:
  extensions: [health_check, file_storage, pprof]
  telemetry:
    metrics:
      level: normal
      address: 0.0.0.0:8888
    logs:
      level: info
```

## Step 3 — Emit YAML

Produce a deployable config. Multi-tier setups get one config block per tier (clearly labeled `# Agent (DaemonSet)` / `# Gateway (Deployment)`).

# Output format

Always produce these sections in this order:

```markdown
## Topology
<one-paragraph description: tiers, deployment mode, signal flow>

## Distribution required
`otel/opentelemetry-collector-contrib:<pinned-version>` (or core, with rationale).
Components from contrib: <list>.

## Configuration

### `<tier-name>` (e.g. `agent`)
```yaml
<full deployable YAML>
```

### `<tier-name>` (e.g. `gateway`, if multi-tier)
```yaml
<full deployable YAML>
```

## Kubernetes RBAC (if applicable)
```yaml
<ClusterRole + ServiceAccount + ClusterRoleBinding for k8sattributes / k8s_events>
```

## Rationale
- **Why contrib distro:** <component(s) that forced it>
- **Processor ordering:** <if non-obvious choices were made>
- **Sizing assumptions:** <memory limits, batch sizes, sampling rates>
- **Notable defaults applied or overridden:** <list>

## TODO before deploying
- <e.g. "Confirm Tempo endpoint TLS expectation">
- <e.g. "Set `DD_API_KEY` as env var in the pod spec">
- <e.g. "Pin to a specific contrib version after testing">
- <e.g. "For tail sampling, ensure the agent tier loadbalancing exporter sees the gateway pods (Service or DNS round-robin)">
```

# Examples of intent → topology mapping

**Intent:** "K8s logs + Java traces to Tempo + Loki."
→ contrib distro. DaemonSet agent: `filelog` + `otlp` receivers, `k8sattributes`, `resourcedetection(env, system, k8snode)`, two pipelines (logs → Loki via `otlphttp`; traces → Tempo via `otlp` directly, or forward to a gateway if tail sampling is added later). Include `file_storage` extension for filelog checkpoint persistence.

**Intent:** "VM fleet, Prom scrape + host metrics + OTLP traces to Mimir + Tempo."
→ contrib distro. Single daemon collector per VM. `prometheus`, `hostmetrics`, `otlp` receivers. `resourcedetection(env, system)`. Two pipelines: metrics → `prometheusremotewrite` to Mimir (with `headers.X-Scope-OrgID`); traces → `otlp` to Tempo.

**Intent:** "Two-tier with tail sampling to Datadog."
→ contrib distro. **Agent**: `otlp` receiver → `memory_limiter` → `batch` → `loadbalancing` exporter routing by traceID to the gateway Service. **Gateway**: `otlp` receiver → `memory_limiter` → `tail_sampling` (errors policy + probabilistic 5 %) → `batch` → `datadog` exporter (api key from env).

# Skill self-pitfalls

- Don't invent processors or fields. If a component isn't real, the config breaks at startup.
- Pin a version (e.g. `0.110.0`). Schemas drift between minor releases.
- For K8s, always include the RBAC manifests for `k8sattributes` / `k8s_events` — without them, the config silently lacks attribution.
- For DaemonSets reading `filelog`, mention the volume mounts (`/var/log/pods` and the `/var/log/containers` symlinks) and the `file_storage` extension for offset persistence.
- Loki natively accepts OTLP since 3.x — prefer `otlphttp` over the deprecated `loki` exporter unless the user is explicitly on an older version.
- Don't add components the user didn't ask for. A pipeline that does only what the intent describes is easier to validate, smaller, and cheaper to run.
- When tail sampling is requested, the agent tier **must** use `loadbalancing` exporter routing by traceID to the gateway. Otherwise tail sampling silently drops or duplicates decisions across replicas — flag this explicitly in the rationale.
- Provide the `TODO before deploying` section honestly — point at things the user must verify (endpoints, secrets, RBAC, version pin) before applying.
