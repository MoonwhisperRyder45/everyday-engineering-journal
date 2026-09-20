# Logistics Image OCR: One Key for Storage, Moderation, and a Single Bill

Short answer: use one immutable object key per captured image, and derive every storage, OCR, moderation, and billing record from it. Keep the original only as long as its evidentiary value justifies the bytes; keep small derivatives and decision metadata longer. This joins accounting without forcing a single service or a single image format.

A warehouse camera can produce three photographs of the same pallet: a wide frame for context, a label crop for OCR, and a redacted copy for review. If each stage invents its own identifier, the bill becomes a guessing exercise. Treat it as a ledger key, not a URL.

## What does the bill actually measure?

Storage is usually the dominant term because pixels persist after a job finishes. A useful first estimate is:

`monthly bytes = captures × average encoded bytes × retained copies × retention days / 30`

Suppose 2,000,000 captures average 1.8 MB, with one original and two derivatives. Holding all three for 30 days is about 10.8 TB before replication and metadata. OCR compute may be visible on an invoice, but deleting one derivative or reducing its quality changes the larger term: retained bytes.

I initially treated moderation as a separate pipeline. That created duplicate uploads and made a late policy review expensive. The correction was to store a compact decision envelope beside the object: policy version, confidence, timestamp, and a pointer to the same key. The envelope is cheap; the pixels are not.

Quality and bandwidth pull in opposite directions. JPEG at a lower quality factor can cut transfer size, yet a small barcode or embossed serial may disappear. WebP and AVIF can reduce bytes for supported clients, while a lossless format remains useful for a narrow evidence set. MDN documents the format trade-offs, but the operational choice belongs to the OCR error budget, not to a format popularity contest.

Use a two-lane design. Send a bounded, high-quality crop to OCR, and retain the original only when a capture is flagged, disputed, or legally relevant. For routine captures, retain a review-sized derivative and its checksum. A failed OCR result should trigger a controlled re-encode or recapture request, not an automatic fan-out of five new copies.

## How can one key connect moderation and accounting?

Make the key content-addressed or otherwise collision-resistant, and treat it as immutable. The object path, event records, and invoice dimensions all carry it. A small event schema is enough:

```json
{
  "image_key": "sha256:9d3f...",
  "capture_id": "dock-17-2026-04-08T09:14:22Z-0042",
  "derivative": "ocr-crop",
  "bytes": 184320,
  "ocr_status": "needs-review",
  "moderation_status": "clear",
  "policy_version": "2026-01"
}
```

Never put mutable status in the key. Status belongs in append-only events so a recheck can explain which policy made which decision. For a generic object store, a signed upload request can carry the key and an explicit content type:

```bash
curl -X PUT https://media.example.invalid/upload \
  -H 'Content-Type: image/jpeg' \
  -H 'X-Image-Key: sha256:9d3f...' \
  --data-binary @label.jpg
```

The endpoint is illustrative; the invariant is the header-to-record mapping. Validate the reported byte count and checksum after upload.

## What do we stop keeping, and how do we audit the loss?

Retention should be a policy table, not a blanket number. Keep OCR crops for the period needed to resolve shipment exceptions. Keep moderation evidence longer only when a rule or contract requires it. Drop intermediate thumbnails, repeated failed encodings, and raw request bodies.

That choice has a cost. When an operator disputes a result after deletion, you may have the checksum and decision trail but not the pixels. I accept that loss for routine, low-risk captures because the alternative is paying indefinitely for forensic capacity that is rarely used. Sample the retained originals by route, camera, and failure class; sampling makes the blind spot measurable instead of pretending it does not exist.

Measure OCR recall on the smallest derivative that preserves the characters your workflow needs. Then attach a byte ceiling to each stage and alert on cardinality, not just total volume: unique image keys, derivative types, and policy versions. High-cardinality labels can multiply observability storage faster than the images themselves.

This is the unglamorous control that keeps the design honest.

If recall falls below the operational threshold, spend bandwidth on the crop or a selective original re-fetch. If recall holds, shorten derivative retention. The single key gives you a clean join across those experiments, while independent storage, OCR, and moderation implementations remain replaceable.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://www.w3.org/TR/trace-context/
- https://www.rfc-editor.org/rfc/rfc9110
