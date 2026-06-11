---
name: splunk-configure
description: >-
  Generate Splunk Observability Cloud detector and dashboard Terraform from an
  existing otel-audit report. Reads .observe/otel.md, classifies metrics and
  APM readiness coverage into detector/dashboard categories, and outputs
  ready-to-apply HCL with SignalFlow program_text. Use when the user types
  $splunk-configure, asks to "generate detectors", "create alerts from audit",
  "build Terraform for monitors", "set up Splunk detectors", "create
  dashboards", "audit alert coverage", "classify app down vs degraded impact",
  "build blast-radius dashboards", or asks to improve alerting or incident
  localization from observability gaps.
metadata:
  author: otel-studio
  version: 0.1.0
  category: observability
---

# Detect -- Splunk O11y Detector and Dashboard Terraform from Audit Report

## Overview

Read an existing `.observe/otel.md` audit report, classify detected metrics and
APM readiness coverage into detector categories, and generate Terraform
configuration for Splunk Observability Cloud detectors and dashboards. Use
metric detectors for available signals and report missing readiness coverage as
instrumentation prerequisites instead of inventing alerts from absent data.

When a prompt mentions MTTD, faster incident detection, better alerts, easier
incident debugging, or blast-radius visibility, generate detectors and
dashboards that make customer impact, affected workflow, likely fault domain,
blast radius, and release/config correlation faster to detect and localize.

When incident evidence mentions missed, flapping, auto-resolved, or no-data
alerts, treat that as detector reliability evidence. Do not ask app instrumentation
to emit alert lifecycle metrics unless the app owns those events; audit the
detector, dashboard, data quality, and alert coverage behavior in
`alert-coverage-audit` output.

## When to Use

- After running `$otel-audit` to generate `.observe/otel.md`
- When the user wants alerting/detection Terraform for their service
- When creating monitors for RED signals or saturation metrics
- When creating dashboards for API workflows, dependencies, freshness,
  queue/backpressure, customer impact, or release context
- When auditing whether existing or desired alerts cover app-down,
  primary workflow degradation, auth degradation, ingest lag/drops,
  critical workflow delay, dependency failure, blast radius, or capacity
  saturation

**When NOT to use:** If no audit report exists yet, instruct the user to run
`$otel-audit` first.

## Process

### Supported Modes

Use the default detector/dashboard generation path unless the user asks for a
specific mode. Modes share the same `.observe/otel.md` input and should be
implemented as report sections or Terraform output, not as separate skills.

| Mode | Trigger | Output |
|---|---|---|
| `generate` | Generate detectors/dashboards from audit metrics | `.observe/terraform/`, `.observe/detectors.md`, `.observe/dashboards.md` |
| `alert-coverage-audit` | Audit existing or desired alerts/dashboards for incident detection/localization gaps | Coverage matrix comparing readiness areas to detectors/dashboards, including detector reliability evidence for missed, flapping, auto-resolved, or no-data alerts; missing app-owned signals become instrumentation prerequisites |
| `impact-classify` | Distinguish app down from degraded API, workflow, auth, ingest, or workflow-specific impact | Impact detectors/dashboard sections grouped by workflow, outcome, region/environment, dependency, and release context |
| `blast-radius` | Detect region-wide or multi-workflow incidents earlier | Region/environment/workflow rollups and dashboards that show single-service, single-region, multi-region, or all-region blast radius |

If existing Splunk detectors or dashboards are not available in the repository
or through an approved API/source, do not claim they were audited. Generate the
desired-state coverage matrix and clearly label it as based on `.observe/otel.md`
and local Terraform/config evidence only.

### Step 1 -- Locate Audit Report

Look for `.observe/otel.md` in the repository root.

- If the file exists, proceed to Step 2.
- If the file is missing, stop and respond:

> No audit report found at `.observe/otel.md`. Please run `$otel-audit` first
> to generate the observability coverage report.

### Step 2 -- Parse Service Metadata, Metrics, and Readiness Coverage

