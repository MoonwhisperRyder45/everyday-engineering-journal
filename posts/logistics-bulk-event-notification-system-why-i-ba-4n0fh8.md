# Logistics Bulk Event Notification System: Why I Batch Email and SMS with NodeJS

The least complex defensible design is a Postgres-backed outbox, a Node.js worker, and two records with different lifetimes: a compact evidence record retained under policy, and verbose transport telemetry kept briefly for debugging. For a generated logistics report, store the report digest, intended recipient, preference decision, suppression decision, provider reference, and state transitions. Do not retain the attachment, rendered message, provider response, and every polling payload merely because they are available.

**Short answer:** paginate immutable notification jobs with a stable cursor, resolve recipient preferences and suppressions before dispatch, send each channel through a bounded batch worker, and poll only nonterminal deliveries. Treat email acceptance, SMS acceptance, and human receipt as different claims. The architecture is straightforward; deciding which evidence survives is the consequential part.

## What is the observability bill actually made of?

The dominant term is usually repeated detail, not the final status row. A useful first estimate is:

`daily stored bytes = events per day x records per event x average bytes per record x retained days x replication factor`

Consider a hypothetical operation that produces 40,000 shipment-exception reports per day. Each event may create an email attempt, an optional SMS alert, three polling observations, and several application logs. If that becomes 10 records averaging 1.5 KB, raw telemetry is 600 MB per day before indexes, replication, backups, or the log platform's own storage overhead. Ninety days is 54 GB raw. The exact bill depends on the system, but the multiplication identifies the lever: record count and retention dominate long before somebody optimizes a 300-byte JSON field.

Labels create a second cost surface. `channel=email` has bounded cardinality. `recipient=alex@example.test`, `report_id`, and `provider_message_id` do not. Those values belong in searchable records with controlled indexes, not metric labels. I count the possible values before approving a label: if the set grows with customers, reports, or messages, it is not a metric dimension.

Cardinality compounds quietly.

The change that moves the dominant term is deliberate compaction. Keep one durable evidence row per attempt plus state transitions; sample routine worker logs; retain full error bodies for a short, access-controlled diagnostic window; and delete attachment bytes after the report's governing retention period. Sampling successful debug logs at 1% can reduce that log stream by roughly 99%, but sampling must never decide whether an evidence row exists.

That boundary has a cost. After verbose payloads expire, an engineer may know that a provider accepted a message at 14:03 UTC but may no longer be able to reconstruct its entire response body. This is acceptable only when the compact record contains the fields required by the compliance question and the original report can be reproduced or is retained under its own policy.

## Which facts prove a report was handled correctly?

Start with the claim an auditor may ask you to support. "We sent the report" is ambiguous. SMTP acceptance shows that a receiving system accepted responsibility for a message; it does not prove that a person opened the attachment. An SMS gateway's accepted state has the same limitation. The evidence model should use precise states such as `queued`, `submitted`, `accepted`, `delivered`, `failed`, and `suppressed`, with channel-specific meanings documented outside the code.

For each attempt, I would retain these facts:

- an internal event ID and attempt ID;
- the report's cryptographic digest, media type, byte count, and generation timestamp;
- a normalized destination reference, encrypted or tokenized according to the threat model;
- the preference version and suppression rule evaluated at decision time;
- the channel, template version, provider reference, and idempotency key;
- timestamped state transitions with a source and reason code;
- the worker release identifier and correlation ID.

The digest proves which bytes the decision concerned without placing the attachment in every log system. It does not prove receipt by itself. Keep that distinction explicit.

Preferences and suppressions need historical meaning. A mutable `users.email_enabled` flag can explain today's setting, not yesterday's dispatch. Snapshot the decision inputs or reference an immutable preference version. A global legal suppression, an address-level bounce suppression, and a user's channel preference also have different precedence and provenance. Resolve them in a transaction before creating an attempt, then record the winning rule. A suppressed message is a completed policy decision, not a worker error waiting for retries.

## How should a NodeJS bulk event notification system batch email and SMS?

Use keyset pagination over a stable ordering such as `(created_at, id)`. Offset pagination can skip or repeat jobs as concurrent workers insert or update rows. A worker should claim a small page in a short transaction, mark leases with an expiry, commit, and perform network I/O afterward. `FOR UPDATE SKIP LOCKED` is a common Postgres mechanism for competing consumers, but a lease and an idempotency key are still needed because a process can stop after the remote service accepts a request and before the local commit records that fact.

