# Secrets and Information Disclosure

Half of real-world compromise is not a clever exploit; it is finding a credential someone left lying around. Keys in JS bundles, tokens in git history, secrets in error pages, verbose debug endpoints, and metadata leaks are the unglamorous bugs that end in full access. This file is about seeing what the target is accidentally telling you.

## The mental model

Applications leak in two directions: they expose secrets (keys, tokens, credentials, internal URLs) that unlock other systems, and they expose information (versions, stack traces, internal structure, user data) that lowers the cost of every other attack. The mindset is to read everything the target emits, including the parts not meant for humans, and to recognize the shape of a secret or an internal detail when it appears.

## Where it hides / tell-tales

- **Client-side bundles**: webpack/JS bundles and their source maps (`.map`) contain API keys, internal endpoints, feature flags, and sometimes credentials. Modern SPAs ship the whole app; read it.
- **Git and VCS exposure**: `/.git/` served publicly (download and reconstruct the repo with its full history, including secrets since deleted), `.svn`, `.hg`, and `/.git/config` leaking remotes. Git history is where "we removed that key" keys live.
- **Config and backup files**: `.env`, `config.php.bak`, `web.config`, `application.properties`, `settings.py`, `docker-compose.yml`, `.DS_Store` (directory listing), `*.sql` dumps, `*.bak`, `*~`, editor swap files.
- **Verbose endpoints**: Spring `/actuator/env`, `/actuator/heapdump` (a full memory dump containing live secrets and sessions), `/actuator/mappings`; debug toolbars; GraphQL introspection; `phpinfo()`; `/server-status`; stack traces with full paths and framework versions.
- **Cloud and metadata leakage**: S3/GCS bucket listings, public blobs, `.well-known/` descriptors, instance metadata reachable via SSRF (see `../cloud-and-infra/Cloud-Metadata-and-SSRF-to-Cloud.md`).
- **Response and header over-sharing**: PII in API responses (see `REST-API-Security.md`), internal hostnames in headers, email addresses in error messages, user enumeration via differential responses, and version banners.
- **Third-party and public leaks**: secrets in public GitHub repos and gists, Postman collections, CI logs, Docker images on registries, mobile app binaries (see `../mobile/`), and paste sites. GitHub dorking the org name is a first-day task.

## Real material

Reconstructing a git repo from an exposed `/.git/`:

```
# if /.git/ is listable or objects are fetchable:
# tools like git-dumper pull the whole tree; then:
git log --all -p | grep -iE 'api[_-]?key|secret|password|token|AKIA'
git stash list; git show <deleted-commit>   # removed secrets live in history
```

Secret shapes to grep for (in bundles, responses, repos, memory dumps):

```
AKIA[0-9A-Z]{16}                 AWS access key id
aws_secret_access_key = ...      40-char base64-ish
ghp_[0-9A-Za-z]{36}              GitHub personal access token
xox[baprs]-[0-9A-Za-z-]+         Slack token
sk_live_[0-9A-Za-z]{24,}         Stripe secret key
-----BEGIN (RSA|EC|OPENSSH) PRIVATE KEY-----
eyJ...                           JWT (decode it)
mongodb(+srv)?://user:pass@...   connection strings
postgres://user:pass@host/db
```

Spring Boot actuator jackpot:

```
GET /actuator/env       -> environment variables, often with secrets (partially masked, sometimes not)
GET /actuator/heapdump  -> downloads heap; open in a memory analyzer, extract live tokens/sessions
GET /actuator/mappings  -> full route list (feeds BFLA hunting)
```

Source map to original source:

```
# app.min.js references app.min.js.map
# fetch the .map, run a source-map extractor; get original file tree, comments,
# internal API paths, and sometimes committed secrets.
```

## Beyond the basics

- **Treat the JS bundle as the primary recon artifact**: it contains the route table, API bases, client-side feature flags (which reveal unreleased/admin features), third-party keys, and comments. Extracting source maps gives near-source visibility. This single step often finds the endpoint or key that unlocks everything else.
- **Git history over current tree**: the most valuable secrets are the ones that were committed, used, then "removed." They remain in history and in old branches/tags. Always walk `--all` history, not just HEAD.
- **Validate and scope every secret found**: a key is only a finding if it works and if its scope matters. Check what it unlocks (read vs write, which service, which environment) using the provider's safe identity/whoami call, not destructive actions. An over-scoped "public" key (a read key that also writes, a test key that hits prod) is the real severity.
- **Memory and crash dumps**: heap dumps (`/actuator/heapdump`, core dumps, `.hprof`), error pages that echo environment, and debug endpoints expose live secrets and session tokens, not just config. These are often more valuable than config files because they contain runtime state.
- **Information that lowers attack cost**: exact framework/library versions (map to known CVEs and deserialization gadgets), internal hostnames and IPs (SSRF target list), user enumeration (feeds credential attacks), and error-based oracles (feed blind injection). Catalog these even when no single one is "critical."
- **Automated and continuous discovery**: secret scanners (`trufflehog`, `gitleaks`) on repos and bundles, `nuclei` templates for exposed `.git`/`.env`/actuator, and monitoring for new leaks after each deploy. New releases re-leak; watch continuously (see `../00-methodology/Recon-and-Attack-Surface-Mapping.md`).
- **Chaining**: a leaked internal URL feeds SSRF targeting; a leaked framework version feeds a deserialization chain; a leaked API key feeds cloud enumeration; a leaked source map reveals the endpoint for a BOLA. Disclosure is rarely the end, it is the enabler.

## What to watch for

- A string that looks like a secret may be a public/publishable key (some keys are meant to be client-side, like a Stripe publishable `pk_` or a Firebase web config). Confirm it is actually sensitive and actually works before reporting.
- Validate keys with read-only/identity calls; do not perform actions that modify or cost money with a found credential, and do not access data beyond minimal proof.
- Exposed `.git`/backups may be in scope issues on bounty programs or may be considered known; check program rules and avoid downloading large sensitive dumps beyond proof.
- Reporting a secret means it must be rotated; handle it carefully and never retain or reuse it.

## Hunting checklist

- Pull and read every JS bundle; extract source maps; harvest routes, keys, flags, comments.
- Check for `/.git/`, `/.svn/`, `.env`, backups, `.DS_Store`, and reconstruct repos; walk full git history.
- Probe verbose endpoints: actuator (`env`/`heapdump`/`mappings`), debug toolbars, `phpinfo`, `server-status`, GraphQL introspection.
- Read API responses and headers for PII, internal hostnames, and version banners.
- Dork public GitHub/gists/Postman/registries for the org's leaked secrets.
- Validate every secret's scope with safe calls; map each leak to what it chains into.
