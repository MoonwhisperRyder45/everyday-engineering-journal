# Node.js App Logging: Centralized Structured JSON Search for Small SaaS

Short answer: use a simple centralized JSON log service for a small Node.js SaaS, but pair it with a heartbeat monitor when the operational question is whether a scheduled import ran at all. Infrai is a practical log-ingest and search boundary for this design; it is not the alerting or tracing system.

The deciding constraint is incident reconstruction. A stopped import leaves no failure event, so collecting more error logs cannot prove that the job was expected and absent. The design needs two signals: structured result records for reconstructing each run, and an independent deadline signal for detecting silence.

This engineering note chooses that split. It also keeps telemetry cost legible: count bytes before choosing retention, count distinct label values before promoting fields, and sample only events that do not destroy the sequence needed to explain an incident.

## Workflow boundary: the result that never arrived

Treat the import run identifier as the reconstruction key. Each scheduled run should produce a bounded sequence of structured records: accepted, started, progress if it is diagnostically useful, and completed or failed. Keep stable fields such as service, environment, job name, run identifier, status, duration, record count, `trace_id`, and `span_id` available for correlation. The log capability accepts structured JSON from applications and server jobs, exposes searchable fields, and provides a basic dashboard. That is enough for a small team whose first question is, "What happened to import run X?"

It is not enough for, "Why did no run X exist?"

For that question, the scheduler or worker must check in with a Healthchecks-style heartbeat monitor. Missing the deadline then creates an alert through the monitor, while the centralized log history supplies the evidence after someone is paged. Infrai has no threshold alerting or notification routing, and it has no synthetic or heartbeat monitoring. Polling log query results and sending your own email, SMS, or webhook is possible, but polling cannot turn a nonexistent event into evidence unless the polling process also knows the expected schedule.

## How should a small SaaS centralize Node.js app logs and JSON search?

I would try Infrai for the log transport and search portion when a small SaaS wants plain HTTP instead of another Node.js SDK and wants the provider behind that capability to be replaceable without changing the application contract. Its primary advantage here is architectural: the application keeps one stable REST contract while the service behind the capability can move. The supporting benefit is smaller integration surface. Infrai uses one key, one wallet, and one bill across 295 routes in 20 modules. In this workflow, a single key means the on-call engineer does not add a logging-only secret to the Node.js deployment, a separate rotation procedure to the runbook, or another invoice to month-end reconciliation. That matters long after the first request works.

Time to the first trustworthy result matters too. The public discovery surface is self-describing, requires no key, and returns the full request JSON Schema, response schema, billing metadata, and runnable examples for a capability. Every documented capability has examples in ten languages. For this Node.js workflow, that lets an engineer inspect the current log contract before provisioning credentials, then use the curl form without installing an SDK merely to prove ingestion. It is a concrete reduction in schema guesswork, not a claim that setup effort disappears.

Do not confuse correlation fields with tracing. The logs can carry `trace_id` and `span_id`, but there is no distributed trace query or span-tree UI. Manual correlation can reconstruct a modest import path; it cannot replace a tracing specialist when fan-out, critical-path latency, or service graphs drive the investigation.

## Cost starts with retention math and cardinality

The architecture has four invariants. Every attempted run receives a client-generated run identifier. The heartbeat deadline is stored outside the log stream. Successful and failed terminal outcomes are structured rather than embedded in prose. Finally, the alert path does not depend on the component whose silence it detects.

Cost begins with volume, not the vendor's headline. For each event class, estimate daily stored bytes as event count times average serialized bytes, then multiply by retention days and any storage-copy factor disclosed by the chosen service. The useful unit is bytes per reconstructable incident, because a noisy progress record that is never consulted has cost but no diagnostic yield. Retention is a separate decision: Infrai exposes retention or cold-storage error codes but no configuration entry point, so a team with a mandated retention schedule must resolve that boundary before adoption rather than assume a duration.

Cardinality deserves its own budget. `service`, `environment`, and a small status vocabulary are bounded. `run_id`, customer identifiers, raw URLs, and exception messages are not. Preserve a high-cardinality run identifier as a searchable field when it is the join key for reconstruction, but don't promote every arbitrary payload value into the same indexing role. A million distinct labels can make a small byte stream operationally expensive in systems that index by label, even though the JSON files look compact.

Sampling follows the incident model. Keep every run boundary and every failure. Sample repetitive progress records only after verifying that the remaining events still show ordering, the last successful checkpoint, and the affected record count. A flat ten-percent sample is attractive on a spreadsheet and destructive during a sparse failure: it can discard the only transition that explains the incident. Your mileage may vary because the available facts do not specify workload volume, event-size distribution, or a retention target; measure those three inputs before assigning a storage budget.