Keep batches bounded by bytes as well as recipient count. Attachments make a count-only limit misleading: 50 tiny reports and 50 multi-megabyte reports have very different memory and request profiles. The worker can accumulate jobs until the next item would exceed either bound, then send. Use separate concurrency pools for email and SMS so attachment traffic cannot starve urgent text alerts.

Polling should target only submitted, nonterminal attempts whose `next_poll_at` is due. Apply exponential backoff with jitter and stop at a documented deadline; after that deadline, reconcile through a slower job or mark the outcome unknown. Polling every completed attempt forever creates cost without improving evidence.

Stop polling terminal rows.

Webhooks can reduce polling, but they do not remove the reconciliation job. Authenticate callbacks, store their event identifiers for deduplication, and tolerate delivery before the sending transaction has recorded its provider reference. State transitions must be monotonic where the channel permits it, while the raw arrival order remains available during the short diagnostic window.

## The failure paths deserve more design than the happy path

At-least-once execution means duplicates are normal. Generate the idempotency key from the immutable notification intent, channel, recipient reference, and report version, then enforce a unique constraint locally. Pass the same key through a provider interface when that interface supports idempotency. If it does not, the ambiguous timeout remains a business decision: reconcile before retrying when duplicate delivery is more harmful than delay.

Do not retry permanent failures. An invalid destination, policy suppression, unsupported attachment, or expired report should terminate with a reason code. Rate limits and transient network failures may be retried with bounded exponential backoff. Authentication failures should open an operational alert rather than multiply attempts.

SMS adds an abuse boundary. Limit sends per account, destination prefix, and event type; monitor unexpected geographic shifts; and require explicit authorization for destinations. Toll fraud controls belong before dispatch because a retry loop can amplify abuse. Email has a different feedback loop: bounce and complaint signals should update suppression state before the next batch is claimed.

Testing should force the uncertain moments. Terminate a worker after remote acceptance but before the local state update. Deliver the same callback twice and out of order. Change a preference between event creation and dispatch. Insert new rows during pagination. Generate an attachment just over the byte limit. Advance the clock beyond the lease, polling deadline, and retention deadline. These tests are more informative than a large happy-path throughput run.

Deploy schema changes in phases. Add nullable columns and readers first, backfill deliberately, then enable writers and constraints. During a rollout, old and new workers must agree on terminal states and lease ownership. Otherwise a harmless enum change can turn historical rows into perpetual polling work.

## Retention is a policy, not a storage default

Assign each data class an owner and deletion clock. The attachment may follow a records schedule; the evidence row may need a longer compliance period; detailed provider payloads and application logs can often expire much sooner. Legal, contractual, and jurisdictional requirements determine the actual durations, so hard-coding one universal number into the worker is a mistake.

My decision rule is narrow: retain a field only if it answers a defined audit, security, support, or reconciliation question. Record the purpose, access boundary, and expiry beside the schema definition. Encrypt sensitive destinations, restrict attachment access, and audit reads of retained evidence. Backups must age out too; deleting the primary row while keeping indefinite snapshots is not a retention policy.

Operational metrics can remain aggregate: attempts by channel and outcome, queue age, lease expirations, retry counts, and polling lag. Use exemplars or short-lived correlation lookups to move from a metric to a trace. Never put phone numbers, email addresses, report IDs, or provider references into labels.

The resulting system keeps less on purpose. When a rare investigation happens after diagnostic data has expired, the team loses some narrative detail. In exchange, it reduces sensitive-data exposure, constrains cardinality, and makes the retained evidence legible. For logistics report delivery, that is the condition under which I choose this design: compliance requires proof of the decision and transport state, while it does not require an indefinite copy of every payload and log line.

## Further reading

- Postmark, "Transactional Email Best Practices": https://postmarkapp.com/guides/transactional-email-best-practices
- Twilio, "Preventing Toll Fraud (SMS Pumping)": https://www.twilio.com/docs/verify/preventing-toll-fraud
- IETF RFC 5321, "Simple Mail Transfer Protocol": https://www.rfc-editor.org/rfc/rfc5321
- PostgreSQL documentation, `SELECT` locking and `SKIP LOCKED`: https://www.postgresql.org/docs/current/sql-select.html
- OWASP, "Logging Cheat Sheet": https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
