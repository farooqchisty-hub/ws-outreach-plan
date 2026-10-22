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
{ "gift_id": "g_abc", "status": "generating", "page_url": "https://app.winningstatics.com/gift/<token>" }
```

### GET https://api.winningstatics.com/outreach/gifts/:gift_id
```json
{ "gift_id": "g_abc", "status": "generating|ready|needs_review|failed",
  "page_url": "...", "product": { "title": "Glow Serum", "url": "..." },
  "ads": [ { "url": "https://.../1.png", "kind": "new", "concept": "iMessage thread" },
           { "url": "https://.../3.png", "kind": "improved", "source_ad_url": "https://www.facebook.com/ads/library/?id=...",
             "change_note": "bigger hook, product moved up, BFCM offer added" } ],
  "video": { "status": "none|queued|ready", "url": null } }
```
`needs_review` means the quality gate failed after all retries: Closefast escalates, never sends.
Arm A gets 5 ads (2 new + 3 improved options of their best live static, each a different augmentation device; 5 new if they run no statics). Arm B gets the same 5
statics plus a video slot queued for manual production (`video.status = queued`).

### POST https://api.winningstatics.com/outreach/gifts/:gift_id/extra
One more new ad for the gift-ghost follow-up 1. Returns the same shape with the new ad appended.

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
