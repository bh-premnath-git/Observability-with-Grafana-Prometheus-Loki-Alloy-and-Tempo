# Observability with Grafana, Prometheus, Loki, Alloy and Tempo

This stack is a very strong modern open-source observability setup.

---

## High-Level Overview

| Component | Role |
|-----------|------|
| **Grafana** | The UI where you explore dashboards, logs, traces, and alerts; current docs track the Grafana 12 era (12.4.2 as of April 2026), which includes Explore Logs for native log exploration and AI-assisted correlation across signals |
| **Prometheus** | Stores and queries metrics as time series, usually by scraping `/metrics` endpoints |
| **Loki** | Stores and queries logs; designed to correlate well with Grafana, Prometheus, and Tempo; current release line is v3.7.x |
| **Tempo** | Stores and queries distributed traces; can connect traces with logs and metrics |
| **Alloy** | Grafana's collector for metrics, logs, traces, and profiles; its default engine is the stable Alloy runtime combining Prometheus-native collection and OpenTelemetry support; it also includes a bundled OpenTelemetry Collector-compatible engine, though that OTel Engine is currently marked experimental |

---

## Container Images

All images are pinned to specific versions. Alloy, Prometheus, Tempo, and Grafana use **FIPS-compliant Debian 13 builds** from `dhi.io`. Loki uses the official Grafana image (no FIPS variant available).

| Component | Image | Version |
|-----------|-------|---------|
| **Alloy** | `dhi.io/alloy` | `1.15.1-debian13-fips` |
| **Prometheus** | `dhi.io/prometheus` | `3.11.2-debian13-fips` |
| **Loki** | `grafana/loki` | `3.7.0` |
| **Tempo** | `dhi.io/tempo` | `2.10.3-debian13-fips` |
| **Grafana** | `dhi.io/grafana` | `12.4.2-debian13-fips` |

> **Why FIPS?** The `dhi.io` images are built on Debian 13 (Bookworm) with FIPS 140-2 validated cryptographic modules. This makes the stack compatible with environments that require FIPS compliance — including AWS GovCloud and regulated workloads on ECS/Fargate.

---

## Environment Variables

All runtime configuration lives in `.env` at the project root. Docker Compose loads it automatically — no extra flags needed.

```bash
# copy and edit before first run
cp .env .env.local   # optional: keep a local override
```

| Variable | Default | Description |
|----------|---------|-------------|
| `ENVIRONMENT` | `local` | Passed to Alloy; labels telemetry by environment |
| `GF_ADMIN_USER` | `admin` | Grafana admin username |
| `GF_ADMIN_PASSWORD` | `admin` | Grafana admin password — **change before exposing to a network** |
| `GF_AUTH_ANONYMOUS_ENABLED` | `false` | Disable anonymous access to Grafana |
| `GF_UNIFIED_ALERTING_ENABLED` | `true` | Enable Grafana unified alerting |
| `GF_FEATURE_TOGGLES_ENABLE` | *(see `.env`)* | Space-separated Grafana feature flags |

> **AWS ECS:** do not ship `.env` to production. Inject these variables from **SSM Parameter Store** or **Secrets Manager** at task launch. Add `.env` to `.gitignore` to prevent accidental commits.

---

## One Request, Five Signals

Think of one request hitting your system:

- **Prometheus** tells you: request rate, latency, error rate, CPU, memory
- **Loki** tells you: the actual log lines around the failure
- **Tempo** tells you: which service call in the request path was slow or broken
- **Grafana** lets you jump between all of them in one place
- **Alloy** is the pipeline that collects and forwards that telemetry

---

## Simple Mental Model

| Signal | Answers |
|--------|---------|
| **Metrics** | "Is something wrong?" |
| **Logs** | "What happened?" |
| **Traces** | "Where in the request path did it happen?" |
| **Dashboards/Alerts** | "How do I see it early and react fast?" |

---

## Typical Flow

```
App / Service
  → emits metrics, logs, traces
  → Alloy collects/processes them
      → metrics  → Prometheus
      → logs     → Loki
      → traces   → Tempo
  → Grafana reads from all three for dashboards, drill-down, and correlation
```

