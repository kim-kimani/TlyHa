# HTTP Request Smuggling

When a front-end proxy and a back-end server disagree about where one HTTP request ends and the next begins, you can smuggle a partial request that the back-end prepends to the next visitor's request. It is one of the highest-impact web bugs: mass credential theft, cache poisoning, and bypass of front-end controls, all without touching the victim directly.

## The mental model

HTTP uses two ways to delimit a message body: `Content-Length` (byte count) and `Transfer-Encoding: chunked` (terminator sequence). If the front-end uses one to decide the body length and the back-end uses the other (or they parse the same header differently), bytes you intend as a second, hidden request get left in the back-end's buffer and glued to whoever's request comes next on that reused connection. You are poisoning a shared connection.

## Where it hides / tell-tales

- **A proxy chain exists**: CDN to origin, load balancer to app server, API gateway to microservice. Smuggling needs at least two HTTP processors that can disagree. Fingerprint the chain from headers (`Via`, `X-Cache`, `Server`, timing).
- **Both `Content-Length` and `Transfer-Encoding` reach the back-end**: the ambiguity lives in how each hop handles them, especially malformed `Transfer-Encoding` the front-end ignores but the back-end honors (or vice versa).
- **HTTP/2 to HTTP/1 downgrade**: a front-end speaking HTTP/2 that rewrites to HTTP/1 for the back-end can introduce length ambiguities (H2.CL, H2.TE) from the implicit H2 message length. This is the fertile modern variant since many edges are H2-to-H1.
- **Timing differentials**: a smuggling probe that makes the back-end wait for more body bytes causes a measurable delay; a reliable detection signal when responses are not obviously poisoned.

## Real material

Classic CL.TE (front-end uses `Content-Length`, back-end uses `Transfer-Encoding`). The front-end forwards the whole thing as one request; the back-end sees the chunked body end at `0` and treats `SMUGGLED` as the start of the next request:

```http
POST / HTTP/1.1
Host: target.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED
```

TE.CL (front-end honors chunked, back-end honors `Content-Length`):

```http
POST / HTTP/1.1
Host: target.com
Content-Length: 3
Transfer-Encoding: chunked

8
SMUGGLED
0


```

Obfuscated `Transfer-Encoding` to make exactly one hop ignore it:

```
Transfer-Encoding: chunked
Transfer-Encoding : chunked        (space before colon)
Transfer-Encoding: xchunked
Transfer-Encoding:[tab]chunked
Transfer-Encoding: chunked\r\nTransfer-encoding: x
X: X[\n]Transfer-Encoding: chunked  (header folding)
```

The impactful payload: smuggle a prefix that captures the next user's request (steal their cookies/credentials) by making their request body append to a request that stores or reflects it:

```http
POST /comment HTTP/1.1
Host: target.com
Content-Length: 300
Transfer-Encoding: chunked

0

POST /comment HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 400

comment=   <-- the victim's real request (with their Cookie header) gets appended here
```
The victim's headers, including their session cookie, land in your comment, which you then read.

## Beyond the basics

- **HTTP/2 desync (the current frontier)**: H2.CL and H2.TE abuse the rewrite from HTTP/2 (which carries length implicitly) to HTTP/1.1. Inject `content-length` or `transfer-encoding` into H2 pseudo-headers/headers so the downgraded H1 request has a length the back-end computes differently. Also CRLF injection into H2 header values (which H1 cannot represent) splits requests. Tools: Burp's HTTP Request Smuggler and the `h2` engines.
- **Client-side desync and browser-powered smuggling**: some desyncs can be triggered from a victim's browser (no shared connection needed), turning smuggling into a one-click attack against a specific user. This widens impact beyond "whoever is next on the connection."
- **What smuggling buys you**: (1) capture other users' requests and steal session tokens en masse; (2) bypass front-end security controls (WAF, auth at the edge) by smuggling a request the back-end trusts; (3) poison the web cache by smuggling a response-splitting prefix (see `Web-Cache-Poisoning-and-Deception.md`); (4) turn an internal-only endpoint reachable only from the back-end into something you can hit.
- **Detection methodology**: use timing-based probes first (a request that should hang the back-end if the desync works), then confirm with a differential response, then weaponize carefully. Blind, time-based detection avoids corrupting real users' traffic during discovery.
- **Connection reuse is required**: the attack needs the front-end to reuse the back-end connection across clients. If connections are not pooled/reused, the classic "next victim" model does not apply, though self/client-side variants may.

## What to watch for

- This is a high-collateral bug class: a weaponized smuggle can corrupt real users' sessions and the shared cache. Test with timing-based, non-destructive probes; do not run request-capture payloads against live users without explicit authorization and care.
- Many stacks now reject conflicting `CL`/`TE` outright; fingerprint before assuming the classic variants work, and focus on H2 downgrade on modern edges.
- A single delayed response can be noise; confirm the timing differential repeats and matches the expected mechanism.
- Poisoning the cache or capturing requests affects everyone; keep proof minimal and scoped.

## Hunting checklist

- Fingerprint the proxy chain and whether HTTP/2 is downgraded to HTTP/1 at the edge.
- Run timing-based CL.TE / TE.CL / H2.CL / H2.TE detection probes.
- Try the obfuscated `Transfer-Encoding` variants to find the one-hop disagreement.
- Confirm connection reuse before expecting next-victim effects.
- Escalate confirmed desync to control-bypass or cache poisoning with minimal, scoped proof.
- Prefer Burp's HTTP Request Smuggler for safe, systematic probing.
