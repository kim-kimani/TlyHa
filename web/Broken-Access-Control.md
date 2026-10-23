# Broken Access Control

IDOR is object-level. This file is everything else: function-level access, vertical privilege escalation, forced browsing, and the subtle middleware and routing flaws that let a normal user act as an admin or a service.

## The mental model

Authorization is two separate questions that teams conflate: "can this identity use this function" (function-level) and "can this identity touch this object" (object-level, covered in the BOLA file). Function-level control is usually enforced at the routing or middleware layer, which means it fails in interesting structural ways: a route that was never wired into the middleware, a method the matcher did not cover, a path normalization gap between the proxy and the app, a role that is read from a client-controlled place.

## Where it hides / tell-tales

- **The admin UI is a separate SPA hitting the same API**: the API enforces nothing and relies on the admin frontend not being served to you. Find the admin API base in the JS bundle and call it directly with your normal token.
- **Client-side role checks**: `if (user.role === 'admin') renderButton()`. The function exists server-side; the button is the only gate. Call the endpoint.
- **Path-based middleware gaps**: a framework guards `/admin/*` but the sensitive handler also answers at `/Admin/`, `/admin/./`, `/admin%2f`, `/api/admin` (not `/admin`), or a trailing-slash/no-slash variant the matcher treats differently from the proxy.
- **Verb tunneling**: the app supports `X-HTTP-Method-Override: PUT` or `_method=DELETE`. Middleware authorizes based on the real method (`POST`) while the framework dispatches on the overridden one.
- **New feature, old guard**: a feature shipped behind a flag or a new endpoint that the central authorization filter does not know about yet. Fuzz for sibling actions near a known-protected one.
- **Role carried in a mutable place**: role or entitlement read from a JWT claim the server does not verify, a cookie, a hidden field, a request header, or a value you set during signup and can re-set via profile update (mass assignment).
- **Referer / origin as authorization**: some apps gate admin actions on `Referer: /admin`. Forgeable.
- **Step-up that is not enforced server-side**: the UI asks for a password or MFA before a sensitive action, but the final request works without the step-up proof.

## Real material

Mass assignment to escalate role, the quiet classic. Registration or profile update accepts fields the UI never shows:

```http
POST /api/users HTTP/1.1
Content-Type: application/json

{
  "email": "me@test.com",
  "password": "x",
  "role": "admin",
  "email_verified": true,
  "is_staff": true,
  "org_id": "victim-org"
}
```

Rails `permit`, Django serializers with `fields = '__all__'`, Spring `@ModelAttribute` binding, and Express blindly spreading `req.body` into a model are all prone. Watch the response: if it echoes `"role":"admin"`, you won.

Method-override escalation:

```http
POST /api/admin/users/42 HTTP/1.1
X-HTTP-Method-Override: DELETE
Content-Length: 0
```

Path confusion to reach a guarded handler:

```
GET /admin/../admin/dashboard   (proxy normalizes differently than app)
GET /%2e%2f/admin/metrics
GET /..;/admin/   (Java/Tomcat path-parameter trick, the ;segment is stripped late)
```

## Beyond the basics

- **Enumerate the function matrix**: list every action the highest-privilege role can take (from the admin JS bundle, API docs, or `/actuator/mappings`), then replay each one with a low-privilege token. The gaps are the findings.
- **The `..;/` and matrix-parameter tricks**: Tomcat, Spring, and some Java stacks treat `;` segments as path parameters and strip them after the security filter ran on the raw path, or before the servlet matches. `/api/user/..;/admin/panel` can slip past a filter that guards `/admin`. Test the full family of normalization quirks.
- **Proxy versus origin normalization**: nginx may collapse `//` and decode `%2e`, while the backend does not, or vice versa. A path that the WAF/proxy sees as harmless can reach a sensitive route at the origin. This is the same class of discrepancy that powers smuggling and cache attacks; carry it here too.
- **Horizontal to vertical pivot**: a horizontal IDOR that lets you read an admin user's object might leak their API key or a role-granting token, turning a same-level read into vertical escalation.
- **GraphQL field-level gaps**: a query may be authorized as a whole, but an individual field resolver (`user { auditLog }`, `organization { billingInfo }`) was never guarded. Walk the schema field by field, not query by query.
- **Mutation of your own entitlements**: look for any endpoint where you can set `plan`, `role`, `permissions`, `features`, `quota`, `verified`, `trusted`. Then look for the second-order version: you request an upgrade, it is "pending," but the entitlement is granted immediately and only revoked on failed payment.

## What to watch for

- A changed response is not escalation until you prove the privileged effect (data you should not see, an action that lands on a second account). Admin endpoints sometimes return `200` with empty or filtered data even when unauthorized.
- Some path tricks only work against a specific proxy or framework version; fingerprint first so you are testing the right family of quirks.
- Mass-assignment fields may be silently dropped rather than rejected; confirm the field actually took effect, do not trust a `200`.

## Hunting checklist

- Build the full privileged-action list from JS, docs, and mappings; replay each with a low-priv token.
- Test every HTTP method and the override headers (`X-HTTP-Method-Override`, `_method`) on guarded routes.
- Try path-normalization variants: case, `%2e`, `%2f`, `..;/`, trailing slash, double slash, `/api` prefix.
- Attempt mass assignment of `role`, `is_admin`, `verified`, `org_id`, `permissions` on create and update.
- Check GraphQL field by field, not query by query.
- Look for client-side-only role checks and separate admin SPAs sharing the API.
