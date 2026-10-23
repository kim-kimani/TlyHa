# NoSQL Injection

NoSQL did not remove injection; it moved it. Instead of breaking out of a quoted string, you abuse the fact that query languages like MongoDB's take structured objects, and a JSON body lets you smuggle operators where the developer expected a scalar.

## The mental model

In document stores, a query is often an object: `{username: input}`. If the input can itself be an object, you replace the expected scalar with an operator object (`{$ne: null}`, `{$gt: ""}`, `{$regex: ...}`) and change the query's logic. The injection vector is the content type: form-encoded bodies coerce to strings, but JSON bodies preserve types, so flipping to JSON is often step one.

## Where it hides / tell-tales

- **Login and lookup endpoints taking JSON**: `{"username":"admin","password":{"$ne":null}}` authenticates as admin if the backend does `db.users.find({username, password})`.
- **Form field that becomes an object via bracket notation**: `user[$ne]=null` in a form body, which Express/`qs` and PHP parse into a nested object `{user: {$ne: null}}`.
- **Search and filter endpoints**: anywhere a user filter is passed through to the query builder, `$regex`, `$where`, `$gt/$lt` can leak or enumerate.
- **`$where` and `mapReduce`**: these evaluate JavaScript server-side in MongoDB. If user input reaches `$where`, you have JS injection, boolean/time oracles, and sometimes worse.
- **GraphQL/REST over Mongo**: the resolver passes arguments straight into a Mongo query. Operators injected through variables.

## Real material

Authentication bypass by operator injection (JSON body):

```json
{"username": "admin", "password": {"$ne": null}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": "admin", "password": {"$regex": "^a"}}
```
The `$ne` form matches any non-null password; the `$gt` form matches any user and any password, logging you in as the first document.

Form-encoded equivalent (parsed into nested objects by many frameworks):

```
username=admin&password[$ne]=x
username[$ne]=&password[$ne]=
```

Blind data extraction via `$regex`, one character at a time (the response differs when the regex matches):

```json
{"username": "admin", "password": {"$regex": "^s"}}
{"username": "admin", "password": {"$regex": "^se"}}
{"username": "admin", "password": {"$regex": "^sec"}}
```
Walk the alphabet per position; a `200`/login success vs failure is your oracle. Anchors (`^`, `$`) and `.{n}` length probes speed it up.

JavaScript injection through `$where` (boolean and timing oracles, even RCE-ish logic):

```json
{"$where": "this.password.length > 8"}
{"$where": "sleep(5000) || true"}
{"$where": "this.username == 'admin' && this.password[0] == 's'"}
```

## Beyond the basics

- **Content-type pivot is the primary skill**: if an endpoint rejects operators in a form body, resend as `application/json` (or vice versa) and watch types survive. Many "not vulnerable" verdicts are just the wrong content type.
- **Operator fuzzing**: beyond `$ne`/`$gt`/`$regex`, try `$in`, `$nin`, `$exists`, `$or`, `$and`, `$not`, `$expr`. `$expr` enables field-to-field comparisons and aggregation expressions that widen what you can leak.
- **Blind extraction automation**: treat `$regex` matching as a boolean oracle and script a per-character search; add `$where` timing when no boolean signal exists.
- **Beyond MongoDB**: the same "operator smuggling" mindset applies to other stores. Elasticsearch/OpenSearch query DSL injection (inject into a `query_string` or raw query object to pull other indices, use `script` fields for code), CouchDB views, Redis command injection via unescaped input, and Cassandra CQL (which is close to SQL, so SQL-style techniques apply). Identify the store first.
- **Server-side JS in `$where` as a foothold**: boolean/timing oracle for extraction; combined with other weaknesses it can leak environment details. Treat any user input reaching `$where`/`mapReduce` as high severity.
- **Aggregation pipeline injection**: if user input reaches `$match`, `$lookup` (which can join other collections, think cross-collection data theft), or `$merge`/`$out` (which write), the impact jumps from read to cross-collection read and even write.

## What to watch for

- A login that "succeeds" with `$ne` might log you into an arbitrary account (often the first/admin); confirm which identity you got before claiming admin.
- Framework body parsers differ: some sanitize `$`-prefixed keys (Mongoose `sanitizeFilter`, `express-mongo-sanitize`); confirm whether the operator actually reached the query.
- `$regex` extraction can be slow and noisy; use length probes and binary-search-style anchoring, and avoid hammering live auth endpoints.
- Do not use `$out`/`$merge`/`$where` destructive or heavy operations against production data.

## Hunting checklist

- Flip body content type to JSON; inject operator objects into every field.
- Try bracket-notation operator injection in form bodies (`field[$ne]=`).
- Test auth endpoints for `$ne`/`$gt`/`$regex` bypass.
- Use `$regex` as a blind extraction oracle; script per-character.
- Look for `$where`/`mapReduce` sinks (JS injection) and aggregation sinks (`$lookup`, `$out`).
- Identify the actual datastore; switch to Elasticsearch/Redis/CQL techniques if not Mongo.
