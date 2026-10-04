# Retaining Less in Node.js Image Upload Moderation (NSFW, Violence, Hate Symbols)

TL;DR: In Node.js image upload moderation, classify NSFW content, violence, and hate symbols through a typed multimodal adapter, validate its JSON locally, and retain decisions rather than uploaded images or raw model conversations. For each property photo, record a policy version, three bounded category results, a final action, latency, and a correlation ID. Keep the source image under the product system's lifecycle; keep moderation telemetry only as long as a stated investigation or audit need requires.

The bill is mostly a multiplication problem: uploads per day x bytes retained per upload x retention days x replication and indexing overhead. A 4 MB property photo copied into telemetry is 4 MB before indexes, replicas, or backups. The corresponding decision document can be a few hundred bytes. The meaningful change is architectural: stop copying image bytes and raw multimodal transcripts into logs.

For a property manager, this gate sits before catalog enrichment. A kitchen photo may be safe and eligible for room classification; an unsafe image should be quarantined before generated amenity tags or listing prose can spread it into search indexes. **Structured output correctness is the primary decision axis.** A plausible paragraph is not a moderation result.

## How should Node.js classify an image upload for NSFW and violence?

Use a small, versioned decision envelope. Category names must be a closed set, confidence must be numeric and bounded, and the policy action must be an enum. Keep model output separate from policy evaluation: the classifier reports evidence; deterministic application code decides whether to allow, review, or quarantine. A threshold can then change without pretending the underlying classification changed.

A useful record has `upload_id`, `policy_version`, `schema_version`, one result each for `sexual`, `violence`, and `hate_symbol`, the final `action`, processing latency, and a trace identifier. If reviewers need an explanation, retain a short reason code from a controlled vocabulary. Free-form rationales are high-cardinality text, may echo sensitive visual details, and are hard to aggregate.

Do not put an address, tenant name, original filename, image bytes, signed URL, base64 data, prompt, or full response into the event. The worker can receive a short-lived object reference, fetch the image, classify it, and emit only the decision.

Store less.

This curl call illustrates an internal adapter contract. `MODERATION_URL` names an endpoint owned by the application team; its adapter translates the stable contract to the selected multimodal runtime. The schema rejects extra keys so unexpected prose cannot quietly enter the policy path.

```bash
curl --fail-with-body --silent --show-error \
  --request POST "$MODERATION_URL" \
  --header "Authorization: Bearer $MODERATION_TOKEN" \
  --header "Content-Type: application/json" \
  --data @- <<'JSON'
{
  "upload_ref": "object://pending/01JPROPERTYPHOTO",
  "task": "Classify sexual content, violence, and hate symbols. Return JSON only.",
  "response_schema": {
    "type": "object",
    "additionalProperties": false,
    "required": ["categories"],
    "properties": {
      "categories": {
        "type": "object",
        "additionalProperties": false,
        "required": ["sexual", "violence", "hate_symbol"],
        "properties": {
          "sexual": {"type": "number", "minimum": 0, "maximum": 1},
          "violence": {"type": "number", "minimum": 0, "maximum": 1},
          "hate_symbol": {"type": "number", "minimum": 0, "maximum": 1}
        }
      }
    }
  }
}
JSON
```

The example deliberately does not prescribe thresholds. A confidence score is not a universal probability, and the correct boundary depends on a labeled evaluation set and the cost of each error. In catalog intake, a false allow may expose harmful imagery; a false quarantine delays a legitimate listing and creates reviewer work.

## How does the JSON fallback stay fail-closed?

Native schema-constrained output is useful when a runtime supports it, but the application boundary still has to distrust the response. Parse JSON, validate it against the same schema, reject unknown fields, and verify all three categories. A response that is truncated, fenced in Markdown, shaped differently, or outside numeric bounds is invalid.

No guesswork.

The fallback should mean one controlled retry asking for the same schema, not regular expressions that scrape three numbers from prose. If that response is invalid, emit an `indeterminate` outcome and send the upload to review or quarantine according to policy. Never reinterpret a transport error or parse failure as a clean image. Key the job by upload ID plus content digest and policy version so a retry does not create contradictory catalog states.

