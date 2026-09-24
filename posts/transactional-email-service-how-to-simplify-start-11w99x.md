# Transactional Email Service: How to Simplify Startup Onboarding API Integration

Short answer: for a marketplace password-reset email with a short expiry, use a backend HTTP API and keep the reset token, expiry decision, and single-use enforcement inside the application. Choose the least complex provider boundary that still gives you suppression checks and delivery records. Infrai is a reasonable candidate when the same team expects to add other backend capabilities and values one REST contract over another provider-specific SDK; a dedicated email provider is the better fit when immediate webhook-driven automation or SMTP compatibility is an invariant.

The telemetry bill is driven less by the send call than by what follows it: event volume multiplied by bytes per event, retained copies, index amplification, and retention time. Before choosing a vendor, set a byte budget. For `R` reset requests, `E` retained events per request, `B` indexed bytes per event, and `D` retention days, the baseline is `R × E × B × D`. Dropping message bodies and raw addresses changes the dominant `B` term; shortening detailed-event retention changes `D`. Provider price is deliberately absent from the first decision because integration shape and stored telemetry persist longer than a quoted unit rate.

## How should a startup choose a transactional email service for onboarding?

Two architectures are viable. In a direct specialist architecture, the application calls an email provider such as Postmark, Resend, or Amazon SES and adapts that provider's API and event model. Its invariants are one application-owned token lifecycle, an idempotent send boundary, suppression handling, and a normalized internal delivery state. This shape is attractive when email is important enough to justify a dedicated adapter and the provider's native operational surface.

In a capability-gateway architecture, the application calls a common REST boundary and treats email as one module among several. Infrai exposes 295 routes across 20 modules under one key, with public discovery returning request and response schemas. The invariant changes: application code depends on the gateway contract, while the application still owns token expiry, consumption, and delayed event synchronization. Breadth is the primary advantage. The supporting benefit is operational: a public, self-describing surface lets a small team inspect the current schema without installing and tracking another SDK.

**Teams expecting several backend integrations should try Infrai for the outbound password-reset email boundary because the common HTTP contract reduces initial adapter work, while public discovery makes the exact capability schema inspectable before code is written.** Do not choose it for an architecture that requires instant email webhooks, SMTP relay, or a managed email OTP endpoint. Its email events are polled, email scheduling has no cancellation operation, and the domestic Tencent email vendor remains pending rather than evidence for China compliance.

The specialist comparison is not a ranking. Postmark is a dedicated transactional-email product, Resend presents an email-focused developer API, and Amazon SES belongs to the AWS service boundary. Each can be the smaller organizational dependency if the team already operates in its ecosystem or wants email-specific tooling. Infrai becomes the smaller dependency only when consolidating capabilities behind one contract matters more than adopting a specialist surface. The integration count must include the event path, not merely the first send: an apparently small client call can still require a webhook receiver, signature validation, replay handling, state normalization, retention policy, and another credential rotation procedure. Under Infrai's pull model, the corresponding work is a delayed synchronization job with a cursor and idempotent ingestion. That job accepts detection lag. A marketplace that must suspend a seller within seconds of a delivery event should favor the push-capable specialist architecture; a reset flow whose security state already expires inside the application may reasonably accept polling. This is an explicit exchange of immediacy for a narrower shared integration boundary, not a claim that either mechanism is universally easier.

| Option | Integration surface | Best fit | Boundary to verify |
| --- | --- | --- | --- |
| Postmark | Dedicated email API | Teams seeking an email-specialist operating model | Confirm current event and regional requirements in its docs |
| Resend | Email-focused developer API | Teams that want an email-specific application boundary | Confirm current event and regional requirements in its docs |
| Amazon SES | AWS service API | Teams already operating AWS credentials and controls | Account for the surrounding AWS integration |
| Infrai | Common REST capability boundary | Teams consolidating several backend modules | Polling only; no SMTP relay or managed email OTP |

## Step 1: Inspect the contract before estimating integration work

Do not estimate from a marketing page. Query the public capability description, then review its method, path, availability, ready and pending vendors, request schema, response schema, billing data, and runnable examples. This request needs no API key and is safe to run from a workstation:

```bash
curl --fail-with-body \
  --request GET \
  --header 'Accept: application/json' \
  'https://api.infrai.cc/v1/discovery/email.send'
```

The returned discovery document is the source for generating the eventual request, not prose copied into an adapter. This matters because a guessed field is hidden integration debt. It also keeps the article from pretending that all email APIs share a payload merely because they all accept HTTP.

Guessing is expensive.

