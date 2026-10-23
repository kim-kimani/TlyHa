# CSRF (Cross-Site Request Forgery)

CSRF was supposed to be dead once `SameSite=Lax` became a browser default. It is not. The living bugs are in token flaws, method and content-type tricks, `SameSite` gaps, login/logout CSRF, and the many endpoints that opt out of protection.

## The mental model

CSRF makes a victim's browser send an authenticated request the attacker composed. It works because the browser attaches cookies automatically to requests to the target, regardless of who triggered them. The defense is to require proof the request originated from the real app (a CSRF token, or `SameSite` cookies, or an origin check). Every CSRF bug is a hole in that proof.

## Where it hides / tell-tales

- **No token, relying on cookies alone**: any state-changing request with no anti-CSRF token and `SameSite=None` (or no `SameSite`, which older browsers treat as None) is a candidate.
- **Token present but not validated**: remove the token, send an empty token, send another user's token, or reuse a stale token. Many apps check only presence, not correctness or binding.
- **Token not bound to the session**: a global token, or a token that validates for any user, lets the attacker fetch one and reuse it.
- **Method-based gaps**: protection on `POST` but not `PUT`/`DELETE`/`PATCH`, or the endpoint accepts `GET` for a state change (`?action=delete`).
- **Content-type exemptions**: the framework enforces CSRF only on `application/json` but the endpoint also accepts `application/x-www-form-urlencoded` or `text/plain`, which a cross-site HTML form can send without a preflight.
- **`SameSite=Lax` gaps**: `Lax` still sends cookies on top-level `GET` navigations. A GET-based state change, or a flow that starts with a top-level navigation (some OAuth/login), is still reachable cross-site. Also, a freshly set cookie has a ~2 minute window where some browsers treat it leniently.
- **Login and logout CSRF**: forcing a victim to log into the attacker's account (then they enter data the attacker sees) or logging them out; often overlooked because "it does not steal anything."
- **CORS does not stop CSRF**: CORS governs reading responses, not sending requests. A write that ignores the response (most state changes) is unaffected by CORS. People confuse the two.

## Real material

A classic form-based CSRF (works when no token and cookies auto-sent):

```html
<form action="https://app.target.com/account/email" method="POST" id="f">
  <input name="email" value="attacker@evil.tld">
</form>
<script>document.getElementById('f').submit()</script>
```

Content-type bypass: JSON endpoint that also accepts form or `text/plain`. A simple request (no preflight) can carry a JSON-looking body via `text/plain`:

```html
<form action="https://app.target.com/api/profile" method="POST"
      enctype="text/plain">
  <input name='{"email":"attacker@evil.tld","x":"' value='y"}'>
</form>
```
The body becomes `{"email":"attacker@evil.tld","x":"=y"}`, which a lenient JSON parser accepts.

GET-based state change (still works under `SameSite=Lax` via top-level navigation):

```html
<img src="https://app.target.com/account/delete?confirm=yes">
<!-- or force a top-level nav: -->
<a href="https://app.target.com/transfer?to=attacker&amt=1000" ...>  (victim clicks)
window.location = 'https://app.target.com/transfer?to=attacker&amt=1000'
```

Token-not-validated test matrix:

```
1. Send the request with no CSRF token field at all.
2. Send with an empty token.
3. Send with a token from the attacker's own session.
4. Change the token's last character.
5. Swap POST for GET or PUT.
```

## Beyond the basics

- **Chaining self-XSS and CSRF**: a login CSRF logs the victim into the attacker's account, where attacker-controlled data triggers self-XSS that now runs in the victim's browser session. Two "lows" become a real finding.
- **`SameSite` reality check**: do not assume `Lax` closes everything. Enumerate which auth cookies are `Lax` vs `None`, and which sensitive actions use `GET` or top-level navigation. The default protects same-site POST, not GET navigation, not `SameSite=None` cookies, and not subdomains in some configurations.
- **Token-fixation and leakage**: if the CSRF token is reflected in a place an attacker can read (a `GET` response without CORS protection, a referrer leak, a cached page), the token defense collapses. Also test whether the token is the same as a predictable value.
- **Clickjacking as a CSRF cousin**: when a sensitive action lacks `X-Frame-Options`/`frame-ancestors`, frame the real app and trick the victim into clicking the genuine button, which carries their real token. This bypasses token defenses entirely because the victim's own browser supplies the token. Check framability on every state-changing page.
- **Flash/legacy and WebSocket CSRF (CSWSH)**: WebSocket handshakes are plain HTTP and often skip CSRF and origin checks; a cross-site page can open an authenticated socket. See `WebSockets.md`.
- **Multi-step and double-submit flaws**: double-submit cookie defenses break if an attacker can set a cookie on the victim (subdomain cookie injection, see `Session-and-Cookie-Security.md`), because they then control both the cookie and the body token.
- **CORS misconfig turning CSRF into full read**: if the app reflects `Origin` into `Access-Control-Allow-Origin` with credentials, the attacker can both send the request and read the response, including the CSRF token, defeating the token and enabling data theft. See `CORS-Misconfiguration.md`.

## What to watch for

- Modern browsers default to `SameSite=Lax`, so pure cross-site POST CSRF without a token is less common; prove the cookie actually attaches cross-site before claiming it.
- A missing token is not a bug if `SameSite` protects the cookie and there is no GET/navigation path; map the cookie attributes first.
- Login/logout CSRF is real but lower severity; frame impact honestly and look for the chain that raises it.
- Do not perform irreversible actions on real accounts to "prove" CSRF; use your own victim account.

## Hunting checklist

- For each state-changing endpoint, check for a token and validate it is bound and checked, using the test matrix.
- Enumerate auth cookie `SameSite` values; find GET/navigation-based sensitive actions.
- Try content-type downgrades (`text/plain`, form-encoded) on JSON endpoints.
- Test method swaps (`GET`/`PUT`/`DELETE`).
- Check framability for clickjacking on sensitive pages.
- Look for CORS reflection that leaks the token; check WebSocket handshakes for origin checks.
- Consider login/logout CSRF and self-XSS chains.