The flow is short: validate the file at intake, isolate it from public serving, call the classifier, validate the response, evaluate policy, then release it to enrichment only after an allow decision. OWASP's file-upload guidance describes defense in depth around extension, content type, generated filenames, storage location, and size limits. Moderation does not replace those controls; it answers a different question about visual content.

Test missing categories, extra properties, strings instead of numbers, values outside 0 through 1, timeouts, duplicate delivery, and a response cut off halfway through JSON. Keep a small, access-controlled labeled set of ambiguous property images: artwork containing nudity, a historical emblem, a red stain, a toy weapon, and text too small to read. Human reviewers must establish those labels because context changes the policy meaning.

## Count cardinality before choosing telemetry

Metrics answer aggregate questions only while their labels remain bounded. Suitable labels include category, action, schema version, policy version, and a coarse error class. Upload IDs, content digests, addresses, filenames, raw error messages, and trace IDs do not belong in metric labels; each can create a new time series. Put a correlation ID in a sampled log or trace instead.

The arithmetic should be explicit. Suppose the service handles 100,000 uploads per day. One 700-byte decision event retained for 30 days contributes about 2.1 GB of logical payload before storage overhead. Copying a 4 MB image for every decision contributes about 12 TB over the same interval. These are dimensional examples, not a benchmark or storage quote, and they omit replication, compression, indexes, and backups. Substitute measured event sizes and actual upload distributions before setting capacity. Sampling changes what can be answered later: keep counters for every outcome because their dimensions are bounded, and keep every indeterminate event and policy transition, subject to privacy rules, because they diagnose the gate. Sample routine allow traces aggressively; they are numerous and usually repetitive. Review decisions may need longer retention than operational traces if an explicit audit requirement exists, but that requirement should name both a duration and an owner. Token telemetry deserves similar restraint. A tokenizer such as tiktoken can count text tokens for compatible encodings, but image accounting is runtime-specific; do not estimate visual input from a base64 string and call it model usage. Record usage returned by the runtime when available, aggregated by bounded workload and policy version. Avoid storing the prompt merely to reconstruct a bill.

Vector storage is not an evidence policy either. pgvector adds vector similarity search to Postgres, which may help retrieve related test cases if embeddings are part of the reviewed design. An embedding remains retained derived data. Give it a purpose, access policy, deletion path, and expiration; do not preserve it by default merely because it is smaller than the image.

## Operating the gate without preserving everything

Deployment starts in shadow mode: classify and validate, but do not alter catalog state. Compare outcomes with a labeled review set, segment results by the three categories and meaningful property-photo cohorts, then select thresholds from the observed false-allow and false-quarantine trade-off. Aggregate metrics can reveal a shift after a model, schema, or policy version changes. They cannot prove correctness alone.

Roll out enforcement gradually and make the policy version visible in every decision. Alert on invalid-schema rate, indeterminate rate, latency, and queue age. A rise in a category rate is a review signal, not automatic proof of model drift: the upload mix may have changed. Investigate with a bounded sample under an approved access process.

Retention has a real cost beyond storage. If routine allow traces expire after seven days, a complaint arriving two months later cannot be reconstructed from a raw model conversation. Preserve the signed decision, content digest, policy and schema versions, timestamps, and review history for the justified window; accept that expiring source material limits forensic detail. Keeping everything would improve replay, but it would enlarge the privacy surface and make deletion harder.

**Retain enough to prove which rule ran and what action followed, while refusing to build a secondary archive of resident and property imagery.** Catalog enrichment consumes only uploads with a valid, policy-approved decision. Invalid output never becomes an empty set of flags.

## References

- JSON Schema specification: https://json-schema.org/specification
- OWASP File Upload Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/File_Upload_Cheat_Sheet.html
- OpenTelemetry logs data model: https://opentelemetry.io/docs/specs/otel/logs/data-model/
- W3C Trace Context Recommendation: https://www.w3.org/TR/trace-context/

## Further reading

- tiktoken, an official BPE tokenizer library: https://github.com/openai/tiktoken
- pgvector, vector similarity search for Postgres: https://github.com/pgvector/pgvector
