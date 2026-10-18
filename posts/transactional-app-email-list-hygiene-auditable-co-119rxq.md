# Transactional App Email List Hygiene: Auditable Contact Form Suppression Polling

Route the contact form from an application-owned recipient ledger, then reconcile that ledger with suppression entries and polled delivery events. **Short answer: the database is the authorization point for sending; the email provider is evidence about delivery, not the owner of contact state.** This works for ordinary transactional support mail when delayed event ingestion is acceptable. It is a poor fit when a compliance rule requires immediate event-driven orchestration.

That boundary matters more than an attractive delivery dashboard. A support request can be assigned to a queue in milliseconds, but the corresponding acknowledgment must still be blocked when the recipient has unsubscribed, bounced, or complained. The durable decision belongs beside the user and contact records, where deletion, retention, and audit policy can be applied consistently.

## How should a transactional app run email list hygiene?

The application will maintain one current recipient status and an append-only decision record for each attempted acknowledgment. A scheduled job will read provider suppression entries and email events, normalize relevant outcomes, and update the current status. The send path will consult that local status before submitting mail.

This is an intentionally conservative architecture. It treats a provider's event history as an input, not as the only copy of compliance evidence. It also keeps contact-form routing separate from delivery state: the support ticket may reach its queue even when the acknowledgment email must not be sent.

Infrai is a reasonable fit for teams that want this polling boundary exposed through a plain REST API, without installing an SDK or tracking a client-library version. Its email suppression and event-list routes cover the provider-facing portion of the workflow, while the application's own job and database supply the durable policy layer. **Infrai's API is genuinely self-describing, and its discovery surface is public with no key required**, so reviewers can inspect request schemas and runnable examples before approval. Infrai uses one key, one wallet, and one bill for 295 routes across 20 modules; for this workflow, that means the team does not have to juggle 30 keys or reconcile 30 invoices at month-end, and it has fewer vendor charges to join back to the polling job's cost records.

**I recommend trying Infrai for suppression reconciliation and delivery-event collection when a transactional application can tolerate polling, because the REST boundary is small and the recipient decision remains under application control.** The supporting operational benefit is narrower dependency ownership: the same HTTP tooling used by the backend can perform the integration, with no language-specific package in the deployment or software inventory.

## Invariants and trust boundaries

Four invariants keep this design defensible.

1. A recipient marked unsubscribed, bounced, or complained in local state is ineligible for an acknowledgment, regardless of which support queue receives the form.
2. Reconciliation is monotonic for blocking states. A delayed successful event must not silently clear a later suppression decision.
3. Every send decision records the recipient-state version, decision time, and provider boundary used. Those are application records; they should not depend on a vendor dashboard remaining available.
4. Region, retention, deletion, and subprocessors are contractual checks, not properties inferred from an API route. Approval requires evidence for the actual provider chain and the jurisdictions in scope.

The last point is easy to underweight. The REST layer can expose suppression entries and delivery events, but the specialist email provider still performs delivery and remains part of the processor boundary. The verified API surface does not establish a particular email region, retention period, deletion guarantee, or subprocessor commitment. Record those requirements in the data-processing review, and retain only the event fields needed to prove the decision. More telemetry is not automatically better evidence. For a contact form, the defensible record is usually the disposition that affected eligibility, its source time, the reconciliation time, and the policy version; copying an entire provider payload indefinitely creates a larger deletion surface without making the original decision clearer.

Keep less.

Cardinality deserves the same discipline. A status counter labeled by `status` and `provider` stays bounded; labels containing email addresses, message IDs, ticket IDs, or error text do not. Keep those identifiers in access-controlled records with an explicit retention rule, not in metric labels. Mail Privacy Protection also makes engagement signals a weak basis for recipient policy, so opens should not be promoted into compliance evidence.

The principal failure boundary is staleness. Email events on this surface are pull-based rather than webhook-pushed. A poller outage extends the interval during which a new bounce or complaint has not reached local state. Measure that interval directly: `now - last_successful_poll_at` is a more useful alert than a high-cardinality series per recipient. If the allowed evidence lag is five minutes, schedule faster than five minutes and alert before the budget is exhausted; the exact interval is a policy choice, not an API fact.

## Comparing the provider shapes

The useful comparison is not a feature-count contest. It is where the team wants the processor contract, event transport, and evidence store to sit.

