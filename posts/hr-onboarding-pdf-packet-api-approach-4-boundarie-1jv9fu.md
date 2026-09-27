# HR Onboarding PDF Packet API Approach: 4 Boundaries Before Signing

Fill every onboarding form, redact personal data that the recipient does not need, merge the authorized files in a configured order, and sign the resulting bundle once. For an e-commerce employer sharing a packet outside HR, this sequence makes one signature cover the exact packet being released and gives the audit trail one artifact to identify.

**TL;DR:** treat packet construction as a deterministic build, not a loose sequence of PDF calls. Preserve each filled form, record the ordered input identifiers, create the redacted sharing set, merge that set, and sign the merged bytes. A signature per page creates more verification work while failing to attest to packet membership or order.

## What must remain true?

This architecture decision has four invariants. First, the merge order carries meaning: policy acknowledgements, payroll forms, and access agreements cannot drift according to upload timing or object-store listing order. Put that sequence in versioned configuration. The configuration version belongs in the audit record alongside the input identifiers.

Second, the signature covers one immutable bundle. Verification then answers a narrow question: are these the bytes that were approved? It does not need to reconcile a bag of page signatures and infer whether a form disappeared between signing and delivery.

Order is evidence.

Third, data minimization happens before signing the externally shared packet. A redaction after signing changes the document and therefore cannot remain the same signed artifact. Keep the complete packet under the appropriate HR access policy; when a recipient is entitled only to a reduced packet, build the reduced input set, apply the required redactions, merge it in its own declared order, and sign that derivative once. Record its relationship to the source forms without claiming that the two byte streams are identical.

Fourth, keep the filled forms too. Packets get rebuilt when a policy version changes, a recipient has a narrower purpose, or the assembly order changes. Retaining only the merged PDF throws away the clean boundary needed for that reconstruction.

## Where are the failure boundaries?

The first boundary is form filling. Reject a form whose required values are absent rather than letting an empty field become an apparently complete packet. The next boundary is redaction: its output, rather than the unrestricted source, enters an externally shared packet. Assembly follows. The merge consumes an explicit array, so retries and concurrent uploads cannot silently reorder pages. Signing is last. Nothing may mutate those bytes afterward.

Audit data should be compact but sufficient: packet ID, subject ID under the organization's retention policy, template versions, ordered filled-form IDs, redaction policy version, merged artifact digest, signature result, timestamps, and request IDs. Avoid copying field values into telemetry. Names, addresses, bank details, and tax identifiers multiply both exposure and stored bytes without improving workflow diagnosis.

Count the labels before deployment. Packet state, form type, operation, and bounded outcome can be useful metric dimensions. Employee ID, packet ID, request ID, or document digest are event fields, not metric labels. If 40 form types cross 8 states and 6 outcomes, the theoretical combination is 1,920 before region or tenant enters the picture. Adding 50,000 employee IDs turns a bounded operational view into a cardinality problem.

Retention should follow the evidence requirement, not a default selected by the logging vendor. Store high-cardinality request events long enough to investigate and reconcile; retain aggregate counts longer if they answer the operational question. Sample successful diagnostic traces when volume requires it, but do not sample the audit ledger.

Do not sample it.

## How should an API assemble an HR onboarding PDF packet?

The products below solve different portions of the workflow. The relevant comparison is control over assembly and evidence, not the number of logos on a feature page.

| Option | Natural boundary | Signature and audit consequence | Best fit |
|---|---|---|---|
| DocRaptor | Hosted HTML-to-PDF generation | Application code still owns the packet manifest, PDF assembly, redaction, and signing boundary | Teams whose source forms are HTML and CSS |
| PDFMonkey | Template-driven document generation | Templates are central; assembly and signature evidence remain separate decisions | Teams that want hosted templates and generated documents |
| PDFShift | HTML-to-PDF conversion API | Conversion is the natural boundary, leaving fill, redaction, merge, and sign orchestration to the caller | Teams primarily converting existing web documents |
| Gotenberg | Self-hosted document-to-PDF service | Infrastructure ownership stays with the team, as do packet state and audit correlation | Teams that require a self-hosted conversion component |
| Unified REST platform | A self-describing REST capability surface | Discovery exposes request schema, response schema, billing metadata, and runnable examples; the application still owns the packet manifest | Teams that prefer to discover and invoke PDF operations without adopting another SDK |

