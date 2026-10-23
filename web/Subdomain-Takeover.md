# Subdomain Takeover

A subdomain takeover happens when a DNS record points at a third-party service that no longer owns the resource, so you claim it. You then control content on a hostname the organization trusts, which is worth far more than it first appears: cookie theft, OAuth redirect abuse, CSP and CORS trust, and phishing with a real certificate on a real subdomain.

## The mental model

Organizations create DNS records (`CNAME`, `A`, `NS`, `MX`) pointing at cloud services (S3, GitHub Pages, Heroku, Azure, Fastly, a SaaS). When they later delete the resource but forget the DNS record, the record dangles. If the third-party service lets anyone claim that name (an unclaimed S3 bucket, an unregistered GitHub Pages repo, a free Heroku app name), you register it and now serve content at the victim's subdomain. The bug is the dangling pointer plus a claimable backend.

## Where it hides / tell-tales

- **`CNAME` to a deprovisioned service**: `blog.target.com CNAME target.github.io` where the GitHub Pages site returns a 404 "There isn't a GitHub Pages site here." That fingerprint plus a claimable repo is a takeover.
- **Service-specific fingerprints**: each provider returns a distinctive error for an unclaimed resource (`NoSuchBucket` for S3, Fastly's "Fastly error: unknown domain," Heroku's "No such app," Shopify, Tumblr, Zendesk, Surge, Netlify, Azure `cloudapp`/`trafficmanager` NXDOMAIN patterns). The error is the tell.
- **Dangling `NS` records**: delegation to a nameserver/zone you can register at a provider; rarer but gives full control of the subtree.
- **Expired cloud resources**: an `A` record to an elastic IP or VM that was released; if you can re-acquire that IP (hard, but possible on some clouds), you serve from it.
- **Forgotten staging/marketing subdomains**: the stuff spun up for a campaign and never cleaned up, found via CT logs and subdomain enumeration (see `../00-methodology/Recon-and-Attack-Surface-Mapping.md`).

## Real material

Discovery flow:

```
1. Enumerate subdomains (passive: CT logs, subfinder/amass; active: brute).
2. Resolve each; note CNAME targets and A records.
3. For CNAMEs to third-party services, fetch the hostname over HTTP/HTTPS.
4. Match the response against the known "unclaimed" fingerprints.
```

Example fingerprints:

```
S3:        "The specified bucket does not exist" / NoSuchBucket
GitHub:    "There isn't a GitHub Pages site here."
Heroku:    "No such app" / herokucdn.com error page
Fastly:    "Fastly error: unknown domain:"
Azure:     NXDOMAIN on *.azurewebsites.net / *.cloudapp.net / *.trafficmanager.net
Shopify:   "Sorry, this shop is currently unavailable."
Zendesk:   "Help Center Closed" / nonexistent subdomain
```

Claiming (example, S3): the `CNAME` points at `target-assets.s3.amazonaws.com`; that bucket name is free; you create a bucket with exactly that name in the right region, enable static hosting, and serve your content at `assets.target.com`.

## Beyond the basics

- **Impact is about the trust the hostname carries, not the page**: a claimed subdomain lets you (1) set cookies on the parent domain if any cookie uses `Domain=.target.com`, enabling session fixation and stealing cookies scoped to the parent (see `Session-and-Cookie-Security.md`); (2) satisfy CORS allowlists that trust `*.target.com`, reading the main API with credentials (see `CORS-Misconfiguration.md`); (3) host an OAuth `redirect_uri` or be in a CSP `script-src` allowlist that trusts subdomains, turning takeover into token theft or XSS on the main app; (4) phish with a valid TLS certificate on a genuine subdomain; (5) receive emails if an `MX` takeover, intercepting password resets.
- **Second-order subdomain takeover**: the main app loads a script, image, or iframe from a subdomain you took over, so your content executes in the main app's context. Hunt for `<script src>`, `<link>`, and `<iframe>` references to takeover-able subdomains across the primary application.
- **NS takeover is the jackpot**: controlling a delegated nameserver lets you answer for the entire subdomain tree, create arbitrary records (including `A`, `MX`, `TXT` for domain validation), and potentially issue certificates. Rare, high severity.
- **Dangling records in CDN/WAF configs**: a Fastly/CloudFront/Azure Front Door config that references a backend or domain no longer owned; the takeover is at the CDN layer.
- **Cert issuance and domain validation**: a claimed subdomain passing HTTP/DNS validation lets you obtain a trusted certificate, defeating certificate pinning assumptions and enabling convincing MITM-style phishing.

## What to watch for

- A `404` is not always a takeover; confirm the backend resource is genuinely claimable and that claiming it serves content at the victim hostname. Some fingerprints are ambiguous.
- Many providers now require domain verification before serving a claimed name (to fix this exact issue); verify the service still allows unverified claiming before reporting.
- Claiming the resource is the proof, but keep the served content a benign proof-of-ownership page; do not phish or serve malicious content.
- Do not disrupt any legitimately-served content; only claim genuinely dangling resources.

## Hunting checklist

- Enumerate all subdomains (passive and active) and resolve their records.
- Fetch each third-party-pointing host and match against provider "unclaimed" fingerprints.
- Check for `NS` and `MX` dangles, not just `CNAME`.
- For a confirmed claimable resource, claim it and serve a benign proof page.
- Assess impact via parent-domain cookies, `*.target.com` CORS/CSP trust, OAuth redirect allowlists, and references from the main app (second-order).
