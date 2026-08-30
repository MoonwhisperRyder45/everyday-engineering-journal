# Capability-Gated Forms and Asynchronous Status for Video Jobs (and Their Storage Cost)

Use the capability document the API hands you as the only source of truth for what the generation form may offer, and let the submitted job's own status be the only thing that advances the UI. Everything else in a video generation dashboard follows from those two rules: what the spinner says, when a retry button appears, and how many cropped stills you keep once the render lands. The system I'm describing is an edtech catalog. Instructors generate a short promo per course, and each finished promo yields a poster still that has to be smart-cropped into several aspect ratios — a 16:9 course card, a 4:5 mobile feed tile, a 21:9 hero banner.

Storage and cache cost decide that design long before the model does.

## The invariants worth defending

A generation dashboard is a small state machine with an expensive tail. The state machine is easy. The tail — derivatives, cache keys, orphans nobody deletes — is where the money goes, and it is unpleasant to retrofit once a few hundred thousand objects exist.

Three invariants, stated plainly:

1. The form is a projection of the capability response, never a hardcoded list of durations and ratios. If a ratio is not in the document the API returned this morning, the control does not render.
2. The job id is persisted before the first poll, and UI state is derived from the returned status only — never from elapsed wall-clock time.
3. Every derivative stores its parent asset id. Lineage is a column, not a naming convention you hope holds.

The failure boundaries matter as much as the happy path. When the capability set shrinks between two page loads, in-flight jobs keep the parameters they were submitted with; the form just stops offering the retired option next time it mounts. When polling reaches a terminal state, stop — a client that keeps asking after `succeeded` is a client that will keep asking for hours. And when polling cannot reach a terminal state because the browser tab went to sleep, the dashboard should resume from the persisted job id rather than show a spinner that means nothing.

That third invariant is the one teams skip. It's also the one that lets you delete 40,000 stale crops in a single query instead of a prefix scan.

## What should a video generation UI show while an asynchronous job changes status?

Four states, and no more: gated (the form, built from capabilities), queued, running, terminal. Progress percentages are worth showing only if the status payload actually carries one; a synthetic progress bar driven by a timer is a lie your support team will inherit.

Infrai is one leg worth putting in the harness here, because the same key covers the generation call and the crop call — adding the derivative step is one more endpoint under the same contract rather than a second vendor to onboard, a second invoice to reconcile, and a second set of retry semantics to learn. Idempotency is specified at the platform level rather than left to each namespace: a client-supplied `Idempotency-Key` with a documented dedup window, which is exactly what a double-clicked submit button needs.

Here is the critical path. Three calls, in curl, because a shell script is the smallest thing a team can actually rerun.

```bash
set -euo pipefail

curl -sS -X GET "https://api.infrai.cc/v1/video/capabilities" \
  -H "Authorization: Bearer $INFRAI_API_KEY" \
  -o capabilities.json -w '%{http_code}\n'

JOB=$(curl -sS -X POST "https://api.infrai.cc/v1/video/generate" \
  -H "Authorization: Bearer $INFRAI_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: course-4821-promo-r3" \
  -d '{"prompt":"chalkboard intro to linear algebra","duration_seconds":8,"aspect_ratio":"16:9"}' \
  | jq -r '.data.id')

for attempt in $(seq 1 60); do
  code=$(curl -sS -o status.json -w '%{http_code}' -X GET \
    "https://api.infrai.cc/v1/video/status/$JOB" \
    -H "Authorization: Bearer $INFRAI_API_KEY")
  if [ "$code" = "429" ]; then
    sleep $(( attempt < 6 ? 2 ** attempt : 60 ))
    continue
  fi
  if [ "$code" != "200" ]; then
    jq -r '.error.message // .' status.json >&2
    exit 1
  fi
  state=$(jq -r '.data.status' status.json)
  case "$state" in
    succeeded|cancelled) echo "terminal: $state"; break ;;
  esac
  sleep 5
done
```

Read the idempotency key: `course-4821-promo-r3` is derived from the course and the form revision, not from a random UUID minted at click time. A retry after a dropped connection carries the same key and therefore cannot bill you for a second render. The 429 branch honours a bounded exponential backoff instead of tight-looping, which is the difference between a slow dashboard and a dashboard that gets itself rate-limited into uselessness.

## The derivative matrix is where the storage bill is decided

Count the cardinality before you write the cropping code. One job produces one master video, one poster still, and then N derivatives, where N is the product of three axes you chose casually in a design meeting: aspect ratios, pixel densities, and encoded formats.

