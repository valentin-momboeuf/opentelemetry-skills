---
name: ottl-helper
description: Use when writing, explaining, or validating OTTL (OpenTelemetry Transformation Language) statements for the OpenTelemetry Collector's `transform`, `filter`, or `routing` (connector) processors. Triggers on "write an OTTL statement", "redact this attribute", "drop these spans/logs/datapoints", "route metrics by ...", "explain this transform processor", "filter logs where ...", "what does this OTTL expression do", "validate my OTTL", "transformprocessor", "filterprocessor", "routing connector", or any task involving `set()`, `delete_key()`, `keep_keys()`, `replace_pattern()`, `IsMatch()`, `where`, `route()`. Picks the correct context (resource/scope/span/spanevent/metric/datapoint/log), produces a deployable YAML block, and explains semantics including the easy-to-flip filter "drops on match" behavior.
---

# When to use this skill

The user wants to:

- **Write** an OTTL statement from a stated intent ("redact emails in logs", "drop healthcheck spans", "route checkout traces to Datadog").
- **Explain** an existing OTTL block — what it does, in which context, with what side effects.
- **Validate** an OTTL block — find syntax issues, semantic gotchas, mismatches between intent and implementation.

If the user has a question about a *different* collector component (receiver, exporter, k8sattributes, etc.), use `otel-config-validator` or `otel-pipeline-designer` instead. This skill is specifically for the OTTL DSL inside `transform`, `filter`, and `routing`.

# Mental model

OTTL is a small DSL evaluated **per telemetry item** by the contrib collector. Three things shape every statement:

## 1. Processor / connector and its semantics

| Component | What it does | Statement form |
|---|---|---|
| `transform` | **Mutates** telemetry items in place (set / delete / replace fields) | `set(...)`, `delete_key(...)`, `replace_pattern(...)`, etc. |
| `filter` | **Drops** items whose condition matches (note: **DROP**, not keep) | Boolean expression(s); a single `true` drops the item |
| `routing` (connector) | **Selects pipelines** by condition; non-matching items go to `default_pipelines` | `route() where <bool-expr>` — `route()` is the imperative trigger |

The most common bug is treating `filter` as "keep where X". It's the opposite. `filter.traces.span: ['attributes["env"] == "prod"']` drops every prod span. Always restate semantics aloud before writing or reading a `filter`.

## 2. Context — what each statement sees

OTTL only looks at one telemetry shape at a time. The `context:` (or implicit context, depending on processor version) tells it which.

| Signal | Available contexts |
|---|---|
| traces | `resource`, `scope` (a.k.a. `instrumentation_scope`), `span`, `spanevent` |
| metrics | `resource`, `scope`, `metric` (stream-level: name, unit, description, type), `datapoint` (per-point value + attributes) |
| logs | `resource`, `scope`, `log` |

Inside a context, parents are reachable via path prefixes (e.g. from `span` you can read `resource.attributes["service.name"]`). Children are not — a `metric`-context statement cannot see individual datapoints' attributes.

Choosing the wrong context is the second most common bug. To redact a per-request `user_id` from a log, use `context: log` (each record has its own attributes); to set a fixed `env` once per pod, `context: resource` is enough.

## 3. Path expressions

The canonical form uses brackets, especially for keys containing dots:

```
resource.attributes["service.name"]    # OK
resource.attributes.service.name       # ambiguous — avoid
attributes["http.status_code"]         # OK in span/log/datapoint contexts
body                                   # the log body
name                                   # span name or metric name
trace_id, span_id                      # within span context
```

Common roots by context:

- `span`: `name`, `attributes[...]`, `status.code`, `kind`, `trace_id`, `span_id`, `start_time_unix_nano`, `end_time_unix_nano`, `events`, `links`
- `log`: `body`, `attributes[...]`, `severity_number`, `severity_text`, `time_unix_nano`, `trace_id`, `span_id`
- `datapoint`: `attributes[...]`, `value` (or `value_double` / `value_int`), `time_unix_nano`, `start_time_unix_nano`
- `metric`: `name`, `description`, `unit`, `type`
- `resource`: `attributes[...]`
- `scope`: `name`, `version`, `attributes[...]`

