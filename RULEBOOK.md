# WS outreach rulebook (Closefast WS instance)

## Global rules
- One email to a person every 2 days at most, counted across every lane and inbox.
- A real reply ends whatever follow-up ladder is running. Out of office and support-desk
  auto-replies are not real replies.
- Weekdays only, inside 9:00 to 17:00 in the lead's timezone (country, else store currency,
  else US Eastern). No sends on Sunday in any timezone.
- Before every opener: re-check the store's Meta ads (insight must still be true), the email
  is not suppressed, the store is not a WS customer, and no other engine has the address.
- Every outgoing message passes the style gate (COPY.md caps, no dashes, no invented numbers or
  links, no refund promises, one question max, correct greeting and sender name). A failure goes
  to the approval queue, never out.
- Purchase, booked call, opt-out, or escalation stops every lane for that lead immediately.
- Manual takeover from the dashboard stops every path that sends, including Instantly.
- Instantly campaign: one step, delay 0, `{{personalizedSubject}}` / `{{personalizedBody}}`.
  Check the campaign is active (status 1) before every opener. Never add a step 2 in the UI.
  Every follow-up and reply goes out through the reply API so these rules apply.

## Reply categories and actions
| Category | Examples | Action | Then |
|---|---|---|---|
| yes | "sure", "go for it", "send them", names a product | Create the gift (CONTRACT). Arm A: send ads when ready. Arm B: send statics, then the video when uploaded | Gift-ghost lane |
| yes_with_details | "yes, but do the hoodie" / "use our new logo" | Pass the product or note to the gift, confirm in one line | Gift-ghost lane |
| question | "what's the catch", "who are you", "how much", "is this AI" | Answer from the brief in the owner's voice. "is this AI": yes, made with AI and checked by us | Reply-ghost lane |
| objection | "we have an agency", "we do it in house", "too busy" | Answer once, re-offer the free ads with no strings | Reply-ghost lane |
| price_or_haggle | asks for a discount or a custom deal | Arm A: BFCM10 is the only discount the agent can give. Arm B: offer the call | Negotiation, no round cap |
| forward | "talk to my marketer", cc's someone | Thread follows the store: thank them, address the new person | As if they replied |
| not_now | "after BFCM", "January" | Send not-now copy, set a dated wake-up (default 2026-12-02, or the date they gave) | Re-enters cold lane on the date with a January angle |
| hard_no | "not interested", "no thanks" | Send the hard-no copy once (main site link, 5 free ads) | Stops. No follow-ups |
| opt_out | "stop", "unsubscribe", "remove me", "don't email" | Send nothing. Suppress the address and the store domain everywhere | Never contacted again |
| where_from | "how did you get my email" | Reply with the where-from line | Reply-ghost lane |
| data_request | "delete my data", "what data do you hold" (GDPR or CCPA) | Suppress, escalate | Owner handles |
| ooo | out-of-office auto-reply | Pause, resume the day after the return date | Same lane continues |
| auto_ticket | "we received your request #1234", helpdesk bot | Ignore | Same lane continues |
| bounce | hard bounce | Suppress the address | Lead closed |
| hostile | abuse, legal threats | Suppress, escalate, send nothing | Owner handles |
| escalate | contracts, custom terms, partnership or agency offers, anything after an arm B call is booked, classifier confidence under 0.8 | Stop the thread, alert the owner | Waits for the owner |

## Lanes
| Lane | Trigger | Cadence | Ends |
|---|---|---|---|
| Cold | Opener sent, no reply | 3 follow-ups, 2 days apart (COPY) | After 3, lead marked unresponsive |
| Gift ghost | Ads delivered, no reply | 3 follow-ups, 2 days apart: extra ad, BFCM10 code, last nudge | After 3, then nurture |
| Reply ghost | They asked something, we answered, they went quiet | 3 nudges, 2 days apart, each answering what they asked or offering the ads again | After 3 |
| Negotiation | Haggling or objection back-and-forth | Reply to every message, no cap | Hard no, or the ask is beyond the agent's authority (escalate) |
| Clicked, didn't buy | Gift page plans_click or checkout_started without purchase | +1 day, +3 days | After 2, or purchase |
| Nurture | Gift ghost finished | Every 14 days until 2026-12-31: a fresh ad, a new angle | Opt-out, or the end date |
| Not now | Dated wake-up | Re-enters cold with a new angle | As cold |

## Approvals
- The first 5 gift deliveries (positive replies) wait in the approval queue and alert the
  owner. After 5 approvals the engine sends gift deliveries on its own.
- Replies in the question, objection and negotiation lanes are sent on autopilot from day one
  when the classifier confidence is at least 0.8 and the style gate passes; otherwise they wait.

## Arm B specifics
- On yes: create the gift (4 statics + a queued video). Send the statics as soon as they're
  ready. The video is made by hand; Closefast sends it when WS posts `video_ready`.
- If the video isn't ready 30 hours after the yes, alert the owner (never promise a new time
  on its own).
- After the video: every message asks for the call. A booked call stops outreach and alerts
  the owner with a call brief (store, ads, insight, what they said).

## KPIs (dashboard overview, with an arm A / arm B split everywhere)
Sent, delivered, bounced, replies, reply rate, yes replies, yes rate, reply categories,
gifts created, gifts delivered, median yes-to-delivery time, gift page views, ad downloads,
plans clicks, booking clicks, calls booked, signups, checkouts started, trials, purchases,
conversion yes-to-purchase, revenue and MRR, BFCM10 redemptions, cost (ads generated, model
calls) per lead and per customer, unsubscribe rate, escalations open, per-inbox health
(sent today, bounces, replies), per-insight reply rate, per-lane state counts.