Extract from `.observe/otel.md`:

1. **Service metadata** from the report header:
   - Service name (from the `# Observability Report: {service-name}` heading)
   - Language (from the `**Language:**` field)
   - Framework (from the `**Framework:**` field)

2. **Metrics table** from the `### Metrics` section:
   - Each row provides: metric name, source, and type (auto/custom)
   - Record all metrics for classification in Step 3

3. **Gaps** from the `## Gaps` section. This is the current-main
   `$otel-audit` handoff:
   - Treat each bullet as an instrumentation prerequisite candidate.
   - Infer impact from the wording, available metric evidence, and readiness
     sections.
   - Add missing readiness signals to the generated report instead of creating
     detector placeholders for absent data.

4. **Legacy APM Readiness Coverage** from the `## APM Readiness Coverage`
   section when present. Use this only when `## Gaps` is absent:
   - Area
   - Status
   - Evidence
   - Gap
   - Detection/Localization Impact, or the legacy column name MTTD Impact

5. **Incident Readiness** from the `## Incident Readiness` section when present:
   - API/workflow impact
   - dependencies
   - freshness/backpressure
   - auth/edge/capacity/release context
   Missing or partial incident-readiness areas become instrumentation
   prerequisites unless matching metrics already exist in the Metrics table.

6. **Detector reliability evidence** from gaps, readiness sections, local alert
   config, or incident evidence:
   - missed, flapping, auto-resolved, or no-data alerts
   - detectors that cannot distinguish no traffic from no telemetry
   - dashboard or detector group-by keys that hide workflow, region,
     environment, dependency, or release blast radius
   Record these in `alert-coverage-audit` output. Only turn them into
   instrumentation prerequisites when the missing data is app-owned and absent.

If the Metrics section says "No metrics detected.", do not generate detector or
dashboard Terraform from metrics. Continue processing `## Gaps`,
`## APM Readiness Coverage`, and `## Incident Readiness` so the output still
includes `.observe/detectors.md` instrumentation prerequisites and, for
`alert-coverage-audit` mode, an alert coverage matrix. Include an alert
coverage matrix when running `alert-coverage-audit`. If there are no metrics,
no gaps, and no readiness sections, stop and respond:

> The audit report contains no metrics. Detectors require metric data.
> Run `$otel-instrument` to add instrumentation, then re-run `$otel-audit`.

### Step 3 -- Classify Metrics into Detector Categories

When metrics exist, load `references/detector-classification.md` and apply the
classification rules to each metric from Step 2.

Assign each metric to exactly one category. Apply incident-readiness categories
before generic latency, error, throughput, or saturation so customer-impact,
dependency, freshness, backpressure, auth/edge, capacity, and release/config
signals do not collapse into generic RED buckets.

- **latency** -- duration histograms
- **error** -- counters with failure/error/invalid keywords
- **throughput** -- counters without error keywords
- **saturation** -- gauges for connections, buffers, queues, lag,
  disk/filesystem, and resource utilization
- **freshness** -- gauges/histograms for event age, ingest lag, processing lag
- **backpressure** -- queue depth, consumer lag, oldest-message age, rebalance
  count, paused/blocked consumer gauges
- **dependency** -- dependency error, timeout, retry, rate-limit, throttle,
  circuit-breaker, endpoint health, target health, availability, unhealthy
  target count, or operation-duration metrics
- **customer-impact** -- workflow success/error/degraded/timeout counters or
  duration histograms for rendering, transaction, auth, notification, or other
  user-visible workflows
- **impact-classification** -- app/workflow availability, synthetic probe/client telemetry,
  degraded/unavailable impact, or customer-impact summary metrics used to
  distinguish app down from degraded API, workflow, auth, ingest, or
  workflow-specific impact
- **auth-edge** -- login, identity provider, domain routing, token/session, DNS, TLS,
  certificate, gateway, or edge workflow metrics
