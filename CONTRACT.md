# WS outreach: contract between Winning Statics and Closefast (WS instance)

Owner decisions are in DECISIONS.md. Copy is in COPY.md. Engine rules are in RULEBOOK.md.
Both sides build to this file. Change it only in one place and tell the other side.

## Identity

- `lead_ref`: Closefast's lead id, a string. Every call and every event carries it.
- `arm`: `A` (product) or `B` (managed, done for you).
- Shared secret: `~/.ws_outreach_shared_key` (hex, mode 600). Set it as a Worker secret on both
  sides (`OUTREACH_KEY` on WS, `WS_OUTREACH_KEY` on Closefast). Never commit it.
- Requests WS -> Closefast and Closefast -> WS both send
  `x-outreach-ts: <unix seconds>` and `x-outreach-sig: hex(HMAC-SHA256(key, ts + "." + rawBody))`.
  Reject if the signature is wrong or the timestamp is more than 300 s off.

## Closefast -> WS

### POST https://api.winningstatics.com/outreach/gifts
Creates the free ads for a lead who said yes. Idempotent on `lead_ref` (a second call returns
the existing gift, never a second set of ads).

```json
{ "lead_ref": "cf_123", "arm": "A", "store_domain": "lumea.com", "email": "sarah@lumea.com",
  "first_name": "Sarah", "company_name": "Lumea", "product_url": "https://lumea.com/products/glow-serum" }
```
`product_url` is optional: the lead may name a product in the reply; otherwise WS picks the
best-seller. Response `202`:
```json
{ "gift_id": "g_abc", "status": "generating", "page_url": "https://app.winningstatics.com/gift/<token>",
  "short_url": "https://winningstatics.com/for/sarah-lumea" }
```

### GET https://api.winningstatics.com/outreach/gifts/:gift_id
```json
{ "gift_id": "g_abc", "status": "generating|ready|needs_review|failed",
  "page_url": "...", "short_url": "https://winningstatics.com/for/sarah-lumea",
  "product": { "title": "Glow Serum", "url": "..." },
  "ads": [ { "index": 0, "url": "https://api.winningstatics.com/g/<token>/0.png", "preview_url": "https://api.winningstatics.com/g/<token>/0.jpg",
             "ratio": "4:5", "kind": "new", "concept": "iMessage thread" },
           { "index": 2, "url": "https://api.winningstatics.com/g/<token>/2.png", "preview_url": "https://api.winningstatics.com/g/<token>/2.jpg",
             "ratio": "4:5", "kind": "improved", "concept": "your ad, pour into use",
             "source_ad_url": "https://www.facebook.com/ads/library/?id=...", "change_note": "bigger hook, product moved up" },
           { "index": 5, "url": "https://api.winningstatics.com/g/<token>/5.png", "preview_url": "https://api.winningstatics.com/g/<token>/5.jpg",
             "ratio": "9:16", "kind": "tweak", "concept": "your change", "parent": 0, "note": "make it 9:16 for stories" },
           { "index": 6, "url": "https://api.winningstatics.com/g/<token>/6.png", "preview_url": "https://api.winningstatics.com/g/<token>/6.jpg", "ratio": "4:5", "kind": "new",
             "concept": "receipt flat lay", "product": { "title": "Night Mask", "url": "https://lumea.com/products/night-mask" } } ],
  "tweaks": [ { "tweak_id": "tweak1", "ad_index": 0, "note": "make it 9:16 for stories", "ratio": "9:16",
                "status": "generating|ready|failed", "new_ad_index": 5, "reason": "only when failed" } ],
  "tweaks_left": 2, "extras_left": 3,
  "video": { "status": "none|queued|ready", "url": null } }
```
`needs_review` is left for hard failures only (store unreadable, no product, no image from either image provider):
Closefast escalates, never sends. Every ad gets 3 tries and the best one ships, so QA notes never hold a gift.
Arm A gets 5 ads (2 new + 3 improved options of their best live static, each a different augmentation device; 5 new
if they run no statics). Arm B gets the same 5 statics plus a video slot queued for manual production
(`video.status = queued`).

- `short_url` is the link to put in emails: `https://winningstatics.com/for/{first}-{brand}`, lower case words joined by
  hyphens, no random characters. The brand is the store's short name (Brickell Men's Products is `brickell`). With no
  first name it is `/for/{brand}`; a collision takes a word suffix (`-ads`, `-gift`, `-picks`). It answers 302 to
  `page_url` and keeps `?code=BFCM10`. The gift guard and events still key on the gift, never the slug.
- `index` is the ad's position in `ads`; it never changes once delivered. Extras and tweaks are appended.
- `url` and `preview_url` are on our own domain (`https://api.winningstatics.com/g/<token>/<index>.png|.jpg`, never a
  storage host), public, `cache-control: public, max-age=31536000, immutable`. Safe to put in cold emails.
- `preview_url`: a JPEG about 560 px wide (show it at 280 px), under 120 KB. Use it inline in emails; link the image to
  `page_url`. `null` only if the preview could not be made (fall back to `url`).
- `ratio`: `4:5` for every ad except a tweak that asked for `9:16` or `1:1`.

