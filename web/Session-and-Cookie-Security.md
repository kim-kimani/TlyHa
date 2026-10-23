# Session and Cookie Security

Sessions are where authentication becomes persistent state, and persistent state is where things leak, get fixed, get reused, and fail to expire. The bugs here are rarely flashy; they are the quiet foundation under account takeover.

## The mental model

A session is a secret the server issues that stands in for "you already proved who you are." Everything that touches that secret matters: how it is generated (predictable?), where it lives (cookie flags, storage, URL), how it is bound (to IP, device, nothing?), how it is invalidated (logout, password change, idle, absolute), and whether it can be fixed or swapped by an attacker before or during login.

## Where it hides / tell-tales

- **Session not rotated on privilege change**: the session id stays the same across login, from anonymous to authenticated. That is the precondition for session fixation. Set a victim's session to a known value (via a settable cookie, a URL parameter, or a subdomain cookie), get them to log in, and you now share their authenticated session.
- **No server-side invalidation**: logout clears the cookie client-side but the server still honors the old session id. Capture the cookie, log out in the browser, replay the cookie: still works. Same test for password change and "log out all devices."
- **Cookie scope too broad**: `Domain=.target.com` means a cookie set by any subdomain is sent to all of them. A low-trust subdomain (a marketing site, a user-content host) can set or read session-adjacent cookies. This is the cookie-injection precondition for fixation and for several cache and CSRF tricks.
- **Missing flags**: no `HttpOnly` (XSS can steal it), no `Secure` (sent over cleartext), no or wrong `SameSite` (CSRF). `SameSite=Lax` still sends on top-level GET navigations, which matters for GET-based state change and some OAuth flows.
- **Predictable or low-entropy tokens**: sequential, timestamp-derived, short, or encoding guessable content. Collect several, look for structure. If it is a signed blob (`value.signature`), it may be a JWT or a framework session you can attack offline.
- **Session data stored client-side**: signed cookies (Flask, Rails, Express `cookie-session`) carry the session payload in the cookie. If the signing secret is weak or leaked, you forge arbitrary session state (including `admin: true`). Flask sessions are base64 and readable; decode to confirm, then try to crack the secret.
- **Token in the URL**: session id or token in the query string leaks via `Referer`, logs, history, and shared links.

## Real material

A Flask session cookie is readable without the secret, which tells you exactly what to forge:

```
session=eyJsb2dnZWRfaW4iOnRydWUsInVzZXIiOiJhbGljZSJ9.ZaBcD.signature
# first segment base64 decodes to {"logged_in":true,"user":"alice"}
```
If the `SECRET_KEY` is weak or in a leaked `.env`/repo, `flask-unsign --unsign` cracks it and `--sign` lets you mint `{"user":"admin","is_admin":true}`.

Session fixation via a subdomain-settable cookie:

```
1. Attacker sets cookie on victim's browser from a sibling subdomain:
   Set-Cookie: session=ATTACKER_KNOWN_VALUE; Domain=.target.com; Path=/
2. Victim visits app.target.com and logs in.
3. If the app does not rotate the session id on login, the authenticated
   session is now ATTACKER_KNOWN_VALUE, which the attacker also holds.
```

Logout-does-not-invalidate test, mechanically:

```
1. Log in, capture Cookie: session=X
2. Click logout in the browser.
3. Replay a request with Cookie: session=X from a separate client.
   If it returns authenticated content, server-side invalidation is missing.
```

## Beyond the basics

- **The invalidation matrix**: test every invalidation path independently: explicit logout, password change, password reset completion, email change, "log out other sessions," role downgrade, and account deletion. A very common high-severity bug is that password reset does not kill existing sessions, so an attacker who already had a session keeps it after the victim "secures" the account.
- **Idle vs absolute timeout**: many apps have neither, or only idle. A token that never absolutely expires plus any one-time leak equals permanent access.
- **Concurrent session handling**: some apps bind a session to one device but fail open when two use it; others leak whether an account is logged in elsewhere (an oracle).
- **Cookie parsing quirks and injection**: cookie values that are not quoted/encoded can allow injection of additional attributes or splitting. Combined with a `Domain=.target.com` scope and a subdomain you control, you can overwrite or shadow the real session cookie. Duplicate cookie names (`__Host-` vs non-prefixed, or two `session=` cookies) resolve differently across the proxy, the framework, and the browser, which is an exploitable discrepancy.
- **`__Host-` and `__Secure-` prefixes**: their absence on sensitive cookies is a tell that cookie-injection defenses are not in place. `__Host-` forbids `Domain` and requires `Secure` and `Path=/`, which blocks subdomain injection; if the app does not use it where it should, note it.
- **Cross-site via `SameSite` gaps**: `SameSite=None` requires `Secure` and opens full cross-site sending; `Lax` still allows top-level GET. Map which cookies carry auth and what their `SameSite` is before planning CSRF.

## What to watch for

- Clearing a cookie in the browser is not proof of server invalidation; always replay the raw cookie from a separate client.
- A long random cookie is not necessarily opaque; check for a signed-payload format before assuming you cannot forge it.
- Some "session fixation" is actually harmless because the value is regenerated; confirm the post-login id equals the pre-login id before claiming it.

## Hunting checklist

- Compare the session id before and after login (fixation) and after privilege changes.
- Replay captured cookies after logout, password change, and reset to test server-side invalidation.
- Decode cookies for signed-payload formats (Flask, Rails, JWT); attempt to crack the secret if weak.
- Check all cookie flags: `HttpOnly`, `Secure`, `SameSite`, `Domain` scope, `__Host-`/`__Secure-` prefixes.
- Test idle and absolute timeout, and multi-device invalidation.
- Look for session identifiers in URLs, `Referer` leakage, and subdomain cookie injection.
