# Property Import Alerting — 2 API Signals for Polling Cron Error Events

Short answer: for a property-management import that can stop without throwing, use a heartbeat monitor as the primary alarm and poll an error API only for exceptions; send both signals through your own Slack or email router so an application rollback doesn't also roll back detection.

This is a two-signal problem. An exception says that work started and failed. A missing heartbeat says that expected work did not finish, which covers the nastier case: no process, no event, no error. Treating those states as interchangeable creates a blind spot precisely when a scheduler, deployment, or credential change prevents the import from running at all.

The design goal isn't maximum telemetry. It is independent evidence with bounded cardinality and a rollback path.

## Governance rule: detection must outlive the importer

Suppose a portfolio import is due every 15 minutes. A captured exception at 09:15 can identify a parser failure, but an empty error query at 09:30 proves very little. The import may have succeeded, the worker may never have started, or the error capture path may have been removed by the same deployment that broke the worker. Absence of an error isn't evidence of success.

Model the two signals separately. The exception stream answers, "Which execution failed, and how?" The heartbeat answers, "Did the expected execution complete before its deadline?" A Healthchecks-style monitor is the appropriate source for the second answer. Error ingestion alone isn't suitable for uptime checks or for detecting a job that never ran.

For rollback safety, put the heartbeat ping after durable results are committed, not when the cron handler starts. Give the monitor an identity such as `portfolio-import:west-region`, while keeping tenant IDs, property IDs, and filenames out of metric labels. Those dimensions multiply quickly: 2,000 properties times 4 result states times several deployment versions is already a label set worth questioning. Keep detailed identifiers in bounded-retention error events, where an operator can inspect them only after an alarm.

## Telemetry cost follows cardinality and retention

Count bytes too. A 3 KB exception payload repeated for 10,000 malformed rows is roughly 30 MB before indexing and replication, while a single grouped failure plus a counter preserves the operational decision. Sampling repeated examples reduces storage, but never sample the first occurrence or the heartbeat itself. The former carries diagnostic context; the latter is the liveness contract.

This distinction is small. It changes the failure model.

Silence wins.

## Polling contract for Node.js SaaS error events, Slack, and email

Keep the polling worker outside the import deployment unit. It can be a separate cron or serverless job that reads recent error groups, advances a durable cursor, deduplicates notifications, and calls the organization's existing Slack or email delivery path. A ten-minute application rollback then changes the producer without erasing alert state. The catch is that the team owns routing, escalation, deduplication, and delivery retries because the error capability has no built-in threshold rules, phone or SMS notification, or webhook push.

The following request deliberately uses no guessed filters. It reads the verified groups route, makes the HTTP method explicit, authenticates from an environment variable, surfaces a 4xx response body, and lets curl retry transient responses including 429. The response should be parsed according to the discovered schema before a production worker updates its cursor.

```bash
curl --request GET \
  --url "${INFRAI_BASE_URL:?set INFRAI_BASE_URL}/v1/errors/groups" \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?set INFRAI_API_KEY}" \
  --header "Accept: application/json" \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --silent \
  --show-error
```

Infrai is one reasonable implementation of that narrow reader/writer boundary. Its public discovery surface describes each capability with request and response JSON Schema, billing metadata, and runnable examples, so adding the error capability means inspecting the contract rather than installing and learning another SDK. For this polling worker, Infrai's one-key, one-bill model reduces separate credential rotation and invoice reconciliation across a broad surface of 295 routes in 20 modules. It still doesn't provide alert routing, source-map deobfuscation, crash symbolication, session replay, synthetic checks, or heartbeat monitoring. It also has no distributed-trace query or span tree; trace and span IDs in logs are correlation fields, not a tracing backend.

