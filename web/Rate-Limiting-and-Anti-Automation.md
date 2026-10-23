# Rate Limiting and Anti-Automation

Rate limiting is the control that makes brute force, enumeration, OTP guessing, coupon abuse, and scraping impractical. It is also one of the most commonly broken controls, because it is bolted on at one layer and bypassed at another. An experienced hunter treats "is there a limit, and can I evade it" as a question to ask on every sensitive action.

## The mental model

A rate limit counts actions per identifier (IP, account, token, device) over a window and blocks past a threshold. Every bypass is either making your actions look like they come from different identifiers, hitting a path the limiter does not cover, or doing more per request than the limiter counts. The impact is unlocking whatever the limit was protecting: credentials, OTPs, discount codes, private-data enumeration.

## Where it hides / tell-tales

- **Limit keyed on a spoofable identifier**: counting by `X-Forwarded-For`, a client-supplied device id, or a header you control. Rotate it and the counter resets.
- **Limit on one path but not its siblings**: the web login is limited, the mobile/API/`v1` login is not; the UI OTP check is limited, a different endpoint that also verifies the OTP is not.
- **Client-side or soft limits**: the throttle is enforced in JS or only returns a warning; the server still processes requests.
- **Per-window resets you control**: re-requesting an OTP resets the attempt counter; starting a new session/flow gives a fresh budget.
- **Batching defeats per-request counting**: GraphQL alias batching or array batching runs many operations in one HTTP request the limiter counts once (see `GraphQL.md`).
- **Case/format path variants dodge the matcher**: `/Login`, `/login/`, `/login%20`, adding a query string, or an extra path segment the limiter keys on literally.

## Real material

Identifier spoofing (the most common bypass):

```http
POST /api/login HTTP/1.1
X-Forwarded-For: 1.2.3.4          # rotate per request
X-Forwarded-For: 1.2.3.5
X-Real-IP: ...   X-Client-IP: ...   X-Originating-IP: ...   True-Client-IP: ...
Forwarded: for=1.2.3.6
```

Path and method variants that share the backend but not the limiter:

```
/api/login   vs  /api/v1/login   vs  /mobile/login
/login  vs  /login/  vs  /LOGIN  vs  /login?x=1  vs  //login
POST vs PUT; JSON body vs form body
```

GraphQL batching to brute force in one counted request (see the GraphQL file):

```graphql
mutation { a:login(u:"v",p:"1"){t} b:login(u:"v",p:"2"){t} c:login(u:"v",p:"3"){t} }
```

OTP counter reset by re-requesting:

```
1. Request OTP. 2. Guess 5 times (limit hit). 3. Request a NEW OTP (counter resets).
4. Guess 5 more. Repeat -> unlimited guesses across the code space.
```

## Beyond the basics

- **Enumerate every path to the same action**: the richest bypass is finding the unlimited sibling. Map web vs mobile vs partner APIs, versioned endpoints, and alternate verbs; one of them almost always missed the limiter. This ties to `REST-API-Security.md`.
- **Spoof the identity the limiter trusts**: if it keys on `X-Forwarded-For` (common behind proxies that pass it through), rotate it; if it keys on account, rotate accounts/tokens; if it keys on a device header, rotate that. Try the whole forwarded-IP header family.
- **Reset-the-window tricks**: re-requesting OTPs/codes, re-initiating the flow, logging out/in, or rotating the session token to get a fresh per-session budget. A limit you can reset on demand is not a limit.
- **Counting-unit abuse**: do more per request than is counted: batch (GraphQL/REST bulk), multi-value parameters, or an endpoint that processes an array of items with one "request." The limiter sees one; you did a hundred.
- **Race the threshold**: fire many requests in parallel (single-packet, see `Business-Logic-and-Race-Conditions.md`) so they pass the check before the counter updates; useful when the limit is checked non-atomically.
- **Distributed-but-cheap**: where allowed and in scope, many source IPs (cloud, proxies) defeat per-IP limits; note that this is often against program rules, so prefer a logical bypass that proves the control is structurally weak.
- **CAPTCHA and challenge bypass**: CAPTCHA tokens that are reusable, not bound to the action, validated only client-side, or solvable/relayed; a CAPTCHA that can be replayed or skipped is a rate-limit bypass. Check whether the token is one-time and server-verified.
- **Impact framing**: a rate-limit bypass is rarely the finding by itself; it is the enabler. Pair it with the thing it unlocks (credential stuffing success, OTP brute force to account takeover, coupon/gift-card draining, private-data enumeration) for a real report.

## What to watch for

- Confirm the backend actually processes the extra requests (a `200` that still says "invalid" counts); some apps fail open on the limiter but still enforce the real check.
- A reset response might rotate the secret (new OTP invalidates old guesses); confirm your guesses still apply to the current code.
- Spoofing `X-Forwarded-For` only works if the app trusts it over the real socket IP; verify it changes the counter.
- Brute forcing live accounts/OTPs can lock real users and may violate program rules; use your own accounts and demonstrate the bypass minimally rather than fully exploiting.

## Hunting checklist

- For each sensitive action, find every path to it (web/mobile/API/versioned/verb variants); test each for limits.
- Try identifier spoofing: the full forwarded-IP header family, account/token/device rotation.
- Try window resets: re-request codes, restart flows, rotate sessions.
- Try counting-unit abuse: GraphQL/array batching, multi-value params.
- Test CAPTCHA token reuse/binding/server-verification.
- Pair any bypass with the concrete impact it unlocks.
