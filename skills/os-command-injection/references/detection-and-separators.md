# Detection & Command Separators

## Where to look

Command injection lives wherever an application hands user-influenced input to a system shell instead of calling a library function directly. Prioritize:

```
- Network diagnostic tools (ping, traceroute, DNS lookup, port check)
- File/image/PDF/video conversion features (often wrapping ffmpeg,
  ImageMagick, wkhtmltopdf, LibreOffice under the hood)
- Backup, export, and archival features (often shelling out to tar,
  zip, rsync)
- Monitoring/health-check and "test this integration" features
- Legacy integrations and anything with a changelog suggesting it
  predates the current stack (old PHP/Perl/CGI scripts are
  disproportionately likely culprits)
```

Common code-level sinks if you're reviewing source rather than black-box testing: `exec()`, `system()`, `shell_exec()`, `popen()`, `Runtime.exec()`, `ProcessBuilder` (Java), `subprocess.Popen`/`os.system` with `shell=True` (Python), backticks/`` ` `` and `system()` (Ruby/Perl). Note that `ProcessBuilder` and `subprocess.Popen` *without* `shell=True`/an array of arguments are generally **not** vulnerable to shell metacharacter injection — but can still be vulnerable to argument/flag injection if user input becomes one of the array elements; see `filter-and-waf-bypass.md`.

## Command separators

Test each individually around a baseline value the application expects (e.g., an IP address for a ping tool):

```
Works on both Linux and Windows shells:
  &    &&    |    ||

Linux/Unix only:
  ;
  (newline)
  `command`          (backtick inline execution)
  $(command)          (inline execution)
```

A useful starting probe combining several at once, since you don't yet know which separator (if any) the application's shell call respects:
```
127.0.0.1; whoami
127.0.0.1 && whoami
127.0.0.1 | whoami
127.0.0.1 || whoami
127.0.0.1 & whoami &
```

## Reflection and confirmation tests

```
whoami
id
hostname
echo test123
```
`echo test123` is a useful canary: if `test123` appears in the response somewhere it couldn't otherwise have come from, that's a clean confirmation independent of guessing the OS or any real system detail. Once confirmed, move to the OS-fingerprinting commands below to plan further (non-destructive) discovery.

**Distinguish real execution from coincidence:** before concluding `whoami` "worked," check that the returned value looks like a real identity for the platform in question (a plausible Linux username, or a `DOMAIN\user`-shaped Windows identity) — not just that *something* changed in the response, which can also happen for unrelated reasons (caching, rate limiting, a generic error page).

## Indicators something happened, even without clean output

```
- Reflected command output appearing somewhere in the response
- A new/different error message (e.g., a shell syntax error,
  "command not found," or a stack trace naming a process-execution
  function)
- A response status code or content-length change correlated
  specifically with your injected separator, not present for a
  syntactically similar but non-separator character
- An unexpected delay (see `blind-and-oast-techniques.md` if there's
  no other visible signal at all)
```

## Non-destructive discovery commands (use only after execution is confirmed)

**Environment:**
```
Linux:   whoami   id   uname -a   hostname
Windows: whoami   ver   hostname
```

**Network configuration (useful for documenting blast radius — e.g., "this process can also reach internal host X" — describe this in the report rather than actually pivoting further):**
```
Linux:   ifconfig   ip addr   netstat -an
Windows: ipconfig /all   netstat -an
```

**Process listing:**
```
Linux:   ps -ef
Windows: tasklist
```

**Harmless file reads, to demonstrate file-read impact without touching anything sensitive beyond confirming the capability:**
```
Linux:   cat /etc/passwd
Windows: type C:\Windows\win.ini
```
If you go further to check for credential exposure (`.env` files, config files, database credential files), treat any value you encounter as sensitive: note that the capability exists and what kind of secret was reachable in your report, but avoid copying full credential values into the report itself — describe the exposure rather than reproducing secrets verbatim.

## What this phase confirms vs. what it doesn't

Separator and reflection testing confirm injection is real and give you a foothold for discovery — they're not yet a complete report on their own if the application gave you a clean, synchronous, visible response. If instead you got no visible signal at all despite trying every separator, move to `blind-and-oast-techniques.md` before concluding the endpoint isn't vulnerable; absence of visible output is extremely common for async/background execution contexts and does not mean injection failed.