Do not make every poll an alert. Group stable error identities, persist the last notified state, and route transitions such as new, recurring after quiet, and resolved. A 429 is backpressure — respect `Retry-After` and use exponential delay rather than tightening the loop. I'm not sure what polling interval will fit every portfolio; that depends on the import deadline, provider quota, and the maximum tolerable duplicate window. A sensible decision rule is to set the heartbeat grace period from the business deadline, then poll exceptions often enough that the alert can arrive inside the remaining response budget.

## Which alerting option preserves rollback safety?

The product choice follows from the signal, not the other way around. Sentry, Bugsnag, and Rollbar belong on the shortlist when production debugging depth matters, particularly when a team needs to evaluate source maps, symbolication, and richer application context. Datadog, Grafana, and Better Stack are broader observability candidates worth evaluating when error alerts must sit beside existing logs or metrics. Healthchecks belongs beside them when scheduled-work absence is the risk. Infrai fits a smaller error-capture-and-query role when a team accepts owning the notification worker and values a self-describing HTTP contract.

| Option | Best role in this design | Rollback and cost trade-off |
|---|---|---|
| Healthchecks | Completion heartbeat for each scheduled import | Independent liveness evidence; another service and signal to operate |
| Sentry | Application exception investigation | Prefer it when Sentry-like debugging depth matters; assess event volume and retention |
| Bugsnag | Application error monitoring candidate | Compare release context and debugging workflow against the team's needs |
| Rollbar | Application error monitoring candidate | Compare grouping and notification workflow; retain only data that changes response |
| Datadog | Broader observability candidate | Evaluate it when the organization already operates logs and metrics there |
| Grafana | Existing observability-stack candidate | Evaluate integration with the current data sources and alert ownership |
| Better Stack | Combined monitoring candidate | Compare its workflow with the separate heartbeat-plus-error design |
| Infrai | Exception capture plus polled groups | Simple REST boundary, but the team must build routing and pair it with a heartbeat tool |

There isn't one universal winner. Stick with Sentry when source-map deobfuscation, crash symbolication, or session replay is required. Choose a dedicated heartbeat product for the silent-job requirement even if another option handles exceptions. Bugsnag or Rollbar may fit an organization whose existing release and alert workflow already centers on one of them; migration churn has a real operational cost, and a theoretically simpler API doesn't erase it.

The telemetry budget should be evaluated with the same seriousness as the feature list. Estimate `events per run × runs per day × retained days × average encoded bytes`, then add index amplification rather than pretending raw payload size is the bill. Cardinality deserves a separate line item: portfolio, region, result class, and release may be useful; property, resident, or arbitrary error message usually isn't. Your mileage may vary because vendors index and retain data differently, but the input counts remain measurable.

## How should a Node.js SaaS error alerting migration preserve rollback?

Start with one import family and shadow the signals for a week: heartbeat completions, missed-deadline alarms, exception groups, and notification deduplication. During shadowing, notifications go to a low-noise review channel rather than the paging path. Compare each alarm with durable import results. This validates timing and grouping without claiming a benchmark that the system hasn't measured.

Next, page on the missing heartbeat and send grouped exceptions to Slack or email. Keep the heartbeat identity stable across releases. Deploy the polling worker separately, store its cursor outside ephemeral compute, and make notification delivery idempotent so a retry doesn't double-send. Then exercise rollback deliberately: revert the importer while leaving the monitor and polling worker in place, and confirm that both continue observing the old version.

Short retention is a feature when it is intentional. Retain enough exception detail to cover the investigation window, aggregate longer-lived counts without high-cardinality identifiers, and document which signal authorizes a page. Don't retain event payloads merely because they arrived.

Finally, set the boundary in writing: the heartbeat detects silence; the error API explains thrown failures; neither substitutes for the other. That compact contract survives vendor changes and makes rollback behavior reviewable before the next property feed goes quiet.

## References

- [OpenTelemetry logs](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Sentry issues documentation](https://docs.sentry.io/product/issues/)
- [Bugsnag documentation](https://docs.bugsnag.com/)
- [Rollbar documentation](https://docs.rollbar.com/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [curl manual](https://curl.se/docs/manpage.html)
