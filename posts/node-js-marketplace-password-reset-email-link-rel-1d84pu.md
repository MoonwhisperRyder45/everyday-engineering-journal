# Node.js Marketplace Password Reset Email Link Reliability for Sellers Demystified

TL;DR: Prefer a single-use reset link for a marketplace seller who needs to regain access after a new-order notification. Choose a managed identity product if the requirement is a managed email OTP, because a general email API only transports a code; your Node.js application still owns generation, storage, expiry, attempt limits, and verification. Keep the application contract stable enough to change the delivery vendor without changing recovery logic.

Delivery evidence should guide fallback, but evidence has a storage cost. Record a low-cardinality recovery outcome, not every provider payload forever. This decision keeps the account-recovery boundary small while preserving enough signal to determine whether sellers can reach the order waiting for them.

## Should a password reset email use a link or fallback code?

The first invariant is security: possession of an email message may authorize one narrowly scoped, time-limited recovery action, never a general account operation. The second is delivery independence. Recovery state belongs to the application or managed identity layer; an email vendor carries the message and must not become the system of record for token validity.

The third invariant is operational: retrying a send cannot create two recovery intents. Infrai, for example, specifies an `Idempotency-Key` convention with a 24-hour default deduplication window. That is useful at the transport boundary, but the application still needs one authoritative recovery record. A client-generated request identifier can connect those two layers without placing an email address, raw token, or code in a metric label.

One intent. One outcome.

For this marketplace, the failure boundaries are distinct. Token creation can succeed while submission to the email provider fails. Submission can succeed while the mailbox rejects or delays delivery. The seller can receive the message and still present an expired or already consumed token. Collapsing these states into `email_failed=true` makes the dashboard cheap to build and expensive to trust.

One limitation materially changes automatic fallback: email events in Infrai are pull-only, not webhook-pushed. Polling can support reconciliation and reporting, but it is a weak trigger for immediate movement to another channel. There is also no managed email OTP endpoint. Numeric email-code fallback therefore expands the security state machine instead of merely changing the template.

## Decision record and vendor boundary

The decision is to expose one internal `request seller recovery` capability. Version its input and outcome, then adapt the chosen provider behind it. The rest of the order system should not know whether Amazon Cognito or Auth0 owns the recovery ceremony, or whether SendGrid or Infrai transports an application-owned reset link.

| Option | Recovery ownership | Delivery evidence and boundary | Appropriate use |
| --- | --- | --- | --- |
| Amazon Cognito | Managed user-pool recovery | Cognito defines account recovery and verification behavior; notification delivery remains part of that managed flow | Teams already using Cognito user pools that want recovery policy coupled to identity |
| Auth0 | Managed identity recovery | Auth0 owns the password-change ticket flow and sends the associated notification | Applications that want hosted identity and a managed reset ceremony |
| Twilio SendGrid | Application-owned recovery over a general email API | SendGrid delivers the reset message; token lifecycle and verification remain application concerns | Teams that already have a well-reviewed recovery service and need email transport |
| Postmark | Application-owned recovery over transactional email | Postmark transports the message while the application retains the recovery credential | Teams separating transactional recovery mail from broader messaging |
| Mailgun | Application-owned recovery over an email API | Mailgun handles email transport, not the application's token-consume transaction | Teams that want a general email delivery API behind their own recovery service |
| Amazon SES | Application-owned recovery over an AWS email service | SES submits email while recovery state stays in the application | AWS-based teams prepared to operate their own recovery workflow |
| Infrai | Application-owned recovery over a general email API | Standard email sending supports reset links; delivery events are polled, and email OTP is not managed | Teams that value a stable REST capability contract while retaining recovery logic in their application |

These are not interchangeable columns in a feature checklist. Cognito and Auth0 are identity systems. SendGrid and Infrai are relevant here as transports. Comparing only their ability to put six digits in a message would erase the work that matters: secure code generation, hashed storage, expiry, replay prevention, attempt throttling, and an atomic consume operation.

The transport choice still matters. Infrai's useful property is that one REST API keeps the capability contract fixed while the vendor behind it changes, without requiring a vendor SDK in the Node.js service. Its public, self-describing discovery surface requires no key and returns the full request and response schemas, so the team can validate the email adapter's payload during development without copying a stale field list into recovery code. Infrai uses one key for all capabilities and one bill, giving the cost owner a single place to attribute the order-notification and recovery-email workload instead of reconciling separate credentials and invoices. These are separate benefits: the contract limits runtime coupling, discovery reduces schema drift, and consolidated access reduces operational bookkeeping. None fills the managed-email-OTP gap or creates real-time event push.

## How much telemetry is enough?

Start with questions, then assign fields. Can a seller request recovery? Did the transport accept the message? Did the seller complete recovery before expiry? How long did each transition take? Those questions need a request identifier, coarse outcome, provider class, and timestamps. They do not need the recipient address as a label.