---

## Why Alloy Matters Now

Earlier Grafana setups often used separate agents like Promtail or standalone collectors. Alloy is now Grafana's collector, combining **Prometheus-native collection and OpenTelemetry support** in one agent. Its default engine is the stable Alloy runtime. Alloy also ships a bundled OpenTelemetry Collector-compatible engine for teams that want a drop-in OTel experience, but Grafana currently marks that OTel Engine as experimental — the stable default is the Alloy runtime itself.

Alloy is useful when you want:

- One agent instead of many
- Consistent pipelines
- Central processing of labels, relabeling, filtering, batching
- Easier multi-signal collection

---

## What Each Component Does Best

### Prometheus
Strongest for numeric operational data:

- Request count and error count
- Latency (p50, p95, p99)
- CPU / memory
- Queue depth
- Business counters

Uses **PromQL** for alerting and dashboards. Supports counters, gauges, histograms, and summaries. Prometheus 3.x treats **native histograms as stable and mainstream** — they are no longer experimental. Native histograms offer better accuracy and lower cardinality than classic histograms. Use `scrape_native_histograms` to collect them from exporters. As of Prometheus 3.9+, the old `--enable-feature=native-histograms` flag is a no-op; native histogram support is always on. The stack runs **3.11.2**, which includes remote write 2.0 and UTF-8 metric names.

### Loki
Best for cheaper, scalable log aggregation with strong Grafana integration. Supports alerting through its ruler and integration with Alertmanager. Designed to correlate logs with metrics and traces. The current release line is **Loki v3.7.x**. Loki 3.x brought significant query performance improvements and **Bloom filters** for efficient log volume reduction at scale.

### Tempo
Best for microservices and distributed systems where a single user request crosses many services. Can search traces, generate metrics from spans, and link tracing data with logs and metrics.

---

## Best Practical Architecture

For a microservices environment:

1. Instrument apps with **OpenTelemetry** — the core signals and OTLP are stable at the specification and protocol level, making it a safe long-term foundation; note that language-level SDK maturity still varies by signal and language, especially for logs
2. Use **Alloy** as the collector/forwarder
3. Use **Prometheus** for metrics storage and alerting
4. Use **Loki** for centralized logs
5. Use **Tempo** for tracing
6. Use **Grafana** as the single pane of glass

This gives you one clean pipeline and avoids every service directly talking to every backend.

---

## Example: Payment System Incident

Suppose `payment-service` becomes slow:

1. **Prometheus** shows p95 latency increased from 120 ms to 2.8 s
2. **Grafana** alert fires on latency and error ratio
3. You click into **Tempo** and see the slow span is `banking-service → external PSP adapter`
4. From that trace ID, you jump into **Loki** and find timeout/retry logs

Now you know not just *that* latency is bad, but exactly *which downstream dependency* caused it.

---

## Good Beginner Rollout Order

1. Start with **Prometheus + Grafana**
2. Add **Loki**
3. Add **Tempo**
4. Put **Alloy** in front as the unified collector
5. Add correlation links and alerts

That sequence is easier than trying to perfect the whole stack at once.

---

## Important Best Practices

- **Keep labels low-cardinality in Prometheus** — otherwise costs and query load explode
- **Do not log everything blindly into Loki** — parse and label carefully
- **Sample traces thoughtfully in Tempo** for high-volume systems
- **Add trace IDs into logs** so Grafana can pivot from logs to traces
- **Protect Prometheus and Alertmanager endpoints** — these endpoints are sensitive from a security perspective

---

## Very Short Summary

| Component | Purpose |
|-----------|---------|
| Grafana | Shows everything |
| Prometheus | Metrics |
| Loki | Logs |
| Tempo | Traces |
| Alloy | Collection pipeline |

Together, they give you full observability across **what is broken**, **why it broke**, and **where it broke**.

---

## AWS ECS Architecture

For an AWS ECS cluster, this stack fits well, but the design depends on whether you use **Fargate** or **ECS on EC2**.