### POST https://api.winningstatics.com/outreach/gifts/:gift_id/delivered
Make-first outreach (23 Oct 2026): Closefast creates the gift before the opener goes out, and calls this once the email
carrying the gift link has been sent. Body, optional: `{ "at": "2026-10-23T10:00:00Z" }` (the send time; now when absent).
Response `200 { "gift_id": "g_abc", "delivered_at": "..." }`; a second call answers the first time with `"already": true`.
Only a delivered gift blocks its store and email from the main site's Free plan and trial. GET /outreach/gifts/:id
carries `delivered_at` (null until then).

### POST https://api.winningstatics.com/outreach/gifts/:gift_id/extra
One more new ad (gift-ghost follow-up 1, or "make some for my other product"). Body, optional:
```json
{ "product_url": "https://lumea.com/products/night-mask" }
```
`product_url` must be a page of the gift's own store; the new ad is made for that product. Response `202`: the GET
shape with `status: "generating"`; `gift_ready` fires again when the ad lands (`data.added` names its index).
Errors: `400 bad_product_url`, `409 busy` (`retry_after: 60`, the gift is still generating), `409 extra_limit`
(3 per gift), `409 gift_failed`.

### POST https://api.winningstatics.com/outreach/gifts/:gift_id/tweak
A lead asks for a change to one delivered ad. WS remakes that ad with the change, keeping everything else true to the
store (best of 3 tries, Replicate as the image fallback), and appends it as a new ad; the original stays.
```json
{ "ad_index": 0, "note": "make the headline red and add 20% off", "ratio": "9:16" }
```
- `ad_index`: the `index` of the ad to change (any ad, including an earlier tweak).
- `note`: the lead's instruction, 3 to 500 characters, in their words (text, colour, offer, size and so on). An offer
  or price the lead writes is used as written; nothing else is invented.
- `ratio`: optional, `4:5|9:16|1:1`. Without it WS reads the note ("9:16", "story", "reels", "vertical" -> `9:16`;
  "square" -> `1:1`), else keeps the ad's own.

Response `202`: the GET shape with `status: "generating"` plus `"tweak_id": "tweak1"`. When it lands, `gift_ready`
fires again with `data: { "ads": 6, "extra": true, "added": [5], "added_kinds": ["tweak"] }` and the new ad is
`ads[5]` (`kind: "tweak"`, `parent: 0`). If no image could be made, the gift goes back to `ready` unchanged and
`tweaks[].status` is `failed` with a `reason` (no `gift_ready`).
Errors: `400 bad_ad_index | bad_note | bad_ratio | bad_json`, `409 busy` (`retry_after: 60`), `409 tweak_limit`
(3 per gift), `409 gift_not_ready`, `404 gift_not_found`.

### Make-first (owner 23 Oct): gifts made before the opener
Closefast now creates the gift about 1 to 2 hours before a lead's opener is due, before any yes. The create body
carries `"make_first": true`. For such a gift WS must:
- NOT flag the store domain and email for the Free plan / trial guard at creation;
- NOT queue the arm B video at creation (`video.status` stays `none`);
- still generate the 5 statics and answer GET as usual (`ready`, `needs_review`, `failed`).

Two signed calls follow (same HMAC headers; idempotent; answer `200`, or `404 gift_not_found`):

`POST https://api.winningstatics.com/outreach/gifts/:gift_id/accepted`
```json
{ "lead_ref": "cf_123", "at": "2026-10-23T10:00:00Z" }
```
The lead said yes. Arm B: queue the sample video now (`video.status = queued`), exactly as a non-make-first
gift did at creation.

`POST https://api.winningstatics.com/outreach/gifts/:gift_id/delivered`
```json
{ "lead_ref": "cf_123", "at": "2026-10-23T10:05:00Z" }
```
The ads were emailed to the lead. Flag the store domain and email for the guard NOW (not before).
Closefast also sends `delivered` for gifts created the old way (on a yes); treat it as a no-op there.
Closefast retries a non-2xx every 10 minutes and posts a FAILED alert, so a missing route shows up at once.

## WS -> Closefast

### POST https://<closefast-ws host>/api/hooks/ws
```json
{ "event": "gift_ready|gift_page_view|ad_download|plans_click|booking_click|signup|checkout_started|trial_started|purchase|video_ready|cancel",
  "lead_ref": "cf_123", "gift_id": "g_abc", "at": "2026-10-23T10:00:00Z",
  "data": { "plan": "growth", "cadence": "month", "amount_cents": 19900, "currency": "USD", "discount_code": "BFCM10" } }
```
- `purchase` and `booking` stop all outreach for that lead (RULEBOOK).
- Retries: WS retries non-2xx with backoff for 24 h. Closefast dedupes on `(lead_ref, event, at)`.
- `video_ready` (arm B sample or a paying customer's weekly video) carries `data.url`.

## Gift page (WS)

`https://app.winningstatics.com/gift/<token>`: public, no login.
1. Their ads at the top, full size, free to download, no watermark.
2. Paid plans only at the new prices (PRICING in DECISIONS.md). No Free plan, no trial.
3. `?code=BFCM10` pre-applies the 10% code. The code is shown only when present.
4. Secondary: "Rather talk it through?" -> https://cal.com/farooq-magicfit/30min (tracked as booking_click).
5. Checkout creates the account; the gift ads land in their library.

Guard: the store domain and email of every gift are flagged. The main site's Free plan and
trial refuse them (a polite "you've already had your free ads" with the paid plans).

## Discount

Dodo code `BFCM10`: 10% off the first 3 months of any monthly plan (or 10% off a yearly plan),
valid until 2026-12-01, outreach use only.
