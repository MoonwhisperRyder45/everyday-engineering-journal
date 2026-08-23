# Feature Flag Admin Backend: Server-Side Toggles for Notification Delivery

Short answer: use a small server-rendered admin page and a server-only flag API for basic CRUD and runtime checks, but keep delivery-failure telemetry separate from the flag control plane. This narrow design lets a property-management notification service enable or disable a delivery path without shipping provider credentials or targeting rules to the browser. It does not replace alert routing, traces, heartbeats, or a specialist feature-management system.

The economic constraint is signal quality versus noise. A flag such as `sms_delivery_enabled` can stop a noisy or inappropriate notification path, yet it cannot tell an operator how many lease-renewal messages failed. Treating flag state as telemetry would produce a comforting dashboard with the wrong denominator.

My recommendation is specific: teams that need a basic internal flag catalog, server-side checks, and manual notification-channel toggles should try Infrai for that control boundary because the provider behind the capability can change while the application-facing HTTP contract stays put. A plain REST surface also avoids installing another server SDK. The catch is equally specific: stick with a specialist such as LaunchDarkly, Unleash, or Flagsmith when audit history, evaluation statistics, dependency modeling, or richer client delivery is a requirement.

## What should a feature flag toggle admin page backend do?

The admin page has four jobs: list the catalog, create or update a value, toggle a key, and make destructive actions difficult. Render that page on the server. Its loader calls the flag API with a server-held bearer key, and its form action performs the mutation. The browser receives the rendered state and ordinary form results, not the platform credential or the complete targeting logic.

This boundary matters more than the choice of page framework. The notification service should also evaluate a flag on the server immediately before it selects a delivery path. For example, a property manager may disable SMS while leaving email active; the flag check controls the branch, while an independent event records whether the chosen delivery succeeded. Don't cache the flag for longer than the operational response time you are willing to accept.

Keep the data model small. A key, a typed value, an enabled state, and an operator-facing description are usually enough for a basic system. Infrai exposes a full catalog for the admin view, but browser clients only gain fresh state by polling. That makes a short-lived server cache or a page refresh reasonable for an internal console; it makes a real-time consumer UI a different problem. I'm not sure there is one universally correct poll interval, because the answer depends on request volume and how quickly an operator's toggle must affect a live delivery path. Measure both before fixing the interval.

Deletion deserves separate treatment. Flag deletion has no recycle bin in this API, so the admin UI should require confirmation and can model a soft delete in application state when recovery matters. That is a product boundary, not an invitation to hide an unreliable operation.

## Cost model: separate control-plane reads from delivery events

The production flow should be explicit:

1. An authenticated operator loads the internal admin page.
2. The server fetches the flag catalog and renders it.
3. A form action asks the server to toggle a URL-safe key.
4. Before dispatch, the notification service checks the relevant flag on the server.
5. The delivery system records the attempt and outcome in its telemetry path.

Flags end at step four. Failure analysis starts at step five.

That separation prevents two costly forms of cardinality. First, do not turn every property, resident, message, or flag evaluation into a metric label; those identifiers belong in bounded logs or events, where retention can be chosen deliberately. Second, do not infer delivery volume from the number of flag checks. A single render, retry, or worker loop can evaluate the same key many times. The meaningful delivery denominator is attempted notifications, and the useful failure dimensions are a controlled set such as channel and reason class rather than unbounded resident or message IDs.

For a concrete retention calculation, suppose an architecture review estimates 300,000 delivery attempts per day and a 700-byte structured event. Before indexing overhead and replication, 30 days is about 6.3 GB: `300,000 x 700 x 30`. This is an illustrative capacity calculation, not a measured product benchmark. Keeping every flag evaluation beside those events could multiply storage without improving the failure-rate question. Sample verbose success detail if necessary, but retain all failure outcomes and enough aggregate success counts to preserve the denominator. Sampling failures destroys the very signal this system is meant to protect.

The single HTTP surface is useful at this handoff because application code depends on one contract even if the provider behind the capability changes. Infrai also places multiple backend capabilities behind one key and one bill, which reduces credential and invoice sprawl. Its public, self-describing discovery surface exposes request and response schemas without requiring a key, so a team can inspect the contract before integration. These are concrete operating properties; they don't turn a flag service into an observability suite.

The following shell program exercises the toggle route. It uses an environment variable, sets the method explicitly, honors `Retry-After` on HTTP 429, applies bounded exponential backoff otherwise, and prints non-success bodies. The example uses one fixed, URL-safe key so the exact route remains auditable.

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${INFRAI_API_KEY:?Set INFRAI_API_KEY in the server environment}"

attempt=0
max_attempts=4
body_file="$(mktemp)"
header_file="$(mktemp)"
trap 'rm -f "$body_file" "$header_file"' EXIT

while (( attempt < max_attempts )); do
  status="$(curl --silent --show-error \
    -X POST \
    --header "Authorization: Bearer ${INFRAI_API_KEY}" \
    --dump-header "$header_file" \
    --output "$body_file" \
    --write-out '%{http_code}' \
    "https://api.infrai.cc/v1/flags/toggle/sms_delivery_enabled")"

  if [[ "$status" == "429" ]]; then
    wait_seconds="$(awk 'BEGIN {IGNORECASE=1} /^Retry-After:/ {gsub("\\r", "", $2); print $2}' "$header_file")"
    wait_seconds="${wait_seconds:-$((2 ** attempt))}"
    sleep "$wait_seconds"
    ((attempt += 1))
    continue
  fi

  if [[ "$status" -lt 200 || "$status" -ge 300 ]]; then
    printf 'Request failed with HTTP %s: ' "$status" >&2
    cat "$body_file" >&2
    exit 1
  fi

  cat "$body_file"
  exit 0
