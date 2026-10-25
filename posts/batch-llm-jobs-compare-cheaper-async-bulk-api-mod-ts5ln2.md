# Batch LLM Jobs: Compare Cheaper Async Bulk API Moderation with Realtime Limits (10K)

Short answer: batch LLM jobs can be cheaper than realtime handling for media moderation reports that can wait for a nightly or backfill run. They move summarization, tagging, and extraction away from peak-time synchronous handling, while status tracking and result export keep the workflow small enough for a junior team. The important constraint is structured output: a delayed answer that fails the review schema is still a failed answer.

For a planning example, assume 10,000 reports, each with 350 input tokens and an 80-token JSON decision. That is 4.3 million tokens per run before retries. Run it nightly for 30 days and retention stops being a side detail. Raw prompts, full responses, duplicated status events, and verbose logs can outweigh the material used to make the original classification.

## What is the bill actually made of?

I start with bytes, not a vendor price page. The workload has at least four terms: input tokens, output tokens, repeated attempts, and retained telemetry. Batch processing changes the timing and execution model. It does not make malformed reports disappear, and it does not make every log line free.

The first estimate is straightforward:

`10,000 x (350 + 80) = 4,300,000 tokens per run`

That is a planning ceiling, not a production bill. The final amount depends on the selected model, prompt growth, retries, and the provider's billing rules. Estimate against representative reports before submission, including long reports, empty fields, and adversarial text. A small sample that contains only clean reports produces false confidence about both token volume and schema validity.

Retention needs its own arithmetic. Keeping 30 full copies of the assumed input and output gives 129 million token-equivalents passing through the system, before JSON framing, metadata, indexes, and logs. A compact record with report ID, decision, schema version, confidence fields, request ID, and a pointer to restricted raw material has a different storage shape. The trade-off is real: when an appeal arrives, less retained context means more reconstruction work.

I retain raw payloads only for the review window required by policy, then keep derived decisions longer when auditability requires them. Debug logs get sampling and redaction. A request ID is more useful than copying the entire prompt into every line.

## Can batch LLM jobs be cheaper than realtime moderation?

Yes, when “later” is acceptable. A moderation classifier can turn a report into a fixed schema: category, severity, policy IDs, and a human-review flag. The model call should request structured output, and the worker should validate every result before marking it ready for review. JSON that parses but omits a required field is still a failed item.

There is no dedicated moderation endpoint in the verified capability set. The practical fallback is a chat model with a JSON schema contract. That makes validation an application boundary rather than a formatting preference. It also makes the comparison fair: the relevant measure is valid structured decisions on a fixed corpus, not merely whether a request returned HTTP 200.

The operating sequence is plain: submit a bounded set of jobs, persist the batch ID, poll with backoff, fetch results, validate each item, and export an operator-readable artifact. A retry or review lane needs an idempotent item key. Standard queues are at-least-once systems, so the consumer must tolerate duplicate delivery instead of creating two moderation decisions for one report.

Batch helps only where latency is flexible. User-facing chat, an upload screen waiting for immediate classification, and an interactive appeal tool still need normal completion calls. Hiding synchronous work behind a batch label only adds delay.

The submission call can stay a small curl smoke test. Keep the base URL in environment configuration, and keep the request body generated from the discovered schema rather than embedding a guessed payload in a blog example.

```bash
curl --request POST "${INFRAI_BASE_URL}/ai/batch/submit" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: ${BATCH_IDEMPOTENCY_KEY}" \
  --data "${BATCH_REQUEST_JSON}"
```

The worker should treat HTTP 429 as a scheduling signal: back off exponentially, honor `Retry-After` when present, and reuse the same idempotency key. It should inspect non-2xx response bodies instead of assuming success. A write without an idempotency key can duplicate a submission when the client loses the response after the server accepted it.

## Which platform boundary matches the workflow?

The comparison is about operational fit, not a universal winner.

