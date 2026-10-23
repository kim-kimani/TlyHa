# Mobile API and Transport Security

The mobile app is mostly a skin over an API, and that API is where account takeover and mass data theft usually live. Mobile backends are often wider, older, and less tested than the web equivalents, and they lean on client-side assumptions (pinning, a secret in the app, device checks) that fall the moment you proxy the traffic.

## The mental model

To test a mobile backend you must first see its traffic, which means defeating certificate pinning and routing the app through an intercepting proxy. Once you can read and modify requests, the mobile API is just an API, and the entire web methodology applies (BOLA, BFLA, mass assignment, auth, injection). The extra angles are mobile-specific trust: the app assumes only the real client talks to the API, so it ships "secret" keys, signs requests with recoverable algorithms, and trusts device-provided values.

## Where it hides / tell-tales

- **Pinning as the only defense**: once pinning is bypassed, the API often has weaker authorization than the web app because the team assumed the client was trusted. The mobile API is frequently the softest entry to the same data.
- **Wider/older endpoints**: the mobile app hits endpoints the web UI never exposes, including legacy versions, debug routes, and richer objects. Proxy the app and inventory everything (see `../web/REST-API-Security.md`).
- **Client-provided trust values**: device id, `is_jailbroken`/`is_rooted`, app version, GPS, phone number, and entitlement flags sent by the client and trusted by the server. Flip them.
- **Request signing / HMAC with a shippable key**: the app signs requests with a key embedded in the binary; recover the key (static or Frida) and you forge valid signatures, defeating the "only our app can call this" control.
- **Hardcoded API keys and backend secrets**: third-party keys (maps, push, analytics, payment) and sometimes privileged backend keys in the app (see `../web/Secrets-and-Information-Disclosure.md`).
- **Weak or absent transport security**: cleartext endpoints, mixed HTTP/HTTPS, and pinning that only covers the main host while analytics/support subdomains leak tokens.

## Real material

Getting to the traffic:

```
# 1. install your CA on the device (user or system store; Android 7+ needs system store or network_security_config bypass)
# 2. route through Burp/mitmproxy
# 3. bypass pinning:
objection -g com.target.app explore
  android sslpinning disable          # or: ios sslpinning disable
# or a Frida pinning-bypass script; or patch the network_security_config / TrustManager
```

Once intercepted, the request looks like any API call, and you test it like one:

```http
GET /mobile/v1/users/me/wallet HTTP/1.1
Authorization: Bearer <token from the app>
X-Device-Id: abc123
X-App-Version: 4.2.0
X-Is-Rooted: false
```
Attack ideas: change `users/me` to another id (BOLA), set `X-Is-Rooted` however you like (it is client-supplied), downgrade `X-App-Version` to reach old behavior, strip optional security headers to see what is actually enforced.

Recovering and reusing a request-signing key (conceptually):

```
# app computes:  X-Signature = HMAC-SHA256(key, method + path + body + timestamp)
# find `key` via jadx/strings or hook the HMAC call in Frida to dump inputs+key.
# now you can sign arbitrary requests -> the server's "only our app" check is void.
```

## Beyond the basics

- **Pinning bypass is step zero, not the finding**: the report is what you find behind it. Budget most of your time on the API's authorization and logic once you can see the traffic.
- **Hook the crypto, do not reverse it**: instead of statically reversing a custom signing/encryption scheme, use Frida to hook the sign/encrypt function and dump plaintext, keys, and outputs at runtime. This collapses hours of reversing into minutes and reveals exactly how to forge requests.
- **Certificate pinning depth**: check whether pinning covers all hosts or just the primary; secondary hosts (analytics, crash reporting, support chat) often carry the same auth token over unpinned or even cleartext channels, leaking it.
- **Device-trust bypass**: anything the client measures about the device (root/jailbreak status, attestation, emulator checks) and sends to the server can be forged unless backed by hardware attestation (Play Integrity/App Attest) that the server actually verifies. Many servers accept a self-reported "not rooted"; test it.
- **Push, deep-link, and OTP flows**: SMS OTP and push-approval flows that the API exposes can be attacked (OTP brute force, see `../web/MFA-Bypass.md`; deep-link token delivery, see `Deep-Links-and-IPC.md`). The mobile API is where these live.
- **Replay and timestamp windows**: request signing with a wide or unchecked timestamp window allows replay; a nonce that is not enforced allows duplication (ties to `../web/Business-Logic-and-Race-Conditions.md`).
- **GraphQL/gRPC backends**: mobile apps increasingly use GraphQL (see `../web/GraphQL.md`) or gRPC/protobuf. For protobuf, use the app's `.proto` (often in the binary) or infer the schema to craft messages; the authorization gaps are the same.

## What to watch for

- A pinned app that resists objection may use native pinning or custom TLS; escalate to a Frida script or patch the binary, but confirm you are actually seeing traffic before testing.
- Forging the signing key proves the client control is bypassable, but the impact still comes from what unsigned/forged requests let you do; demonstrate that.
- Client-supplied device flags may be cross-checked server-side in some apps; confirm the server actually trusts your forged value.
- Use your own accounts and device; do not access other users' data beyond minimal proof.

## Hunting checklist

- Install your CA, route through a proxy, and bypass pinning (objection/Frida/config patch).
- Inventory every endpoint the app calls, including legacy/debug/richer ones.
- Run full API methodology: BOLA, BFLA, mass assignment, auth, injection.
- Flip client-supplied trust values (device id, root status, app version, entitlements).
- Recover or hook request-signing/encryption to forge valid requests; test replay windows.
- Check pinning coverage across all hosts; hunt token leakage on secondary/cleartext channels.