### ECS on Fargate: Recommended Pattern

Fargate is serverless for containers — you do not manage underlying instances. This means host-level daemon patterns are limited, so the **sidecar pattern** is the natural default.

**Recommended task layout:**

```
ECS Task
├── app container         (your service, instrumented with OTel)
├── Alloy sidecar         (receives OTLP, forwards metrics/traces)
└── FireLens log router   (routes stdout/stderr logs to Loki)
```

**Signal flow:**

```
app → OTLP → Alloy → metrics → Prometheus backend
                   → traces  → Tempo
app stdout → FireLens → Loki
Grafana reads metrics, logs, traces
```

> **Why FireLens for logs?**  
> AWS documents FireLens as the ECS-native mechanism for routing container logs and recommends **Fluent Bit** over Fluentd for lower resource usage. As of April 2026, AWS for Fluent Bit supports the 3.x line, based on Amazon Linux 2023 and Fluent Bit 4.1.1. On Fargate, prefer stdout/stderr log routing with FireLens instead of host-based file scraping patterns. AWS documents FireLens as the native ECS path.  
> **Operational note:** FireLens listens on port 24224 internally. AWS advises not exposing that port outside the task.

### ECS on EC2: Practical Options

On EC2-backed ECS you control the host, so you have two workable choices:

| Option | When to use |
|--------|-------------|
| **Sidecar collector per task** | Good default; consistent with Fargate approach |
| **Daemon-style collector per EC2 node** | Better at scale when tasks are dense |

### Application Instrumentation

Each service should export:

- `/metrics` or OTel metrics endpoint (for Prometheus)
- OTLP traces (for Tempo)
- Structured JSON logs with `trace_id` and `span_id` (for log-to-trace correlation in Grafana)

### Storage and Query Layer Options

| Option | Description |
|--------|-------------|
| **Fully self-hosted** | Prometheus + Loki + Tempo + Grafana on ECS/EC2 |
| **Managed metrics** | AWS Container Insights / Amazon Managed Prometheus (AMP) |
| **Managed Grafana** | Amazon Managed Grafana |
| **Hybrid** | Self-hosted Loki/Tempo + managed metrics/Grafana |

---

## Where AWS-Native Pieces Help

| AWS Feature | Value |
|-------------|-------|
| **Container Insights** | Cluster, service, task, and container-level visibility with enhanced observability |
| **ADOT** | AWS Distro for OpenTelemetry; supported for ECS/Fargate; can collect metrics and traces |
| **FireLens** | Native ECS log routing mechanism; preferred over file-based log collection |

The real decision is usually not "AWS or Grafana stack?" — it is:

> Use AWS-native signals where they are easy and valuable, and use the Grafana stack as the unified observability experience.

---

## Recommended Full Stack for ECS Microservices

| Layer | Component |
|-------|-----------|
| Instrumentation | OpenTelemetry in every service |
| Collection | Alloy sidecar per task (clean default for Fargate); sidecar or node-level collection both valid for ECS on EC2 |
| Log routing | FireLens (Fluent Bit) → Loki |
| Metrics storage | Prometheus-compatible backend |
| Log storage | Loki |
| Trace storage | Tempo |
| UI / Correlation | Grafana |

**Benefits:**

- Good service-level visibility
- Clean multi-signal correlation
- Portable outside AWS later
- Less lock-in than going fully AWS-native
- Easier debugging of service-to-service calls

---

## ECS-Specific Cautions

- **On Fargate, prefer sidecars, not host agents**
- **For logs, prefer `stdout`/`stderr` + FireLens**, not file scraping inside containers
- **Use task IAM roles correctly** — ECS task definitions support task roles and execution roles separately; collectors and routers that need AWS API access must be configured correctly
- **Use OpenTelemetry, not X-Ray SDK/daemon** — as of February 25, 2026, AWS officially placed the X-Ray SDKs and daemon into maintenance mode. End of support is February 25, 2027. Do not start new tracing architecture around X-Ray SDK/daemon patterns; use OTel instrumentation with Alloy or ADOT instead
- **Keep Prometheus labels low-cardinality**, especially `service`, `task`, `container`, `region`, `env`

