# Overlooked and Low-Hanging Bugs

The smallest things get reported least, which is exactly why they are still there. This is a catalog of the easy-to-miss, quick-to-test bugs that experienced hunters sweep for on every target: individually small, frequently chainable, and occasionally the whole finding. Treat it as a rapid pass to run alongside the deeper topic files.

## The mental model

Every app has dozens of tiny trust assumptions that nobody wrote a test for: a header that is not set, a parameter that is reflected, a value that is trusted, a flow that is not re-checked. None are glamorous, but each is a real bug, and they combine. The habit is a fast, wide sweep: touch every small thing, note the anomalies, and look for the ones that chain into something bigger.

## The catalog (each is a quick test)

### Tokens, links, and leakage
- **Secrets in `Referer`**: sensitive tokens (reset, OAuth code, API key) in a URL leak to third-party scripts and external links via the `Referer` header. Check reset and callback pages for third-party includes.
- **Tabnabbing**: `target="_blank"` without `rel="noopener noreferrer"` lets the opened page rewrite `window.opener.location` to a phishing page. Check external links.
- **Tokens in URLs**: session ids, API keys, and tokens in query strings leak via history, logs, and `Referer`.
- **Verbose errors and stack traces**: one leaky endpoint reveals the stack, paths, and versions (see `Secrets-and-Information-Disclosure.md`).
- **Autocomplete and cached sensitive pages**: `Cache-Control` missing on authenticated pages so they persist in shared caches/proxies (ties to `Web-Cache-Poisoning-and-Deception.md`).

### Cookies and headers
- **Missing cookie flags**: no `HttpOnly`/`Secure`/`SameSite` on session cookies; broad `Domain` scope (see `Session-and-Cookie-Security.md`).
- **Cookie bombing/overflow**: setting many/large cookies on a shared parent domain to break a victim's access to the site (a DoS via cookies), possible when a subdomain can set parent-domain cookies.
- **Missing security headers as chain enablers**: no `X-Frame-Options`/`frame-ancestors` (clickjacking), weak/missing CSP (XSS impact), missing `X-Content-Type-Options: nosniff` (content sniffing to XSS on uploads).

### Reflection and content-type
- **Reflected content without execution**: a reflected value that is not XSS today but lands in a JSON/JS/attribute context elsewhere; always check other contexts.
- **Content sniffing**: an endpoint returning user content with the wrong/missing `Content-Type` and no `nosniff`, letting the browser render it as HTML (stored XSS from a non-HTML endpoint, see `File-Upload.md`).
- **SVG/HTML in "image"/"document" fields**: uploads or avatars that render as markup.
- **PDF/HTML email injection**: user input rendered into generated PDFs or HTML emails (XSS in the PDF viewer, link/CSS injection in emails).

### Parameters and logic
- **HTTP Parameter Pollution (HPP)**: `?a=1&a=2` where the app and a filter/WAF disagree on which wins; can bypass validation or change behavior.
- **Hidden/debug parameters**: `?debug=true`, `?test=1`, `?admin=1`, `?format=json`, `?callback=` unlocking behavior (see `../00-methodology/Recon-and-Attack-Surface-Mapping.md`).
- **Mass assignment of small flags**: `verified`, `newsletter`, `role`, `beta` (see `Broken-Access-Control.md`).
- **Numeric/boundary quirks**: negative, zero, huge, and decimal values in quantity/amount/limit fields (see `Business-Logic-and-Race-Conditions.md`).
- **IDOR on the unglamorous object**: a notification, an export, a preview, a thumbnail, a `.pdf` variant (see `IDOR-and-BOLA.md`).

### Identity and email
- **Email normalization**: `+` tags, Gmail dots, unicode/homoglyph, trailing dot/space treated inconsistently across registration/login/recovery (see `Account-Takeover-Methodology.md`).
- **Username/display-name injection**: markup/emoji/RTL-override characters in names that render in other users' views or admin panels (second-order XSS, UI spoofing).
- **Unicode and case folding**: `ß`/`ss`, Turkish `I`/`ı`, and NFC/NFKC normalization that collapses distinct inputs into the same value in auth/uniqueness checks.

### Redirects and navigation
- **Open redirect**: low alone, chains into OAuth/SSRF/token leak (see `Open-Redirect-and-URL-Parsing.md`).
- **Self-XSS**: low alone, chains with CSRF/clickjacking to become stored/victim XSS.
- **CSV/formula injection** in exports (see `LDAP-XPath-and-Less-Common-Injections.md`).

### Rate and automation
- **No rate limit on cheap/sensitive actions**: OTP, login, invite, email send, search (see `Rate-Limiting-and-Anti-Automation.md`).
- **User/account enumeration**: differential responses/timing on login, registration, and reset, which feeds every other attack.

## Real material

A few one-liners to sweep with:

```
# tabnabbing: find external target=_blank without rel=noopener in the HTML/JS.
# HPP: send duplicated params and diff behavior:
/search?q=a&q=b         /transfer?to=me&to=victim
# content sniffing: upload text with no/x-wrong Content-Type and no nosniff; request it.
# email normalization collision across flows:
register: victim+x@gmail.com   login/reset: victim@gmail.com
# referer leak: load the reset page; check Network tab for third-party requests
#              carrying the token in Referer.
# cookie scope: can a subdomain set Domain=.target.com cookies the main app reads?
```

## Beyond the basics

- **Chaining is where the value is**: almost every item here is low on its own and real when combined. Open redirect + OAuth = token theft. Self-XSS + clickjacking = stored XSS. Email normalization + SSO linking = ATO. Missing `nosniff` + an upload = stored XSS. Cookie injection from a subdomain + double-submit CSRF = CSRF bypass. Train yourself to ask "what does this small thing unlock."
- **Second-order is the common thread**: names, filenames, metadata, and log lines that are inert where you inject them but dangerous where they are later rendered (admin panels, exports, log viewers, emails, PDFs). Plant canaries and hunt the render sites.
- **The sweep is cheap, run it everywhere**: these tests take seconds each; make them a reflex on every endpoint and every input. The payoff is both the occasional standalone finding and the chain components that upgrade your bigger bugs.
- **Mobile and API share these**: the same small bugs appear in mobile backends and APIs, often less guarded (see `../mobile/` and `REST-API-Security.md`).

## What to watch for

- Low-severity findings are easy to over-report; lead with impact or a chain, and group minor hygiene issues rather than filing each alone, to match program expectations.
- A missing header is only a bug if something exploits its absence; pair it with the attack it enables.
- Confirm reflections and normalization collisions actually produce an effect before claiming them.
- Prove on your own accounts; avoid spamming email/SMS and avoid DoS-style cookie bombing beyond minimal proof.

## Hunting checklist

- Sweep every page for missing security headers and cookie flags, and pair each gap with its attack.
- Check external links for tabnabbing; check reset/callback pages for `Referer` token leaks.
- Test HPP, hidden/debug params, and small mass-assignment flags.
- Test content-type sniffing and markup in image/document/name fields.
- Test email/username normalization collisions across registration, login, and recovery.
- Note every low finding as a potential chain component and look for the upgrade.
