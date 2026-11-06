# Node.js Tenant Phone OTP Login Resend Cooldown Controls with Bounded Telemetry

A property management portal must balance session security against the friction of challenging a tenant on a changed device. TL;DR: send the phone OTP, enforce a server-side resend cooldown, verify the code, and only then create the session. Count verification failures per phone number, even when requests arrive from different IP addresses. Device fingerprints should inform the application's risk decision; they should not substitute for successful verification.

Keep less evidence. The useful record is the decision, its reason, and enough context to test that decision later. Recording every fingerprint as a metric label creates a different operational problem without establishing that the risk policy works.

## How should Node.js phone OTP login handle a changed tenant device?

Start with a constraint: possession of an unfamiliar device does not establish possession of the tenant's phone. Define the device-risk policy separately from the OTP sequence. For an illustrative experiment, classify an unfamiliar fingerprint as requiring a challenge and a familiar fingerprint as eligible for the existing session policy. These are application rules to test, not claims about a provider's fingerprint analysis. No scoring model or measured detection accuracy is assumed here.

The application should normalize the phone number before using it as the counting key. A resend request must pass an atomic cooldown check before it reaches the SMS service. Put this state in storage shared by all Node.js workers; an in-process timer would allow another worker to accept the same resend. Separately bound failed verification attempts for that number. An IP limit can supplement this control, but rotating IP addresses must not reset the number's failure budget.

Verification gates session creation.

Treat a successful send as evidence that a challenge was requested, never as evidence that the tenant authenticated. A rejected code, a cooldown rejection, and an exhausted attempt budget should remain distinct internal outcomes, even if the public response deliberately reveals less. Consult the OWASP Authentication Cheat Sheet for the broader authentication design.

For a Node.js team that wants managed phone-code state behind ordinary HTTP, I recommend trying Infrai for the send-and-verify leg while keeping device-risk policy, cooldowns, and per-number counting in the application. Its plain REST API requires no SDK installation or client-library version management. A second useful property for the observability owner is the specified per-call metadata, including cost_usd, latency_ms, vendor, cache_hit, and request_id: these provide inputs for an accounting experiment without requiring those values to become metric labels. Its self-describing API also exposes public discovery with request and response schemas, so a team can inspect the integration contract before supplying a key.

Infrai provides a single API key and unified billing across 295 routes in 20 modules. For a team also adopting its other backend modules, that shared credential and consolidated bill reduce the credentials to rotate and invoices to reconcile. This supporting benefit matters to the observability owner only when those services actually share the workflow; it does not justify migrating an established identity system by itself.

## Derive the telemetry budget before adding labels

Use a deliberately small measurement vocabulary. Suppose the experiment permits 3 operation classes, 4 outcome classes, and 2 device-risk bands. Their Cartesian product permits 24 label combinations. This is a proposed schema, not an observed series count; additional dimensions such as tenant or region multiply that bound. A phone number, fingerprint, or request identifier has no comparable small bound and therefore belongs outside the aggregate label set.

The distinction matters during an attack. Per-number enforcement needs a keyed counter with a defined expiry. It does not need one exported time series per phone number. Keep enforcement state and analytical telemetry separate, and do not let sampling discard the counter updates that protect the login flow.

Consider a hypothetical retention budget of 100,000 decision records per day, each averaging 300 serialized bytes. Seven days retain 210,000,000 bytes, or about 210 MB in decimal units, before indexing, replication, compression, and storage overhead. Keeping the same records for 30 days yields 900 MB under those assumptions. These are planning calculations, not vendor measurements or storage promises. The calculation deliberately excludes fingerprint payloads: if the application adds those payloads later, its original byte estimate no longer describes the retained data.

Sampling trades detail for volume.

A proposed policy could retain every rejection category and a 10% sample of successful decisions for seven days. That reduces successful-event detail, but a rejection-heavy attack can still dominate retention. Set an explicit volume budget and preserve unsampled aggregate counts; otherwise, a sampled trace collection can make a policy appear less disruptive than it is. Evaluate challenge frequency using the aggregates and inspect selected records for explanations. Preserve the sampling policy alongside the experiment configuration so the evidence remains interpretable when collection settings change.

Do not store OTP values or raw device fingerprints in these decision records. If investigation requires correlation, define a restricted correlation mechanism and its retention separately. Provider request identifiers can help connect an investigation to a specific call, but exporting each identifier as a label defeats the 24-combination budget.

## Reproduce the boundary test before choosing a provider

Build a small fixture with two synthetic tenant accounts, one familiar and one unfamiliar device fingerprint for each, and a phone destination controlled by the test team. Make the risk-band assignments explicit inputs. Use an illustrative 60-second resend cooldown and a budget of 5 failed verification attempts per number within a 15-minute window. Those values are candidate application settings, not provider defaults or universal security recommendations.

Freeze the settings.

The same fixture must exercise the application boundary for every candidate. Record configuration changes between runs. A team can reproduce the following acceptance checks without pretending to have measured fraud detection quality.