---

## Next Steps

For a concrete ECS/Fargate production architecture covering where Alloy, FireLens, Prometheus, Loki, Tempo, and Grafana should physically run, see the architecture guides in this repository.

---

## Alloy Configuration — Minimal Production Starter

The rest of this document describes Alloy as the backbone. Here is a minimal `.alloy` file that wires all three signals. Without something like this, the stack does not run.

```alloy
// alloy.alloy — minimal multi-signal starter (April 2026)

// ── OTLP receiver: accepts metrics, logs, and traces from instrumented apps ──
otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  http { endpoint = "0.0.0.0:4318" }

  output {
    metrics = [otelcol.processor.batch.default.input]
    logs    = [otelcol.exporter.loki.default.input]
    traces  = [otelcol.exporter.otlphttp.tempo.input]
  }
}

otelcol.processor.batch "default" {
  output {
    metrics = [otelcol.exporter.prometheus.default.input]
  }
}

// ── Metrics: scrape local /metrics endpoints (Prometheus pull model) ──
prometheus.scrape "app" {
  targets = [
    {"__address__" = "localhost:8080", "job" = "my-service", "env" = "prod"},
  ]
  scrape_native_histograms = true   // collect native histograms (Prometheus 3.x)
  forward_to               = [prometheus.remote_write.backend.receiver]
}

// ── Metrics: push scraped and OTLP metrics to Prometheus-compatible backend ──
prometheus.remote_write "backend" {
  endpoint {
    url = env("PROMETHEUS_REMOTE_WRITE_URL")  // e.g. http://prometheus:9090/api/v1/write

    // Enable exemplars so Grafana can pivot from histogram buckets to traces
    metadata_config { send_interval = "1m" }
  }
}

otelcol.exporter.prometheus "default" {
  forward_to = [prometheus.remote_write.backend.receiver]
}

// ── Traces: forward to Tempo ──
otelcol.exporter.otlphttp "tempo" {
  client {
    endpoint = env("TEMPO_ENDPOINT")  // e.g. http://tempo:4318
  }
}

// ── Logs: forward OTLP logs to Loki ──
otelcol.exporter.loki "default" {
  forward_to = [loki.write.backend.receiver]
}

loki.write "backend" {
  endpoint {
    url = env("LOKI_PUSH_URL")  // e.g. http://loki:3100/loki/api/v1/push
  }
}
```

**Key points:**
- Secrets (`PROMETHEUS_REMOTE_WRITE_URL`, `TEMPO_ENDPOINT`, `LOKI_PUSH_URL`) come from environment variables, not hardcoded values — inject via ECS secrets or SSM Parameter Store
- `scrape_native_histograms = true` is required to collect native histograms from exporters
- The OTLP receiver on 4317/4318 is what your app containers point at via `OTEL_EXPORTER_OTLP_ENDPOINT`

---

## Alertmanager — Routing Alerts to the Right Place

Prometheus evaluates alert rules and fires to Alertmanager. Alertmanager handles routing, deduplication, silencing, and inhibition. Without it, alerts fire nowhere useful.

### Minimal Alertmanager Config

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: pagerduty-critical
  group_by: [alertname, env, service]
  group_wait:      30s
  group_interval:  5m
  repeat_interval: 4h

  routes:
    # Low-severity alerts go to Slack only
    - matchers: [severity="warning"]
      receiver: slack-warnings

    # Business-hours-only team gets a separate route
    - matchers: [team="payments"]
      receiver: pagerduty-payments
      active_time_intervals: [business-hours]

inhibit_rules:
  # If a critical alert is firing, suppress warnings for the same service
  - source_matchers: [severity="critical"]
    target_matchers: [severity="warning"]
    equal: [service, env]

receivers:
  - name: pagerduty-critical
    pagerduty_configs:
      - routing_key: $PAGERDUTY_KEY
        severity: critical

  - name: pagerduty-payments
    pagerduty_configs:
      - routing_key: $PAGERDUTY_PAYMENTS_KEY

  - name: slack-warnings
    slack_configs:
      - api_url: $SLACK_WEBHOOK_URL
        channel: "#alerts-warnings"
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

