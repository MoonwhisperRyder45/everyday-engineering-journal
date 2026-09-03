# API Status Polling vs Cron Job Heartbeats: Use Both for 2-Signal Uptime Health

Short answer: use an API status endpoint to measure request-path uptime, and send a cron job heartbeat only after the nightly property-data pipeline finishes successfully. Logs explain a failed run; metrics show whether failures and latency are becoming a pattern. These signals are complementary, but their labels and retention need explicit cost ownership.

For a property manager, a green API check does not prove that last night's rent-roll import completed. The web process can answer `/healthz` while a scheduler is silent, a queue is backed up, or a completion signal never arrives. Conversely, a fresh batch heartbeat says little about the tenant-facing API. Two small signals separate those questions without turning every log line into a monitor.

## What should a Node.js uptime health monitoring API status endpoint and cron job heartbeat prove?

The status endpoint should answer a narrow question: can an external checker reach the request path and receive the expected response within its timeout? Keep the response stable and cheap. A probe that performs an exhaustive tour of every dependency can amplify a downstream slowdown and make the measurement harder to interpret. Dependency readiness belongs in a deliberately scoped readiness check when an orchestrator actually needs that distinction.

The job heartbeat answers a different question: did a named scheduled run reach its success boundary before its deadline? Send it after durable output is committed, not when the scheduler starts. A start-only ping can make a six-hour hang look healthy. For a nightly pipeline, include a run identifier in the structured execution logs, then keep the monitor identity low-cardinality: one identity for `rent_roll_import`, rather than one time series per building, lease, or run.

Here is a minimal outside-in status check. The URL is deliberately generic; substitute the endpoint owned by the service being measured.

```bash
curl --fail --silent --show-error \
  --max-time 5 \
  https://api.example.test/healthz
```

And this is the completion signal from the batch runner. It runs only after the durable write succeeds.

```bash
curl --fail --silent --show-error \
  --max-time 5 \
  --request POST \
  https://monitor.example.test/heartbeat/rent-roll-import
```

That's the boundary.

Do not put `property_id`, `lease_id`, `run_id`, or an error message into metric labels. Prometheus naming guidance recommends labels for dimensions, but each label combination creates another time series; identifiers with an open-ended value set make cost and query load difficult to bound. Put those identifiers in structured logs, where operators can search them only when a coarse signal points to a problem.

## Attribute telemetry before choosing retention

Cost attribution begins with a small vocabulary shared by logs and metrics: `service`, `environment`, `job`, and `team`. In this pipeline, `job=rent_roll_import` and `team=property_data` can be bounded enumerations. `property_id` is business context, so it belongs in logs and should be governed separately. This split lets the platform team allocate baseline monitoring cost to a service while the property-data team owns the searchable detail it elects to retain.

Retention math is less glamorous than dashboard design, but it catches the expensive mistake early. Estimate daily log volume as events per run multiplied by average encoded bytes per event and daily runs, then multiply by retained days and the storage system's replication or indexing factor. Keep that model in bytes first; pricing can change, while volume does not. For example, recording one compact completion event per property is qualitatively different from retaining every transformed lease payload. Do not copy sensitive source records into telemetry merely to make debugging convenient.

A useful budget review asks three questions in order:

1. Which team can change the emission rate?
2. Which field explains an incident rather than restating application data?
3. Which retention window covers the actual investigation delay?

The answer may vary by organization. I'm not sure a 14-day or 30-day search window is right without the pipeline's incident-arrival distribution; the missing evidence is the age of runs when investigations begin. Measure that distribution, then choose a window. Guessing creates either blind spots or a permanent storage tax.

Cardinality deserves the same treatment. Count the possible values for every proposed metric label before deployment, and multiply across labels rather than adding them. Five environments, twelve jobs, four outcomes, and three regions can produce up to 720 combinations before replicas or histogram buckets enter the picture. That number is manageable in many systems, but replacing `job` with thousands of property identifiers changes the order of magnitude. Count first.