# Authoring workflow

## Mode A — Write OTTL from intent

1. **Identify the signal** (traces / metrics / logs) and the **context** (where does the field live?). Per-pod field → `resource`. Per-request / span / datapoint → leaf context.
2. **Pick the processor**:
   - Modify a field → `transform`
   - Discard a subset → `filter`
   - Send to a different pipeline → `routing` (connector)
3. **Write the statement**, using the function reference below.
4. **Wrap in YAML** with `error_mode: ignore` in production. Without it, a single bad expression on a single item drops the entire batch — every other downstream pipeline pays the cost of one malformed log line.
5. **Add a comment** above each statement. OTTL blocks are notoriously hard to read three months later — a one-line `# why this exists` saves real time.

## Mode B — Explain an OTTL block

1. **Identify the processor** (`transform` / `filter` / `routing`) and its semantic (mutate / drop / select).
2. **Identify the context** (signal + level).
3. **Walk each statement** in order: target field, predicate (if any), result.
4. **State side effects**: what gets dropped, mutated, routed; what passes unchanged.
5. **Flag gotchas**: filter inversion, missing `route()`, unanchored regex, type coercion between string and int.

## Mode C — Validate

For each issue below, scan the user's block and report:

- **Filter inversion** — condition reads like a positive selector. Confirm out loud: `filter` *drops* the matched items, is that what they want?
- **Missing `route()`** in routing connector — every `routing.table` entry must use `route() where ...`. Without `route()`, the statement is silently a no-op.
- **Wrong context for the field** — e.g. `body` referenced under `context: span` (it's logs only).
- **Bracket access on dotted keys** — `attributes.http.status_code` won't parse; use `attributes["http.status_code"]`.
- **Type mismatch in comparisons** — `attributes["http.status_code"] == 200` works only if the producer set it as int; many SDKs set it as string. Wrap with `Int()` to be safe: `Int(attributes["http.status_code"]) == 200`.
- **Regex anchoring** — `IsMatch(name, "/healthz")` matches anywhere in the name. Anchor with `^` / `$` if exact match was intended.
- **Trace-level filtering** — `filter` is per-span. "Drop the whole trace if any span errors" is impossible there — the right tool is `tail_sampling` at a gateway tier.
- **`error_mode` defaulting to `propagate`** — set `error_mode: ignore` (or `silent`) in prod.

# Quick function reference

The catalog below covers ~90 % of real-world use. The OTTL function set evolves between collector versions; if a function is unfamiliar to the user, suggest verifying against their target contrib version's docs.

**Mutators** (transform only):
- `set(target, value)` — assign / upsert
- `delete_key(map, "key")` / `delete_matching_keys(map, "regex")`
- `keep_keys(map, ["k1","k2"])` / `keep_matching_keys(map, "regex")`
- `replace_pattern(field, "regex", "replacement")` / `replace_all_patterns(map, "key"|"value", "regex", "replacement")`
- `truncate_all(map, max_len)` — caps every string value in the map
- `merge_maps(target, source, "insert"|"update"|"upsert")`

**Type / parsing**:
- `Int(x)`, `Double(x)`, `String(x)`, `Bytes(x)`
- `ParseJSON(field)` — returns a map; combine with `merge_maps` to flatten
- `Hex(int)`, `Sha256(string)`

**String**:
- `Concat([a, b, c], "separator")`
- `Substring(s, start, length)`
- `ConvertCase(s, "upper"|"lower"|"snake"|"camel")`
- `Trim(s, "chars")` / `TrimSpace(s)`

**Boolean predicates** (filter / `where`):
- `IsMatch(field, "regex")` — Go RE2; no lookaround
- `IsString(x)`, `IsInt(x)`, `IsDouble(x)`, `IsBool(x)`, `IsMap(x)`, `IsList(x)`
- Comparators: `==`, `!=`, `<`, `<=`, `>`, `>=`
- Combinators: `and`, `or`, `not`
- Trace helpers: `IsRootSpan()`, `TraceID()`, `SpanID()`

**Time**:
- `UnixNano()`, `Now()`
- Date parts: `Hour()`, `Minute()`, `Day()`, `Month()`, `Year()`

# Recipe book

### Redact PII in log body and headers

```yaml
processors:
  transform:
    error_mode: ignore
    log_statements:
      - context: log
        statements:
          # Redact email addresses anywhere in the body
          - replace_pattern(body, "[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}", "<EMAIL>")
          # Redact bearer tokens in the Authorization header attribute
          - replace_pattern(attributes["http.request.header.authorization"], "Bearer [A-Za-z0-9._\\-]+", "Bearer <REDACTED>") where attributes["http.request.header.authorization"] != nil
```

### Drop healthcheck spans

```yaml
processors:
  filter:
    error_mode: ignore
    traces:
      span:
        # DROPS spans whose URL ends in /healthz or /readyz.
        # Anchored with $ to avoid matching paths like /api/v1/healthz/details.
        - 'attributes["http.url"] != nil and IsMatch(attributes["http.url"], "/(healthz|readyz)$")'
```

### Tag every span with its env from a resource attribute

```yaml
processors:
  transform:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          - set(attributes["env"], resource.attributes["deployment.environment"])
```

### Drop high-cardinality keys before they hit Mimir

```yaml
processors:
  transform:
    error_mode: ignore
    metric_statements:
      - context: datapoint
        statements:
          - delete_matching_keys(attributes, "^(request_id|trace_id|span_id)$")
```

### Route by service.name (routing connector)

```yaml
connectors:
  routing:
    default_pipelines: [traces/default]
    error_mode: ignore
    table:
      - statement: route() where resource.attributes["service.name"] == "checkout"
        pipelines: [traces/checkout]
      - statement: route() where IsMatch(resource.attributes["service.name"], "^(payments|billing).*")
        pipelines: [traces/finance]

service:
  pipelines:
    traces/in:
      receivers: [otlp]
      exporters: [routing]            # the connector acts as exporter on the input pipeline
    traces/checkout:
      receivers: [routing]            # ... and as receiver on each output pipeline
      processors: [batch]
      exporters: [otlp/checkout]
    traces/finance:
      receivers: [routing]
      processors: [batch]
      exporters: [otlp/finance]
    traces/default:
      receivers: [routing]
      processors: [batch]
      exporters: [otlp/default]
```

# Output format

Match the format to the user's mode.

## Write mode

```markdown
## Intent
<one sentence — what the user wants done>

## Context choice
<signal + context, with the why>

## Statement(s)
```yaml
<the OTTL block, properly nested in the processor/connector config>
```

## Notes
- error_mode: <chosen value, with rationale>
- <gotchas — type coercion, regex anchors, missing fields>
- <ordering, if multiple statements>
```

## Explain mode

```markdown
## Processor & semantic
<transform mutates / filter DROPS / routing selects pipeline>

## Context
<signal + context — what each statement sees>

## Statement-by-statement
1. `<statement>` — <plain English: condition + effect>
2. `<statement>` — ...

## Side effects
- <what gets dropped, mutated, routed>
- <what passes through unchanged>

## Concerns / gotchas
- <if any>
```

## Validate mode

```markdown
## ❌ Issues
- [<line / statement>] <problem>: <what's wrong>; <suggested fix>

## ⚠️ Concerns
- <semantic risks, untested assumptions>

## ✅ Looks correct
- <what was checked and passed>
```

# Skill self-pitfalls

- Don't fabricate functions. If unsure, restrict to the catalog above and tell the user to verify against the contrib version they run.
- Don't omit `error_mode: ignore` in production examples. It's the cheapest reliability improvement for OTTL configs.
- For `filter`, every example must explicitly state "this DROPS items where the condition is true" so the user can't invert the logic by mistake.
- For `routing`, every entry must include `route()` — without it the statement is a silent no-op.
- Anchor regex (`^` / `$`) when matching paths or names — unanchored regex over-matches in subtle ways.
- Trace-level decisions ("drop the whole trace if any span errors") cannot be done in `filter`. Point the user to `tail_sampling`.
- When the user asks "how do I do X" but X is impossible in the chosen processor, say so and suggest the right one — don't invent a workaround.