time_intervals:
  - name: business-hours
    time_intervals:
      - weekdays: [monday:friday]
        times: [{ start_time: "08:00", end_time: "18:00" }]
```

**What each piece does:**

| Concept | Purpose |
|---------|---------|
| `group_by` | Batches related alerts into one notification instead of flooding |
| `group_wait` | Waits 30s before first notification to collect related alerts |
| `repeat_interval` | Re-notifies every 4h if alert stays firing |
| `inhibit_rules` | Suppresses warning noise when a critical alert already covers the same failure |
| `active_time_intervals` | Routes to on-call only during business hours for non-critical teams |

Loki's ruler can also fire alerts from log queries (e.g., error rate from log volume). Route those through the same Alertmanager for unified alert management.

---

## FireLens / Fluent Bit Configuration — ECS to Loki

FireLens is recommended throughout this document. Here is what the actual config looks like.

### ECS Task Definition (relevant sections)

```json
{
  "containerDefinitions": [
    {
      "name": "log_router",
      "image": "public.ecr.aws/aws-observability/aws-for-fluent-bit:3",
      "essential": true,
      "firelensConfiguration": {
        "type": "fluentbit",
        "options": { "enable-ecs-log-metadata": "true" }
      },
      "memoryReservation": 50
    },
    {
      "name": "my-service",
      "image": "my-service:latest",
      "logConfiguration": {
        "logDriver": "awsfirelens",
        "options": {
          "Name":        "grafana-loki",
          "Url":         "http://loki:3100/loki/api/v1/push",
          "LabelKeys":   "container_name,ecs_task_definition,ecs_cluster,source",
          "RemoveKeys":  "container_id,ecs_task_arn",
          "LineFormat":  "json",
          "AutoKubernetesLabels": "false"
        }
      }
    }
  ]
}
```

### What the labels become in Loki

With `enable-ecs-log-metadata: true`, FireLens injects ECS metadata into each log record. The `LabelKeys` option promotes specific fields into Loki labels:

| Loki label | Source |
|------------|--------|
| `container_name` | ECS container name |
| `ecs_task_definition` | Task definition family:revision |
| `ecs_cluster` | Cluster name |
| `source` | stdout or stderr |

Keep `LabelKeys` short. High-cardinality fields (`ecs_task_arn`, `container_id`) go into the log line or Loki structured metadata — not labels. See the Loki Label Schema section below.

**Operational reminder:** FireLens listens on port 24224. Do not expose this port outside the task network. The `my-service` container reaches the log router only via localhost within the task.

---

## Exemplars — The Metrics-to-Traces Bridge

Exemplars are the actual mechanism that lets you click from a high-latency histogram bucket directly to the offending trace in Tempo. They are the missing link in the payment incident workflow described earlier in this document.

### How it works

When your app records a histogram observation (e.g., request latency), it can attach an exemplar — a sample data point that includes the `trace_id` of the request that produced that observation.

```
histogram observation: 2.8s latency
  + exemplar: { trace_id = "abc123", span_id = "def456" }
```

Prometheus stores exemplars alongside the metric. Grafana renders them as dots on histogram panels. Clicking a dot opens that trace in Tempo.

### Requirements

1. **Instrumentation** emits exemplars. The OTel SDK does this automatically when a trace context is active.
2. **Prometheus** must have exemplar storage enabled:

```yaml
# prometheus.yml
feature_flags:
  - exemplar-storage   # enabled by default in Prometheus 3.x
```

3. **Alloy** scrapes with native histograms enabled (already shown above: `scrape_native_histograms = true`)
4. **Grafana data source** must have exemplars enabled:

```yaml
# Grafana Prometheus datasource provisioning
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    jsonData:
      exemplarTraceIdDestinations:
        - name: trace_id
          datasourceUid: tempo-uid   # points to your Tempo datasource