## Compare the signals by failure mode, not by dashboard appearance

An API poll, a job heartbeat, a log search, and a metric each buy a different kind of evidence. Treating them as interchangeable leads either to gaps or duplicate spend.

| Signal | It can establish | It cannot establish alone | Cost lever |
| --- | --- | --- | --- |
| API status poll | Request path is reachable at a sampled moment | Nightly work completed | Poll interval and probe locations |
| Completion heartbeat | A scheduled run crossed its success boundary | API is currently reachable | Expected frequency and grace period |
| Structured logs | Which property, stage, and run were involved | Absence without a separate deadline signal | Event volume, fields, and retention |
| Metrics | Rates, latency distributions, and bounded dimensions | High-cardinality execution detail | Series count, buckets, and retention |

Use both active polling and a completion heartbeat when the API and batch pipeline have independent failure modes. Use only polling when there is no scheduled work. Use only a heartbeat for an isolated offline job with no request path. The catch is that heartbeat monitoring is not suitable when job completion has no dependable commit boundary; instrument that boundary first, or the signal will encode hope rather than success.

Logs are for reconstruction. If the import misses its deadline, query by `run_id`, then narrow by stage and property. A compact event sequence such as `download_started`, `validation_completed`, and `commit_completed` is usually more useful than repeated free-form messages. Preserve errors that support diagnosis, but avoid logging every successful record by default. One anomalous run should not force permanent ingestion of routine payload detail.

Metrics are for aggregation. A counter for completed runs, a counter for failed runs, and a duration distribution can expose trends without indexing each record. A metric name should describe one unit and meaning; keep the outcome in a bounded label or use separate counters where that is clearer. A metric named after a dashboard panel tends to age badly because it hides what was actually measured.

## Sampling cannot replace the deadline signal

OpenTelemetry distinguishes head sampling, decided near the start of a trace, from tail sampling, decided after more of the trace is available. Tail sampling can retain traces with errors or unusual latency, but it requires infrastructure to observe and buffer trace data before deciding. Head sampling is operationally simpler, yet it can discard the one unusual execution an investigator needs.

Neither method proves that a cron run occurred. No trace exists when the scheduler never starts the job, and a sampled-away successful run is a poor source of deadline truth. Keep the heartbeat deterministic and tiny. Sample richer traces according to diagnosis value, then use logs and metrics to bridge from the missed deadline to the relevant execution evidence.

There is a real trade-off here: retaining every trace may be appropriate during a short rollout or for a very low-volume job, while probabilistic head sampling fits steady high-volume paths whose aggregate behavior matters more than any ordinary request. Tail sampling fits cases where errors or long duration are rare and valuable enough to justify the collector state. Your mileage may vary because traffic shape, investigation latency, and backend indexing behavior determine the actual cost.

## How should teams roll out two signals without doubling alert noise?

Start with the status poll and successful-completion heartbeat in observe-only mode. For several nightly cycles, record arrival times and calculate the normal completion distribution. Set the missed-run deadline from that evidence plus an explicit operating margin, rather than copying the cron schedule into an alert rule. Then route the API alert to the service owner and the missed heartbeat to the property-data owner; shared channels blur accountability.

Next, add bounded metrics and a small structured event schema. Test that a failed command does not send the success heartbeat, that a retry preserves a single logical `run_id`, and that duplicate completion pings do not create duplicate pages. Deployment validation should also confirm the external status probe traverses the intended public path. An in-process check from the same host measures something else.

Finally, review volume after one full retention window. Remove unused fields, reduce repetitive success events, and document who approves new metric labels. Keep the architecture when both request uptime and nightly freshness matter. Stick with one signal when only one of those promises exists; extra telemetry without a distinct decision is just unassigned cost.

## Further reading

- https://prometheus.io/docs/practices/naming/
- https://opentelemetry.io/docs/concepts/sampling/