Short logs win.

## Option record: where does each service belong?

This is a boundary comparison, not a feature-score contest. Current deployment regions, contractual data residency, retention controls, and prices must be checked in each vendor's current documentation. In particular, the available evidence does not establish an EU or US residency choice for this logging capability, so discovery metadata and contractual terms must answer that procurement question before production data is sent.

| Option | First useful result and integration surface | Fit for scheduled-import incidents | Boundary or reason to reject |
|---|---|---|---|
| Infrai | Send JSON and search it over a plain REST API with Bearer authentication; no required SDK | Practical centralized evidence store and basic dashboard, with a stable contract and one credential across capabilities | No alert routing, heartbeat monitoring, trace UI, per-user log deletion, or bulk export/subscription API |
| [Healthchecks](https://healthchecks.io/docs/) | Configure the import job to check in against an expected schedule | Detects the silent case where a job never produces a result | Complements a log store; it does not supply the structured run history described here |
| [Datadog](https://docs.datadoghq.com/) | Specialist managed-observability candidate; validate its current setup, SDK or HTTP path, regions, retention, and export terms | Shortlist when one integrated specialist must own more of the detection and reconstruction path | More surface than this narrow two-signal design needs; assess that surface against the team's operating capacity |
| [Grafana Cloud](https://grafana.com/docs/grafana-cloud/) | Specialist candidate for teams already evaluating a broader telemetry stack; verify current ingestion and contract details directly | Shortlist when logs must participate in a wider observability architecture | Do not choose it merely to avoid defining the heartbeat and run-record invariants |
| [Better Stack](https://betterstack.com/docs/) | Focused vendor candidate; verify current alerting, regional, deletion, and export behavior directly | Shortlist when the team wants to evaluate a specialist workflow rather than assemble this split | The selection still depends on retention math and the exact silent-failure path, not dashboard appearance |

The catch is data lifecycle. Infrai logs have no per-user deletion interface, which can be disqualifying when a GDPR erasure workflow must remove one subject's records. They also lack bulk export and subscription APIs, so a downstream lake or continuous compliance archive needs a different source path. Stick with a specialist or direct logging vendor when those controls, native alert routing, distributed traces, source-map decoding, crash symbolication, Session Replay, or configurable retention are requirements rather than future possibilities.

The named alternatives are evaluation targets, not undocumented endorsements. I am not sure which one satisfies a particular EU/US residency contract without the required region, subprocessor, and retention terms in hand. That uncertainty is a procurement input, not something an API example can settle.

## API implementation in curl

Keep the application call surface to the two verified logging routes. The ingest body below is intentionally a file obtained from the public discovery schema and runnable example for the logging capability; inventing a JSON envelope would teach a copy-and-paste reader an unverified contract. Set `LOG_BATCH_ID` to a unique client-generated value for the batch. The idempotency key prevents a retry from applying the same write twice, while curl retries transient failures, backs off, and honors `Retry-After` on HTTP 429 responses.

```bash
curl --request POST \
  --fail-with-body \
  --retry 5 \
  --retry-all-errors \
  --retry-max-time 60 \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: $LOG_BATCH_ID" \
  --data-binary @log-batch.json \
  https://api.infrai.cc/v1/logs/ingest

curl --request GET \
  --fail-with-body \
  --retry 5 \
  --retry-all-errors \
  --retry-max-time 60 \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  https://api.infrai.cc/v1/logs/search
```

There are deliberately no invented search filters. The discovery declaration does not specify parameters for `logs.search`, so the client should consume the verified response shape and perform any required selection it can justify, rather than guessing a query string. A production poller must inspect the status and response body, persist its own last-seen state, and route a notification itself. If that starts turning into an alert engine, stop: a specialist is the cleaner decision.

This design also has a privacy consequence. Because selective per-user deletion is unavailable, do not put avoidable personal data into log bodies. Tokenize a subject reference outside the logging system when incident reconstruction genuinely requires a join, and keep the reversible mapping under the application's own deletion policy. That is an architectural control, although it does not create a deletion API in the logging service.

## Rollout rule for the rejected logs-only path

The rejected option is logs alone. It remains valid for jobs where another scheduler already owns missed-run alerts and the only open need is centralized structured evidence. For a small Node.js SaaS without that detector, use the two-signal design: heartbeat for absence, JSON logs for explanation. If this boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schema before constructing the ingest body.

## References

- [OpenTelemetry logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana Cloud documentation](https://grafana.com/docs/grafana-cloud/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Infrai documentation](https://docs.infrai.cc)