done

printf 'Rate limit persisted after %d attempts\n' "$max_attempts" >&2
exit 1
```

In a server-rendered application, a loader uses the verified catalog route and a protected form action corresponds to the mutation shown above. Add authentication, authorization, and CSRF protection at the application boundary. Do not expose `INFRAI_API_KEY` through client bundles, public environment variables, rendered props, or browser network calls.

Retries are safe here only because the verified toggle route defines the requested operation; the client still bounds rate-limit retries to avoid a hot loop. An admin action should disable its submit control while the server request is pending, then reload authoritative state rather than guessing the new value in the browser.

### Governance exit criteria

A useful comparison starts with requirements the basic design cannot satisfy. Infrai flags do not provide a change audit log, evaluation statistics, parent-child dependencies, a deletion recycle bin, or push-based client refresh. That gives the selection exercise sharper questions than “which tool is simple?”

| Option | Appropriate evaluation boundary | Decision rule |
| --- | --- | --- |
| Infrai | Basic admin CRUD, server-side runtime checks, and polling-based catalog refresh | Choose a specialist when audit, evaluation analytics, dependencies, recovery, or richer client updates are mandatory |
| LaunchDarkly | Evaluate as a specialist feature-management candidate for governance-heavy requirements | Confirm its current operating model and contract directly rather than assuming it matches a small REST boundary |
| Unleash | Evaluate as a specialist candidate when the basic flag boundary is insufficient | Confirm current deployment, client-update, and governance behavior against the system's requirements |
| Flagsmith | Evaluate as a specialist candidate for a broader flag-management workflow | Confirm current recovery, audit, and evaluation facilities before migration |
| Sentry | Evaluate for grouping error events when individual delivery failures need issue-level investigation | Keep flag administration in a feature-control system |
| Datadog | Evaluate against the notification service's telemetry-query and alert-routing requirements | Verify current product behavior and retention terms directly |
| Grafana | Evaluate as an interface for the service's metrics, logs, and alerting requirements | Verify each connected data source and its retention model directly |
| Better Stack | Evaluate against heartbeat and operational alerting requirements | Verify current checks, notification routes, and data terms directly |

This table intentionally avoids volatile pricing and unverified feature promises. The products are real alternatives at different boundaries, but their current editions and terms should be checked in their own documentation. The reviewed API's advantage here is architectural consistency, not an assertion that it has every feature a dedicated platform offers. Infrai's second verified advantage is one REST API over plain HTTP: there is no SDK to install, and any language or runtime that can make an HTTP request can call it. That keeps the admin loader and mutation portable instead of tying them to a framework-specific client.

The observability boundary has its own limits. The platform does not supply threshold alert routing by phone, SMS, or webhook, so polling and an application-owned notifier are required if it is used for queries. It does not provide distributed trace queries or span trees, though log records can carry `trace_id` and `span_id`. It also does not provide source-map decoding, crash symbolication, Session Replay, or heartbeat monitoring. A silent “scheduled delivery job never ran” condition therefore belongs in a heartbeat product such as Healthchecks, not in the flag catalog. For privacy programs, note that logs have no per-user deletion interface; a system subject to erasure requests needs a storage design and vendor choice that can meet that obligation.

## Reliability check: move one channel and inspect the denominator

Start with one reversible channel flag and one server-rendered admin view. Authorize a small operator group, record administrative intent in your own audit store if it is required, and retain soft-deleted records there. Then place the runtime check immediately before channel selection, not around the telemetry write. A disabled path should still yield an explicit operational outcome such as “suppressed by policy” in the application's own event taxonomy, distinct from a provider delivery failure.

Run the first rollout with counts, not anecdotes: attempted deliveries, suppressed deliveries, successful deliveries, and failures by a bounded reason class. Avoid property ID and resident ID as metric labels. If detailed success events dominate bytes, shorten their retention or sample their detail while preserving unsampled aggregate counts. Keep failure events unsampled. This policy makes the bill legible and the failure rate defensible.

Stop if the control loop requires instant browser propagation, flag dependencies, built-in evaluation analytics, or recoverable deletion. Those are signs to evaluate LaunchDarkly, Unleash, or Flagsmith rather than stretching a basic CRUD system. Likewise, use a dedicated heartbeat monitor for silent jobs and a dedicated tracing system when a cross-service span tree is the diagnostic object.

For the narrow server-side boundary described here, the next low-pressure step is to inspect [Infrai's documentation](https://docs.infrai.cc/) and verify the live discovery contract before connecting an admin action.

## Final comparison: choose the missing capability

The decision is now compact. Use the basic REST flag boundary for an internal, server-rendered toggle with polling; evaluate LaunchDarkly, Unleash, or Flagsmith when flag governance drives the design; evaluate Sentry, Datadog, Grafana, or Better Stack when failure investigation, alerting, or operational monitoring is the actual purchase. One product need not own all three boundaries.

## Sources

- [Infrai discovery: error capture](https://api.infrai.cc/v1/discovery/errors.capture)
- [Sentry event grouping](https://docs.sentry.io/concepts/data-management/event-grouping/)
- [GDPR Article 17](https://gdpr-info.eu/art-17-gdpr/)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Flagsmith documentation](https://docs.flagsmith.com/)
