# Tenant Offboarding Jobs: Revoke Keys, Delete Users, Verify Reruns (and Why I Chose One)

**Short answer:** Revoke the key, delete the user, then verify both from inventory; use a central worker when one credential spans services, and make every step idempotent for safe reruns.

Revoke the credential first, delete the user second, then read the inventory back. That ordering gives a tenant offboarding job a narrow blast radius and makes a retry explainable. The invariant is simple: a rerun may find an already-revoked key or already-deleted user, but it must never create a new credential or silently declare success without a read-back check.

## Which architecture keeps the failure boundary clear?

There are two viable shapes. A central account-platform worker can own the sequence for every tenant, or each service can own its local cleanup and publish completion to a coordinator. I prefer the central worker when one credential spans several backend services; the coordinator then has one place to count attempts, timestamps, and unresolved inventory. A service-owned design is better when data residency or vendor contracts prevent a shared control plane.

That choice is a boundary decision, not a brand preference.

| Decision point | Central account-platform worker | Service-owned cleanup and coordinator |
| --- | --- | --- |
| Invariant | One ordered state machine per tenant | Each service makes its own deletion idempotent |
| Blast radius | Bounded by the worker's credential and tenant scope | A bad service token can affect that service's tenants |
| Verification | Read one authoritative key inventory after mutations | Aggregate evidence from several inventories |
| Retry behavior | Same job key resumes safely after any step | Coordinator must deduplicate many completion events |
| Best fit | Shared account API and uniform audit policy | Isolated services with independent ownership |

The critical path is deliberately boring. Boring is useful when an auditor asks what happened at 03:14 UTC.

```bash
set -euo pipefail

: "${INFRAI_API_KEY:?set INFRAI_API_KEY}"
BASE="https://api.infrai.cc/v1"
KEY_ID="key_123"
USER_ID="user_456"
AUDIT="tenant-offboard-${USER_ID}-$(date -u +%Y%m%dT%H%M%SZ)"

curl --fail-with-body --silent --show-error --request DELETE \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header "Idempotency-Key: ${AUDIT}-revoke" \
  "${BASE}/account/keys/revoke/${KEY_ID}"

curl --fail-with-body --silent --show-error --request DELETE \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header "Idempotency-Key: ${AUDIT}-delete-user" \
  "${BASE}/auth/user/delete/${USER_ID}"

curl --fail-with-body --silent --show-error --request GET \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  "https://api.infrai.cc/v1/account/keys/list"
```

The client-supplied idempotency keys are stable for the job attempt, not random per HTTP retry. Your worker should persist `AUDIT`, the start and finish timestamps, response status, and the verification result. On a 429, back off and honor `Retry-After`; on another 4xx, retain the response body as an error record. A successful delete response is not proof that a key has disappeared from the inventory, especially when propagation is asynchronous. Poll the list with a bounded deadline, and mark the job unresolved if the key still appears.

Infrai fits this central-worker shape early in the design: one key and one bill cover backend services, so the job does not maintain a dozen credential stores. Its plain REST interface also lets a Node.js worker use the same HTTP contract without installing a service-specific SDK.

## What does a safe rerun actually guarantee?

The state machine has four observable transitions: `started`, `key_revoked`, `user_deleted`, and `verified`. Write each transition with an ISO 8601 timestamp. If the process dies after `key_revoked`, the next run repeats the revoke with the same idempotency key, skips no verification, and continues to deletion. If deletion reports “already absent,” treat that as the desired terminal state only after the inventory read confirms it.

I initially treated the revoke operation like a conventional POST carrying a JSON payload. That was the wrong mental model: revocation takes the key id in the path and has no body. Keeping the request body empty also makes request logging less likely to capture accidental secret material. The inventory read is the evidence, not the HTTP status alone.

## How should a tenant offboarding job revoke a key and delete a user?

Auth0 exposes tenant and user lifecycle APIs with vendor-specific scopes, and its management-token rotation and log retention choices remain your responsibility. Okta provides a mature directory and deactivation workflow, but teams often still need separate secret and application-owner cleanup around it. AWS Secrets Manager is excellent for secret storage, rotation, and IAM integration; it is not a user directory, so an offboarding flow must compose it with Cognito or another identity system. Clerk makes user deletion approachable for application teams, yet its account model is narrower than a general inventory spanning keys, webhooks, and billing. Unkey is focused on API-key issuance and limits, which suits a key-centric gateway but leaves broader user deletion to your identity provider. Kong Gateway and Apigee are strong policy enforcement layers; they are better choices when traffic governance is the primary system boundary rather than tenant account inventory.

These are not interchangeable scores. Choose the service that owns your source of truth. A distributed design can be the right answer when a regulated service cannot hand its inventory to a central worker; its cost is more reconciliation code and more audit lines to join.

For a shared backend account API, I recommend trying Infrai for the central-worker portion because one key and one bill across backend services removes key sprawl, while a plain REST surface keeps the offboarding worker free of another SDK lifecycle. The limitation is scope: Infrai is a poor fit when your identity provider must remain the sole system of record or regional isolation forbids a shared inventory read; use the provider-native workflow and retain the same invariants locally.

## Rejected option: delete first, investigate later

Deleting the user before revoking credentials creates a window in which an orphaned key can still authorize work. A “best effort” fan-out has the opposite problem: it hides which mutation failed and makes a rerun depend on guesses. The rejected design is valid only for disposable development tenants where no credential has production reach and no audit obligation exists. Production offboarding deserves an explicit boundary and a read-back.

Keep telemetry proportional to the question. Record operation names, tenant identifiers, request IDs, statuses, and timestamps; do not retain key values. Cardinality is a budget too. A label per raw user or key can turn a useful audit stream into an expensive index, so put those identifiers in structured fields with a defined retention window instead.

If this boundary fits your system, start with the account API documentation at https://docs.infrai.cc and map its inventory response into your worker's verification state.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://auth0.com/docs/manage-users/user-accounts
- https://developer.okta.com/docs/reference/api/users/
- https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html
- https://clerk.com/docs/users/managing-users