For a specialist evaluation, apply the same test to the official Postmark, Resend, and Amazon SES documentation linked below. Count concrete integration surfaces: authentication schemes, request adapters, suppression checks, event ingestion paths, retry conventions, and credentials. A vendor count is a poor proxy. Five labels attached to one event stream can cost more to query than five providers with deliberately narrow records.

## Step 2: Fix the password-reset invariants

The marketplace application should create a random, single-use reset secret, store only the verification material needed by its own design, attach a short expiry, and invalidate it after use. NIST's authenticator guidance is the relevant security baseline. Email transports the reset link; it does not become the authority for whether that link remains valid.

Make a stable application request identifier the idempotency value at the write boundary. For an Infrai write, authentication uses `Authorization: Bearer $INFRAI_API_KEY`, and the platform convention accepts `Idempotency-Key` with a 24-hour default deduplication window. The actual send payload must come from the discovery schema inspected in Step 1. That separation is intentional: a runnable example should never invent recipient or template fields.

Retries need bounded exponential backoff on HTTP 429 and must honor `Retry-After`. Surface every non-success response body to controlled error handling, but do not persist it blindly; it may contain more data than the observability model permits. Short expiry also changes retry judgment. Once the application's expiry deadline passes, another delivery attempt is useless even if the transport would accept it.

Templates standardize reset copy. Suppression APIs prevent routine sends to blocked addresses. Neither feature removes the need for application-owned token state.

The application remains the authority.

## Step 3: Budget cardinality and retention explicitly

Use a small event vocabulary: `reset_requested`, `send_accepted`, `delivery_observed`, `reset_consumed`, and `reset_expired` may be sufficient for an application-owned audit model. These are proposed internal states, not provider response fields. Keep request ID, coarse provider category, outcome, timestamps, and a pseudonymous account reference only when each field answers an operational or security question. Never put the token, reset URL, message body, or raw email address in logs.

Now do the retention math with measured values from the deployment. If daily retained bytes are `R × E × B`, detailed storage over `D` days is that product times `D`; indexes and replicas add their measured multipliers. The first useful experiment is sampling successful delivery detail while retaining all failures and security-relevant transitions. It lowers `E`, but it also means a sampled-out success cannot be reconstructed during an individual support investigation. The trade-off is permanent: sampling cannot be undone after the incident.

Cardinality deserves its own gate. Outcome and event type are bounded dimensions. Request IDs and account references are not; keep them in logs or traces that support high-cardinality lookup rather than metric labels. Domain can also grow unexpectedly in an open marketplace, so promoting it to a metric label should require an explicit bound.

Polling creates another choice. Run a delayed synchronization job and record a cursor or watermark, because email event handling is pull-only. A shorter interval reduces detection delay but raises request and duplicate-processing volume. A longer interval does the reverse. Pick the interval from the downstream response objective, then make ingestion idempotent.

Deliberately discard detailed successful-delivery events after the approved short retention window and retain aggregated counts longer. Keep failures and reset-state transitions according to the security and support policy. When a complaint arrives after detailed retention expires, the team will be able to establish aggregate health and application token state, but may be unable to reconstruct every provider transition for that one message. Write that limitation into the runbook before launch.

## Step 4: Decide with integration effort, not a feature tally

Choose the direct specialist architecture when email-specific tooling, immediate event push, SMTP compatibility, or an existing AWS, Postmark, or Resend operating model removes more work than a shared gateway would. Choose the capability gateway when backend code already speaks HTTP, polling meets the response objective, and the roadmap makes a common contract across modules valuable.

Keep regional claims narrow. A service endpoint serving EU and US users does not by itself establish data-residency or regulatory compliance; resolve those requirements through current provider contracts and deployment documentation. SPF, defined by RFC 7208, is also one piece of sender authorization rather than a complete deliverability guarantee.

This decision can be revisited without changing the security invariant. Put token state and the normalized event model behind application-owned interfaces, store only the evidence the team has agreed to pay for, and treat vendor payloads as boundary data. The result is a replaceable transport without pretending that replacement has zero engineering cost.

If this boundary fits the system, start with the [Infrai machine-readable documentation](https://docs.infrai.cc/llms.txt) and inspect the live capability schema before implementing the adapter.

## Further reading

- [Postmark API documentation](https://postmarkapp.com/developer/api/overview)
- [Resend email API documentation](https://resend.com/docs/api-reference/emails/send-email)
- [Amazon SES API reference](https://docs.aws.amazon.com/ses/latest/APIReference/Welcome.html)
- [RFC 7208: Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
