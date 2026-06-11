# Detector Classification Rules

Rules for mapping metrics from an otel-audit report into detector categories.
Apply these rules in order; the first match wins.

## Classification Rules

Apply incident-readiness categories before generic RED categories. Impact
classification, auth/edge, customer impact, freshness, backpressure,
dependency, and capacity signals are usually more useful for incident detection
and localization than generic throughput when the metric names and dimensions
make the domain clear.

Detector reliability evidence is not a service metric category. If incident
evidence mentions missed, flapping, auto-resolved, or no-data alerts, capture
that in `alert-coverage-audit` output and only create an instrumentation
prerequisite when the service lacks an app-owned signal needed by the detector.

### Impact Classification

A metric is an **impact-classification** detector candidate when:

- The metric name contains one of: `impact`, `availability`, `synthetic`,
  `client_telemetry`, `workflow.state`, `workflow.outcome`, `degraded`,
  `unavailable`
- AND the metric can be grouped by low-cardinality workflow, region/environment,
  service, dependency, or release dimensions

These metrics distinguish app down from degraded API, workflow, auth, ingest,
or workflow-specific impact. They should use critical thresholds for unavailable
impact and major/critical thresholds for degraded workflow impact.

### Auth/Edge

A metric is an **auth-edge** detector candidate when:

- The metric name contains one of: `login`, `auth`, `identity_provider`,
  `token`, `session`, `domain_route`, `domain.routing`, `domain-routing`,
  `dns`, `tls`, `cert`, `certificate`, `gateway`, `edge`, `edge.route`,
  `gateway.route`
- AND the metric measures duration, success, error, timeout, expiry, or
  unavailable outcomes

Auth/edge metrics should prioritize authentication, domain-routing, TLS, and
certificate failures because they often appear as generic HTTP failures unless
the service emits a more specific outcome or failure class.

### Customer Impact

A metric is a **customer-impact** detector candidate when:

- The metric name contains one of: `workflow`, `user_flow`, `journey`,
  `operation`, `render`, `load`, `transaction`, `customer_impact`
- AND the metric measures duration, success, error, degraded, timeout, or
  unavailable outcomes

These metrics answer whether customers are down or degraded and should use
critical workflow success/error and latency thresholds.

### Freshness

A metric is a **freshness** detector candidate when:

- The metric name contains one of: `freshness`, `newest_event_age`,
  `event.age`, `ingest.lag`, `processing.lag`, `data.age`, `staleness`
- The metric measures age or lag as a gauge or histogram

Freshness metrics should use static warning/critical thresholds because stale
data is often customer-impacting before request RED signals move.

### Backpressure

A metric is a **backpressure** detector candidate when:

- The metric name contains one of: `queue.depth`, `queue.size`,
  `consumer.lag`, `oldest_message_age`, `rebalance`, `paused_consumer`,
  `blocked_consumer`, `backpressure`
- The source is a queue, worker, stream consumer, or async task processor

Backpressure metrics should use static thresholds for lag/depth/age and
sudden-change detection for rebalance count.

### Dependency

A metric is a **dependency** detector candidate when:

- The metric name contains one of: `dependency`, `client`, `external`,
  `datastore`, `database`, `search`, `cache`, `broker`, `stream`, `cloud`,
  `endpoint_health`, `target_health`, `availability`, `unavailable`,
  `unhealthy`
- AND the metric measures `.duration`, `error`, `timeout`, `retry`,
  `rate_limit`, `throttle`, `circuit_breaker`, endpoint health, target health,
  availability, unhealthy target count, or operation failure

Dependency metrics should be grouped by low-cardinality dependency and
operation dimensions when those dimensions exist.

### Capacity Saturation

A metric is a **capacity-saturation** detector candidate when:

- The metric name contains one of: `capacity`, `utilization`, `memory`, `heap`,
  `cpu`, `disk`, `filesystem`, `fs.`, `jvm`, `threadpool`, `thread_pool`,
  `worker.active`, `inflight`, `concurrency`, `desired`, `healthy`,
  `readiness`, `startup`, `healthcheck`, `quota`, `throttle`, `rate_limit`,
  `restart`, `crashloop`, `pod`, `node`, `task`, `process`, `hpa`, `asg`
- The metric is a gauge/up-down counter, or a counter for throttled/rejected
  work, restarts, crash-loop events, readiness/startup failures, healthcheck
  failures, desired-vs-healthy gaps, or quota/rate-limit breaches

Capacity-saturation metrics should use static thresholds for utilization/quota,
disk/filesystem pressure, startup/readiness/healthcheck failures, and
sudden-change detection for throttles/restarts.
For runtime CPU, prefer normalized utilization gauges such as
`process.cpu.utilization`, `process.runtime.*.cpu.utilization`,
`jvm.cpu.recent_utilization`, or a runtime-specific equivalent. When
source-backed CPU utilization is available, these are capacity-saturation
signals and should produce CPU-specific dashboards and a CPU saturation detector.
Do not use thread count, heap usage, memory usage, GC duration, or
worker counts as CPU coverage. For cumulative CPU time metrics such as
`process.cpu.time`, `jvm.cpu.time`, or runtime-specific CPU time counters, use
diagnostic rate charts with `rollup='rate'` or equivalent only unless the audit
also proves a normalized CPU utilization signal or a safe normalization formula.

### Release Context

A metric or metric dimension is a **release-context** candidate when:

- The metric or dimension name contains one of: `service.version`,
  `deployment.environment`, `deployment.region`, `deployment.platform`,
  `container.image.tag`, `artifact.version`, `artifact_version`,
  `config.version`, `config_version`, `feature_flag`, `canary`, `rollout`,
  `build.version`, or `image.tag`