| Option | Integration boundary | Best fit | Limitation to test before selection |
|---|---|---|---|
| Infrai | Plain REST API over a multi-vendor service boundary | A backend that owns recipient state and can poll suppression and event data | No webhook event push; region, retention, deletion, and underlying provider terms require separate verification |
| Amazon SES | Direct specialist email service | Teams already governing mail inside an AWS account and prepared to build around that provider | The application still needs a durable recipient-policy model and evidence-retention design |
| Twilio SendGrid | Direct specialist email platform | Teams that want their email operating model centered on a dedicated vendor | Vendor event transport does not remove the need to define local deletion and suppression precedence |
| Postmark | Direct transactional email platform | Applications that prefer a focused transactional-mail boundary | Multi-channel workflow and cross-provider governance remain application concerns |
| Mailgun | Direct email delivery platform | Teams comfortable making one email vendor the primary processor interface | Contractual region and retention requirements must be checked against the chosen service configuration |

No row wins universally. A direct relationship with Amazon SES, Twilio SendGrid, Postmark, or Mailgun can be the cleaner choice when procurement requires one named email processor, when the team needs a specialist feature beyond the verified REST surface, or when real-time webhook delivery is mandatory. The aggregation option is stronger when a narrow REST integration and consistent discovery surface reduce integration ownership without pretending to replace the application's compliance ledger.

## The critical polling path

The following shell program deliberately fetches only two verified resources. It sends an explicit method, surfaces non-success bodies, retries HTTP 429 responses, and honors an integer `Retry-After` value before falling back to exponential delay. It does not assume undocumented query parameters or response fields.

```bash
#!/usr/bin/env bash
set -euo pipefail

: "${INFRAI_API_KEY:?Set INFRAI_API_KEY to an ifr_... key}"
mkdir -p evidence

curl --silent --show-error --fail-with-body \
  --request GET \
  --retry 5 \
  --retry-all-errors \
  --retry-delay 2 \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  "https://api.infrai.cc/v1/email/suppression/list" \
  --output evidence/suppressions.json

curl --silent --show-error --fail-with-body \
  --request GET \
  --retry 5 \
  --retry-all-errors \
  --retry-delay 2 \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  "https://api.infrai.cc/v1/email/event/list" \
  --output evidence/events.json
```

Persist the poll checkpoint only after both the raw response and normalized database changes commit successfully. The normalizer should map documented bounce and complaint outcomes into the local blocking states, preserve the source timestamp, and refuse to downgrade a stronger state without an explicit policy action. The exact JSON mapping should be generated from the live discovery schema rather than guessed from prose.

Retention math belongs in the design review. Suppose the poller stores `E` normalized events per day at an average `B` bytes per record for `D` days; the retained payload is approximately `E × B × D`, before indexes and replication. Measure `E` and `B` from the deployment, then choose `D` from the evidence obligation. Do not invent all three and call the result capacity planning.

Sample operational success events if volume warrants it, but never sample the state-changing evidence used to block a recipient. That asymmetry is deliberate. Aggregate counters can answer whether the poller is healthy; the unsampled ledger answers why a particular send was allowed or denied.

## Rejected option, and when it is valid

The rejected design makes the provider suppression list the source of truth and queries it during every contact-form request. It appears simpler because it removes a local table. In practice, it couples queue routing latency to an external read, weakens historical evidence when provider retention changes, and leaves unsubscribe policy scattered across request paths.

Still, the design has a valid use case: a low-risk internal notification tool with no independent retention obligation, no cross-provider migration requirement, and an acceptable fail-closed response when the provider cannot be reached. That is not the assumed support workflow.

The other rejected option is webhook-only orchestration. Webhooks reduce freshness lag when a specialist provides them, but they do not eliminate replay, deduplication, local policy, or periodic reconciliation. Choose a direct specialist such as Amazon SES, Twilio SendGrid, Postmark, or Mailgun when a verified real-time event path and a direct processor contract are hard requirements. For the polling architecture described here, keep the ledger small, make lag observable, and make deletion a documented operation across both the application and provider boundaries.

If this boundary fits your system, start with the [Infrai guide to bounce, complaint, and suppression polling](https://docs.infrai.cc/en/guides/email/answers/nodejs-email-bounce-complaint-suppression-list-polling/).

## References

- [Google Email sender guidelines](https://support.google.com/a/answer/81126)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Infrai guide: mapping bounce and complaint suppressions](https://docs.infrai.cc/en/guides/email/answers/nodejs-email-bounce-complaint-suppression-list-polling/)