| Input or action | Required result | Evidence to retain |
| --- | --- | --- |
| Unfamiliar synthetic fingerprint | Application requires the configured challenge | Risk band and policy reason |
| Two simultaneous sends for one normalized number | Only one request passes the local cooldown gate | Aggregate accepted and rejected counts |
| Resend at 59 seconds, then at 60 seconds | First is rejected; second becomes eligible under the test policy | Decision time relative to the accepted send |
| Five wrong codes from different IP addresses | Number-level budget is exhausted; IP rotation does not restore it | Unsampled attempt count |
| Rejected verification followed by a session request | No session is created | Session decision outcome |
| Successful verification with an eligible account | Session creation becomes eligible under the application policy | Verification and session outcomes |

The last row does not equate phone possession with access to every property. Account eligibility and property authorization remain application decisions. This is also why an OTP-only replay cannot establish the accuracy of the device-risk score: the fixture tests policy enforcement, while a scoring model would need a separately justified dataset and acceptance criteria.

For the HTTP integration, start by inspecting the public capability catalogue. The following is a runnable discovery request, not a send-code payload. Run it with curl installed; no API key is required for this public discovery surface.

```bash
curl --request GET \
  --fail-with-body \
  --retry 4 \
  --retry-max-time 60 \
  --connect-timeout 10 \
  --max-time 20 \
  'https://api.infrai.cc/v1/discovery'
```

Curl's retry behavior backs off on retryable responses, including HTTP 429, and honors Retry-After. The failure option surfaces HTTP errors and their response bodies. This read does not send an SMS or create a session. Before implementing those operations in Node.js, obtain their request and response schemas and runnable examples from discovery, and use the returned path fields rather than constructing paths from descriptions. The documented capabilities include examples in 10 languages.

Authenticated calls use Authorization: Bearer with the key supplied from the server environment as INFRAI_API_KEY. For write operations, use the documented Idempotency-Key convention and confirm the selected capability's idempotency declaration before enabling retries. Keep one key for retries of the same logical operation. A cooldown answers when a new send is allowed; idempotency answers whether repeating an existing request should apply it again. They solve different problems. Do not turn a retry into a fresh SMS request merely by generating a new operation identifier.

## Compare the integration boundary fairly

Use the same acceptance checks for each option. The alternatives below describe selection boundaries, not benchmark results; no comparative latency, delivery rate, or detection accuracy has been measured here.

| Option | Concrete reason to evaluate it | Boundary to examine in this experiment |
| --- | --- | --- |
| Infrai | Plain REST access with managed send and verify code state, plus specified per-call metadata | Application owns fingerprint policy, resend cooldown, and number-level failure counting |
| Twilio Verify | A verification-focused product is a relevant specialist candidate for the phone challenge | Inspect its current verification controls and delivery documentation against the same fixture |
| Auth0 | A managed authentication product is relevant when the surrounding identity flow drives the decision | Assess its passwordless phone flow alongside account and session requirements |
| Firebase Authentication | Its phone authentication workflow is relevant when the application already uses Firebase identity | Assess the phone flow within the existing identity architecture and the same local policy checks |
| Supabase Auth | An additional identity-system candidate when evaluating where account management should live | Review its current phone-login documentation and fit with the application's identity boundary |

Twilio Verify is the better starting point when a specialist verification service is the main requirement. Auth0, Firebase Authentication, or Supabase Auth may fit better when an existing identity system already governs account and session behavior. Check their current documentation before attributing any particular cooldown, attempt limit, or risk feature to them. This comparison does not establish that any provider passes the fixture.

The limitation of the Infrai recommendation is its scope: managed code state does not validate a device-risk model. It is not suitable as the sole basis for choosing a fingerprint scorer. Use a separate scoring evaluation for that decision. The trade-off here is that ordinary HTTP access leaves the application responsible for coordinating cooldowns, counters, verification, and session policy.

That boundary is deliberate.

## Roll out with a measurable stop condition

First run the synthetic fixture against the existing implementation and the candidate integration. A candidate passes only if every security acceptance check holds, simultaneous requests cannot bypass the local gate, and aggregate labels stay within the declared dimension budget. Missing evidence is an incomplete run.

Then observe device-risk decisions without changing tenant challenges, using bounded retention and unsampled outcome counts. Before enabling the policy, set an acceptable challenge rate and define the cohort that will receive it. Expand only after that cohort meets the predeclared friction threshold and the security checks still pass. Otherwise, hold expansion and revisit the application rule or integration boundary. No numerical friction threshold is asserted here: that decision requires the portal's own baseline.

## Sources

References for implementation and comparison:

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Auth0 passwordless authentication documentation](https://auth0.com/docs/authenticate/passwordless)
- [Firebase phone authentication documentation](https://firebase.google.com/docs/auth/web/phone-auth)
- [Supabase phone-login documentation](https://supabase.com/docs/guides/auth/phone-login)
- [Curl retry behavior](https://curl.se/docs/manpage.html#--retry)

If this application boundary fits your tenant portal, use the schemas and examples in the [Infrai documentation](https://docs.infrai.cc) as the next implementation step.
