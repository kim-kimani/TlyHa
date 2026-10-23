# Business Logic and Race Conditions

No scanner finds these. They are bugs in what the application is allowed to do, not in how it parses input. You find them by understanding the business, modeling the intended flow, and then doing something the developer assumed nobody would. The payoff is often money, free goods, or privilege, with clean-looking traffic.

## The mental model

Every feature encodes assumptions: steps happen in order, each happens once, values stay within ranges, a check and the action it guards are atomic, and the client follows the happy path. Logic bugs break an assumption about sequence, quantity, identity, or timing. Race conditions specifically break the "check then act" atomicity: between the check and the act, you squeeze in a second operation.

## Where it hides / tell-tales

- **Multi-step flows**: checkout, signup, KYC, onboarding, password reset, transfers. Skip a step, repeat a step, reorder steps, resume an expired step, or run two instances concurrently.
- **Anywhere a limited resource is counted**: balance, coupon uses, gift-card value, inventory, invites, API quota, votes, loyalty points. If the check (is there enough?) and the decrement (take some) are not atomic, parallel requests each pass the check before any decrement lands.
- **Price, quantity, and currency on the client**: price sent from the client, negative quantities, quantity of zero, currency swap to a weaker one, discount stacking, decimal/rounding abuse (`0.001` that rounds to free), integer overflow on totals.
- **State flags you can set early**: `paid=true`, `verified=true`, `status=approved`, `trial=false` set or echoed by the client, or reachable via a step that should be gated.
- **Idempotency and replay**: replaying a "confirm payment," "apply coupon," or "redeem" request; the lack of an idempotency key means the action repeats.
- **Different limits on different paths**: the web UI enforces a max transfer, the mobile or API path does not; one flow verifies ownership, a parallel flow does not.

## Real material

The classic limit-overrun race (apply one coupon many times, or withdraw a balance twice). Fire N identical requests in parallel so they all pass the "check" before any "decrement":

```http
POST /api/wallet/withdraw HTTP/1.1
Content-Type: application/json

{"amount": 100, "to": "attacker"}
```
Send 50 of these simultaneously (single-packet attack: send all with the last byte withheld, then release together, so they hit the server in the same window). If the balance check is not atomic with the debit, you withdraw more than you had.

Client-side price/quantity tampering:

```json
{"items":[{"sku":"TV-55","price":1,"qty":1}]}      // price trusted from client
{"items":[{"sku":"TV-55","qty":-1}]}               // negative qty credits you
{"couponCode":"SAVE10","couponCode2":"SAVE10"}     // stacking via duplicate fields
{"amount":"0.001","currency":"BTC"}                // rounding / currency confusion
```

Step-skip in a reset or purchase flow (jump straight to the final state-changing step):

```
Intended: /checkout/cart -> /checkout/pay -> /checkout/confirm
Attack:   POST /checkout/confirm   directly, with a cart that was never paid
```

Coupon/gift-card enumeration and reuse:

```
POST /api/coupon  {"code":"GIFT-0001"} ... brute the space; or redeem the same
one-time code twice via a race.
```

## Beyond the basics

- **Model the economy first**: write down every way value enters, moves, and leaves the system, and every counter. The bug is usually where two of those interact (refund plus re-purchase, cancel plus keep-the-goods, transfer plus reverse). Think like someone trying to get something for free, not like someone sending payloads.
- **The single-packet attack for races**: HTTP/2 lets you send many requests in one TCP packet, eliminating network jitter so they arrive truly simultaneously. This makes narrow race windows (microseconds) exploitable where sequential requests would not collide. This is the modern upgrade over "fire a burst with a thread pool."
- **Races beyond money**: double-spend of a one-time token (use a reset/invite/coupon twice), bypassing uniqueness (register two accounts with the same email in the gap between check and insert), 2FA/limit resets, promoting yourself while a downgrade is in flight, and uploading-then-using a file before the AV deletes it (see `File-Upload.md`).
- **TOCTOU on authorization**: check ownership, then act, with a window to change ownership in between. Or validate a value, then re-read it from a mutable store that you changed concurrently.
- **Partial-completion and rollback gaps**: force an error midway through a multi-write transaction so some writes persist and others roll back, leaving an inconsistent state in your favor (goods shipped, charge reversed).
- **Negative and boundary economics**: negative quantities as credits, refunds exceeding the purchase, applying a percentage discount to a negative line item, integer overflow turning a huge total into a small one, and time-based abuse (back-dating, timezone gaps, trial resets by changing the clock or the account's region).
- **Workflow authorization**: an action valid for role A in state X performed by role B or in state Y; approvals you can self-approve; a "request" that is auto-granted; a reversible action (cancel) that does not reverse the benefit.

## What to watch for

- Logic bugs are easy to misjudge; confirm the real-world effect (money moved, goods granted, privilege gained) rather than a suggestive response.
- Race conditions are probabilistic; repeat to confirm, and establish the baseline (does it ever happen without concurrency?) so you are not fooled by a flaky backend.
- Many apps use database constraints or locks that quietly prevent the race; a failed attempt does not mean the logic is sound, but a successful double-effect does mean it is broken.
- These often move real money or inventory; use minimal amounts, your own accounts, and reverse anything reversible; get explicit authorization before value-moving tests on a live program.

## Hunting checklist

- Map the full economy and every counter/limit; list how value enters, moves, and exits.
- For each multi-step flow, try skip, repeat, reorder, resume-expired, and concurrent execution.
- Test every limited resource with the single-packet parallel attack for non-atomic check-then-act.
- Tamper client-controlled price, quantity (including negative/zero), currency, and discount stacking.
- Look for replayable actions lacking idempotency keys.
- Compare limits and checks across web, mobile, and API paths for the weakest one.