- **capacity-saturation** -- memory, CPU, disk/filesystem, JVM,
  worker/thread-pool utilization, inflight/concurrency, queue saturation,
  quota, throttling, crash-loop/restart, desired-vs-healthy,
  startup/readiness/healthcheck failure, HPA/ASG, pod, task, process, or node
  capacity metrics
- **release-context** -- `service.version`, `deployment.environment`,
  `deployment.region`, `deployment.platform`, `container.image.tag`, artifact
  version, config/canary/rollout metadata used as dashboard filters and
  detector dimensions, not as standalone alert metrics
Skip metrics that match the exclusion rules (auto-instrumented library metrics
that duplicate custom signals).

For every `## Gaps` entry that is still missing a metric, add an entry to the
generated report's "Instrumentation Prerequisites" section. Do not generate a
detector for a missing signal. Recommend `$otel-instrument` with the specific
coverage area that must be added first.

For legacy APM readiness rows, add prerequisites for every area with status
`missing` or `partial` when no `## Gaps` entry covers the same area.

For every incident-readiness area with status `missing` or `partial`, add a
prerequisite unless equivalent metrics are present. Do not generate detectors
from desired impact, dependency, freshness, backpressure, auth/edge, capacity,
or release/config rows unless the Metrics table contains the corresponding
metric evidence.

### Step 4 -- Generate Terraform

Create the output directory `.observe/terraform/` if it does not exist.

Generate detector Terraform using `references/terraform-templates.md`. Also
generate dashboard Terraform or, when a dashboard panel cannot be expressed
confidently from available metrics, a dashboard specification in
`.observe/dashboards.md`.
Dashboard Terraform should use `signalfx_dashboard_group`,
`signalfx_dashboard`, and chart resources such as `signalfx_time_chart`,
`signalfx_single_value_chart`, `signalfx_table_chart`, or `signalfx_list_chart`
when enough metric evidence exists.

#### `.observe/terraform/detectors.tf`

For each classified metric, emit a `signalfx_detector` resource block:

```hcl
resource "signalfx_detector" "<category>_<sanitized_metric_name>" {
  name        = "${var.service_name} <Category> - <metric_name>"
  description = "Detects <category> anomalies for <metric_name>"

  program_text = <<-EOF
    <SignalFlow program from template>
  EOF

  rule {
    description  = "<Category> threshold breached"
    severity     = "<severity from template>"
    detect_label = "<label from template>"

    notifications = [var.notification_channel]
  }
}
```

Sanitize metric names for HCL identifiers: replace dots and hyphens with
underscores, strip leading digits.

#### `.observe/terraform/variables.tf`

```hcl
variable "realm" {
  description = "Splunk Observability Cloud realm"
  type        = string
}

variable "api_token" {
  description = "Splunk Observability Cloud API token"
  type        = string
  sensitive   = true
}

variable "service_name" {
  description = "Service name for detector naming"
  type        = string
  default     = "<service-name from report>"
}

variable "notification_channel" {
  description = "Notification target for detector alerts"
  type        = string
}

# Per-detector threshold overrides
<one variable block per detector with its default threshold>
```

#### `.observe/terraform/dashboards.tf`

Generate a service dashboard with sections for every detector category that has
metrics:

| Section | Charts |
|---|---|
| API workflows | request rate, p99 latency, 5xx/error rate by route/status |
| External dependencies | dependency latency, error/timeout/retry/rate-limit rate, circuit-breaker state, endpoint health, target health, availability, and unhealthy target count when present |
| Data freshness | newest event age, ingest lag, processing lag, accepted/dropped by reason |
| Queue/backpressure | queue depth, consumer lag, oldest message age, rebalance count, paused consumers |
| Customer impact | user workflow duration, success/error/degraded/timeout by workflow |
| Impact classification | app-down vs degraded workflow counts, synthetic probe/client telemetry/API/freshness/dependency correlation, impact by region/environment |
| Auth/edge workflows | auth success/error/latency, identity provider failures, domain-routing/DNS/TLS/gateway failures |
| Capacity saturation | memory/CPU/disk/runtime/thread-pool/concurrency/quota/throttle/restart, desired-vs-healthy, startup/readiness/healthcheck failure, and traffic target health where metrics exist |
| Blast radius | impacted workflows and regions over time, grouped by service, deployment region, environment, platform, release, and dependency |
| Release context | dashboard filters or event overlays for service version, deployment environment/region/platform, container image tag, artifact version, config version, and canary/rollout dimensions |