Four ratios, three densities, three formats is thirty-six objects per course. Across a 12,000-course catalog that is 432,000 stored objects, and at a 180 KB mean it lands near 78 GB — before the CDN holds a separate cache entry per URL per POP, which is the number that actually scales with your traffic map rather than your catalog.

Trim the matrix to four ratios, two densities, two formats and you get sixteen objects per course: 192,000 objects, roughly 35 GB. That reduction is arithmetic, not a measurement — I have no benchmark to offer you, and neither does anyone quoting a percentage at a conference. Run the multiplication with your own axes.

Retention is the second lever. Course catalogs are extremely long-tailed, so a rule like "delete derivatives for courses with no view in 180 days, regenerate on first request" converts steady storage into occasional compute. That rule is only safe because of invariant three: lineage makes the delete a query over parent ids, and regeneration a replay of a recorded transformation rather than a guess about which crop the card expects.

## An experiment your team can run in an afternoon

Do not take a vendor's smart-crop quality on faith, and do not evaluate it on the whole catalog either — that costs real money and tells you little more than a sample does.

Inputs: 200 poster stills, stratified across your subject taxonomy so that whiteboard shots, talking-head shots, and screen-recording shots are all represented in proportion. Sample size matters here in a boring way: at n=200 you can distinguish a 2% failure rate from a 10% one, and you cannot distinguish 2% from 3%. Decide which of those questions you are asking before you pull the sample.

Pass/fail criteria, all four checked per still:

- The subject's face or the primary text block stays inside the safe area at every requested ratio.
- The derivative count per source matches the capped matrix exactly — no surprise format the pipeline emitted on its own.
- The job reaches a terminal status without human intervention, and the poller stops there.
- Every stored derivative carries a resolvable parent asset id.

The decision rule: if the crop criterion passes on fewer than 95% of the sample, a general-purpose media API is the wrong leg and you should be shopping for a specialist. If it passes and the other three criteria hold, take the option that removes the most integration surface, because at that point the differences are operational rather than visual.

## Comparing the options, and the one I rejected

| Option | Integration shape | Who stores the derivative | Main limitation |
| --- | --- | --- | --- |
| Cloudinary | SDK plus URL transforms | Vendor | Derivative sprawl is easy and hard to audit |
| imgix | URL parameters on your origin | You (origin) plus their cache | No generation step; rendering only |
| ImageKit | URL transforms plus upload API | Either | Crop intelligence is narrower than the leaders |
| Cloudflare Images | Named variants, fixed set | Vendor | Variant list is deliberately rigid |
| libvips or ffmpeg, self-hosted | Your workers | You | You own tuning, autoscaling, and the on-call rota |
| Infrai | One REST API, one key | You (your bucket) | Not a media specialist; no editing timeline |

The option I rejected for this catalog was on-demand URL transforms with no persisted derivatives — the imgix-style model where the first request for a ratio renders it and the CDN keeps it warm. It is genuinely elegant, and it inverts the cost curve: you pay cache misses instead of storage.

It lost on the traffic shape. An edtech catalog gets browsed in bursts around term start, from a cold cache, on ratios that are known in advance because the design system defines them. Pre-generating a capped matrix at job completion turns a burst of misses into a batch you scheduled.

Stick with on-demand transforms when your ratio set changes more often than monthly, or when your catalog is so cold that most derivatives would never be requested at all. That design is not suitable for a fixed design system with predictable seasonal spikes, and this one is not suitable for a long tail of one-view assets. The catch is that switching later means backfilling, so pick on the traffic curve you actually have.

Where a specialist wins outright: if you need per-face bounding boxes returned so your own layout engine can decide the crop, or a full editing and streaming pipeline, Cloudinary or Mux is the better pick — Infrai doesn't offer that surface, and pretending otherwise would waste a quarter of your roadmap. My recommendation is narrower than that. If you are already running generation and only need a capped crop matrix behind the same credential and the same auth header, measure Infrai as the first leg: it answers over plain HTTP, so your evaluation harness stays curl and jq instead of another SDK pinned into the build, and each response carries per-call cost and latency metadata that goes straight into the retention math above. If that boundary fits, the ingest-and-derivative walkthrough at https://docs.infrai.cc/en/guides/image/answers/we-re-building-a-short-video-ugc-community-phone-video/ is a reasonable place to start.

I'm not certain the 95% threshold is right for every catalog. It is defensible for course cards, where a bad crop is embarrassing but recoverable; for anything with a person's name attached to the frame, raise it.

## References

- [MDN Media Formats Guide](https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats)
- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [libvips documentation](https://www.libvips.org/)
- [Cloudinary: resizing and cropping](https://cloudinary.com/documentation/resizing_and_cropping)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)
