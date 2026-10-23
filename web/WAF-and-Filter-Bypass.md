# WAF and Filter Bypass

A WAF is a pattern matcher sitting between you and the app, and the app parses input differently than the WAF does. Every bypass exploits that gap: you encode, fragment, or relocate the payload so the WAF sees something benign while the app sees the attack. This file is the cross-cutting evasion mindset that supports every injection topic.

## The mental model

The WAF and the application are two parsers in a row that do not agree. The WAF normalizes and inspects a representation of your request; the app normalizes and interprets a (possibly different) representation. Bypass is about finding a payload that is attack in the app's view and noise in the WAF's view. The levers are: encoding, parser quirks, protocol features the WAF mishandles, moving the payload to a location the WAF inspects less, and reaching the origin directly.

## Where it hides / tell-tales

- **The WAF blocks a payload but not its encoding**: double URL-encoding, unicode, HTML entities, overlong UTF-8, mixed case, and whitespace alternatives often pass.
- **The WAF inspects some request parts and not others**: body but not headers, query but not body, JSON values but not JSON keys, form but not multipart, first parameter but not duplicates. Relocate the payload.
- **Content-type and parsing confusion**: send `application/json` the WAF parses shallowly, or `multipart/form-data` with tricks, or `charset` declarations that change interpretation. Parameter pollution (`a=benign&a=payload`) where WAF and app pick different values.
- **Origin reachable directly**: if you find the origin IP behind the CDN/WAF (historical DNS, SPF, cert SANs, Shodan, misconfigured vhost), you skip the WAF entirely.
- **Rate/size limits and chunking**: very large requests, chunked bodies, or many small fragments that the WAF truncates or skips.

## Real material

Encoding ladders (apply to any injection keyword/metacharacter):

```
union select     -> UnIoN SeLeCt                 (case)
                 -> UNION/**/SELECT               (inline comment)
                 -> %55nion %53elect              (URL-encode)
                 -> %2555nion                      (double URL-encode)
                 -> uni%00on                        (null byte)
                 -> union%0aselect                  (newline whitespace)
<script>         -> <ScRiPt>  <scr%00ipt>  <svg/onload=>  <img/src/onerror=>
../              -> ..%2f  ..%252f  ....//  %2e%2e%2f
```

Relocating the payload (WAF inspects less here):

```
# move SQLi from query to JSON body, header, or cookie
X-Forwarded-For: ' OR 1=1-- -
Cookie: tracking=' OR 1=1-- -
{"filter":{"$where":"..."}}          (nested JSON the WAF skims)
# HTTP parameter pollution
?id=1&id=1'+OR+'1'='1                 (WAF reads first, app reads last, or concatenates)
```

Content-type and multipart tricks:

```
# JSON with unexpected structure the WAF does not fully parse
{"a":"union","b":"select","c":" ... "}   then the app concatenates a+b
# multipart with the payload in the filename or an extra part
Content-Disposition: form-data; name="x"; filename="' OR 1=1-- -"
# charset confusion
Content-Type: application/x-www-form-urlencoded; charset=ibm037   (EBCDIC smuggling)
```

Finding the origin to skip the WAF:

```
# historical DNS (SecurityTrails, DNS history), SPF/MX records, cert SANs (crt.sh),
# Shodan "ssl.cert.subject.CN:target.com", favicon hash, and sending the Host header
# to the discovered IP:
curl -H 'Host: app.target.com' https://ORIGIN_IP/ -k
```

## Beyond the basics

- **Parser differential is the whole discipline**: the same mindset as `HTTP-Request-Smuggling.md`, `Open-Redirect-and-URL-Parsing.md`, and SSRF allowlist bypass. The WAF is just another parser in the chain to disagree with. Think about how it normalizes versus how the backend does.
- **Tailor to the detected WAF**: fingerprint it (response headers, block-page fingerprints, `wafw00f`) and use its known weak spots. Cloudflare, Akamai, AWS WAF, Imperva, F5, ModSecurity CRS each have characteristic gaps and characteristic regexes you can step around. Different vendors miss different encodings.
- **Keyword fragmentation and logical equivalents**: replace blocked constructs with equivalents the app accepts: `OR` to `||`, `=` to `LIKE`/`REGEXP`, `AND` to `&&`, space to `/**/`/`%09`/`%0c`/`+`, string literals to hex/`CHAR()`, comments to break keywords (`SEL/**/ECT`). For XSS, swap tags and handlers (`<svg onload>`, `<details ontoggle>`), use SVG/MathML namespaces, and HTML-entity-encode inside attributes.
- **Mutation and context exploitation**: payloads that are inert as sent but become dangerous after the app's own processing (template rendering, HTML sanitizer mutation, URL decoding, JSON unescaping). The WAF sees the inert form; the app creates the dangerous form. This overlaps with mutation XSS and second-order injection.
- **Request size and timing**: pad the request with junk before the payload so the WAF's inspection buffer is exhausted (some WAFs only inspect the first N kilobytes), or use chunked transfer to split a signature across chunks.
- **Blind and OOB to avoid signature-heavy responses**: when the WAF also inspects responses or rate-limits on error patterns, switch to blind, time-based, and out-of-band techniques that produce clean traffic.
- **Protocol downgrade and alternate channels**: HTTP/2 vs HTTP/1 differences, WebSocket frames (often uninspected, see `WebSockets.md`), and GraphQL batching (see `GraphQL.md`) all reach the app through paths the WAF covers less.

## What to watch for

- A WAF block (403/captcha/challenge) is not proof the underlying bug exists; it means your signature matched. Confirm the vulnerability with a bypass, do not report "WAF blocked me" as a finding.
- Bypassing the WAF does not create impact by itself; you still must demonstrate the actual vulnerability behind it.
- Aggressive bypass attempts can trip rate limits and get your IP blocked on a live program; go slow and change one variable at a time.
- Hitting the origin directly can be out of scope or against program rules; check the program's policy before bypassing the CDN.

## Hunting checklist

- Fingerprint the WAF and look up its known gaps.
- Apply the encoding ladder (case, comments, URL/double-URL, unicode, null, whitespace) to the core payload.
- Relocate the payload (headers, cookies, JSON keys/nested, multipart filename, duplicate params).
- Try content-type and charset confusion and parameter pollution for parser differentials.
- Use logical equivalents and keyword fragmentation specific to the injection type.
- Consider buffer-exhaustion padding, chunking, HTTP/2, WebSocket, and GraphQL-batch channels.
- Where allowed, find and test the origin directly to bypass the WAF entirely.
