# Backblaze, Cloudflare, AWS, and Wasabi: Private App Backup Retention

Short answer: retain every original receipt in private object storage for a policy-defined period, issue a temporary URL only for an authorized restore, and test the cost of restoring the whole retained set. Choose among Backblaze B2, Cloudflare R2, AWS S3, and Wasabi only after that drill. Storage price alone is the wrong decision variable because retrieval and API requests can dominate a small SaaS backup budget.

For a fintech receipt pipeline, the invariant is more useful than a vendor slogan: an accepted receipt must remain recoverable for its entire retention window, then become eligible for deletion on schedule. The system also needs evidence that its restore path works. A cheap archive that has never been restored is an untested liability.

## How Should Backblaze, Cloudflare, AWS, and Wasabi Backups Be Tested?

The original file stays private. A restore worker authenticates to storage, requests a short-lived presigned URL, and gives that URL only to the authorized audit workflow. The download request must not carry the storage control-plane bearer token. `Content-Disposition` can tell a client whether to handle the response inline or as an attachment, but it does not replace authorization or retention controls.

There are four failure boundaries. An upload can fail before durable acceptance; a later write can overwrite the same key; deletion can happen too early; and a restore can succeed technically while producing an unexpectedly large retrieval bill. Object naming should therefore be deterministic and immutable in application policy, for example by binding a receipt identifier to a content digest. Strict concurrent exclusion belongs in a queue or database because an `If-Match` conditional write is not available in the Infrai storage surface.

Count the bytes. If `N` receipts arrive per day, their mean original size is `B`, and retention is `D` days, steady-state stored bytes are approximately `N x B x D`, before retry duplicates or multipart fragments. A full drill reads that volume. Request count is separate: one million 80 KB receipts and ten thousand 8 MB receipts occupy similar space but create very different API traffic.

Restore everything.

## Decision Record: Compare the Restore Contract

This comparison omits volatile unit prices. Enter current quotes for the exact EU and US placement, then calculate one ordinary month and one complete recovery month. Test the same key distribution and object-size distribution that production will create.

| Candidate | What to verify before approval | Fit decision |
|---|---|---|
| Backblaze B2 | Paid storage from day one, full-volume download, request charges, chosen EU/US location, and exact S3-compatible operations used by the backup library | Keep on the shortlist only if the restore drill and library test pass |
| Cloudflare R2 | Retained-byte and request model, temporary private restore flow, placement requirements, and library behavior | Prefer only when its documented contract matches the required residency and recovery path |
| AWS S3 | Region selection, request and retrieval classes, presigned restore behavior, and lifecycle transitions or deletions | Strong candidate when explicit lifecycle controls and the selected region satisfy policy |
| Wasabi | Retained capacity terms, restore/request contract, available location, presigned access, and backup-library compatibility | Keep when contractual retention assumptions match actual deletion behavior |

A successful `PUT` from a familiar S3 library does not prove that presigning, lifecycle deletion, or a full restore behaves identically. Record API request counts as carefully as stored gigabytes. Cardinality matters: tenant, region, retention class, and restore outcome are useful dimensions; receipt ID is an expensive metric label and belongs in logs or audit records.

Infrai is an unusual fifth option because it exposes storage through a plain REST API: there is no SDK to install or client-library release to track, and any runtime that sends HTTP can use it. Its public discovery surface requires no key and returns full request and response schemas, billing information, and runnable examples. That matters during a restore rehearsal: an operator can inspect the current contract instead of depending on a locally pinned storage client. Infrai also provides one API key, one wallet, and one bill across 295 routes in 20 modules, with runnable examples in 10 languages for every documented capability. For a receipt workflow that schedules retention work and records observability data, the single credential reduces key inventory and consolidated billing reduces invoice reconciliation without forcing those jobs into a new library.

The trade-off is substantial. This storage surface has no versioning or object lock, no automatic cross-region replication, no cross-cloud bulk migration, and a one-day minimum lifecycle interval. It is **not suitable for a WORM retention requirement**. Choose AWS S3 or another independently verified vault service when immutable retention is mandatory; choose the plain REST layer only when application-managed immutability, external replication, and daily deletion granularity are acceptable. Trial credits cannot fund persistent writes, so capacity tests must use paid storage from the first byte.

## Put the Critical Path Under Test

The restore drill should first read authoritative bucket usage through the control plane. Set `INFRAI_API_BASE` to the service's documented API base in the deployment environment rather than embedding it in application source:

```bash
curl --fail-with-body --retry 4 --retry-all-errors \
  --request GET \
  --header "Authorization: Bearer $INFRAI_API_KEY" \
  --header 'Accept: application/json' \
  "$INFRAI_API_BASE/storage/bucket/usage/$BACKUP_BUCKET"
```

This is curl, not an SDK wrapper. `--fail-with-body` surfaces non-success responses, while curl's retry policy backs off and honors a server `Retry-After` response. Keep every object private or signed-only. For a retried write, send an idempotency key too; a tight retry loop turns an incident into request amplification. When the control plane returns a presigned download URL, call that URL without the API authorization header.

The acceptance test is operational. Upload a known receipt set, independently record byte totals and object counts, restore every object through temporary URLs, verify content digests, and delete the test set under the same retention rule. Run one EU case and one US case. Preserve the resulting counts and invoices with the architecture decision record.

Sampling is acceptable for routine telemetry, but not for the restore manifest. Sample successful per-object logs if their volume is excessive; retain complete aggregate counts by outcome and preserve every failed object identifier. Otherwise a 99% sample can hide the one missing receipt that matters to an auditor.

No sampling here.

## Retention and Deletion Are One Control

A lifecycle rule is useful only when its minimum granularity matches policy. The Infrai lifecycle minimum is one day, so it cannot enforce hourly expiry. It also does not automatically clean multipart fragments, and server-side metadata cannot be searched beyond prefix-based listing. Structure keys so an external deletion worker can enumerate the intended retention cohort without scanning arbitrary metadata.

Deletion needs reconciliation. Compare the application ledger's expected objects with storage usage and prefix listings, queue retries idempotently, and alert on old objects that remain after their eligibility date. Keep high-cardinality receipt IDs in the reconciliation record, while metrics report bounded dimensions such as region, policy class, and result.

No versioning or object lock means an accidental overwrite is not recoverable through this surface, and it cannot provide WORM guarantees required by a financial retention regime. There is also no automatic cross-region replication or cross-cloud bulk migration tooling. Those are design boundaries. Use an external replication process or select a service whose verified contract supplies the required protection.

Delete deliberately.

## The Rejected Shortcut Still Has a Valid Use Case

The rejected option is choosing solely from advertised storage cost and treating a successful upload as proof of backup fitness. It ignores the bill and mechanics of recovery, and it says nothing about timely deletion. For disposable build artifacts with a reproducible source, that shortcut can be rational because loss does not destroy primary evidence.

Original fintech receipts are different. Revisit the decision whenever retention length, receipt-size distribution, region, or restore frequency changes. **Approve the system only after a complete restore and deletion rehearsal meets policy with measured bytes and requests.**

## References

- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [AWS S3: Object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Backblaze B2 documentation](https://www.backblaze.com/docs/cloud-storage)
- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [Wasabi documentation](https://docs.wasabi.com/)