- AND the value is stable and low-cardinality enough for dashboard filters,
  event overlays, or detector dimensions

Release-context data should be used to correlate incidents to releases,
config changes, platforms, regions, images, and canary/rollout batches. It is
not a standalone alert metric unless another rule also matches a health,
latency, error, dependency, or capacity signal. When a metric has both release
context and a detector-worthy signal, classify the metric by the detector-worthy
signal and use the release context as a dashboard filter or detector dimension.
Classify pure release/config/version metadata as `release-context` only after no
detector-worthy category matches.

### Latency

A metric is a **latency** detector candidate when:

- The metric name contains `.duration` (e.g. `http.server.request.duration`,
  `rpc.server.duration`, `db.client.operation.duration`)
- The metric type is histogram

These metrics measure response time and are best monitored with p99 percentile
thresholds.

### Error

A metric is an **error** detector candidate when:

- The metric name ends in `.total` or `.count` AND contains one of these
  keywords: `error`, `errors`, `failure`, `failures`, `failed`, `invalid`,
  `rejected`, `timeout`, `exception`
- Examples: `http.server.errors.total`, `rpc.server.failure.count`,
  `orders.invalid.total`

These metrics measure failure rates and are best monitored with sudden-change
detection against recent baselines.

### Throughput

A metric is a **throughput** detector candidate when:

- The metric name ends in `.total` or `.count` AND does NOT contain any of the
  error keywords listed above
- Examples: `http.server.requests.total`, `orders.processed.count`,
  `messages.consumed.total`

These metrics measure request/event volume and are best monitored with
sudden-change detection (both drops and spikes).

### Saturation

A metric is a **saturation** detector candidate when:

- The metric type is gauge (observable gauge, up-down counter)
- The metric name contains one of: `connections`, `pool`, `buffer`, `queue`,
  `lag`, `utilization`, `capacity`, `active`, `pending`, `heap`, `memory`,
  `disk`, `filesystem`, `goroutines`, `threads`
- Examples: `db.pool.connections.active`, `process.runtime.go.goroutines`,
  `kafka.consumer.lag`, `jvm.memory.heap.used`

These metrics measure resource consumption and are best monitored with static
thresholds.

## Exclusion Rules

Skip a metric (do not generate a detector) when:

1. **Auto-instrumented library duplicates** -- If both a library auto-instrumented
   metric and a custom metric measure the same signal, prefer the custom metric.
   Common library sources to skip when custom equivalents exist:
   - `redisotel` metrics when custom Redis metrics are present
   - `otelhttp` metrics when custom HTTP metrics are present
   - `otelgrpc` metrics when custom gRPC metrics are present

2. **Runtime/host metrics without actionable thresholds** -- Skip generic
   runtime metrics that lack meaningful static thresholds unless the user
   explicitly requests them:
   - `process.runtime.go.gc.count`
   - `process.runtime.go.mem.heap_alloc` (unless saturation threshold is defined)
   - `process.cpu.time` or `jvm.cpu.time` when no normalized CPU utilization
     signal or safe normalization formula is available

3. **Informational-only metrics** -- Metrics that are purely informational
   and not suitable for alerting:
   - `process.uptime`
   - Version gauges

## Decision Flowchart

```
metric name contains impact/availability/synthetic/client telemetry keyword?
  -> YES -> impact-classification detector
  -> NO

metric name contains specific auth/edge keyword?
  -> YES -> auth-edge detector
  -> NO

metric name contains customer workflow keyword?
  -> YES -> customer-impact detector
  -> NO

metric name contains freshness/lag/age keyword?
  -> YES -> freshness detector
  -> NO

metric name contains queue/backpressure keyword?
  -> YES -> backpressure detector
  -> NO

metric name contains capacity/disk/readiness/healthcheck/quota/throttle/restart keyword?
  -> YES -> capacity-saturation detector
  -> NO

metric name contains ".duration"?
  -> YES -> customer-impact or dependency keyword present?
    -> YES -> customer-impact or dependency detector
    -> NO  -> latency detector
  -> NO

metric name ends with ".total" or ".count"?
  -> YES -> customer-impact or dependency keyword present?
    -> YES -> customer-impact or dependency detector
    -> NO  -> contains error keyword?
      -> YES -> error detector
      -> NO  -> throughput detector
  -> NO

metric type is gauge AND name matches saturation keywords?
  -> YES -> capacity-saturation when incident/capacity keyword is present, otherwise saturation detector
  -> NO

metric or dimension contains service.version/deployment.region/deployment.platform/container.image.tag/artifact version/config/canary/rollout keyword?
  -> YES -> pure release-context dashboard filter or detector dimension, not standalone alert
  -> NO  -> skip (no detector)
```

## Priority Order

When a metric could match multiple categories (rare), use this priority:

1. Impact Classification (answers app-down vs degraded first)
2. Auth/Edge (auth/domain-routing/certificate failures are high-impact)
3. Customer Impact (answers user-visible workflow health)
4. Freshness (stale data is often invisible to request RED)
5. Backpressure (lag and queue pressure are early incident indicators)
6. Dependency (root-cause signals for downstream failures)
7. Capacity Saturation (resource/quota pressure before drops/outage)
8. Latency (generic duration histograms)
9. Error (generic error counters)
10. Throughput (general counters)
11. Saturation (generic gauges)
12. Release Context (dashboard filters and health-signal dimensions only after no detector-worthy category matches)
