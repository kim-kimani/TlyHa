# Host Header Attacks

The `Host` header (and its forwarding cousins) is attacker-controlled input that applications treat as trusted context: the site's own domain. When code builds links, cache keys, password-reset URLs, or routing decisions from it, you control those outputs. It is a small, often-missed input with outsized reach.

## The mental model

The server receives `Host` from the client and typically assumes it is the real site name. But the client sets it, and behind proxies the backend may instead trust `X-Forwarded-Host`, `X-Forwarded-Server`, `X-Host`, or an absolute URL in the request line. Anywhere the app reflects "its own" hostname into a security-relevant output, you redirect that output to your domain by lying about the host. The classic impact is stealing password-reset links; the broader impact is cache poisoning, SSRF-ish routing, and auth bypass.

## Where it hides / tell-tales

- **Password reset and email links built from the host**: the single highest-impact case (see `Password-Reset-and-Account-Recovery.md`). The reset email links to `Host`, so a poisoned host sends the token to you.
- **Absolute URLs in responses**: canonical tags, `og:` meta, redirects, API `self` links, and script/asset `src` built from the host.
- **Cache key vs routing mismatch**: the cache keys on the real host while the backend reflects an unkeyed `X-Forwarded-Host` into content, enabling poisoning (see `Web-Cache-Poisoning-and-Deception.md`).
- **Virtual-host routing and access control**: the backend routes or authorizes based on `Host`; a spoofed host reaches an internal vhost, a different tenant, or an admin virtual host.
- **The forwarding-header family is trusted**: when `Host` is validated, the backend still honors `X-Forwarded-Host` / `X-Original-URL` / `X-Rewrite-URL`.

## Real material

Basic host override and the forwarding-header family:

```http
POST /password/forgot HTTP/1.1
Host: evil.tld
...
{"email":"victim@target.com"}
```
```http
POST /password/forgot HTTP/1.1
Host: app.target.com
X-Forwarded-Host: evil.tld
X-Forwarded-Server: evil.tld
X-Host: evil.tld
Forwarded: host=evil.tld
...
```

Variations that slip past naive `Host` validation:

```
Host: app.target.com:@evil.tld
Host: app.target.com evil.tld            (duplicate/space)
Host: evil.tld
X-Forwarded-Host: evil.tld
Absolute URL in request line:
GET https://evil.tld/password/forgot HTTP/1.1
Host: app.target.com
Two Host headers (front-end reads one, back-end the other)
```

Routing/access bypass via vhost:

```http
GET /admin HTTP/1.1
Host: internal-admin.target.local      (reach an internal virtual host from outside)
```

## Beyond the basics

- **Password-reset poisoning is the money finding**: test the whole forwarding-header family on the reset endpoint and read the resulting link; programs accept this because the token genuinely leaks (see the reset file for the full chain, including when `Host` is pinned but `X-Forwarded-Host` is not).
- **Proxy/back-end disagreement**: the front-end validates `Host` but the back-end trusts a forwarding header, or they parse duplicate/space/`@`-laden hosts differently. This parser differential is the same mindset as smuggling and open-redirect (see `Open-Redirect-and-URL-Parsing.md`); the front door and the app disagree about the hostname.
- **Cache poisoning via unkeyed host headers**: if `X-Forwarded-Host` is reflected into an absolute URL (a script src, a redirect) and is unkeyed, you poison the shared cache so every visitor loads your host. One of the most reliable cache-poisoning primitives.
- **SSRF and internal routing**: a spoofed `Host`/`X-Forwarded-Host` can route to internal vhosts, trigger the backend to fetch using the attacker host, or satisfy a "trusted internal hostname" check. Combine with the SSRF file.
- **Web cache deception and canonical abuse**: poisoned canonical/`og:` tags can be used for SEO poisoning and to make shared links point at attacker content.
- **Authentication and tenant decisions on Host**: multi-tenant apps that pick the tenant from the host can be pushed to the wrong tenant; some apps gate "internal only" features on an internal `Host` value you can simply send.

## What to watch for

- Confirm the spoofed host actually reaches a security-relevant output (a link, a cache entry, a routing decision); a reflected host in a harmless place is not impactful.
- When `Host` is validated, the win is usually in a forwarding header; test the full family, since the backend often trusts one you did not expect.
- Reset-link poisoning requires the victim to click; frame impact honestly, but it is widely accepted.
- Do not poison production caches beyond minimal proof, and use your own accounts for reset tests.

## Hunting checklist

- On the password-reset endpoint, test `Host` and every forwarding header; read the generated link.
- Find absolute-URL reflections (canonical, og, redirects, asset src) and try to point them at your host.
- Test forwarding headers when `Host` is validated; try duplicate/space/`@`/absolute-URL forms.
- Check for cache poisoning via unkeyed `X-Forwarded-Host`.
- Probe vhost routing and `Host`-based access/tenant decisions.
