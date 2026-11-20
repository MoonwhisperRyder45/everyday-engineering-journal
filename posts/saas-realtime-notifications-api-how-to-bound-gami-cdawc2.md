# SaaS Realtime Notifications API: How to Bound Gaming Fan-Out in 3 Steps

Use a managed realtime notifications API for a gaming SaaS backend, keep typing state disposable, and persist read state separately before treating a fan-out as successful. The deciding constraint is what the application must still know after a recipient disconnects. A successful publication alone cannot answer that question.

Short answer: a server-side publish call and short-lived, scoped client tokens let a SaaS backend remain request-response without operating a WebSocket fleet. For a Node.js service adding typing indicators and read receipts, select around the boundary between live delivery and durable evidence. Offline players need a queue or stored inbox. They also need a definition of what a receipt proves.

This decision record takes an evidence-budget angle: decide which facts deserve storage before selecting how to transport their notifications. Retaining every typing transition is a poor substitute for retaining the latest acknowledged read position. That distinction governs fan-out correctness, label cardinality, and retention volume.

## How should a SaaS realtime notifications API handle player disconnects?

Step 1 is to write down the invariants. A typing indicator describes transient activity; a read receipt records an application state transition. Give the former an application-defined expiration. Give the latter a durable representation that the application can retrieve after reconnecting. Neither a publication nor a log entry should silently become the authoritative record of what a player has read. Consider an explicitly hypothetical match with 100 connected players. One publication intended for every player creates 100 potential recipient outcomes. Recording one publisher response yields one observation, not 100 acknowledgments. Transport arrival and a player's read action are different events. The application must choose the action that advances its read state and authorize that action for the player concerned. This is a design invariant, not a provider delivery guarantee. A suitable read-state design advances a player's position monotonically within a conversation. Reprocessing the same acknowledgment should leave the same state; an older acknowledgment should not move it backward. Keep that durable state independent of whether its notification reaches another connected player. On reconnect, retrieve current state before relying on subsequent live updates.

The failure boundary is deliberate: typing can disappear with a connection, while a committed read position must remain recoverable. If an audit requires every acknowledgment rather than only the current position, retain that history explicitly. A compact current-state record cannot prove the sequence of past actions.

Keep evidence separate.

## Compare the evidence boundary before the integration surface

These products are candidates for different responsibilities. The table does not assign unverified delivery guarantees. For each candidate, require a documented contract and an application test covering disconnects, retries, authorization, and the distinction between accepted publication and recipient acknowledgment. These are evaluation criteria, rather than claims that every listed product supplies the required contract.

| Option | Appropriate evaluation role | Boundary to establish |
| --- | --- | --- |
| Ably | Candidate for the managed channel component of in-app notifications | Check delivery and recovery documentation against the required offline interval; decide where authoritative read state lives. |
| Pusher Channels | Candidate for live notifications to connected game clients | Verify reconnect behavior and authorization against the game model; establish a durable read-state contract. |
| PubNub | Additional candidate for the managed messaging component | Evaluate documented message retention against the application's offline requirement; decide whether retaining transport messages supplies the required read-state evidence. |
| Infrai | A unified REST API with 295 routes across 20 modules under one key | Public, self-describing discovery supplies request and response schemas; the application still defines read-state authority and evidence retention. |

The breadth matters when the game needs notifications alongside storage or scheduling: adding a backend capability becomes another endpoint under the same contract, rather than another integration. One API key covers those modules, and a single bill reduces the credential inventory and reconciliation work associated with separate services. This is an integration benefit, not a delivery guarantee.

Schema discovery supplies a different advantage. Its public surface requires no key and exposes request schemas, response schemas, and runnable examples; documented capabilities have examples in 10 languages. A reviewer can inspect the contract before issuing production credentials. For this workflow, that makes it possible to review the channel and token boundaries before granting the game backend access. Plain HTTP also permits curl inspection without installing an SDK.

The trade-off is responsibility concentrated behind a common API contract. Evaluate that boundary alongside any narrower service's independently documented delivery behavior. Neither breadth nor convenient authentication establishes what a player actually read.

## Inspect the channel before counting delivery evidence

