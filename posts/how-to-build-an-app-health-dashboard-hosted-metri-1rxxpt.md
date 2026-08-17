# How to Build an App Health Dashboard — Hosted Metrics Evidence Costs

Short answer: choose a hosted health API that accepts standard metric shapes, keeps region and retention explicit, and lets every stored byte be attributed to a service and environment; for a small fintech startup, a modest evidence set with bounded labels is more useful than a large dashboard that cannot reconstruct a customer incident.

The governing trade-off is evidence versus cardinality. A Node.js process can expose or push hundreds of measurements without Prometheus, but collecting everything is not the same as retaining what an incident reviewer needs. Start with four questions: which customer-facing failure must be reconstructed, which measurements establish its sequence, where may those measurements reside, and who pays for their retention? The dashboard comes later.

## What should a small startup require from a hosted app health API?

Require an ingestion contract, not a particular collector. The contract should preserve a metric name, timestamp, numeric value, unit, and a deliberately small attribute set. OpenTelemetry defines a metric as a measurement captured at runtime and describes metric data points as carrying attributes, start and end timestamps, and values. That gives a useful neutral model even when the startup does not run Prometheus.

For an EU-US fintech service, region must be a deployment decision rather than a label casually attached after ingestion. Send EU workloads to the approved EU endpoint and US workloads to the approved US endpoint, then keep a single logical schema across both. A dashboard may query the two stores and combine aggregates, but raw evidence should not cross a boundary merely to make one graph easier to draw. The exact residency and transfer requirements depend on contracts and legal advice; the architecture should make the chosen policy enforceable instead of assuming it.

The minimum API evaluation is quite plain:

| Test | Evidence to request | Rejection condition |
| --- | --- | --- |
| Ingestion | Documented HTTPS request, authentication, timestamp behavior | Hidden agent dependency or ambiguous timestamp handling |
| Attribution | Query or export by service, environment, and region | Spend can be seen only as one account-wide total |
| Retention | Configurable retention or documented deletion lifecycle | Retention cannot match the incident-review window |
| Portability | Export that preserves metric names and attributes | Data leaves only as screenshots or flattened reports |
| Failure handling | Defined success and client-error responses | The sender cannot distinguish accepted data from rejected input |

Keep the first probe copyable. This generic request uses a pseudonymous endpoint and an OpenTelemetry-shaped payload; the actual field encoding must be matched to the selected service's documented ingestion contract.

```bash
curl --fail-with-body --silent --show-error \
  --request POST \
  --header "Authorization: Bearer ${TELEMETRY_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{
    "resourceMetrics": [{
      "resource": {"attributes": [
        {"key": "service.name", "value": {"stringValue": "payments-api"}},
        {"key": "deployment.environment", "value": {"stringValue": "production"}},
        {"key": "region", "value": {"stringValue": "eu"}}
      ]},
      "scopeMetrics": [{"metrics": [{
        "name": "payment_authorization_duration",
        "unit": "ms",
        "gauge": {"dataPoints": [{
          "timeUnixNano": "1786838400000000000",
          "asDouble": 184.0,
          "attributes": [{"key": "outcome", "value": {"stringValue": "approved"}}]
        }]}
      }]}]
    }]
  }' \
  https://telemetry.example/metrics
```

This is only a contract test. It doesn't prove that the resulting dashboard is financially or operationally sound.

## Derive the evidence set before drawing the dashboard

Take one concrete incident: a customer says a card authorization stalled, then appeared twice. The health view does not need the customer's email, card number, or a transaction identifier as a metric label. It needs enough aggregate evidence to determine whether authorization latency rose, whether retry volume changed, which release was active, and whether one region or payment path was affected. Detailed per-transaction reconstruction belongs in an access-controlled evidence system with its own retention and erasure policy, not in a high-cardinality metric series.

That separation matters because labels multiply. Suppose the initial dashboard records 8 measurements for 3 services, 2 environments, 2 regions, 4 outcomes, and 5 payment paths. The upper bound is `8 x 3 x 2 x 2 x 4 x 5 = 1,920` series before replicas or releases enter the model. Add a `transaction_id` with 100,000 daily values and the theoretical bound becomes 192 million series. The exact active count will be lower when combinations do not occur, but that observation does not rescue the schema: an unbounded customer or transaction label makes cost attribution and query behavior depend on traffic rather than on an intentional budget.

No customer IDs.

Use a low-cardinality release identifier only if the incident question genuinely requires it, and define when old release values disappear. Keep path labels templated, such as `/payments/{id}`, rather than recording concrete URLs. Record outcomes from a finite vocabulary. These decisions preserve the dimensions that explain a service change while excluding values that grow with customers, requests, or money movements.

