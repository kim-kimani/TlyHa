# Account Takeover Methodology

Account takeover (ATO) is the outcome, not the bug. It is where many smaller findings converge, so experienced hunters attack the account lifecycle as a system and ask, at every stage, "how could someone else end up in control here." This file is the synthesis that ties the auth-related topics together into a hunting plan aimed squarely at owning accounts.

## The mental model

An account is controlled by whoever holds a valid credential or session for it. ATO happens anywhere the system hands that control to the wrong person: at registration (claim someone's identity), at login (bypass or brute force), at recovery (steal the reset), at linking (attach your login to their account or theirs to yours), at session level (fixate, steal, or forge a token), or at the profile level (change the email/phone that controls recovery). Map the lifecycle, then probe each transition.

## The lifecycle, with the ATO question at each stage

### Registration

- **Pre-registration / account pre-hijack**: register an account with the victim's email before they do; when they later sign up (often via SSO), they may inherit or merge into your account, or you retain access. Variants: unverified-email registration, then the victim "claims" it; or you set a password on a victim's half-created SSO account.
- **Email/identifier confusion**: `victim@gmail.com` vs `victim+x@gmail.com`, dots in Gmail, unicode/homoglyph normalization, trailing dots/spaces, and case differences that the app treats as the same account in one place and different in another.

### Login

- **Auth bypass and brute force**: response manipulation, SQL/NoSQL injection in login, weak lockout/rate limits (see `Rate-Limiting-and-Anti-Automation.md`), and credential stuffing where no protection exists.
- **MFA bypass**: the whole `MFA-Bypass.md` surface (session issued before MFA, code not bound to user, alternate login paths skipping MFA).

### Recovery (the richest ATO surface)

- **Reset-link theft**: host-header poisoning, `Referer` token leak, token in response, weak/predictable or JWT reset tokens (see `Password-Reset-and-Account-Recovery.md`).
- **Token not bound to account**: submit your valid reset token with the victim's id/email.
- **No session invalidation on reset/password change**: attacker's pre-existing session survives the victim "securing" the account.
- **OTP brute force / reuse** on SMS/email recovery.

### Account linking / SSO

- **OAuth/OIDC linking flaws**: auto-link by email (unverified), code/token substitution, `redirect_uri` theft, `id_token` not validated (see `OAuth-and-OIDC.md`). Link your social login to the victim's local account, or the reverse.
- **Merge logic**: merging accounts by a shared identifier that you can set.

### Session

- **Fixation**: set a known session before login that persists (see `Session-and-Cookie-Security.md`).
- **Token theft via XSS** (same-origin in-session actions even under HttpOnly), **forgeable JWT/signed cookie** (see `JWT-Attacks.md`), and **session not invalidated** after logout/reset.

### Profile (the control-plane of the account)

- **Email/phone change without re-auth or verification**: change the recovery address, then reset at leisure. The single most direct ATO once you have any foothold (XSS, CSRF, IDOR).
- **Mass assignment** of `email`/`role`/`verified` (see `Broken-Access-Control.md`).
- **IDOR that writes to another user's profile** (set their email to yours), or that reads their reset token/session.

## Real material

Chained ATO patterns experienced hunters look for:

```
A) Enumeration -> host-header reset poisoning -> no session invalidation
   = durable takeover that survives the victim's password change.

B) Self-XSS -> login/clickjacking CSRF to land it on the victim -> in-session
   "change email" call -> password reset to the new email = full ATO.

C) OAuth: register victim@email locally (unverified) -> victim "Sign in with Google"
   auto-links to your seeded account = you are in their account.

D) IDOR on PATCH /users/{id} with {"email":"attacker@evil"} on the victim's id
   -> reset to your email = ATO.

E) Response/flag manipulation at MFA step -> session already valid = bypass.
```

The "change email" request is the keystone; find it and check what it requires:

```http
POST /api/account/email HTTP/1.1
Cookie: session=<victim-foothold>
Content-Type: application/json

{"email":"attacker@evil.tld"}       # does it require the current password? re-auth? email confirm?
```

## Beyond the basics

- **Attack the lifecycle, not one endpoint**: the biggest ATOs chain a weak link in one stage with a weak link in another (enumeration + reset poisoning + no invalidation). Diagram the whole flow for the target and test every transition and every identifier transformation.
- **The email-change endpoint is the crown jewel**: once you have any foothold (XSS, CSRF, a session, an IDOR), changing the recovery email/phone converts it to full, persistent ATO. Always check whether sensitive changes require re-authentication and out-of-band confirmation; frequently they do not.
- **Identifier normalization bugs are quietly powerful**: differences in how email/username are canonicalized across registration, login, and recovery let you create collisions and confusion that map your credential onto their account. Test Gmail dots/plus, unicode, case, and trailing whitespace across all three stages.
- **Session invalidation is where "secure" accounts stay compromised**: verify that logout, password change, password reset, email change, and "log out all devices" each kill existing sessions server-side. A gap here turns a temporary leak into permanent access.
- **Pre-hijacking is underrated**: several large sites were vulnerable to account pre-hijack variants; test registering ahead of a victim and what happens when they later sign up or SSO-link.
- **Prove impact cleanly**: the gold-standard PoC is taking over a second account you created, via a chain that would work on any user, without touching real victims.

## What to watch for

- Confirm the chain actually yields access to the victim account, not just a suggestive response; demonstrate end to end on your own two accounts.
- Many links are individually low; the report value is the full ATO chain, so document each step and the trust boundary it breaks.
- Recovery and email-change testing can send real emails/SMS; use your own addresses and avoid spamming.
- Never take over real users' accounts; prove with controlled test accounts.

## Hunting checklist

- Diagram the full account lifecycle: registration, login, MFA, recovery, SSO/linking, session, profile.
- At each transition, ask how control could pass to the wrong person; map to the relevant topic file.
- Test identifier normalization across registration/login/recovery for collisions.
- Find the email/phone-change endpoint; check for required re-auth and confirmation.
- Verify session invalidation on logout/password-change/reset/email-change.
- Build and demonstrate the end-to-end ATO chain on your own accounts.
