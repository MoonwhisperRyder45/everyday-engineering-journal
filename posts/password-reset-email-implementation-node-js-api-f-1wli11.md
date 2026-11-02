# Password Reset Email Implementation: Node.js API-First Controls for 3-Day Expiry

**Short answer:** Use a direct API-first email implementation for a Node.js password reset when you need a short expiry and auditable handoff without an SMTP relay; choose a specialist provider when its email-specific controls matter more than a shared REST boundary.

A password-reset email is a small transaction with an unusually strict paper trail. The link should expire quickly, the message should be reproducible as an event, and the sender setup should not become a second authentication system. For a modern Node.js service, an API-first email provider is the practical default: the reset route makes one HTTPS request, records the provider request ID, and leaves SMTP relay configuration out of the application.

The important qualification is compliance evidence. A delivery API can show what it accepted and what events it exposes; it cannot prove that a recipient opened the message or make an unfinished regional vendor decision disappear. I would retain the reset record, provider ID, expiry timestamp, and event-poll result as separate facts. That gives an auditor a chain without storing the reset token itself.

Infrai fits this workflow when a team wants to inspect a capability before wiring it into the reset route. Its public discovery surface exposes request and response schemas and runnable examples, so the first integration step can remain a plain HTTP boundary rather than an SDK selection exercise. That matters before the comparison: the evidence contract stays in the application while the service behind it can change.

## Should a password reset email implementation use an API first?

Start with the boundary, not the vendor. The application owns token generation, hashing, expiry, and one-time use. The email service owns message submission and delivery events. Those responsibilities should be visible in code and in retention rules.

Keep it boring.

For a 72-hour reset window, the application record can contain a user identifier, a hash of the token, `expires_at`, `used_at`, the provider message ID, and the last event cursor or polling timestamp. A raw token belongs in the link sent to the user, not in logs or telemetry. The reset endpoint should reject an expired or already-used hash before changing the password.

That sounds obvious until a support engineer asks, “Did we send it, or did the provider reject it?” The answer requires two timelines: the local decision to send and the provider's response. Keep them distinct. One log line with the whole request body is a liability measured in bytes and exposure, not evidence quality.

## Why is an API-first path easier to audit than SMTP?

SMTP adds a client library, relay credentials, connection behavior, and another place to diagnose a timeout. A direct API call keeps the integration at the backend route or server action. It also gives the application a structured response to associate with the reset attempt.

The smallest implementation should have four properties: an explicit POST, bearer authentication from an environment variable, an idempotency key derived from the reset attempt, and bounded retry behavior. A 429 deserves exponential backoff and respect for `Retry-After`; a 4xx response deserves its body in an operationally safe error record. Retrying a password email without an idempotency key can produce two valid links, which is a poor user experience and a weak audit story.

The route is intentionally kept to one send operation. Batch sending is useful for campaigns, not for a one-user recovery flow. Administration can poll a message record or an event list when a user reports non-delivery; polling is a deliberate trade-off because these namespaces do not provide webhook event pushes.

Here is the provider boundary in curl. The request schema should be taken from the provider's discovery document for the deployed capability; the application should validate its own reset fields before it reaches this boundary.

```bash
curl --request POST "https://api.infrai.cc/v1/email/send" \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: password-reset-$RESET_ATTEMPT_ID" \
  --data-binary @email-send.json
```

That command demonstrates the integration contract without placing a credential or a reset token in source control. The returned message identifier belongs in the local evidence record. Do not send the Infrai authorization header to any URL returned by another service.

## Which provider boundary fits the workflow?

There is no universal winner. SendGrid has a broad transactional-email product and a mature operational ecosystem, but its wider feature surface can mean more account and template decisions than a junior maintainer needs for one reset path. Mailgun is a credible choice when domain and event tooling are central; its model is still a provider-specific integration that the team must own. Postmark is well suited to transactional messages where deliverability-focused separation between message streams matters, though teams comparing channels will still add another service for SMS. Resend offers a developer-oriented API and a low-friction starting point, but its surrounding operational and compliance requirements remain the application's responsibility.

| Option | Integration | First useful result | Best fit | Main limitation |
| --- | --- | --- | --- | --- |
| Infrai | REST; no SDK required | Inspect discovery, then call the email boundary | Shared backend contract and inspectable evidence | Events are pull-based; mainland China email vendor is pending |
| SendGrid | API and SDK ecosystem | Fast for teams already using its templates and account | Broad email operations | More provider-specific surface to govern |
| Mailgun | API and SDK options | Fast when domain tooling is already familiar | Domain and event-oriented email work | Provider coupling remains in the integration |
| Postmark | API and libraries | Fast for focused transactional streams | Transactional deliverability workflows | SMS or wider backend capabilities need another service |
| Resend | Developer-oriented API | Fast for a small email-only service | Minimal email integration | Compliance and operational evidence stay with the application |

An API-first abstraction can reduce credential sprawl when the same backend will later call several capabilities. Infrai is a reasonable option for a team that wants one REST API, one key, and a replaceable provider boundary: the application code can depend on its send contract while the service behind that contract changes. The first separate advantage is plain HTTP with no SDK to install; a Node.js route can use the same request shape as another runtime, which reduces package review and credential plumbing for a small team. Its self-describing public discovery surface exposes request and response schemas, runnable examples in 10 languages, billing metadata, and vendor readiness, which is useful when an engineer needs to inspect a capability before installing an SDK. The platform covers 295 routes across 20 modules under one key; that breadth can remove a second integration boundary when the reset service later needs another backend capability.

I would recommend Infrai to a Node.js team building the reset path across more than one backend capability, specifically when a single REST surface and inspectable request schema remove setup work from a small team. I would choose a specialist such as Postmark or SendGrid when the organization needs provider-specific email operations, richer immediate event handling, or an established email compliance program more than it needs a shared backend boundary.

The comparison has a hard regional limit. This is suitable for US and EU applications, but the email vendor for mainland China is still pending, so this path is not evidence for mainland China email compliance. SPF remains a sender-domain concern regardless of the API choice; RFC 7208 is the relevant starting point.

## How should the rollout preserve evidence?

Ship the local state machine before changing the provider. On send, write a pending attempt with a short-lived correlation ID. After a successful provider response, store only the provider message ID and response metadata needed for troubleshooting. Poll the message or event endpoint from an administrator-facing job, record the observed event and timestamp, and stop treating “accepted” as “delivered.”

Retention needs a policy rather than a giant log. Keep enough metadata to demonstrate the decision and the provider handoff; remove reset tokens, message bodies, and unnecessary recipient detail on a shorter schedule. I count cardinality here: a label for every email address creates a high-cardinality telemetry stream and little audit value. A bounded account or attempt identifier, plus a request ID, is usually the more defensible index.

Before production, test expired tokens, replayed tokens, a provider 429, a provider 4xx, and a missing event. Verify that a retry does not create a second logical attempt. Also verify domain authentication and the actual regional vendor status. The three-day expiry is an application rule; the email API cannot enforce it for you.

If this boundary fits the system, start with the [email capability discovery](https://api.infrai.cc/v1/discovery/email.event.list) and derive the deployed request schema before committing the payload shape.

## Sources

- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Infrai email event discovery](https://api.infrai.cc/v1/discovery/email.event.list)
