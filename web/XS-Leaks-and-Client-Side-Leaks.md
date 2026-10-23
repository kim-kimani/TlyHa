# XS-Leaks and Client-Side Leaks

Cross-site leaks extract small bits of information about a logged-in victim from another site by measuring side effects the browser exposes across origins, even though the same-origin policy forbids reading the actual responses. They are subtle, often low-to-medium individually, but they leak things like "is this user an admin," "does this account exist," or "does the victim's inbox contain X," which can be serious.

## The mental model

You cannot read a cross-origin response, but you can observe its side effects: how long it took, whether it errored, whether it changed the number of frames, whether it was cacheable, whether it triggered a resource load. Each observable is an oracle that answers a yes/no question about the victim's authenticated state on the target. Chain enough yes/no answers and you extract real information. These are browser-behavior bugs plus an application that returns observably-different responses based on private state.

## Where it hides / tell-tales

- **Responses that differ by private state in an observable way**: a search endpoint that returns 2 results vs 0 (different size/timing), an admin page that redirects vs renders, an endpoint that errors for non-owners, a page whose number of iframes depends on account type.
- **Status-code and redirect differences cross-origin**: you can often detect whether a cross-origin request returned an error vs success, or redirected, via events (`onload`/`onerror`), even without reading the body.
- **Timing differences**: a request that does more work (results found, permission check passed) takes measurably longer.
- **Cache probing**: whether a resource is already in the victim's cache reveals whether they visited/own something.
- **Framing-count and window-property leaks**: `window.frames.length`, `window.length`, and named-window access change based on the target page's content.
- **postMessage leakage**: a target that `postMessage`s sensitive data to `*` (any origin) leaks it to an attacker frame (see `Cross-Site-Scripting.md` for the sink side).

## Real material

Existence/permission oracle via `onload`/`onerror` (a resource that loads for owners, errors otherwise):

```html
<script src="https://app.target.com/api/secret-report?id=123"
        onload="know('exists/authorized')"
        onerror="know('missing/forbidden')"></script>
<!-- or with an <img>/<iframe>/<link> depending on the content type -->
```

Frame-count oracle (page renders a different number of frames by account state):

```js
const w = window.open('https://app.target.com/dashboard');
setTimeout(() => { report(w.frames.length); }, 1500);   // length reveals a state bit
```

Timing oracle (search returns slower when it matches the victim's private data):

```js
const t0 = performance.now();
await fetch('https://app.target.com/search?q=secret', {mode:'no-cors', credentials:'include'});
const dt = performance.now() - t0;   // longer => results found for this victim
```

Cache-probe (is a resource already cached for the victim, implying prior visit/ownership):

```js
const t0 = performance.now();
await fetch('https://app.target.com/private-asset.png', {mode:'no-cors', credentials:'include', cache:'force-cache'});
// very fast => served from cache => victim had accessed it
```

## Beyond the basics

- **Think in oracles and bits**: each technique yields one yes/no about the victim. Design the target request so the answer encodes the secret you want (does account X exist, is the victim an admin, does their inbox contain a keyword), then automate across many queries to extract more.
- **The modern XS-Leak toolbox**: error events, redirect detection, frame-counting, `performance.getEntries()` resource timing, cache probing, COOP/COEP-based cross-origin-opener detection, and browser-specific oracles. The field is well catalogued (the XS-Leaks wiki); match the oracle to what the target distinguishes.
- **Search-based and inbox-based leaks**: email/search features that reflect private content in response size/timing let an attacker binary-search for whether the victim's data contains a value (a classic "is there an email from X in your inbox" leak), which is a real privacy breach.
- **Defenses tell you what is exploitable**: the mitigations are `SameSite` cookies (which cut off credentialed cross-site requests), `Cross-Origin-Resource-Policy` / `Cross-Origin-Opener-Policy` / `Cross-Origin-Embedder-Policy`, `X-Frame-Options`/`frame-ancestors`, `Vary`, `Cache-Control`, and fetch-metadata (`Sec-Fetch-*`) checks. Where these are missing, the corresponding oracle works; their absence is the tell.
- **DOM clobbering (a related client-side gadget)**: injecting named HTML elements (`<a id=x>`, `<form name=y>`) to overwrite JS variables the page reads (`window.x`, `document.y`), turning a markup-injection with no script into a logic/XSS primitive when code trusts a clobberable global. Hunt for `innerHTML` sinks that later reference globals by name.
- **Connection-pool and hardware oracles**: advanced timing side channels (socket pool exhaustion, GPU/`performance` timers) can leak cross-site state even with good headers; mostly research-grade but worth knowing the ceiling.
- **Fetch-metadata and SameSite have shrunk this**: default `SameSite=Lax` kills many credentialed XS-Leaks; the surviving ones use top-level navigations, `SameSite=None` cookies, or no-credential oracles. Confirm the cookie/header posture before assuming an oracle fires.

## What to watch for

- Individually these are often low severity; lead with a concrete private bit you can extract (admin status, account existence, inbox content) to make the case.
- Oracles can be noisy (timing especially); repeat and use statistical confidence, and confirm the response genuinely differs by private state.
- `SameSite=Lax` defaults and COOP/CORP increasingly neutralize these; verify the target actually sends credentials cross-site before building the attack.
- Extract only your own test account's state as proof; do not harvest real victims' private data.

## Hunting checklist

- Find endpoints whose response differs by private state (size, timing, status, redirect, frame count).
- Build the matching oracle (error events, frame-count, resource timing, cache probe).
- Check cookie `SameSite` and the COOP/CORP/CORB/fetch-metadata headers; missing ones enable specific oracles.
- Look for `postMessage` to `*` and DOM-clobbering-prone globals.
- Chain oracles to extract a real private bit; automate for multi-bit extraction on your own account.