Every dashboard must include a `service.name` filter. Include
`deployment.environment`, `deployment.region`, `deployment.platform`,
`service.version`, `container.image.tag`, artifact version, config version, and
rollout/canary filters when the metrics or audit evidence show those dimensions
exist and are low-cardinality.
Prefer exact metric/resource attribute names proven by the audit, such as
`cloud.region` or platform-provided container image attributes. Do not create
duplicate filters only to force generic alias names.
Dashboard variables that apply globally across multiple charts must use
`apply_if_exist = true`. This is required for optional dimensions such as
environment, region, namespace, platform, version, image tag, config version,
rollout, dependency, and custom realm, because not every chart metric has every
dimension. Never generate wildcard variables for optional
dimensions with `apply_if_exist = false`; they can silently filter all data out
of charts whose metric lacks that dimension.
Keep the Splunk Observability Cloud API `realm` variable separate from service
telemetry dimensions. Do not use `var.realm` as a SignalFlow filter for
`sfx_realm`, `deployment.region`, `cloud.region`, or any application/runtime
dimension unless live metric metadata or the audit proves that exact dimension
and value. Prefer dashboard variables for dimensions such as environment,
region, namespace, platform, version, image tag, and custom realm so users can
select the active stream. If a chart needs a fixed dimension value, the value
must come from proven metric metadata or an explicit user choice, not from the
provider/API realm.
Before writing chart `program_text`, verify every filter and group-by dimension
against audit evidence, local metric metadata, or approved Splunk API metadata.
If a dimension is not present on the target metric, omit the filter/group-by and
document the missing dimension in `.observe/dashboards.md`. Never group by a
dimension solely because instrumentation code intended to emit it.
Only generate a dashboard panel for a metric name that is present in the audit
or verified through live metric metadata. If using a provider-derived,
precomputed, or transformed metric name instead of the audited OTel name, record
the live metadata provenance in `.observe/dashboards.md` and mark the panel as
verified.
Live metric metadata alone is not enough to claim source-backed coverage when
the repository does not contain the emitter. If the metric is found only in
build output, generated artifacts, `target/`, `build/`, `.class`, jar, coverage,
or stale runtime files, treat it as stale/unowned evidence: do not generate the
panel by default, and list it as a verification issue or cleanup prerequisite.
Use source files, checked-in IaC/config, audited instrumentation evidence, or an
explicit user-approved external source as provenance.
Do not combine mixed-unit signals on one chart. Split boolean readiness,
availability, ratios/percentages, bytes, counts, rates, cumulative counters,
and durations into separate panels unless the dashboard type has explicit
per-series units or axes and the report documents the unit handling. For
example, database readiness and connection-pool utilization should be separate
panels, not one shared-axis time chart.
For runtime capacity, generate separate dashboard panels and detectors for each
source-backed resource class instead of substituting adjacent runtime metrics.
When source-backed CPU utilization metrics such as `process.cpu.utilization`,
`process.runtime.*.cpu.utilization`, `jvm.cpu.recent_utilization`, or a
runtime-specific equivalent are present, generate a CPU utilization panel and
a CPU saturation detector. Memory/heap usage, thread/goroutine/worker count, GC
pressure, concurrency, disk, and quota signals belong in separate panels with
their own units. Do not use thread count, heap usage, or GC metrics as a CPU
proxy. If only cumulative CPU time is available, chart it as a diagnostic rate
with `rollup='rate'` or equivalent and list normalized CPU utilization as a
missing signal before creating a CPU saturation detector.
For pre-aggregated percentile metrics, including names such as `.p99`, `.p95`,
`p50`, `quantile`, or metrics marked as already-quantized, do not average
series for the headline chart. Use `max()` for current worst-case values and
`max(by=[...])` for breakdowns. Use `.percentile(pct=99)` only for raw
duration/histogram distributions. Match units to metric names and metadata:
convert nanoseconds to seconds or milliseconds, convert ratios to percent only
when the metric is a ratio, and label bytes/counts/rates honestly.
For cumulative counters or cumulative timers, including names that end in
`.total`, `.count`, `.time`, or metrics whose metadata shows cumulative
temporality, do not chart the raw cumulative value as current health. Use
`rollup='rate'`, a delta/rate transform, or a true duration histogram/summary.
If neither is available, label the chart as cumulative and mark it unverified
instead of presenting it as a latency, utilization, or health-rate panel.
After generating dashboard Terraform, run a value sanity check when a Splunk API
token or local metric query path is available. Execute each chart's SignalFlow
over a recent window with known traffic or a representative historical window
and confirm it returns non-empty series with plausible magnitude and expected
dimensions. If live verification is unavailable, mark the dashboard report as
unverified and call out the exact metric names, filters, units, and group-bys
that still need validation before `terraform apply`. When validating in the UI
after an update, reload the dashboard URL without a stale `configId` parameter
or reset saved dashboard overrides so the browser uses the updated dashboard
definition.