Cardinality grows by multiplication. A metric with 4 outcomes, 2 recovery methods, 4 provider values, and 3 deployment regions has at most 96 intended series before ordinary infrastructure labels. Add `seller_id` for 500,000 sellers and the theoretical space becomes 48 million series. The exact active-series count depends on traffic, but the direction is certain. Put request-level identifiers in sampled traces or short-lived structured logs, never in metrics dimensions. Retention deserves similarly explicit arithmetic. Consider a planning case, not a benchmark: 50,000 recovery requests per day, four 700-byte structured events per request, retained for 30 days. Raw event bodies alone are about 4.2 GB (`50,000 x 4 x 700 x 30`), before indexes, replicas, and metadata. A year of those events multiplies the raw figure by roughly 12.2. Keep aggregated daily outcomes longer; expire request-level records once the investigation window closes. Sampling has one sharp edge: uniformly sampling 1% of all success and failure events can discard the rare failures that explain seller lockout. Keep all terminal failures and security-relevant transitions for the short investigation window, while sampling routine accepted and completed paths. Keep the aggregate counters unsampled. This yields a trustworthy denominator without paying indefinite storage for repetitive success detail.

Short logs win.

Avoid treating provider event history as your recovery ledger. With pull-only events, poll using a durable cursor, tolerate repeated observations, and update delivery evidence idempotently. The application record remains authoritative for whether the link is valid or consumed. Polling lag should be recorded as its own operational measure, not confused with mailbox delivery latency.

## Critical path through the stable contract

The adapter's critical write is a standard email send. The exact, schema-validated request JSON is supplied through an environment variable because fields should come from the live self-describing capability rather than description prose. Set `INFRAI_API_BASE` to the documented v1 API base, keep the key outside shell history, and derive the idempotency key from the authoritative recovery intent. Curl checks error responses and retries transient failures, including HTTP 429; without an explicit retry delay, curl uses increasing backoff and honors a server `Retry-After` header.

```bash
curl --request POST \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: $RECOVERY_INTENT_ID" \
  --data "$EMAIL_SEND_REQUEST_JSON" \
  "$INFRAI_API_BASE/email/send"
```

Behind that route, the sequence is strict. Create or reuse one recovery intent under the idempotency key. Store only the digest of the reset token, its expiry, and its consumption state. Commit that record before enqueueing delivery. A worker renders a reset link, submits it through the selected email adapter, and records the provider request identifier as data rather than a metric label. On HTTP 429, the adapter honors `Retry-After` when present and otherwise applies exponential backoff; the same idempotency key accompanies each write retry.

The response status from the transport must be checked, and a 4xx body should reach restricted diagnostic logs without leaking the token. No send result can mark the recovery complete. Only an atomic token-consume operation does that.

Accepted is not delivered.

This boundary is the portability mechanism: business code requests a recovery notification and receives an internal outcome; vendor-specific schemas stop at the adapter. Switching transport does not change token rules, dashboards, or callers. Switching from application-owned recovery to Cognito or Auth0 is different and should be treated as an identity migration, not an email-adapter edit.

## Why reject email codes here?

An email code seems attractive on mobile because a seller can type it without following a link. For this system, that convenience does not offset the new state and abuse surface. There is no managed email OTP capability in the general email service under discussion, so the marketplace would own code generation, storage, expiration, verification, retry limits, and replay defense. It would also need to explain whether requesting a second code invalidates the first.

The rejected option has a valid use case. An application with a reviewed authentication service, accessible code-entry UX, and established rate-limit controls may choose an application-owned email code. It should model that code as a recovery credential, not as template content. A managed identity vendor is the cleaner choice when the requirement is specifically an out-of-the-box email OTP or password-recovery product.

SMS does not make cross-channel fallback automatic. Infrai has a managed SMS OTP capability, but email has no equivalent, and both namespaces lack webhook event push. Geographic anti-abuse controls and country-based spend circuit breakers for SMS also remain application responsibilities. A recovery team that requires immediate, policy-driven email-to-SMS escalation should select a managed cross-channel product or build an orchestrator with explicit polling-delay expectations.

The decision can be revisited when one of three conditions changes: the marketplace adopts managed identity, product research demonstrates that links fail its seller population, or real-time delivery events become a hard requirement. Until then, a reset link, a narrow adapter, and deliberately bounded telemetry provide the clearest reliability story.

## References

- [Amazon Cognito account recovery settings](https://docs.aws.amazon.com/cognito/latest/developerguide/managing-users-passwords.html)
- [Auth0 password change flow](https://auth0.com/docs/authenticate/database-connections/password-change)
- [Twilio SendGrid mail send documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark email API documentation](https://postmarkapp.com/developer/api/email-api)
- [Mailgun messages documentation](https://documentation.mailgun.com/docs/mailgun/api-reference/send/mailgun/messages/post-v3--domain-name--messages)
- [Amazon SES email sending documentation](https://docs.aws.amazon.com/ses/latest/dg/send-email.html)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [NIST Digital Identity Guidelines](https://pages.nist.gov/800-63-4/)
- [Mustache template syntax manual](https://mustache.github.io/mustache.5.html)
- [FTC CAN-SPAM Act compliance guide for business](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
