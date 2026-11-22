# PR Preflight Reference

## Convention Discovery

Search for existing usage before suggesting anything. Run these against the repo (excluding `node_modules`, `vendor`, `.venv`, build output) and open two or three hits to learn the pattern.

### Dependency manifests (fastest signal)

Check `go.mod`, `package.json`, `pyproject.toml` / `requirements*.txt`, `pom.xml` / `build.gradle*`, `Cargo.toml`, `Gemfile`, `*.csproj` for:

| Concern | Common libraries |
|---------|------------------|
| Logging | Go: `log/slog`, `zap`, `zerolog`, `logrus` · JS/TS: `pino`, `winston`, `bunyan` · Python: `logging`, `structlog`, `loguru` · Java: `slf4j`, `logback`, `log4j2` · Rust: `tracing`, `log` · .NET: `Serilog`, `Microsoft.Extensions.Logging` |
| Metrics | `prometheus/client_golang`, `prom-client`, `prometheus_client`, `micrometer`, `statsd`/`hot-shots`, `datadog`/`dd-trace`, `opentelemetry` metrics SDKs, `metrics` (Rust) |
| Tracing | `go.opentelemetry.io/otel`, `@opentelemetry/*`, `opentelemetry-sdk` (Python), `dd-trace`, `jaeger-client`, `zipkin`, Rust `tracing` + `tracing-opentelemetry` |
| Tests | `testing` + `testify`/`gomock`, `jest`/`vitest`/`mocha`, `pytest`/`unittest`, `JUnit`/`Mockito`, `cargo test`, `RSpec`, `xUnit`/`NUnit`; integration: `testcontainers`, `docker-compose.test.yml`, `httptest`, `supertest` |

### Code search patterns

```bash
# Metrics registration and usage
rg -n "NewCounter|NewGauge|NewHistogram|promauto\.|prom-client|Counter\(|Histogram\(|Gauge\(|meter\.create|\.counter\(|\.histogram\(|statsd\.|metrics\.(Inc|Observe|Gauge)" --type-not lock

# Tracing
rg -n "tracer\.Start|startActiveSpan|start_as_current_span|@WithSpan|tracing::instrument|#\[instrument|otel\.Tracer|trace\.get_tracer"

# Logging (find the dominant style and field names)
rg -n "logger\.(Info|Warn|Error|Debug)|log\.(info|warn|error|debug)|slog\.|zap\.|zerolog|structlog|logger\.(info|warning|error)"

# Alert rule locations
rg -l "alert:|expr:|groups:" --glob "*.y*ml"          # Prometheus rules
rg -l "datadog_monitor|newrelic_alert|grafana_rule|aws_cloudwatch_metric_alarm" --glob "*.tf"
fd -i "alert|monitor|slo" -e yml -e yaml -e json -e tf

# Test layout
fd -e go "_test" | head; fd "test_.*\.py|.*_test\.py" | head; fd ".*\.(test|spec)\.(t|j)sx?" | head
rg -n "//go:build integration|@pytest.mark.integration|@Tag\(\"integration\"\)|describe\.integration" | head
```

Fall back to `grep -rn` if `rg`/`fd` aren't installed.

### What to extract

- **Metric names:** prefix, separator, unit suffix (`_seconds`, `_bytes`, `_total`), whether names include the service.
- **Labels:** the standard set (`service`, `method`, `status`, `route`), and how outcomes are encoded.
- **Histogram buckets:** reuse the repo's bucket definitions instead of library defaults.
- **Log fields:** key names (`err` vs `error`, `userId` vs `user_id`), how the logger gets request context.
- **Span names:** `package.Function`, `HTTP GET /route`, `db.query`, etc.

## Alert Patterns

Templates in PromQL. Translate to the repo's alerting format (Datadog, CloudWatch, New Relic, Grafana) if one is in use. Thresholds are starting points and must be tuned against real traffic.

### Error rate (API / RPC)

```yaml
- alert: OrdersCreateHighErrorRate
  expr: |
    sum(rate(orders_requests_total{op="create",outcome="failure"}[5m]))
      / sum(rate(orders_requests_total{op="create"}[5m])) > 0.05
  for: 10m
  labels: { severity: page }
  annotations:
    summary: "More than 5% of order creations are failing"
    runbook: "<link>"
```

### Latency SLO

```yaml
- alert: OrdersCreateSlow
  expr: |
    histogram_quantile(0.99,
      sum by (le) (rate(order_create_duration_seconds_bucket[5m]))) > 1
  for: 15m
  labels: { severity: ticket }
```

### Job not running (absence)

```yaml
- alert: NightlyReconcileStale
  expr: time() - reconcile_last_success_timestamp_seconds > 2 * 86400
  labels: { severity: page }
```

Requires a gauge set to the Unix time of the last successful run; suggest adding one in Phase 2 for any cron or scheduled job.

### Queue backlog / consumer lag

```yaml
- alert: OrderEventsBacklogGrowing
  expr: deriv(order_events_queue_depth[15m]) > 0 and order_events_queue_depth > 1000
  for: 15m
  labels: { severity: ticket }
```

### Dead-letter queue

```yaml
- alert: OrderEventsDLQNonEmpty
  expr: order_events_dlq_depth > 0
  for: 5m
  labels: { severity: ticket }
```

### Dependency failure

```yaml
- alert: InventoryClientErrors
  expr: sum(rate(inventory_client_requests_total{outcome=~"error|timeout"}[5m])) > 1
  for: 10m
  labels: { severity: ticket }
```

### Saturation

```yaml
- alert: DBPoolSaturated
  expr: db_pool_in_use / db_pool_max > 0.9
  for: 10m
  labels: { severity: ticket }
```

### Severity guidance

- **Page:** user-facing impact now, or will be soon without action (error rate, SLO burn, critical job missed).
- **Ticket:** needs attention within a working day (slow degradation, DLQ entries, non-critical dependency errors).
- If nobody would act on it, it shouldn't be an alert. Put it on a dashboard instead.
