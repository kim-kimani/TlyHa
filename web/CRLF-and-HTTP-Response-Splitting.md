# CRLF Injection and HTTP Response Splitting

A carriage-return/line-feed in the wrong place lets you inject headers, split responses, poison caches, set cookies, and sometimes land XSS, all from a value that looks like it just goes into a redirect or a log. It is an easy bug to overlook because the payload is two invisible characters.

## The mental model

HTTP headers are separated by CRLF (`\r\n`), and a blank line (`\r\n\r\n`) separates headers from the body. If user input is reflected into a response header (a `Location`, a `Set-Cookie`, a custom header) without stripping CRLF, you inject your own header lines, and with a double CRLF you inject a whole response body. The same injection into request headers or logs has its own effects. It is header injection, and response splitting is its most powerful form.

## Where it hides / tell-tales

- **Input reflected into a response header**: redirect targets (`Location`), `Set-Cookie` values built from input, `Content-Type`/filename in downloads, custom `X-*` headers echoing a parameter, and language/region headers.
- **URL parameters in 30x redirects**: `?url=`, `?next=`, `?return=` that land in `Location:` are the classic CRLF sink (and overlap with open redirect, see `Open-Redirect-and-URL-Parsing.md`).
- **Request-side injection**: CRLF in a value the server forwards into a backend request header (SSRF-adjacent), or into a log line (log injection/forging).
- **Encoding tells**: the app blocks literal `\r\n` but not `%0d%0a`, `%0D%0A`, `%E5%98%8A%E5%98%8D` (unicode-encoded CR/LF that some parsers normalize), or a lone `%0a`.

## Real material

Header injection and response splitting via a redirect parameter:

```
# inject a Set-Cookie:
/redirect?url=https://x%0d%0aSet-Cookie:%20session=attacker
# split the response and inject a body (XSS) on older/permissive stacks:
/redirect?url=%0d%0aContent-Length:%200%0d%0a%0d%0aHTTP/1.1%20200%20OK%0d%0aContent-Type:%20text/html%0d%0a%0d%0a<script>alert(document.domain)</script>
# encoding variants when literal CRLF is filtered:
%0d%0a   %0D%0A   %0a   %E5%98%8A%E5%98%8D   \r\n   嘊嘍
```

Cookie injection (set a cookie you control, enabling fixation or overwriting, see `Session-and-Cookie-Security.md`):

```
/set?lang=en%0d%0aSet-Cookie:%20session=FIXED;%20Domain=.target.com
```

## Beyond the basics

- **Cache poisoning via split responses**: if a cache sits in front and you can split a response, the injected response may be stored and served to other users (overlaps with `Web-Cache-Poisoning-and-Deception.md`). This upgrades a reflected CRLF to an everyone-affecting attack.
- **XSS where normal XSS is filtered**: response splitting injects a full body with your own `Content-Type: text/html`, so you control the document even if the app's HTML contexts are well-escaped. A header-level XSS.
- **Session fixation and cookie overwriting**: injecting `Set-Cookie` lets you fix a session, overwrite CSRF double-submit cookies, or drop a `Domain=.target.com` cookie from a lower-trust context.
- **Request splitting / smuggling-adjacent**: CRLF into a header the server uses to build an upstream request can split that request (related to `HTTP-Request-Smuggling.md`), reaching internal endpoints or desyncing the backend. HTTP/2 to HTTP/1 downgrades can reintroduce CRLF effects that HTTP/2 framing normally prevents.
- **Log injection and forging**: CRLF into a logged value forges log entries (hiding activity, injecting fake entries), and if logs are rendered in a web dashboard, your injected line can carry XSS into the admin view (second-order).
- **Open-redirect combo**: the same redirect parameter is often both an open redirect and a CRLF sink; test both, and chain the redirect for token theft plus the CRLF for cookie/cache effects.

## What to watch for

- Modern servers and frameworks strip CRLF from header values by default; confirm the injection actually reaches the response (observe your header/line in the raw response) before claiming it.
- Full response splitting is rarer now; header injection (one `Set-Cookie` or one extra header) is more common and still impactful. Scope the finding to what actually lands.
- Encoding matters: the sink may decode `%0d%0a` at one layer but not another; test the variant set.
- Prove with benign injected headers/cookies; do not poison shared caches on live targets beyond minimal proof.

## Hunting checklist

- Find every reflection into a response header (redirects, cookies, custom headers, download filenames).
- Inject CRLF (and encoded variants) to add a header (`Set-Cookie`, a marker); confirm it appears in the raw response.
- Attempt full response splitting for body/XSS control where the stack is permissive.
- Check for a front cache to escalate to poisoning.
- Test request-side CRLF into upstream headers and into logs (forging, second-order XSS).
- Combine with open redirect on the same parameter.