#### `.observe/terraform/terraform.tfvars.example`

Generate a `.tfvars.example` file the user copies and fills in to apply:

```hcl
realm                = ""   # Splunk Observability Cloud realm
api_token            = ""   # Splunk O11y API token (org-level, detector write)
service_name         = "<service-name from report>"
notification_channel = ""   # e.g. "Email,team@example.com" or PagerDuty routing key
```

Do NOT include per-detector threshold variables in this file -- they already have
sensible defaults in `variables.tf`. Include `realm`, `api_token`, and
`notification_channel` (which have no defaults) plus `service_name` for
convenience (it has a default from the report but users often override it).

### Step 5 -- Generate Detectors and Dashboards Report

Create `.observe/detectors.md` and `.observe/dashboards.md` as human-readable
companions to the Terraform files. The detector report documents every detector
that was generated, its classification rationale, thresholds, and which metrics
were skipped. The dashboard report documents generated panels, filters, and any
readiness coverage that still requires `$otel-instrument`.

Use the following structure:

```markdown
# Detectors Report: <service-name>

**Language:** <lang> | **Framework:** <framework> | **Date:** <YYYY-MM-DD>
**Source:** `.observe/otel.md` | **Output:** `.observe/terraform/`

## Summary

| Category   | Count | Severity | Detection Method |
|------------|-------|----------|------------------|
| Latency    | N     | Warning  | P99 static threshold |
| Error      | N     | Critical | Sudden change (mean + stddev) |
| Saturation | N     | Warning  | Static threshold |
| Throughput | N     | Major    | Sudden change (mean + stddev) |
| Freshness  | N     | Critical | Static lag/age threshold |
| Backpressure | N   | Major    | Static lag/depth threshold |
| Dependency | N     | Major    | Latency/error/timeout threshold |
| Customer Impact | N | Critical | Workflow success/error/latency threshold |
| Impact Classification | N | Critical | App-down/degraded workflow rollup |
| Auth/Edge | N | Critical | Login/edge success/error/latency threshold |
| Capacity Saturation | N | Major | Resource/quota/throttle/restart threshold |
| **Total**  | **N** | | |

## Latency Detectors

P99 percentile against a static threshold. Default: **1.0s**.

| # | Detector | Metric | Source | Threshold Variable | Default |
|---|----------|--------|--------|-------------------|---------|
| 1 | `latency_<id>` | `<metric>` | <source> | `latency_<id>_threshold` | 1.0 |

## Error Detectors

Sudden-change detection using `against_recent.detector_mean_std` (above baseline).
Default: **3.0 stddev**, 5m current window vs 1h history.

| # | Detector | Metric | Source | Threshold Variable | Default |
|---|----------|--------|--------|-------------------|---------|
| 1 | `error_<id>` | `<metric>` | <source> | `error_<id>_stddev` | 3.0 |

## Saturation Detectors

Gauge value against a static threshold. Default: **85.0**.

| # | Detector | Metric | Source | Threshold Variable | Default |
|---|----------|--------|--------|-------------------|---------|
| 1 | `saturation_<id>` | `<metric>` | <source> | `saturation_<id>_threshold` | 85.0 |

## Throughput Detectors

Sudden-change detection using `against_recent.detector_mean_std` (out-of-band).
Default: **3.0 stddev**, 5m current window vs 1h history.

| # | Detector | Metric | Source | Threshold Variable | Default |
|---|----------|--------|--------|-------------------|---------|
| 1 | `throughput_<id>` | `<metric>` | <source> | `throughput_<id>_stddev` | 3.0 |

## Skipped Metrics

| Metric | Reason |
|--------|--------|
| `<metric>` | <why it was not classified> |

## Instrumentation Prerequisites

| Area | Audit Status | Missing Signal | Why No Detector Was Generated | Next Step |
|------|--------------|----------------|-------------------------------|-----------|
| Data freshness | missing | newest event age, ingest lag, dropped records by reason | No metric exists in `.observe/otel.md` | Run `$otel-instrument` to add data freshness signals |
| Dependency health | missing | endpoint health, target health, availability, timeout/rate-limit count, or unhealthy target count | No matching dependency health metric exists in `.observe/otel.md` | Run `$otel-instrument` or configure platform telemetry for dependency health signals |
| Capacity health | missing | disk saturation, desired-vs-healthy, startup/readiness/healthcheck failure, restart count, or traffic target health | No runtime/platform metric exists in `.observe/otel.md` | Run `$otel-instrument` or add platform telemetry before creating detectors |

## Alert Coverage Matrix

Use this section for `alert-coverage-audit` mode and include it in the detector
report when readiness coverage is partial or missing.

| Incident Pattern | Existing/Generated Coverage | Missing Signal or Dashboard | Detection/Localization Risk | Next Step |
|---|---|---|---|---|
| Primary workflow unavailable | {detector/dashboard or "none found"} | {workflow impact metric, dependency metric, or synthetic/client telemetry signal} | {why detection/debugging remains slow} | {configure detector or instrument signal} |
| Ingest lag/drops | {detector/dashboard or "none found"} | {freshness/drop/lag signal} | {risk} | {next step} |
| Auth/domain-routing/edge | {detector/dashboard or "none found"} | {auth/edge workflow signal} | {risk} | {next step} |
| Critical business workflow | {detector/dashboard or "none found"} | {workflow outcome signal} | {risk} | {next step} |
| Multi-region blast radius | {detector/dashboard or "none found"} | {region/environment/workflow rollup} | {risk} | {next step} |
| Dependency endpoint health | {detector/dashboard or "none found"} | {endpoint health, target health, unavailable, timeout, rate-limit, or unhealthy target signal} | {risk} | {next step} |
| Capacity saturation | {detector/dashboard or "none found"} | {CPU/memory/disk/quota/throttle/concurrency/restart/readiness/desired-vs-healthy/platform signal} | {risk} | {next step} |
| Release/config correlation | {dashboard filter/event overlay or "none found"} | {service.version, deployment.region, deployment.platform, container.image.tag, artifact version, config version, or rollout/canary id} | {risk} | {next step} |
| Detector reliability | {detector/dashboard evidence or "none found"} | {missing no-data handling, anti-flap tuning, auto-resolve guard, data-quality signal, or alert route evidence} | {risk} | {tune detector, fix alert coverage, or instrument app-owned missing signal} |

## Classification Rules Applied

<include the decision flowchart from references/detector-classification.md>

## Terraform Output

| File | Contents |
|------|----------|
| `detectors.tf` | N `signalfx_detector` resources with inline SignalFlow |
| `dashboards.tf` | dashboard group, dashboard, and panel resources for classified metrics when metric evidence exists |
| `variables.tf` | 4 required + N threshold variables |
| `terraform.tfvars.example` | Template for required variables |

## Next Steps

1. Copy and fill in credentials
2. Review threshold defaults and override as needed
3. `cd .observe/terraform && terraform init && terraform plan && terraform apply`
4. Tune thresholds based on production baselines

---
*Generated by splunk-configure on <YYYY-MM-DD>*
```

