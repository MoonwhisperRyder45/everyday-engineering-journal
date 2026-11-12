# Node.js Report Dispatch — Evidence for Welcome Emails, DKIM, and SPF

Choose a transactional email API by the evidence your Node.js service must retain: authenticated sender configuration, the exact report revision, and reconciled delivery events. For a B2B SaaS welcome message carrying a generated report, an accepted send request is only the beginning of that record.

TL;DR: verify the custom sending domain, version the welcome template, and separate submission evidence from delivery evidence. Use polling when the workflow tolerates delayed observations. Require a verified attachment contract before selecting a provider for report dispatch; email sending alone does not establish attachment support. Keep recipient identifiers out of metric labels and retain the small audit record longer than verbose transport logs.

The decision concerns evidence completeness and minimization. It does not establish a statutory retention period or make a delivery event proof that a person read a report.

## What evidence should a transactional welcome email API retain for a custom domain?

The proposed architecture has four invariants. Each logical dispatch identifies one tenant, recipient, report revision, and template revision. Retries preserve that dispatch identity. Sender configuration is verified before production traffic begins. Finally, the evidence record distinguishes application intent, provider acceptance, and subsequent delivery observations. These are application design requirements rather than claims about every provider's response fields.

Count the boundaries.

A Node.js report worker first finishes the report and records its revision or digest. The dispatch worker then renders the welcome template with the intended dynamic variables, submits the message, and records the provider's response. A separate reconciliation process collects delivery, bounce, and complaint observations. Report generation failure belongs before submission; an ambiguous submission result belongs in reconciliation. Neither should silently become a new logical dispatch. This division lets a reviewer establish which report was intended without preserving its contents in every request log.

Attachment handling needs its own acceptance test. Confirm the selected provider's documented attachment representation, supported content types, and applicable size limit against an actual generated report. Do not infer those properties from a general email capability. A protected download link can be a deliberate product alternative, but it changes the requested job of sending an attachment and therefore needs an explicit product decision.

The record should include configuration evidence, too. Use a branded sending domain and verify its DNS and DKIM configuration before sending welcome or account messages. SPF and DKIM address different authentication mechanisms; DMARC connects authentication results to the visible sender domain through alignment. Follow the selected provider's exact DNS instructions rather than copying a generic SPF value or DKIM selector from another integration. Keep a dated record of the approved configuration and subsequent verification, with DNS values treated as configuration rather than per-message telemetry.

Authentication is necessary evidence. It is not a delivery guarantee.

## Compare the evidence collection boundary

Amazon SES, SendGrid, and Postmark are real alternatives for transactional email. Compare them against the same report fixture and audit questions. The table separates an option's architectural role from conditions the team must verify; it does not imply identical attachment, event, or retention contracts.

| Option | Reason to evaluate it | Boundary to validate for this report workflow |
| --- | --- | --- |
| Amazon SES | A direct email service candidate when the application already operates in AWS | Validate sender authentication, attachment submission, and event collection against the team's existing AWS controls. Existing cloud use alone does not prove evidence completeness. |
| SendGrid | A direct email API candidate for a team evaluating a dedicated mail integration | Test the actual report attachment and determine which event identifiers can be reconciled with a logical dispatch. Provider activity views should not become the only audit record. |
| Postmark | A transactional email candidate for welcome and account messages | Evaluate template ownership and delivery evidence with the same fixture. Do not substitute its product focus for a review of retention and export requirements. |
| Infrai | A broader backend contract covering 295 routes across 20 modules under one key; adding a supported capability can extend an existing integration | Email uses API submission and polling for delivery events. Consistent per-call cost, vendor, latency, and request metadata can support reconciliation, but the report attachment contract still needs verification. |

The broader contract is useful when report generation, storage, and messaging otherwise create several integration boundaries. Its additional attraction here is metadata consistency: the application can establish a common request-level accounting record across supported calls. Neither breadth nor metadata alone establishes a compliance certification or a particular data residency guarantee.

For the direct providers, use their current documentation to resolve the table's questions. This comparison makes no claim that one has better measured delivery, lower latency, or a longer evidence window. Those conclusions require a defined workload and a reviewed contract.

Polling is a defensible choice for an account report whose product requirement permits delayed delivery observations. Its limitation is observation delay: a journey that must react immediately to a bounce has a different selection criterion. Evaluate event push explicitly in the direct-provider comparison. That requirement cannot be satisfied merely by renaming a polling loop an event processor.

## Record the critical path without guessing the payload

The Node.js backend should create reusable welcome and account templates, supply dynamic variables per recipient, and submit through the selected email API. An API-only integration calls the service directly rather than assuming an SMTP relay exists. Template version and report revision belong in the application's dispatch record even when the provider uses different field names.