| Option | Useful distinction | Boundary to check |
| --- | --- | --- |
| OpenAI Batch API | Natural for teams already using OpenAI request formats and deferred bulk execution. | Confirm the result-file lifecycle and structured-output validation fit the moderation review pipeline. |
| AWS Bedrock batch inference | Fits teams already operating model access, identity, storage, and audit controls in AWS. | The surrounding AWS data path may be a larger integration surface than the model call. |
| Google Vertex AI batch prediction | Fits a GCP-centered data and ML workflow where batch artifacts belong beside other Vertex jobs. | Confirm the selected model and response handling match the review schema. |
| Infrai batch capability | Useful when one REST API, one key, and one bill are more valuable than separate provider credentials; pure HTTP requires no provider SDK, while status and export support the bulk workflow. | Validate model readiness, schema behavior, retention policy, and regional requirements before production. |

OpenAI is a narrow fit when the rest of the application is standardized there. Bedrock and Vertex can be the better organizational choice when cloud IAM, storage, and governance already live in that cloud. A unified runtime has a different advantage: one plain REST interface can be called from a shell job, a Node.js worker, or a backfill tool without installing an SDK for each provider. That reduces the integration friction around this particular flow, where the same team needs submission, status, result retrieval, and export rather than a new client abstraction for each step.

There is a second, independent benefit beyond credential and billing consolidation: the public discovery surface is self-describing and available without a key, so the team can inspect request and response schemas before writing the worker. Documented capabilities also ship runnable examples in 10 languages. As of the 2026-10-25 capability snapshot, that combination makes a schema check and a curl smoke test easier to hand off for a small moderation operation.

The breadth is verified at 295 routes across 20 modules under one key. Breadth matters here only when the shared request convention removes a real boundary: adding token estimation before submission, for example, need not introduce another provider-specific client. It is not a reason to route every workload through one runtime.

The evidence I would collect before selecting a platform is specific: schema-valid rate, retry rate, reviewer agreement, end-to-end latency, and telemetry volume on the same fixed corpus. The winner should produce acceptable moderation decisions with an operating shape the team can support. A convenient API does not compensate for an invalid policy label.

## What should be kept when an appeal arrives?

Retain enough to answer four questions: which report was classified, which schema version was requested, which model route produced the result, and why the worker accepted or rejected it. Raw reports may need stricter access controls and a shorter retention window than derived moderation decisions. Sensitive content should not be copied into ordinary logs merely because a failed job is inconvenient to diagnose.

The cost appears during an appeal or incident. Reconstructing a missing prompt is slower than reading a retained payload. Still, retaining every payload forever multiplies exposure, storage, and the number of places an operator must search. I prefer a deliberate review window, sampled diagnostics afterward, and immutable decision metadata pointing to the relevant audit record.

This is where one key and one bill can be operationally useful, but the argument is consolidation rather than a promise that batch makes every model call inexpensive. The batch workflow still needs a measured retention policy, a schema validator, and a clear owner for rejected items.

## A launch rule for structured moderation batches

Choose batch when the report can wait, the output has a testable schema, and retries can be made idempotent. Start with normal reports, missing fields, long text, and policy-edge cases. Estimate tokens before submission. Record status transitions without duplicating the full payload in every transition log.

Choose synchronous completion when a human is blocked on the answer. For that path, optimize response handling and validation instead of forcing deferred work into an interactive request.

The strongest cost control is often the material you deliberately stop retaining. That choice has a price: less context during an incident. Document that reconstruction cost, define the review window, and let the moderation policy determine how long raw material must remain available.

## Further reading

References:

- OpenAI, Batch API guide: https://platform.openai.com/docs/guides/batch
- AWS, Amazon Bedrock: https://aws.amazon.com/bedrock/
- Google Cloud, Vertex AI batch prediction: https://cloud.google.com/vertex-ai/docs/predictions/batch-predictions
- eCFR, 45 CFR Part 164 Security and Privacy Rules: https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164