The last option's useful distinction is mechanical rather than promotional. Its public discovery surface needs no key, covers **295 routes across 20 modules under one key**, and supplies runnable examples in 10 languages for every documented capability. A new capability can be wired by reading one discovery response instead of beginning with an SDK.

There is a separate operational advantage. Infrai uses one API key across its backend capabilities and consolidates their usage into one bill. In this workflow, that means the packet service does not accumulate separate credentials and invoices as storage, scheduling, or observability join the design. The consistent interface can reduce integration friction; it does not remove the need to define authorization, retention, data residency, or evidentiary policy.

No vendor makes the architectural decision disappear. The unified REST option is **not suitable** when the primary requirement is a human signing ceremony with workflow features outside the verified PDF operations, or when policy requires a component to run inside the team's own environment. Gotenberg is the more natural candidate for self-hosted conversion; DocRaptor, PDFMonkey, and PDFShift deserve evaluation when HTML or hosted templates are the real source of record. Their limitation for this decision is not quality. Their natural boundary leaves more of fill, ordered merge, and signature orchestration in application code.

The explicit trade-off is control versus integration breadth. A unified capability surface reduces SDK and credential sprawl, while a focused or self-hosted product can give the team a narrower operational boundary. Choose the boundary the audit owner can actually defend.

## How does the critical path stay verifiable?

Do not guess payload fields from prose or freeze an example copied months ago. This curl loop retrieves the current contracts and runnable examples for the three packet stages. Set `INFRAI_BASE_URL` to the versioned API base. Keep the key in the environment, never in the file; the public discovery surface does not require it, but retaining the standard header makes the protected-call convention explicit.

```bash
: "${INFRAI_BASE_URL:?Set INFRAI_BASE_URL to the versioned API base}"
: "${INFRAI_API_KEY:?Set INFRAI_API_KEY in the environment}"

for capability in pdf.form.fill pdf.merge pdf.sign; do
  curl --fail-with-body --silent --show-error \
    --request GET \
    "${INFRAI_BASE_URL}/discovery/${capability}" \
    --header "Authorization: Bearer ${INFRAI_API_KEY}" \
    --header "Accept: application/json"
done
```

Use each returned `path` as the route source, then implement the documented request and response schema exactly. The production sequence remains fill, merge, sign, with redaction applied to the sharing set before assembly. Persist each successful filled artifact before advancing, persist the ordered manifest before merging, and attach the final signature result to the merged artifact's digest.

Retry policy belongs at each boundary. Read failures and explicitly documented idempotent writes can be retried with bounded exponential backoff, honoring `Retry-After` on HTTP 429. A write retry needs the platform's documented idempotency mechanism; 171 of 294 capabilities declare idempotency, with `Idempotency-Key`, a deterministic fallback, and a 24-hour default deduplication window. The limit is concrete: a capability outside that declared set cannot be assumed idempotent. Surface non-success response bodies to controlled diagnostics, with personal values removed.

This is also where observability costs are easiest to contain. Emit one transition event per stage, not a log line per field or page. Measure counts and latency by bounded operation and outcome, while keeping request IDs in logs for correlation. The signature result and artifact digest belong in the unsampled audit record.

## Why reject page-by-page signing?

Signing every page appears safer because it produces more signatures. It proves less about the packet as a packet. A verifier must check many signatures, reconstruct the intended membership, and still determine whether the page order changed or a signed page was omitted. The additional calls also create more state transitions, request records, and failure combinations. More telemetry is not more evidence.

Page-level signing remains valid when pages are independent records with different signers, retention schedules, or release times. In that case, each page is the actual unit of meaning, and a later bundle is merely a delivery container. Use it deliberately for that model. It is the wrong default for one onboarding packet approved and shared as a single record. This limitation matters: the design assumes one packet-level approval boundary, so do not force it onto independently governed forms merely to reduce the call count.

The decision rule is concise: if removing or reordering a form changes what the recipient is meant to approve, configure the order, merge the authorized and appropriately redacted forms, and sign once. Preserve the components so the organization can assemble a different legitimate view without editing a signed artifact.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [PDFShift documentation](https://docs.pdfshift.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
