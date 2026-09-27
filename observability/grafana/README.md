# Grafana Observability Stack

Setup and profiles are covered in [GRAFANA_SETUP.md](GRAFANA_SETUP.md). This file records the improvements made to the collector configs and the Temporal Java/Go SDK dashboard, and explains why they were needed.

Local endpoints (`--profile local`):

| Service | URL |
|---|---|
| Grafana | http://localhost:4000 (admin/admin) |
| Prometheus | http://localhost:9090 |
| OTel Collector | OTLP gRPC `:4317`, OTLP HTTP `:4318`, health `:13133` |

Grafana is mapped to host port 4000 so it doesn't collide with web apps on port 3000.

## Collector: `job` label reflects the real source

All three collector configs (`otel-collector-config*.yaml`) used to upsert `service.name: temporal-jumpstart-dotnet` in the `resource` processor. Prometheus remote write maps `service.name` to the `job` label, so every app, Java included, showed up as one .NET job.

That override is removed. `job` now comes from the receiver:

- Scraped targets use the scrape `job_name`, for example `temporal-worker-spring` or `temporal-worker-standard`.
- OTLP pushers use the `service.name` the app reports.

Individual apps are still identified by the `service` and `worker_instance` target labels, for example `service="apps-worker"`. Series recorded before this change keep `job="temporal-jumpstart-dotnet"` until retention removes them.

## Dashboard: Temporal Java/Go SDK (Tally) Metrics

File: `provisioning/dashboards/temporal/temporal-go-java-sdks-tally.json`

### The 30s histogram ceiling

Java SDK timers go through Micrometer, which publishes percentile-histogram buckets only up to 30s (the default `maximumExpectedValue`). Anything slower falls into the `+Inf` bucket, and `histogram_quantile` then returns the highest finite bound. **Any p95 above 30s shows as exactly 30s.**

The upstream dashboard in [temporalio/dashboards](https://github.com/temporalio/dashboards/blob/master/sdk/temporal-go-java-sdks-tally.json) uses the same queries and has the same problem. A dashboard can't fix bucket sizes because the emitting app decides them.

### Changes

| Panel | Before | After | Why |
|---|---|---|---|
| **Long Poll Latency (avg / max)** | p95 of `temporal_long_request_latency` | Mean (`_sum`/`_count`) as solid lines, `_max` gauge as dashed lines | Idle long polls wait about 60s for the server timeout, which is above the ceiling, so p95 was stuck at 30s. Real data: avg about 52s, max about 60s for idle activity pollers. |
| **Every other p95 panel** (RPC, workflow end-to-end, workflow task schedule-to-start/execution/replay, activity and local activity execution/E2E, activity schedule-to-start) | p95 only | p95 plus a dashed max series from the timer's `_max` gauge; titles end in "(p95 / max)" | If p95 flattens at 30s while max goes higher, the ceiling is cutting off the real value. |
| **CPU usage** | Placeholder text | `process_cpu_usage` per service and worker instance (share of all host cores), with host `system_cpu_usage` dashed | Puts SDK behavior next to resource usage. |
| **Memory usage (JVM heap)** | Placeholder text | Heap used per service, with committed heap dashed | Same. |
| **Slots Used (sampled peak) vs Capacity** and **Avg Busy Slots (from execution time)** | One raw `temporal_worker_task_slots_used` panel, `decimals: 0` | Two panels, both counted in slots: sampled peak used against capacity (used + available), and average busy slots from execution time (Little's law). Idle series hidden, legends sorted by max | See below. |
| **Local Activity Throughput** | `rate(..._bucket)` summed | `rate(..._count)` | Adding up buckets counted each event once per bucket, which inflated the rate by the number of buckets. |
| **Variables** | `datasource`, `namespace` | Adds `service` (multi-select, All = `.*`) to every query | Scope the dashboard to one app now that `service` is reliable. |

### Why "Slots Used" read zero while work was processing

`temporal_worker_task_slots_used` is a gauge that Prometheus samples once per scrape (15s). Workflow tasks and activities in this system finish in about 5ms, so a slot is almost never occupied at the moment of a scrape, and the gauge reads 0. The metric itself is correct. In the last hour it read 3 once, for the fulfillment Nexus worker. `decimals: 0` also rounded any fractional value down to 0.

The single panel was replaced by two panels. Both are measured in slots, the same unit as *Slots Available*, so values map directly onto the available count. Every series rolls up per worker type and task queue across worker instances.

| Panel | Series | Query | Good at | Blind spot |
|---|---|---|---|---|
| **Slots Used (sampled peak) vs Capacity** | used (solid) | `max_over_time(slots_used[$__rate_interval])` | Staying correct for long-running tasks, because it reads the gauge | Rarely catches millisecond tasks the scrape never sees in flight |
| | capacity (dashed) | `slots_used + slots_available` | Showing the ceiling that used climbs toward: 200 per worker type today | |
| **Avg Busy Slots (from execution time)** | one line per worker type and task queue | `rate(<task>_execution_latency_seconds_sum[$__rate_interval])` for workflow, activity, local activity and Nexus tasks (Little's law) | Counting every task, however short | Counters only increase on completion, so a long task shows up as a spike when it finishes instead of steady occupancy |

Idle series are hidden with `> 0`. The capacity line is only drawn for worker types that have used slots. That filter cut the slot view from about 75 series to a handful, and removes worker types on the API services that never do work. Legends are sorted by max, descending.

An earlier version put a utilization % series and a busy-slot count in the same panel, on two axes. Each legend row was then formatted in its own series' unit, so percent and plain numbers were mixed in one column. It also couldn't be read against *Slots Available*. Keeping one unit per panel avoids both problems.

Reading them:
- **Light load, short tasks:** *Slots Used* is mostly empty, and *Avg Busy Slots* shows the real activity. Today most worker types stay below 0.01 busy slots out of 200, and fulfillment Nexus peaks around 0.5.
- **Heavy load:** used climbs toward the capacity line. Sustained values near capacity mean the worker type needs more slots or more worker instances.
- **Long activities:** trust *Slots Used*. *Avg Busy Slots* will look spiky.

### Remaining gap: accurate percentiles above 30s

To fix p95 at the source, raise the histogram ceiling in each Java app. For example, with Spring Boot:

```java
@Bean
MeterFilter temporalLongPollHistogram() {
  return new MeterFilter() {
    @Override
    public DistributionStatisticConfig configure(Meter.Id id, DistributionStatisticConfig config) {
      if (id.getName().startsWith("temporal_long_request_latency")) {
        return DistributionStatisticConfig.builder()
            .maximumExpectedValue((double) Duration.ofSeconds(70).toNanos())
            .build().merge(config);
      }
      return config;
    }
  };
}
```

Widen the name match to cover other timers that can run past 30s, such as `temporal_workflow_endtoend_latency` or schedule-to-start. Check the exact meter names your apps register. This snippet has not been tested against these apps.
