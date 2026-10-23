# MFA Bypass

MFA is a second gate bolted onto a first gate that already works. The bugs are almost never in the cryptography of the OTP; they are in the state machine around it, the places the app forgets the second gate exists, and the recovery paths that let you walk around it.

## The mental model

After a password check, the server enters a "pending MFA" state and should refuse to issue a full session until the second factor is proven. Every bypass is one of: the server issues a usable session before MFA (state machine flaw), the MFA check can be brute forced or skipped (enforcement flaw), the factor can be enrolled or reset by the attacker (recovery flaw), or a different entry point to the same account does not require MFA at all (coverage flaw).

## Where it hides / tell-tales

- **The response after password already contains a session**: log in with password, and before entering the code, inspect the response and cookies. If a working session token is issued at the "enter your code" step, the second factor is cosmetic. Try hitting an authenticated endpoint directly.
- **Forced browsing past the MFA page**: the MFA prompt is a frontend route; the backend already considers you authenticated. Navigate straight to `/dashboard` or call an API endpoint.
- **MFA flag is client-controlled**: the verify response or a cookie contains `mfa_passed=false` or `requires_mfa=true` that the client echoes back; flip it.
- **No rate limit on the code**: 6-digit TOTP is a million guesses, but with no limit and no lockout, and a 30 to 90 second window plus skew, it is brute forceable. Watch for per-session limits you can reset by restarting the flow, or limits enforced only on the UI.
- **Code not bound to the user or session**: request a code for your account, then submit it in the victim's pending-login session. Or the same code works for multiple accounts.
- **Backup/recovery codes weak**: short, sequential, low-entropy, not rate limited, or generated predictably.
- **Enrollment without verifying the current factor**: you can add a new TOTP secret or phone number to an account you partially control without proving the existing factor.
- **Alternate login paths skip MFA**: OAuth "Sign in with Google," the mobile app, an older API version, a "remember this device" cookie, SSO, or password reset. Any path that logs you in without the second factor is the bypass.
- **Password reset clears MFA**: completing a reset drops the MFA requirement, so steal-the-reset chains to full bypass.

## Real material

The "session already issued" tell. The verify step is a formality:

```http
POST /api/login HTTP/1.1
Content-Type: application/json

{"username":"victim","password":"correct"}
```
Response sets `Set-Cookie: session=...` and body `{"mfa_required":true}`. Now test:

```http
GET /api/account/settings HTTP/1.1
Cookie: session=<the one just issued>
```
If that returns the victim's data, MFA never gated the session.

Client-trusted MFA state:

```json
// verify response
{"status":"ok","mfa":{"verified":false,"method":"totp"}}
// subsequent request the client sends; flip verified to true
{"action":"getProfile","mfa_verified":true}
```

Code-not-bound-to-victim (cross-session OTP):

```
1. Attacker has partial creds for victim (password known, MFA unknown).
2. Attacker triggers their OWN account's OTP in a parallel session.
3. Attacker submits that OTP into the victim's pending-MFA session.
   If the server only checks "is this a currently-valid OTP" without
   binding it to the account, it passes.
```

"Remember device" cookie that is forgeable or reusable:

```
Set-Cookie: mfa_device=eyJ1c2VyIjoidmljdGltIiwidHJ1c3RlZCI6dHJ1ZX0=
# base64 {"user":"victim","trusted":true}  -> unsigned, swap the user
```

## Beyond the basics

- **Map every authentication entry point first**: web, mobile, desktop, SSO, OAuth, API tokens, legacy endpoints, password reset. MFA is usually enforced on the main web login and forgotten on at least one of the others. The forgotten one is the finding.
- **Response manipulation**: change the verify response from `{"success":false}` to `{"success":true}` with an intercepting proxy if the client makes the trust decision. Sounds naive; still works on SPAs that gate on the response body.
- **Status-code and step confusion**: some flows advance on any non-error; send a malformed code and see if it fails open, or replay the "mark as verified" request out of order.
- **OTP autofill and leakage**: the code echoed in a response, sent to a logged third party, or returned by a sibling endpoint (`/api/mfa/debug`, a `resend` endpoint that returns the code in JSON).
- **Brute force within tolerance**: TOTP validators accept a window of codes (previous/next step) for clock skew, slightly widening the guess space. Combine with no rate limit for a realistic attack.
- **Enrollment and downgrade**: add a weaker second factor (SMS over TOTP, email over app) if the app lets you pick, then attack the weaker channel. Or enroll your own factor onto the victim via an unprotected enrollment endpoint.
- **Backup-code enumeration**: if the login accepts both TOTP and backup codes at the same prompt without distinguishing, you can throw backup-code guesses at the TOTP endpoint.

## What to watch for

- A `200` on a wrong code is not a bypass until you confirm you reached authenticated state; some apps return `200` with `{"success":false}`.
- "Remember device" working on your own device is expected; the bug is when the cookie is forgeable, not account-bound, or portable to another device.
- Brute forcing a live user's OTP on a real program can lock their account; stick to your own test accounts and note the enforcement behavior rather than hammering.

## Hunting checklist

- Enumerate every login path; test each for MFA enforcement, especially mobile, OAuth, SSO, and legacy APIs.
- After the password step, inspect for an already-usable session; try authenticated endpoints and forced browsing.
- Test rate limiting and lockout on the code, including whether restarting the flow resets the counter.
- Test whether the code is bound to the account (cross-session submission).
- Inspect the verify response for client-trusted MFA flags; flip them.
- Check "remember device" cookies for forgeability and portability.
- Test enrollment of a new factor without proving the existing one, and whether password reset clears MFA.
