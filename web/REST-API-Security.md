# REST API Security

APIs are where the real logic and the real data live, and they are tested less than the UI that fronts them. The OWASP API Top 10 exists because API bugs are their own species: object and function level authorization, mass assignment, excessive data exposure, and versioning debt. This file ties those together with the mindset to find them.

## The mental model

An API exposes operations on resources, often more operations than any UI uses, across multiple versions, consumed by web, mobile, and partners. Each consumer may hit different endpoints; the mobile app often reaches older or richer ones. Authorization must be enforced server-side on every object (BOLA) and every function (BFLA), the response must not over-share, and the input must not over-bind. The gaps are systematic because APIs are generated, versioned, and documented inconsistently.

## Where it hides / tell-tales

- **Excessive data exposure**: the endpoint returns the full object and the UI hides fields client-side. `GET /api/users/me` returns `passwordHash`, `ssn`, `internalNotes`, `role`, other users' data in an embedded list. Read the raw JSON, not the rendered page.
- **BOLA and BFLA**: object-level (see `IDOR-and-BOLA.md`) and function-level (see `Broken-Access-Control.md`) authorization failures are the top two API risks. Every id you can change and every admin action you can call is a candidate.
- **Mass assignment**: the API binds the whole JSON body to the model, so `"role":"admin"`, `"verified":true`, `"balance":9999` take effect even though no field exists in the UI.
- **Old versions still routable**: `/api/v1/` next to `/api/v3/`. v1 predates a control v3 added. Also undocumented endpoints found in the mobile app or JS bundle.
- **Inconsistent auth across endpoints**: most endpoints require a bearer token; one internal/debug/health endpoint does not, or accepts a static key found in the app.
- **No rate limiting on expensive or sensitive ops**: OTP, login, search, export, and anything that costs money or sends messages.
- **Verbose errors and schema leakage**: stack traces, ORM errors, and `/swagger.json` / `/openapi.json` exposing the full surface.

## Real material

Excessive data exposure, the quiet leak. The app shows a name; the API returns everything:

```http
GET /api/v2/users/me HTTP/1.1
Authorization: Bearer <token>

-- response --
{ "id":1024, "name":"Me", "email":"me@x.com",
  "passwordHash":"$2b$...", "role":"user", "mfaSecret":"JBSWY3DPEH...",
  "internalRiskScore":0.2, "referredUsers":[{"id":1025,"email":"other@x.com"}] }
```
The `mfaSecret`, `passwordHash`, and other users' emails should never be here.

Mass assignment on update:

```http
PATCH /api/v2/users/me HTTP/1.1
Content-Type: application/json

{ "displayName":"x", "role":"admin", "emailVerified":true, "accountBalance":100000 }
```

Finding and hitting an older version that skipped a control:

```
POST /api/v3/transfer   -> enforces per-transaction limit and step-up auth
POST /api/v1/transfer   -> same backend account, no limit, no step-up  (still live)
```

Discovering the surface from the schema:

```
GET /swagger.json  /openapi.json  /api-docs  /v2/api-docs  /graphql (introspection)
GET /actuator/mappings   (Spring: lists every route)
```

## Beyond the basics

- **Build the endpoint inventory from every client**: proxy the mobile app and the web app, pull the JS bundle routes, read `swagger`/`openapi`, and diff versions. The mobile and partner APIs routinely expose operations the web UI never shows, and those are the least-tested. This inventory is the foundation for BOLA/BFLA testing.
- **Test the authorization matrix exhaustively**: for every endpoint x every role x every object owner, does the server enforce? Automate replaying each request with (a) no token, (b) a low-priv token, (c) another tenant's token. The gaps are the findings. Tools like Autorize/AuthMatrix systematize this.
- **Mass assignment discovery**: diff the object you get from a `GET` against what a `PATCH`/`POST` accepts; try setting every field you saw plus guessed sensitive ones (`role`, `isAdmin`, `verified`, `owner`, `tenantId`, `permissions`, `credits`). Confirm the field actually took effect, not just a `200`.
- **Versioning and deprecation debt**: enumerate `v1..vN`, beta/`/internal/`/`/admin/` prefixes, and date-based versions (`?version=2019-01-01`). Old versions are a time machine to pre-fix behavior.
- **Content-type and parser tricks**: send XML to a JSON endpoint (XXE), send arrays where scalars are expected (NoSQL operators, type confusion), send extra/duplicate parameters (last-wins vs first-wins differences), and test JSON parser quirks (duplicate keys, deep nesting, big numbers).
- **Secrets in the API surface**: API keys embedded in mobile apps, static keys that are not rotated, keys that are over-scoped (a read key that also writes), and JWTs that over-trust claims (see `JWT-Attacks.md`). Check what a leaked key can actually do.
- **Rate limiting bypass**: rotate `X-Forwarded-For`, change casing/trailing slash on the path, use GraphQL batching, parallelize, or hit a different version/host that shares the backend but not the limiter.

## What to watch for

- Over-exposure in a response is a real finding even without further escalation (PII leak), but confirm the sensitive fields are genuinely present and belong to others or should be secret.
- A `200` on a mass-assignment attempt does not mean the field stuck; verify the privileged effect.
- Old-version endpoints may be deprecated but still live; confirm they actually execute, not just return a deprecation notice.
- Replaying another tenant's requests requires a second account you control; do not use real users' tokens.

## Hunting checklist

- Inventory every endpoint from web, mobile, partner clients, JS bundles, and `swagger`/`openapi`/`actuator`.
- Read raw responses for excessive data exposure (secrets, other users' data, internal fields).
- Run the full authorization matrix (no token / low-priv / cross-tenant) across all endpoints and methods.
- Test mass assignment by setting every observed and guessed sensitive field; verify effect.
- Enumerate versions and internal/admin prefixes; test older versions for missing controls.
- Test rate limiting on sensitive/expensive ops and the common bypasses.
- Check embedded API keys and JWT claim trust.
