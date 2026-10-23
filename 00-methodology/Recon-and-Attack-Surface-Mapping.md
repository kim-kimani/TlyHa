# Recon and Attack Surface Mapping

Recon is not a one-time phase that ends when testing starts. It is a continuous habit of expanding what you know is reachable. The hunters who find the critical bugs are usually the ones who found the endpoint nobody else did.

## The mental model

You are building a map of everything the target trusts to be unreachable or unknown. Every host, path, parameter, old version, forgotten subdomain, and third-party integration is a door. Most testers hammer the front door the vendor shipped; the money is in the doors the vendor forgot they left open.

## Horizontal versus vertical expansion

- **Horizontal**: find more assets owned by the same org. Reverse-WHOIS on registrant email and org name, ASN enumeration to find owned IP ranges, favicon hashing (`shodan` and similar index by favicon mmh3 hash), TLS certificate pivoting via `crt.sh` and censys (a cert SAN list often leaks internal hostnames and sibling domains), tracking analytics and ad IDs shared across properties, copyright footer strings, and acquisitions (newly bought companies run on their old insecure stack under the parent's scope).
- **Vertical**: go deeper on one asset. Subdomain brute force plus passive sources (certificate transparency, `subfinder`, `amass`, VirusTotal, Rapid7 Sonar data), then resolve and probe, then screenshot everything.

## Tell-tales that most people miss

- **Certificate transparency logs leak staging early**: when a team provisions a cert for `new-billing.api.target.com` before launch, the hostname is public in CT logs the moment the cert is issued, often weeks before any auth is in place.
- **Wildcard DNS hides and reveals**: a `*.target.com` wildcard means brute force returns everything; pivot to content differences and vhost fuzzing (`Host:` header) against the resolved IPs instead of relying on DNS.
- **The `.js` bundle is a sitemap**: modern SPAs ship their entire route table, API base URLs, feature flags, and sometimes hardcoded keys in webpack bundles. Pull every script, run it through a source-map extractor if `.map` files are present (they often are in staging), and grep for `api`, `/v1/`, `secret`, `token`, `admin`, `internal`.
- **Old content lives forever**: the Wayback Machine, `gau`, and `waybackurls` surface dead endpoints and parameters that the backend still routes. Old parameters frequently skipped a validation that the new ones enforce.
- **`robots.txt`, `sitemap.xml`, `security.txt`, and `.well-known/`**: `robots.txt` is a list of things they did not want indexed, which is a list of things worth looking at. `.well-known/` can leak OAuth and OIDC configuration (`openid-configuration`) that hands you every endpoint and supported flow.
- **Error pages fingerprint the stack**: a default Tomcat, Werkzeug, or ASP.NET error page tells you the framework, and often the version, which maps to known deserialization or path-handling quirks.
- **Headers leak internals**: `Via`, `X-Powered-By`, `X-Backend-Server`, `X-Cache`, `Server`, and custom `X-*` headers reveal proxies, origin hostnames, and cache layers that matter for smuggling and cache poisoning.

## Content and parameter discovery

- Directory and file fuzzing with a wordlist tuned to the stack (`raft` lists, framework-specific lists). Watch response length and status transitions, not just `200`; a `403` on `/admin` proves the path exists and is gated, which is a lead.
- Parameter mining with `arjun`, `param-miner`, or by diffing responses when you add guessed parameters. Hidden parameters (`debug=true`, `admin=1`, `test=1`, `format=json`, `callback=`) frequently unlock behavior.
- API schema discovery: look for `/swagger.json`, `/openapi.json`, `/api-docs`, `/graphql` with introspection, `.well-known` descriptors, and `/actuator` (Spring Boot) which can expose `/actuator/env`, `/actuator/heapdump` (full memory dump with secrets), and `/actuator/mappings`.

## Fingerprinting that changes your attack plan

Knowing the stack narrows the payload set:

- CDN and WAF in front (Cloudflare, Akamai, Fastly) changes whether you pursue cache poisoning and origin-IP discovery (check for direct origin exposure via historical DNS, SPF records, and services like Shodan).
- Reverse proxy chain (nginx to gunicorn, Apache to Tomcat, a CDN to an origin) is the precondition for request smuggling; note every hop.
- Framework (Rails, Django, Laravel, Express, Spring) maps to default routes, mass-assignment behavior, deserialization formats, and template engines.

## Keeping it continuous

Set up monitoring, not a one-off scan: a scheduled subdomain diff, CT-log watching for new hostnames, and re-crawl of JS bundles after releases. New deploys are where fresh, untested code lands, and the first week after a launch is often the softest the target will ever be.

## Hunting checklist

- Enumerate assets horizontally (WHOIS, ASN, favicon, certs) and vertically (subdomains, CT logs).
- Pull and read every JS bundle and source map; extract routes, API bases, keys.
- Mine historical URLs and parameters from Wayback and `gau`.
- Hit `.well-known/`, `swagger`, `openapi`, `graphql` introspection, `/actuator`.
- Map the proxy chain and WAF from headers before planning smuggling or cache attacks.
- Diff the mobile app's traffic against the web app's; the mobile backend is usually wider.