### Step 6 -- Chat Summary

After generating all files (Terraform + reports), present a summary:

```
## Detectors Generated

| Category    | Count |
|-------------|-------|
| Latency     | N     |
| Error       | N     |
| Saturation  | N     |
| Throughput  | N     |
| Freshness   | N     |
| Backpressure | N    |
| Dependency  | N     |
| Customer Impact | N |
| Impact Classification | N |
| Auth/Edge | N |
| Capacity Saturation | N |

**Output:** `.observe/terraform/`

Files:
- `.observe/terraform/detectors.tf` -- N detector resources
- `.observe/terraform/dashboards.tf` -- service dashboard and panel resources when metric evidence exists
- `.observe/terraform/variables.tf` -- realm, api_token, service_name, notification_channel + N threshold variables
- `.observe/terraform/terraform.tfvars.example` -- copy to `terraform.tfvars`, fill in credentials
- `.observe/detectors.md` -- full detectors report with classification details
- `.observe/dashboards.md` -- dashboard sections, filters, and readiness prerequisites

Next:
1. `cp .observe/terraform/terraform.tfvars.example .observe/terraform/terraform.tfvars`
2. Fill in `realm`, `api_token`, and `notification_channel` in `terraform.tfvars`
3. `cd .observe/terraform && terraform init && terraform plan`
4. `terraform apply`
```

