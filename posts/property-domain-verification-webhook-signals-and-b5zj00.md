# Property Domain Verification: Webhook Signals and Polling for Mail Authentication

A property-management onboarding flow should treat mail authentication as an evidence problem, not a button state. Record the intended SPF, DKIM, and DMARC configuration, accept asynchronous observations as evidence, and derive completion only when the observed DNS state matches the intended state. Use a webhook to reduce latency, then use bounded polling to repair missed or stale notifications.

The important boundary is this: a notification says that something changed; it does not prove that the current records satisfy the onboarding contract. I keep the event small and the evidence durable. That choice makes the system easier to audit, and it keeps telemetry from becoming an unpriced second database.

One version. One verdict.

## Architecture decision record: what must remain true?

For a property manager, an incomplete mail setup has a concrete consequence: lease notices, maintenance updates, and payment messages may be sent before the sending domain is aligned with its policy. The onboarding record therefore needs four invariants. The requested domain is normalized before comparison. The intended record set is versioned. Each observation is tied to the version it evaluated. A completion transition is monotonic for one intent version, but a new edit reopens verification.

The failure boundary sits between intent and observation. A webhook can arrive twice, late, or not at all. DNS can expose an old answer during propagation. A polling result can be current but evaluated against yesterday's requested DKIM selector. None of those cases should silently turn into “verified.”

| Option | What it tells the workflow | Failure boundary | Appropriate role |
| --- | --- | --- | --- |
| Webhook only | An external observer reported a change | Delivery loss, duplication, or stale event | Fast path |
| Polling only | A scheduled read produced evidence | Delay, repeated reads, and unnecessary retention | Backstop or simple low-volume flow |
| Webhook plus bounded polling | A change signal plus a fresh read | Requires idempotency and a stopping rule | Default for multi-step onboarding |

I choose the third option when completion affects customer-visible mail. The choice is not about making every check immediate. It is about separating notification latency from correctness.

## Should domain verification use webhook signals or bounded polling?

No green badge yet.

The verifier should model the three authentication mechanisms as separate predicates. SPF needs the expected policy record. DKIM needs the expected selector and public key record. DMARC needs a policy record at the `_dmarc` label; RFC 7489 defines the record's policy and reporting vocabulary. A green state means all predicates for the current intent version passed, not that one convenient lookup returned a value.

The intent ledger can be compact: domain, selector, expected normalized values, intent version, last evidence timestamp, and state. Do not store every raw response forever. Store the normalized result needed to explain the transition, the source timestamp, and a short reason for rejection. Raw payloads belong behind a retention policy, because every byte retained becomes an operational cost and every label creates another cardinality decision.

A notification handler should acknowledge quickly, deduplicate by event identity, and enqueue a verification job. The job reads the current intent version before evaluating anything. That read closes a common race: a customer edits the selector while an older notification is still in flight.

The critical path can be represented with generic endpoints. These are illustrative interfaces, not a vendor-specific API. The `intentVersion` is the concurrency guard; the verifier must reject evidence for any other version.

Consider the race explicitly. A property-management operator enters a domain and selector, the system stores intent version 7, and a change signal schedules a check. Before the check runs, the operator corrects a typo and the system stores version 8. If the worker trusts the event's embedded values, it can record a successful result for version 7 and advance the tenant anyway. If it reads the current intent first, version 7 becomes stale evidence and version 8 remains pending. The extra read is small; the semantic boundary it protects is not.

```bash
curl -X POST https://example.invalid/mail-verification/jobs \
  -H 'Content-Type: application/json' \
  -d '{"domain":"portfolio.example","intentVersion":7,"reason":"dns-change-signal"}'

# The worker records evidence only when the version still matches.
curl -X POST https://example.invalid/mail-verification/evidence \
  -H 'Content-Type: application/json' \
  -d '{"domain":"portfolio.example","intentVersion":7,"spf":"pass","dkim":"pass","dmarc":"pass"}'

# A bounded retry asks for fresh evidence, rather than trusting the notification.
curl --get https://example.invalid/mail-verification/status \
  --data-urlencode 'domain=portfolio.example' \
  --data-urlencode 'intentVersion=7'
```

The production implementation should also make the transition idempotent. A repeated “pass” does not create a new customer action. A later “fail” for the same version must be retained as contradictory evidence and investigated according to the resolver's freshness rules, rather than overwriting history without explanation.

The trade-off is real. A webhook-led design gives quicker feedback but inherits delivery, ordering, and duplicate-event problems. A polling-led design is easier to reason about in a tiny system, but its bounded retries add delay and read load. The combined design is a poor fit for a one-off administrative check where a person is already watching the screen; it earns its complexity when several tenant mail records must converge without a hidden false-green state.

## What does a useful stopping rule look like?

Stop cleanly.

Polling without a stopping rule is a hidden retry queue. Set a maximum verification window, a maximum number of reads, and a next action when the window closes. The action might be “still pending,” “needs DNS correction,” or “manual review.” Those states are more useful than an endless spinner because support staff can distinguish absent records from a verifier that never ran.

Sampling matters here. I would emit every state transition and every terminal reason, but sample repetitive pending observations. A metric label containing the full domain turns tenant names into high-cardinality telemetry; a label containing the intent state and reason is usually more bounded. Keep the domain in a controlled log field only when an investigation requires it, and apply access controls and retention separately from aggregate metrics.

The same discipline applies to the webhook. Count received, duplicate, rejected-version, and expired events. Do not count a webhook as a successful verification. The useful ratio is the share of intents reaching a terminal state within the allowed window, split by reason. That tells an architect whether the problem is customer DNS configuration, notification delivery, or the verifier itself.

## The rejected option and where it still fits

I reject webhook-only completion for this scenario. It has an attractive happy path: receive an event, mark the domain verified, and move the onboarding flow forward. Its weakness is semantic. The event is evidence that a change was noticed, not a complete assertion about SPF, DKIM, and DMARC at the time the tenant's mail is sent. Missed delivery and an edit racing with a stale event are enough to produce a false green state.

Polling-only remains valid for a small internal workflow with low onboarding volume, a short bounded window, and no need for immediate feedback. It can also be the recovery path after notification delivery is degraded. Its cost is operational rather than merely financial: repeated reads create load, increase latency, and make the state machine harder to explain unless each attempt has a reason and a limit.

## Decision

Publish the three records from an explicit intent, observe them independently, and make completion a versioned comparison. Use webhooks for prompt work scheduling; use bounded polling to recover from missed signals and to obtain fresh evidence. Retain enough history to explain a transition, then discard repetitive payloads.

That design keeps the important question visible: does the DNS state match what the property-management system intended for this tenant's mail? Everything else is a delivery optimization or an observability choice.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://www.rfc-editor.org/rfc/rfc7208
- https://www.rfc-editor.org/rfc/rfc6376