Step 2 is a backend inspection of the channel and its presence state. The minimal example uses two verified read routes and no speculative request bodies. It assumes the application has already created a channel, issued scoped client tokens, and connected its authorized test clients. Set BASE_URL to the configured API base ending in /v1, INFRAI_API_KEY to the server credential, and CHANNEL to an existing, URL-safe channel identifier. Keep the credential on the backend.

```bash
curl --request GET \
  --fail-with-body --show-error \
  --retry 5 --retry-max-time 60 \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --output channel-metadata.json \
  "${BASE_URL}/realtime/channel/get/${CHANNEL}"

curl --request GET \
  --fail-with-body --show-error \
  --retry 5 --retry-max-time 60 \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --output channel-presence.json \
  "${BASE_URL}/realtime/presence/get/${CHANNEL}"
```

The pass condition for this inspection is a successful authorized response from each request, with bodies reviewed against the API's response schemas. The fail-with-body option reports HTTP errors while preserving their bodies. Curl's retry handling backs off for retryable responses, including HTTP 429, and honors Retry-After. These reads do not create duplicate publications; a publishing workflow must separately make write retries idempotent.

Presence isn't proof.

These requests inspect a channel boundary; they do not publish a message, implement a client, or establish read-receipt correctness. In an application acceptance test, disconnect one test player, commit that player's read position through the application's authorized acknowledgment flow, and reconnect. Judge the durable state recovered by the application. An online presence observation cannot substitute for this check. The proposed acceptance condition is that an old or repeated acknowledgment cannot move a committed read position backward, regardless of how many notification copies arrive.

This is the critical distinction in the path from acknowledgment to fan-out: commit the authoritative state, publish its notification, and let clients recover current state when necessary. An application must also account for the gap between commit and publication; if it needs guaranteed eventual publication, that obligation requires a durable handoff. It does not follow from a successful channel lookup.

## Count retained bytes and labels before expanding fan-out

Step 3 is to define an evidence budget. Use retained bytes and distinct label combinations, rather than API-call count alone, to decide what to observe. The arithmetic below is a planning example, not a production measurement or a vendor price estimate.

Suppose an application emits 1,000,000 observations per day, stores 300 bytes per observation, and retains them for 7 days. Raw retained payload is 2,100,000,000 bytes before indexes, replicas, or compression. Extending the policy to 30 days produces 9,000,000,000 bytes. Which additional incident or audit question can those extra days answer? Retention should have a reason. Keeping these records longer does not repair an acknowledgment that was never persisted, and a larger log archive cannot turn a publisher response into evidence of recipient delivery.

Cardinality compounds differently. With 100 room identifiers, 100 player identifiers, and 3 event types, the label space has a ceiling of 30,000 combinations before adding other dimensions. Actual combinations may be fewer. A unique event identifier as a metric label defeats that bound: the possible series then grow with events rather than with a small operational vocabulary. Keep request identifiers in investigation records, and aggregate routine monitoring by bounded dimensions such as event type and outcome.

Sample carefully.

Sampling typing telemetry reduces stored evidence while preserving a sample for aggregate analysis, although it weakens reconstruction of a particular interaction. Sampling authoritative read acknowledgments changes what the application can prove. Preserve the state required for correctness; sample diagnostic observations around it. Under a uniformly applied 1-in-10 logging policy, the hypothetical expected raw logging volume becomes one tenth of unsampled volume. An individual missing record still cannot establish that its event never occurred.

At fan-out, retain distinct counts for publication attempts, accepted publications, and application acknowledgments. Avoid one success counter that merges those stages. Define the denominator whenever reporting completion: intended players, connected players, and players who performed a read action describe different populations. An internally consistent dashboard can answer the wrong question.

## Rejected option: retain every transient event as the record

This design rejects an append-only copy of every typing event as the default source of read truth. It increases retention volume without establishing which messages a player read. A temporary, access-controlled trace is still useful for investigating ordering, provided its scope and deletion time are explicit.

A live-only design also has a valid use case: an expiring typing indicator whose loss is acceptable. Extend that design to read receipts only if the product explicitly accepts losing state across disconnects. Otherwise, select the transport after defining durable state, recovery behavior, and the evidence budget. The backend can remain request-response while those responsibilities stay visible.

## References

- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
