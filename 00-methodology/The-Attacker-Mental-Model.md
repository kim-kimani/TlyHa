# The Attacker Mental Model

The difference between a scanner and a hunter is not the payload list. It is the set of questions you ask about a feature before you touch it. This file is the lens you carry into every other topic.

## The core questions

For any request, feature, or endpoint, run this loop:

1. **Who is this request trusting, and why?** Every bug is a trust boundary that was assumed but not enforced. A cookie, a JWT claim, a hidden form field, a header, a referer, a client-supplied role. Find what the server believes and ask whether the client controls it.
2. **What does the server do with each input, and who else sees it?** An input that reaches a template is SSTI, a shell is command injection, a SQL string is injection, a URL fetcher is SSRF, an HTML sink is XSS, a filesystem path is traversal. The payload class is a function of the sink, not the field name.
3. **What state exists, and can I reach it out of order?** Multi-step flows (checkout, signup, password reset, KYC) assume you walk step 1, 2, 3. Replay step 3 alone, skip step 2, run two step-2s in parallel, or come back to a step after the state should have expired.
4. **What happens at the boundary?** Zero, negative, empty, null, very large, duplicated keys, the same object twice, a different content type than expected. Boundaries are where developers stop writing branches.

## The trust-boundary map

Draw this for every target before deep testing:

- **Identity**: how does the server know who you are? Session cookie, bearer token, API key, mTLS, IP. Where is it re-checked, and where is it checked once then trusted?
- **Authorization**: object-level (can I read object X?) versus function-level (can I call admin action Y?). Most real findings are object-level because function-level is usually gated at the router while object ownership is checked deep in a service, if at all.
- **Tenancy**: in multi-tenant SaaS, which parameter selects the tenant? `org_id`, a subdomain, a path segment, a claim. Cross-tenant reads are the highest-value class in B2B.
- **Server-to-server**: internal calls between microservices often carry a weaker or absent auth model than the edge. If you can reach them (SSRF, exposed internal host, a leaked internal API), they frequently trust the caller completely.

## How value compounds

A single bug is usually a low. Hunters think in chains:

- Open redirect alone is informational. Open redirect in an OAuth `redirect_uri` is token theft and account takeover.
- Self-XSS is nothing. Self-XSS plus a login CSRF plus cookie-scoped session is stored XSS on the victim.
- An IDOR that reads a user object is a medium. The same IDOR returning a password reset token or a session identifier is account takeover.
- SSRF to a 200 is interesting. SSRF to `169.254.169.254` or a cloud metadata service is credentials, and credentials are the whole account.

Every topic file ends by asking "what does this chain into." Train the habit of not stopping at the first anomaly.

## Reading a target like a hacker

Tell-tales that a target is soft, before you send a single attack payload:

- **Inconsistent object references**: some endpoints use UUIDs, others use integers, others use email addresses. The integer and email endpoints are where access control was bolted on late.
- **Verbose errors in one place**: if one endpoint leaks a stack trace, the team's error handling is inconsistent, so other endpoints probably leak too.
- **Client-side authorization**: buttons hidden with CSS or JS `if (user.isAdmin)` tell you the admin functions exist and are reachable; the UI is the only thing stopping you.
- **Old API versions**: `/api/v1/` still routable next to `/api/v3/` usually means v1 missed a security control that v3 added.
- **Mixed frameworks**: a legacy PHP path next to a modern SPA backend means two security models, and the seam between them is where assumptions break.
- **Mobile and web share a backend**: the mobile app often hits richer or older endpoints than the web UI exposes. Proxy the app to find them.

## Discipline that keeps you out of rabbit holes

- Timebox. If a lead has not produced a signal of real impact in a set window, note it and move on; come back with fresh eyes.
- Prove impact, do not assume it. A reflected value is not XSS until it executes in a real browser context. A different response is not injection until you can explain the mechanism.
- Minimize blast radius. Use your own second account as the victim, read instead of write, use canary values you can search for, and never run destructive payloads on shared production data.
- Write the report as you go. If you cannot explain the trust boundary that broke, you do not understand the bug yet.