```

With native histograms (Prometheus 3.x), exemplars attach at the histogram level, giving you per-bucket trace links — not just one exemplar per metric name.

---

## Trace ID Injection — Making Log-to-Trace Correlation Work

"Add trace IDs into logs" is only useful if your logging setup actually injects them. This is not automatic in most SDKs unless you wire it up explicitly.

### How it works

The OTel SDK maintains a trace context on the current thread/goroutine/async context. The logging bridge reads that context and injects `trace_id` and `span_id` into each log record while a span is active.

### Per-language setup

**Java (Logback)**
```xml
<!-- logback.xml -->
<appender name="OTEL" class="io.opentelemetry.instrumentation.logback.appender.v1_0.OpenTelemetryAppender"/>

<root level="INFO">
  <appender-ref ref="OTEL"/>
</root>
```
Add dependency: `io.opentelemetry.instrumentation:opentelemetry-logback-appender-1.0`

**Python**
```python
from opentelemetry.instrumentation.logging import LoggingInstrumentor
LoggingInstrumentor().instrument(set_logging_format=True)
# Now every log record includes otelTraceID and otelSpanID
```

**Node.js (Winston)**
```js
const { trace, context } = require('@opentelemetry/api');

const addTraceContext = winston.format((info) => {
  const span = trace.getActiveSpan();
  if (span) {
    const ctx = span.spanContext();
    info.trace_id = ctx.traceId;
    info.span_id  = ctx.spanId;
  }
  return info;
});
```

**Go**
```go
span := trace.SpanFromContext(ctx)
sc  := span.SpanContext()
logger.Info("payment processed",
  slog.String("trace_id", sc.TraceID().String()),
  slog.String("span_id",  sc.SpanID().String()),
)
```

Once `trace_id` appears in the JSON log line, Grafana's Loki datasource can be configured to extract it and render a link to Tempo automatically.

---

## Production Architecture — Gaps to Address Before Going Live

### Prometheus HA and Long-Term Storage

A single Prometheus instance is a single point of failure and has limited retention. For production:

| Option | When to use |
|--------|-------------|
| **Two Prometheus + Thanos** | Self-hosted HA with object storage (S3/GCS) for long-term retention and global query |
| **VictoriaMetrics** | Drop-in Prometheus-compatible backend with better compression and higher ingest throughput |
| **Amazon Managed Prometheus (AMP)** | Managed option; removes operational burden on AWS; scales automatically |

Thanos minimal setup adds a sidecar to each Prometheus that uploads blocks to S3 and a Thanos Query layer in front of both:

```
Prometheus-A ─┐
              ├─ Thanos Query (deduplication) ─ Grafana
Prometheus-B ─┘
      │
  Thanos Sidecar → S3 (long-term storage)
```

### Loki and Tempo Object Storage Backends

Local disk is only suitable for development. In production, both Loki and Tempo need object storage.

**Loki (S3 backend)**
```yaml
# loki-config.yml
storage_config:
  aws:
    s3: s3://my-loki-chunks-bucket
    region: us-east-1
  boltdb_shipper:
    shared_store: s3

schema_config:
  configs:
    - from: 2024-01-01
      store:    boltdb-shipper
      object_store: aws
      schema:   v13
      index:
        prefix: loki_index_
        period: 24h
```

**Tempo (S3 backend)**
```yaml
# tempo-config.yml
storage:
  trace:
    backend: s3
    s3:
      bucket: my-tempo-traces-bucket
      region: us-east-1
