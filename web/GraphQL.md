# GraphQL

GraphQL hands the client a flexible query language against the whole data graph, which means the attack surface is the schema itself. The bugs are authorization applied per-endpoint instead of per-field, introspection leaking the map, batching defeating rate limits, and injection flowing through resolvers into the same old sinks.

## The mental model

One endpoint (`/graphql`) answers arbitrary queries and mutations against a typed schema. Traditional per-URL authorization does not fit, because every operation hits the same URL; authorization must live in each resolver. The common failure is that query-level or endpoint-level auth passes while an individual field, a nested relationship, or a mutation resolver was never guarded. Also, the client controls the shape and volume of work, which breaks rate limiting and cost assumptions.

## Where it hides / tell-tales

- **Introspection enabled in production**: the full schema, including hidden types, admin mutations, and deprecated fields, is one query away. Even if the UI never calls them, they are callable.
- **Per-field authorization gaps**: `user { email }` is fine, but `user { auditLog }`, `user { paymentMethods }`, or `organization { members { ssn } }` were never checked. Walk every field.
- **Nested relationship traversal**: from an object you can read, follow edges to objects you should not. `myOrder { customer { otherOrders { ... } } }` leaks across users.
- **Mutations without authorization or with mass-assignment**: `updateUser(input:{role:ADMIN})`, `deleteAnyObject(id:...)`, hidden `adminX` mutations found via introspection.
- **Batching and aliases defeat rate limits**: one HTTP request containing many aliased operations (or a JSON array of queries) runs hundreds of logins/OTP attempts/IDOR reads while the rate limiter counts one request.
- **Injection through resolvers**: arguments flow into SQL/NoSQL/OS/SSRF sinks exactly as in REST; GraphQL is just the transport. See the relevant injection files.
- **Verbose errors**: GraphQL error messages often leak field names, types, and backend details even when introspection is off ("Did you mean ...?" suggestions enumerate the schema).

## Real material

Introspection to dump the schema:

```graphql
query {
  __schema {
    types { name fields { name args { name type { name } } } }
    queryType { name } mutationType { name }
  }
}
```
Short form to list mutations (the sensitive ones):

```graphql
{ __schema { mutationType { fields { name args { name } } } } }
```

Batching to brute force past rate limits (aliases run many ops in one request):

```graphql
mutation {
  a: login(user:"victim", pass:"guess1") { token }
  b: login(user:"victim", pass:"guess2") { token }
  c: login(user:"victim", pass:"guess3") { token }
  # ... hundreds more, all counted as one HTTP request
}
```
JSON-array batching (another form many servers accept):

```json
[ {"query":"{ user(id:1){email} }"}, {"query":"{ user(id:2){email} }"}, ... ]
```

BOLA via a forgeable node id (see `IDOR-and-BOLA.md`):

```graphql
{ node(id: "VXNlcjoyMDAx") { ... on User { email phone } } }  # base64("User:2001")
```

Field-level over-read:

```graphql
{ me { id organization { members { id email role lastLogin } } } }
```

Deep nesting for DoS (cyclic relationships amplify cost):

```graphql
{ user { friends { friends { friends { friends { id } } } } } }
```

## Beyond the basics

- **Reconstruct the schema even with introspection off**: use field-suggestion errors ("Cannot query field 'x', did you mean 'y'?") to brute-force field and type names, and use known tools/wordlists (`clairvoyance`-style) to rebuild the schema from suggestions. Introspection-off is not security.
- **Authorization matrix per field and per edge**: do not test queries, test fields. For every type a low-priv user can reach, request every field and follow every relationship to confirm each resolver checks ownership. This is where the real BOLA/BFLA findings are.
- **Batching abuse is the signature GraphQL bug**: it turns any rate-limited action (login, OTP, password reset, coupon redeem, IDOR enumeration) into a bulk attack in a single request. Always test whether aliasing and array-batching are enabled; if so, every rate-limit defense in the app is suspect.
- **Mutation hunting**: introspection or suggestion-enumeration reveals mutations the UI never exposes (bulk ops, admin actions, internal tooling). Call them directly with your token; function-level authorization is frequently missing on the ones not wired into the UI.
- **Query cost and DoS**: deeply nested cyclic queries, huge `first:`/`limit:` values, and expensive aliased fan-out can exhaust the backend. Report as DoS; do not actually take the service down.
- **CSRF on GraphQL**: if the endpoint accepts `application/x-www-form-urlencoded` or `GET` queries and relies on cookies, it may be CSRF-able despite "JSON only" assumptions. Test content-type downgrades (see `CSRF.md`).
- **Injection lives in arguments**: a `filter`, `search`, `orderBy`, or `where` argument that reaches a database is SQL/NoSQL injection; a URL argument that the resolver fetches is SSRF. GraphQL does not sanitize for you.

## What to watch for

- A field that errors with "unauthorized" for you is still worth probing via a different path/relationship; resolver auth is inconsistent.
- Introspection disabled does not mean the schema is secret; enumerate via suggestions.
- Batching limits and query-depth limits may be in place; confirm before assuming brute force works.
- Cost/DoS queries can crash shared infrastructure; keep them small for proof.

## Hunting checklist

- Run introspection; if disabled, reconstruct the schema from field-suggestion errors.
- List every query and mutation, especially ones the UI never calls.
- Test per-field and per-relationship authorization for every reachable type.
- Test alias batching and array batching against rate-limited actions.
- Decode and tamper node ids for BOLA.
- Probe arguments for SQL/NoSQL/SSRF injection and test CSRF via content-type downgrade.