The incident evidence set can remain small: request rate, error count, duration distribution, retry count, queue age, dependency availability, and deployment markers. A gauge may represent a current value, while sums and histograms answer different questions over an interval; the OpenTelemetry metrics model is useful here because it distinguishes metric instrument and aggregation concepts without requiring one storage vendor. Pick the type that matches the question. A duration average alone, for example, can conceal a slow tail, while a distribution retains evidence about that shape.

There is also an erasure boundary. GDPR Article 17 describes a right to erasure under specified conditions. Avoiding direct identifiers in operational metrics reduces the chance that a metric store becomes another place where a customer record must be found and erased, but it does not by itself establish compliance. I'm not sure any generic retention number can settle that question for every fintech; counsel, contracts, and the purpose of processing determine the rule. What engineering can do is inventory attributes, document purpose and retention, and test deletion paths for systems that do hold personal data.

## Put retention math beside every sampling decision

Retention is a multiplication problem, so write the assumptions down before comparing hosted APIs. Consider an illustrative, uncompressed planning case: 1,920 active series, one point every 60 seconds, and 48 bytes per stored point after accounting for the value, timestamp, and an estimated share of indexing overhead. The estimate is not a vendor claim; serialization, compression, index structure, and billing units will change it.

`1,920 x 1,440 x 48` gives about 132.7 MB per day. Thirty days gives about 3.98 GB, before replication, query scans, export, or network charges. Measure the actual encoded and billed volume during a trial because your mileage may vary. The useful result is not the provisional gigabyte figure. It is the attribution equation: series by service and environment, multiplied by collection frequency, bytes per point, and retained days.

Sampling changes the evidence. A 10% random sample of requests can reduce event volume, yet it may discard the only attempt associated with a rare payment failure. Aggregated metrics are different: the application or collector can count all requests during an interval and export one aggregate rather than sampling individual successes. For incident reconstruction, retain complete low-cardinality counters and duration distributions for the short review window; sample verbose diagnostic events according to risk, and never imply that a sampled event stream is a complete transaction ledger.

This is the catch: a hosted metrics API is not suitable when the investigation requires immutable, per-transaction audit evidence. Use a dedicated audit store for that evidence, with access controls and lifecycle rules suited to financial records, while metrics describe system behavior. Conversely, a self-managed metrics stack may be the better choice when the team needs control over storage topology, already has operational staff, or cannot accept the hosted service's residency and export boundaries. Hosted ingestion removes infrastructure work, but it transfers dependency, quota, and data-governance questions into a contract.

Cost attribution should survive that choice. Allocate ingestion and retention by `service.name`, deployment environment, and region; then review the top series creators. Don't allocate by customer label. If shared platform measurements cannot be assigned directly, state the allocation rule, such as proportional request count, and keep shared cost visible rather than quietly spreading it across teams. A dashboard with a total spend line but no accountable dimensions is an invoice viewer, not a control system.

## Compare contracts with a bounded trial

Run the same seven-day trial against every viable implementation, using synthetic or non-personal data and the same metric schema. Record accepted points, rejected client requests, encoded bytes, active series, query latency at the required lookback, export fidelity, deletion behavior, and region routing. Seven days will not predict every seasonal peak, but it will expose schema mistakes and provide measured inputs for the retention equation. Extend the trial when weekly traffic is not representative.

Do not rank candidates on dashboard polish first. Weight the contract against the job: incident evidence completeness, cost attribution, residency, retention control, portability, and sender behavior during throttling or network loss. The simple option is the one whose failure semantics and bill can be explained, not necessarily the one with the shortest signup flow.

A sender also needs a bounded local queue, retry rules for retryable failures, and a policy for overload. Those controls should be tested by temporarily denying network access in a staging environment and verifying that application request handling stays independent of telemetry delivery. Observe queue depth and dropped telemetry as operational signals, but cap both memory and disk use. Health reporting must not become the reason the payment path is unhealthy.

## Roll out without losing the incident trail

Begin with one production service and a shadow dashboard. For one retention window, compare the new aggregates with the existing operational record, verify region routing, and rehearse a reconstruction using a synthetic incident. Then freeze the approved attribute vocabulary, add a cardinality budget to code review, and alert on unexpected series growth.

Move the remaining services in small groups. Keep export validation and cost attribution in the acceptance checklist, and remove the old path only after the new evidence set has covered a complete review window. This rollout is intentionally boring — a health dashboard earns trust by preserving the right evidence under stress, not by collecting the most data.

## References

- https://opentelemetry.io/docs/concepts/signals/metrics/
- https://gdpr-info.eu/art-17-gdpr/