```

Both require an ECS task role with `s3:PutObject`, `s3:GetObject`, `s3:ListBucket` on the target bucket.

### Loki Label Schema Design

Label design is the most common Loki footgun. Getting it wrong causes cardinality explosions that degrade query performance.

**Rule:** labels are for filtering streams. Everything else goes in the log line or Loki structured metadata.

| Label (low cardinality — use as label) | Structured metadata / log line (high cardinality — do not use as label) |
|----------------------------------------|-------------------------------------------------------------------------|
| `env` (prod, staging, dev) | `trace_id` |
| `service` (payment-service) | `user_id` |
| `container_name` | `request_id` |
| `ecs_cluster` | `order_id` |
| `source` (stdout/stderr) | `session_id` |

`trace_id` specifically should be in the log line as a field, not a label. Loki 3.x structured metadata lets you store it alongside the stream without inflating label cardinality, and Grafana can still use it for Tempo linking.

Aim for fewer than 10 unique values per label. If a label field has thousands of values in prod, it is the wrong thing to be a label.

### Loki Retention and Compactor

Without compactor configuration, logs accumulate and disk/object storage fills unboundedly.

```yaml
# loki-config.yml
compactor:
  working_directory: /loki/compactor
  shared_store:      s3
  retention_enabled: true

limits_config:
  retention_period: 30d     # global default
  # Per-tenant or per-stream overrides also possible
```

Run the compactor as part of your Loki deployment (it can run embedded in monolithic mode, or as a separate component in microservices mode). Without it, retention is never enforced even if you set `retention_period`.

### Cardinality Budget — Concrete Guidance

"Keep labels low-cardinality" is not actionable on its own. Here are concrete numbers and tools.

**Targets for Prometheus:**
- Aim for fewer than **1 million active time series** per Prometheus instance
- No single label should have more than **100–200 unique values** in the active series set
- A `user_id` or `request_id` label will immediately blow past both limits

**Audit commands:**

```promql
-- Count total active series
count({__name__=~".+"})

-- Find the highest-cardinality metrics
topk(10, count by (__name__)({__name__=~".+"}))

-- Find what label values exist for a specific metric
count by (service) (http_request_duration_seconds_bucket)
```

**Targets for Loki:**
- Total active streams: aim for fewer than **100,000 unique label combinations** per Loki instance
- Query: use Loki's `/loki/api/v1/label` endpoint or `grafana-agent` cardinality tooling

---

## Security — What "Protect Endpoints" Actually Means

The best practices section says "protect endpoints." Here is what that requires in practice.

### Authentication Between Components

**Prometheus web config (basic auth)**
```yaml
# prometheus-web-config.yml
basic_auth_users:
  alloy: $2y$12$...  # bcrypt hash of the password
```
```bash
prometheus --web.config.file=prometheus-web-config.yml
```

**Alloy remote_write with basic auth**
```alloy
prometheus.remote_write "backend" {
  endpoint {
    url = env("PROMETHEUS_REMOTE_WRITE_URL")
    basic_auth {
      username = env("PROM_USER")
      password = env("PROM_PASS")
    }
  }
}
```

**mTLS between Alloy and Loki/Tempo** (recommended for production)
```alloy
loki.write "backend" {
  endpoint {
    url = env("LOKI_PUSH_URL")
    tls_config {
      ca_file   = "/etc/alloy/certs/ca.crt"
      cert_file = "/etc/alloy/certs/client.crt"
      key_file  = "/etc/alloy/certs/client.key"
    }
  }
}
```

### Secrets Management — Not Environment Variables in Compose Files

Grafana datasource credentials, Alertmanager PagerDuty keys, and remote write passwords should never be hardcoded in config files or docker-compose env blocks.

| Platform | Recommended approach |
|----------|---------------------|
| **AWS ECS** | ECS secrets integration with AWS SSM Parameter Store or Secrets Manager; inject as env vars at task launch, not in task definition JSON |
| **Self-hosted** | Vault Agent sidecar writes short-lived secrets to a tmpfs mount that Alloy and Grafana read at startup |
| **Grafana datasource provisioning** | Use `secureJsonData` fields and reference `$__env{VAR}` — Grafana will not expose these values through the API |

```yaml
# Grafana datasource provisioning — correct way
datasources:
  - name: Loki
    type: loki
    url: http://loki:3100
    basicAuth: true
    basicAuthUser: alloy
    secureJsonData:
      basicAuthPassword: $__env{LOKI_PASSWORD}  # injected from SSM, not hardcoded
```

**Grafana itself** should have its `admin` password, `secret_key`, and database credentials sourced from Secrets Manager or Vault, not from a `.env` file committed near the repo.