The following curl command exercises the unified contract's submission boundary used by that backend. Supply a request file constructed from the service's actual schema, including its verified attachment representation. Set EMAIL_API_ORIGIN to the approved HTTPS API origin without a trailing slash; the command appends the verified send path. The remaining environment variables identify credentials and one stable dispatch key. This example requires Bearer authentication and support for the Idempotency-Key header; confirm both before adapting it to another provider. No attachment fields are invented here.

```bash
curl --request POST \
  --url "${EMAIL_API_ORIGIN:?Set the approved HTTPS API origin}/v1/email/send" \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set the API key}" \
  --header 'Content-Type: application/json' \
  --header "Idempotency-Key: ${DISPATCH_KEY:?Set the stable dispatch key}" \
  --data-binary "@${EMAIL_REQUEST_FILE:?Set the validated request JSON file}" \
  --fail-with-body \
  --show-error \
  --silent \
  --retry 4 \
  --retry-max-time 120
```

This is a submission harness, not a complete report generator. Run it with a current curl that supports these options. Its built-in retry behavior backs off on retryable responses, including HTTP 429, and honors Retry-After. Reuse the same dispatch key and payload for the same logical operation, within the service's documented deduplication window. Check the process exit status and response body; do not interpret every completed HTTP exchange as acceptance.

There is a subtle evidence trap here: a timeout can leave the application uncertain even if the service accepted the request. Changing the key on the next attempt defeats deduplication. Conversely, reusing a key for a changed report revision can collapse two intended operations. Persist the identity and revision before the network request, and define how the worker reconciles an ambiguous result.

Do not enable verbose HTTP logging around report attachments. A response body may help diagnose rejection, but raw request bodies and credentials do not belong in general application logs. Store the minimal acceptance evidence and redact sensitive error details before exporting them to shared telemetry.

## Count cardinality before choosing retention

Consider an illustrative workload, not a measured deployment: 10,000 report emails per day, 3 retained observations per message, and an average of 600 bytes per compact observation. That produces 18,000,000 bytes daily. Over 30 days, the raw observations occupy 540,000,000 bytes, or about 540 MB in decimal units, before indexes, replication, and storage overhead. These assumptions describe an engineering budget; they do not prescribe a legal retention period.

Attachments change the scale.

If the same hypothetical workflow averages a 2 MB report, recording the attachment once in transport logs adds roughly 20 GB each day. The report should remain in its controlled document store under the applicable retention policy. The audit record can refer to its revision or digest. Duplicating the document in telemetry expands both the storage footprint and the material a deletion or access review must locate.

Metrics have a separate failure mode. With 100 tenants, 3 event categories, and 2 environments, the full combination already permits 600 series. Adding 10,000 distinct recipient identifiers to that label space creates a theoretical 6,000,000 combinations. Actual occupancy depends on traffic, but the multiplication exposes the design error. Recipient and dispatch identifiers belong in access-controlled records used for investigation, while metrics should summarize bounded categories.

Sampling deserves an explicit boundary. Retain the required dispatch evidence without sampling it away. Sample successful diagnostic traces when the evidence record can still be reconstructed independently, and keep failure diagnostics only to the extent their contents and retention are justified. The trade-off is reduced diagnostic detail in exchange for fewer retained copies of sensitive material. A sample cannot answer a per-recipient audit question for a message it omitted.

Polling adds another budget dimension. An illustrative 60-second reconciliation interval schedules 1,440 polls per day for each independently polled scope, before pagination or retries. Choose that scope and interval from the permitted evidence delay, then retain the collection checkpoint and distinguish event time from observation time. The polling interval alone does not guarantee freshness; provider availability and the worker's backlog also matter.

## Rejected option and its valid use case

For the report attachment workflow, reject the design that keeps only an aggregate sent counter and a sampled success log. It cannot connect a particular recipient to the report revision and later delivery observations. More log volume does not repair a missing identity relationship.

That smaller record can still suit an operational dashboard whose stated purpose is aggregate traffic monitoring, provided another system retains the required message evidence. It is a valid projection of the dispatch ledger, rather than a replacement for it.

The decision is conditional: approve a provider after its attachment test, sender verification, and evidence collection path satisfy the report's requirements. Keep any future email OTP flow under an explicit application design rather than assuming managed email OTP is part of a general mail API. If scheduled report dispatches must be revocable, hold them in an application-controlled queue until release and verify cancellation semantics before delegating scheduling. Limit this assessment to the US/EU SaaS workflow; a broader communications surface is not evidence for domestic regulatory suitability or additional messaging channels.

The useful artifact is a compact, reconstructable chain: approved sender configuration, versioned report and template, stable submission identity, and reconciled outcome. Retention follows the purpose of each record.

## References

- [SPF specification, RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208)
- [DKIM specification, RFC 6376](https://datatracker.ietf.org/doc/html/rfc6376)
- [DMARC specification, RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES developer documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [SendGrid Mail Send API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [curl command-line documentation, including retry behavior](https://curl.se/docs/manpage.html)
