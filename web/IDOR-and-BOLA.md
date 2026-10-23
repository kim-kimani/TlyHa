# IDOR and BOLA (Beyond Sequential IDs)

Incrementing `id=1` to `id=2` is the kindergarten version. Real object-level authorization bugs (BOLA, the API-world name for IDOR) hide behind UUIDs, hashes, nested objects, and secondary identifiers. This file is about finding them when the obvious case is locked down.

## The mental model

The server receives a reference to an object and an identity. The bug is that it resolves the object by the reference without checking that the identity owns or may access it. The reference can be anything the server uses as a key: a numeric id, a UUID, an email, a username, a filename, a slug, a composite key, a row in a batch, a field deep in a JSON body, a value in a JWT it failed to re-verify.

The question is never "is the id guessable." It is "is ownership checked at all, and checked on this exact code path." A UUID you cannot guess is still an IDOR if the UUID leaks somewhere else (a different endpoint, an email, a referrer, a log, an autocomplete).

## Where it hides / tell-tales

- **UUIDs that leak**: the object uses a UUID, so the team assumed it was safe and skipped the ownership check. Then the UUID shows up in a list endpoint, a notification, a shared link, an export, or a `GET /users/me/contacts` response. Harvest identifiers from every endpoint that returns them, then feed them to the endpoint that does not check.
- **Secondary and indirect references**: `GET /api/invoice?id=UUID` is checked, but `GET /api/invoice/UUID/pdf` or `GET /api/invoice/UUID/download` or the `export`, `print`, `email`, `preview` variants were added later and missed the check.
- **Nested object references in bodies**: `PATCH /api/cart` with `{"items":[{"product_id":X,"price":Y}]}` where `price` is honored, or `{"user_id":"victim"}` buried in an update body. People test the URL and the obvious top-level field; they miss the nested one.
- **HTTP method confusion**: `GET /api/doc/123` is authorized, but `PUT`, `DELETE`, `PATCH`, or `POST` to the same path uses a different handler with no check. Always test every method.
- **Mass operations and batch endpoints**: `POST /api/documents/batch` with `{"ids":[mine, victims]}` often checks the first id or none. GraphQL batch queries and REST bulk endpoints are classic.
- **Blind / write-only IDOR**: no data comes back, but the write lands. `POST /api/user/123/avatar`, `DELETE /api/comment/456`, `POST /api/org/9/invite`. You confirm with a second account, not with the response.
- **The "me" to "id" swap**: `GET /api/users/me/settings` works, now try `GET /api/users/{otherId}/settings`. The `me` alias often routes to a lenient handler.
- **Identifier in a cookie, header, or JWT the server re-reads**: `X-Account-Id: 123`, `X-User-Context`, a tenant header. If the server trusts it over the session, you switch accounts by editing a header.

## Real material

A clean BOLA via a nested field in a JSON body, the kind people miss:

```http
PATCH /api/v2/subscriptions/self HTTP/1.1
Host: app.target.com
Authorization: Bearer <your-token>
Content-Type: application/json

{
  "plan": "pro",
  "billing_account_id": "acct_9d2f1a-VICTIM",
  "seats": 50
}
```

If the server charges `billing_account_id` or grants seats against it without checking you own that account, that is a cross-tenant financial IDOR.

A GraphQL node-id BOLA (Relay-style global ids are base64 of `Type:id`, trivially forgeable):

```graphql
query {
  node(id: "VXNlcjoxMDA3") {   # base64("User:1007")
    ... on User { email phone ssnLast4 }
  }
}
```

Decode, increment the inner id, re-encode, and you are iterating users even though the id "looked" opaque.

A wrapped-identifier case. The app sends base64 or hex, so people assume it is signed. It is not:

```
GET /api/report?ref=eyJ1c2VyIjoiMTIzIiwidHlwZSI6Imludm9pY2UifQ==
# decodes to {"user":"123","type":"invoice"}  -> change user to a victim, re-encode
```

## Beyond the basics

- **Harvest then replay**: treat BOLA hunting as two phases. Phase one collects every identifier the app will give your low-privilege account (list endpoints, search, autocomplete, exports, notification payloads, `Location` headers on create). Phase two replays those identifiers against every read and write endpoint, with every HTTP method. Most "unguessable" systems fall here.
- **Cross-tenant over cross-user**: in B2B SaaS the high-value variant is reaching another organization's data, not another user in your org. Find the tenant selector (`org_id`, subdomain, a claim) and try to point it at a tenant you created separately.
- **Predictable "random"**: UUIDv1 encodes a timestamp and MAC; sequential-ish. Short tokens, `base36(autoincrement)`, timestamp-derived ids, and MongoDB ObjectIds (which embed a 4-byte timestamp, a machine id, and an incrementing counter) are far more predictable than they look. If you can create two objects and see their ids, you can often interpolate others.
- **IDOR into token material**: the win condition is when the object you can read is itself a credential. Password reset tokens, email-change confirmation tokens, invite tokens, API keys, session ids returned in a profile object, signed URLs. Reading one of those is account takeover, not a data leak.
- **Second-order**: you set `email=victim@x.com` somewhere it is not checked, and a later flow (reset, notification, merge) acts on it. The bug fires one step removed from the injection.

## What to watch for

- A `403` is not always a dead end; try the same object via a different endpoint, method, or representation (JSON vs form vs multipart), and via the `me`/alias route.
- Rate limits and account lockout can make harvesting noisy; pull identifiers from endpoints that return many at once rather than brute forcing one by one.
- Confirm write-only IDOR with a clean second account so you do not fool yourself with your own cached state.
- Do not corrupt shared data. Read where possible; for writes, use benign canary values and your own victim account.

## Hunting checklist

- Map every object type and every identifier format the app uses; note where each id is emitted.
- For each read endpoint, try every other identifier you harvested; for each, try every HTTP method.
- Test nested and secondary references in bodies, not just the URL.
- Test batch and bulk endpoints with mixed (yours plus victim) id sets.
- Decode every "opaque" token (base64, hex, GraphQL node id, ObjectId) before assuming it is unguessable.
- Prioritize any object that is itself a credential or that selects a tenant.
