# JWT Attacks

A JWT is a trust boundary wearing base64. The entire security model rests on signature verification being done correctly, on the right key, with the right algorithm. Every real JWT bug is one of those three going wrong.

## The mental model

A JWT is `header.payload.signature`, each part base64url. The header declares the algorithm and often a key hint (`kid`, `jku`, `x5u`, `x5c`, `jwk`). The server is supposed to: pick the algorithm it expects (not the one the token declares), fetch the correct key, verify the signature, then and only then trust the claims. Bugs happen when the server trusts the token's own header to choose the algorithm or key, or skips verification, or uses a key it should not.

Decode first, always. Read the claims (`sub`, `role`, `scope`, `aud`, `iss`, `exp`, `iat`, custom tenant and permission claims). The claims tell you what lying would buy you.

## Where it hides / tell-tales

- **`alg` is attacker-chosen**: if you can change `alg` and the token is still accepted, verification trusts the header.
- **`alg: none`**: the "unsecured JWT." Some libraries still accept it, or accept `None`, `NONE`, `nOnE` due to case handling.
- **`kid` is a path or an injection point**: `kid` often indexes a key store. If it is a filename, it is path traversal (`kid: "../../dev/null"` makes the key an empty file, so an empty-signature HMAC validates) or a lookup that can be SQL-injected to return an attacker-known value.
- **`jku` / `x5u` point to a URL the server fetches**: if the server fetches the signing key from a URL in the header and does not pin the host, you host your own key and sign with it. SSRF and allowlist bypass territory.
- **`jwk` embedded in the header**: the token carries its own public key and the server uses it to verify. You sign with your own key, embed the matching public key, done.
- **HMAC vs RSA confusion (`RS256` to `HS256`)**: the server verifies `RS256` with a public key. If you switch the header to `HS256`, a naive library uses the same public key as the HMAC secret. The public key is, by definition, public, so you can forge.
- **Weak HMAC secret**: `HS256` with a guessable or default secret (`secret`, `changeme`, a leaked `.env`). Crack offline with `hashcat` mode 16500.
- **Claims trusted without the matching check**: `aud` not validated (token for service A accepted by service B), `iss` not pinned, `exp` not enforced, or `kid`/tenant claims used for authorization without re-checking ownership.

## Real material

A decoded token before the attack:

```json
// header
{ "alg": "RS256", "typ": "JWT", "kid": "prod-key-2024" }
// payload
{ "sub": "1024", "email": "me@test.com", "role": "user",
  "org": "acme", "iat": 1739000000, "exp": 1739003600 }
```

The `alg: none` forge (note the trailing dot, empty signature):

```
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiIxMDI0Iiwicm9sZSI6ImFkbWluIn0.
```
That is `{"alg":"none","typ":"JWT"}` . `{"sub":"1024","role":"admin"}` . (empty).

The `RS256` to `HS256` confusion: take the server's RSA public key (from the JWKS endpoint, usually `/.well-known/jwks.json`, or `/oauth2/v3/certs`), convert it to PEM, then HMAC-sign a forged payload using that PEM text as the secret with `HS256`. If the server shares one verify function, it validates.

Embedding a self-signed `jwk` in the header:

```json
{
  "alg": "RS256",
  "typ": "JWT",
  "jwk": { "kty": "RSA", "n": "<your-public-modulus>", "e": "AQAB" }
}
```
Sign the payload with your private key; a server that trusts the header's `jwk` verifies against your public key and accepts it.

`kid` path traversal to force an empty key:

```json
{ "alg": "HS256", "kid": "../../../../../../dev/null" }
```
If the server reads the key file at `kid` and that file is empty, HMAC with an empty key lets you compute a valid signature.

`jku` pointing at your server:

```json
{ "alg": "RS256", "kid": "x", "jku": "https://target.com/.well-known/jwks.json" }
```
If the server fetches `jku` and the host check is weak, host your own JWKS. Bypasses often chain an open redirect or SSRF on the target so the URL "looks" in-domain.

## Beyond the basics

- **Find the real verifier**: multiple services may share tokens but verify differently. The edge might verify strictly; an internal or older microservice might accept `alg: none`. If you can reach the weaker verifier (via a path, an old API version, or SSRF), forge for that one.
- **JWKS host allowlist bypass**: when `jku` is restricted to the target domain, chain it with an open redirect, a path on the target that reflects your content, or an SSRF that proxies to your JWKS. The goal is a URL that passes the allowlist but returns your key.
- **`aud`/`iss` cross-acceptance**: collect tokens the target issues for different audiences (web, mobile, partner, internal). A token minted for a low-trust audience sometimes validates on a high-trust one because `aud` is unchecked.
- **Replay and expiry**: if `exp` is not enforced, old captured tokens live forever. If logout does not revoke server-side, the token outlives the "session." Test whether a token keeps working after logout and after a password change.
- **`kid` SQL injection**: `kid: "aaa' UNION SELECT 'attacker-known-secret'-- -"` can make the key lookup return a value you control, then you sign with that value.
- **Nested / encrypted (JWE) tokens**: `alg` plus `enc` in the header means JWE. Attacks shift to the key-management algorithm; watch for `dir`, `RSA1_5` (Bleichenbacher), and weak content encryption.

## What to watch for

- Many modern libraries fixed `alg: none` and `RS256`-to-`HS256` years ago; confirmation matters. A `200` on a forged token might be the server ignoring the token and falling back to another identity source, not accepting your forge. Verify by changing a claim that provably changes behavior (your `sub` to a victim, your `role` to `admin`) and seeing the privileged effect.
- A forged token rejected at the edge may still be worth trying against internal verifiers; do not conclude from one endpoint.
- Cracking `HS256` only works on genuinely weak secrets; do not burn hours on a strong one.

## Hunting checklist

- Decode every token; note `alg`, `kid`, `jku`, `jwk`, `x5u`, and all claims.
- Try `alg: none` and case variants.
- Pull the JWKS; attempt `RS256` to `HS256` confusion with the public key as the HMAC secret.
- Test `kid` for path traversal and SQL injection; test `jku`/`x5u`/`jwk` for attacker-controlled keys.
- Crack `HS256` secrets offline if the algorithm is HMAC.
- Verify `exp`, `aud`, `iss` enforcement, token behavior after logout and password change.