## Output Templates

### detectors.tf Shape

```hcl
terraform {
  required_providers {
    signalfx = {
      source  = "splunk-terraform/signalfx"
      version = "~> 9.0"
    }
  }
}

provider "signalfx" {
  auth_token = var.api_token
  api_url    = "https://api.${var.realm}.signalfx.com"
}

resource "signalfx_detector" "latency_http_server_request_duration" {
  name        = "${var.service_name} Latency - http.server.request.duration"
  description = "Detects high p99 latency for http.server.request.duration"

  program_text = <<-EOF
    A = data('http.server.request.duration', filter=filter('service.name', '${var.service_name}')).percentile(pct=99).publish(label='P99 Latency')
    detect(when(A > threshold(${var.latency_http_server_request_duration_threshold}))).publish('P99 Latency Too High')
  EOF

  rule {
    description  = "P99 latency exceeds threshold"
    severity     = "Warning"
    detect_label = "P99 Latency Too High"

    notifications = [var.notification_channel]
  }
}
```

### variables.tf Shape

```hcl
variable "realm" {
  description = "Splunk Observability Cloud realm"
  type        = string
}

variable "api_token" {
  description = "Splunk Observability Cloud API token"
  type        = string
  sensitive   = true
}

variable "service_name" {
  description = "Service name for detector naming"
  type        = string
  default     = "my-service"
}

variable "notification_channel" {
  description = "Notification target for detector alerts"
  type        = string
}

variable "latency_http_server_request_duration_threshold" {
  description = "P99 latency threshold in seconds for http.server.request.duration"
  type        = number
  default     = 1.0
}
```

## Red Flags

- Audit report has no metrics section
- All metrics are auto-instrumented library duplicates (nothing to detect on)
- Service name contains characters invalid for SignalFlow filter values
