# SAML and SSO

SAML is XML signatures carrying identity assertions. The bugs are canonicalization quirks, signature-scope confusion, and XML parser differences between the thing that verifies the signature and the thing that reads the assertion. If those two disagree about what the document says, you forge identity.

## The mental model

An Identity Provider (IdP) signs an XML assertion that says "this is user X." The Service Provider (SP) verifies the signature against the IdP's certificate, then reads the `NameID` and attributes to log the user in. The attack surface: make the SP verify a signature over one thing while reading a different thing, or make it accept an assertion that was not signed by the real IdP, or exploit the XML layer itself (XXE, comment injection, canonicalization).

## Where it hides / tell-tales

- **Signature wrapping (XSW)**: the response contains one signed assertion and one attacker-injected assertion. The signature check validates the legitimate one; the application logic reads the injected one. The classic SAML break.
- **Comment injection in `NameID`**: `<NameID>admin@target.com<!---->.evil</NameID>`. Some XML libraries return only the text before the comment when reading the node, so the signature covers the full string but the app sees `admin@target.com`. This broke many SAML libraries at once.
- **Unsigned or partially signed responses**: the `Response` is signed but individual `Assertion` elements are not, or vice versa, and the SP checks the wrong one. Or neither is signed and the SP accepts it.
- **`xmlsec` and canonicalization mismatches**: differences in how whitespace, namespaces, and transforms are canonicalized let two parsers read different content.
- **XXE in the SAML endpoint**: the SP parses attacker-supplied XML (the SAMLResponse is base64 in a POST). If the parser resolves external entities, you get SSRF and file read before any identity logic runs.
- **IdP-initiated flow with no `InResponseTo` binding**: allows replay and unsolicited assertions.
- **Certificate trust too loose**: SP accepts any valid-looking cert, a self-signed cert, or does not pin the IdP cert, so you sign with your own.

## Real material

The SAMLResponse is a base64-encoded (and in redirect binding, deflated) XML blob in a form POST:

```http
POST /saml/acs HTTP/1.1
Content-Type: application/x-www-form-urlencoded

SAMLResponse=PHNhbWxwO1Jlc3BvbnNl...&RelayState=/dashboard
```

A simplified assertion showing where identity lives:

```xml
<samlp:Response ... >
  <saml:Assertion ID="_a1">
    <ds:Signature>...</ds:Signature>
    <saml:Subject>
      <saml:NameID>user@target.com</saml:NameID>
    </saml:Subject>
    <saml:AttributeStatement>
      <saml:Attribute Name="role"><saml:AttributeValue>user</saml:AttributeValue></saml:Attribute>
    </saml:AttributeStatement>
  </saml:Assertion>
</samlp:Response>
```

Signature wrapping idea: keep the signed assertion intact so the signature validates, and add a second, unsigned assertion with `NameID=admin@target.com` positioned so the application's XPath selects it instead of the signed one. The exact placement (sibling, child, before, after, wrapped in an `Extensions` element, matching or shadowing the signed `ID`) depends on how the SP selects the assertion to read; try the known XSW permutations.

Comment-injection NameID:

```xml
<saml:NameID>admin@target.com<!---->.attacker@evil.tld</saml:NameID>
```
Signed as the full string, but a vulnerable text-extraction returns `admin@target.com`.

XXE in the parser, before signature logic:

```xml
<?xml version="1.0"?>
<!DOCTYPE samlp:Response [ <!ENTITY x SYSTEM "file:///etc/passwd"> ]>
<samlp:Response>...&x;...</samlp:Response>
```

## Beyond the basics

- **Decode and re-sign is the first test**: base64-decode (and inflate for redirect binding) the response, modify `NameID` to a victim or `admin`, re-encode, and send. If it is accepted, the SP is not verifying the signature at all. You would be surprised how often this works on internal SPs.
- **XSW permutation sweep**: there are roughly eight canonical XML signature wrapping variants differing in where the forged assertion sits relative to the signed one and the signature. Run them all; SPs differ in which XPath they use to locate the assertion.
- **The two-parser discrepancy**: identify whether the signature library and the business logic use the same XML parser. If the SP verifies with one library and reads attributes with another (common in polyglot stacks), canonicalization and comment handling diverge, and that gap is the exploit.
- **`InResponseTo` and replay**: capture a valid assertion and replay it; check whether `InResponseTo`, `NotOnOrAfter`, and one-time-use are enforced. IdP-initiated SSO often is not bound to a request.
- **RelayState open redirect and XSS**: `RelayState` is reflected into a redirect or page; test it as an open redirect and injection sink independent of the SAML logic.
- **Metadata poisoning**: if the SP fetches IdP metadata from a URL or lets you upload it, a malicious metadata document can register your certificate as trusted.

## What to watch for

- SAML is fragile; a malformed document often just errors. Keep a known-good response as your baseline and change one thing at a time.
- Many mature SP libraries (modern `python3-saml`, `ruby-saml`, Shibboleth) patched XSW and comment injection; confirm the version and behavior rather than assuming.
- Deflate/inflate and base64 and URL-encoding order differs between redirect and POST bindings; get the encoding right or you will think a bypass failed when you simply sent garbage.

## Hunting checklist

- Decode the SAMLResponse; try modifying `NameID`/attributes without re-signing (tests whether signatures are checked at all).
- Run the full XSW permutation set.
- Test comment injection in `NameID` and other identity fields.
- Test the ACS endpoint for XXE before identity logic.
- Check `InResponseTo`, `NotOnOrAfter`, and replay protection.
- Test `RelayState` as open redirect and XSS.
- Check whether IdP metadata/cert can be supplied or poisoned.
