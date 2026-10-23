# OAuth and OIDC

OAuth is a delegation protocol that everyone implements slightly wrong. The bugs are rarely in the spec; they are in the `redirect_uri` handling, the state parameter, the token exchange, and the gap between OAuth (authorization) and OIDC (authentication) that developers blur.

## The mental model

OAuth 2.0 grants a client access to a resource on a user's behalf. The dangerous object is the authorization code or token, and the dangerous moment is the redirect that carries it back. If an attacker can make the authorization server send that code to a URL they control, or can swap their own code into a victim's session, they take over the account. OIDC adds an `id_token` (a JWT asserting who the user is); when apps use that `id_token` or the OAuth access token to log users in, any flaw in validating it is authentication bypass.

## Where it hides / tell-tales

- **`redirect_uri` validation is the whole game**: exact-match is safe; anything looser is a bug. Prefix match, substring match, allowing subdomains, allowing paths, allowing open-redirect-capable endpoints on an allowed host, honoring the `redirect_uri` on the token endpoint differently than on the authorize endpoint.
- **Missing or unbound `state`**: `state` is CSRF protection for the callback. If it is absent, not tied to the user's session, or not checked, you can perform login CSRF and force-link accounts.
- **`response_type` and `response_mode` confusion**: implicit flow (`token` in the fragment) leaks tokens to the browser and to anything reading the URL; mixing flows enables code and token substitution.
- **Account linking without verification**: "Sign in with Google" that matches on email and auto-links to an existing local account. If the OAuth provider did not verify the email, or the app trusts an unverified email claim, you register an account with the victim's email at a sloppy provider and link into their account.
- **`id_token` accepted without validating `aud`, `iss`, signature, or nonce**: the classic OIDC authentication bypass. The app takes the `id_token`, reads `email`, logs the user in, and never checks the token was issued for this client by the expected issuer.
- **Authorization code leaks**: code in the `Referer` header to third-party scripts on the callback page, code in browser history, code logged, code reused (not single-use) or not bound to the client via PKCE.
- **PKCE absent or downgradeable**: public clients without PKCE, or servers that accept a request with `code_verifier` missing when `code_challenge` was sent.

## Real material

The redirect_uri theft primitive. Attacker crafts an authorize URL with a redirect they control that still passes a weak allowlist:

```
https://auth.target.com/oauth/authorize?
  client_id=webapp&
  response_type=code&
  scope=openid%20email&
  redirect_uri=https://app.target.com/callback/../../redirect?to=https://evil.tld&
  state=xyz
```
If `redirect_uri` is validated by "starts with `https://app.target.com`" and `app.target.com` has an open redirect, the code lands on `evil.tld`.

Common weak-allowlist bypasses to try against `redirect_uri`:

```
https://app.target.com.evil.tld/cb          (suffix trick)
https://evil.tld/app.target.com             (substring match)
https://app.target.com@evil.tld/cb          (userinfo confusion)
https://app.target.com/cb/../../oauth/leak  (path traversal on allowed host)
https://app.target.com//evil.tld            (protocol-relative after host)
https://app-target.com/cb                    (hyphen vs dot)
https://app.target.com%2f@evil.tld
```

An OIDC `id_token` whose claims reveal the bypass target:

```json
{
  "iss": "https://accounts.google.com",
  "aud": "123-webapp.apps.googleusercontent.com",
  "sub": "117...",
  "email": "victim@gmail.com",
  "email_verified": true,
  "nonce": "n-0S6_WzA2Mj"
}
```
If the app reads `email` and logs you in without checking `aud` equals its own client id, `iss` is the expected provider, the signature is valid against that provider's JWKS, and `nonce` matches what it sent, then forging or swapping this token is account takeover. See `JWT-Attacks.md` for the signature side.

## Beyond the basics

- **Code / token substitution (the "cut-and-paste" attack)**: start a login with your own account, intercept the callback, and splice your valid `code` into the victim's active login flow (or vice versa) when `state` is not bound to the session. Result: victim logs into your account (you now see what they type, add a payment method, etc.) or you log into theirs.
- **`redirect_uri` on authorize vs token endpoint**: RFC requires the `redirect_uri` sent to the token endpoint to match the one from authorize. Some servers skip this, enabling a mix-and-match where a code obtained with one redirect is redeemed with another.
- **Mix-up attack**: with multiple identity providers, if the client does not track which IdP a code came from, an attacker can make the client send a code issued by a malicious IdP to the honest IdP's token endpoint, or leak the code. Mitigated by `iss` in the response; test whether the client checks it.
- **Scope upgrade**: request broader `scope` than the UI does and see if the consent is remembered or silently granted; a previously consented client may get new scopes without re-prompting.
- **Dynamic client registration**: if the authorization server allows open registration (`/register`), register a client with a `redirect_uri` you control and a trusted-looking name, then phish or chain.
- **Logout and token lifetime**: refresh tokens that are not rotated or revoked on logout, access tokens that outlive sessions, `id_token` replay. Capture and replay after logout.
- **The confused "login with" linking**: many account-takeover chains are: pre-register the victim's email locally, victim later signs in with Google which auto-links to the attacker-seeded account, or the reverse where unverified email from a minor provider links into a verified local account.

## What to watch for

- An open redirect that only works when logged out may not carry the code; confirm the token or code actually lands on your host.
- `state` present does not mean `state` validated; test by replaying a callback with a mismatched or reused `state`.
- Providers increasingly send `iss` and enforce exact `redirect_uri`; confirm the behavior on this implementation rather than assuming the old bugs.

## Hunting checklist

- Enumerate every `redirect_uri` accepted; test suffix, substring, userinfo (`@`), traversal, protocol-relative, and open-redirect chaining on allowed hosts.
- Remove, reuse, and mismatch `state`; test for login CSRF and code substitution.
- Check `id_token` validation: signature, `aud`, `iss`, `nonce`, `exp`.
- Test account linking with unverified or attacker-controlled emails.
- Look for implicit flow, missing PKCE, non-single-use codes, and code leakage via `Referer`.
- Test refresh and access token behavior after logout and password change.
