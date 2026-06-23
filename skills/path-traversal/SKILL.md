---
name: path-traversal
description: Detect and exploit path traversal / local file inclusion (LFI) — basic and nested ../ sequences, absolute-path and null-byte bypass, single/double/Unicode/overlong-UTF8 encoding bypass, server quirks (Tomcat ..;/, ASP.NET cookieless session IDs), Windows tricks (backslash/forward-slash mixing, UNC shares, 8.3 short names, NTFS alternate data streams), and escalation to RCE via Zip Slip archive-extraction traversal, PHP stream wrappers (php://filter, php://input, expect://, zip://), log poisoning, and session-file poisoning — for authorized bug bounty and pentest work. Covers a validation gate distinguishing confirmed file disclosure from a normalized/blocked attempt, a responsible-scope principle for stopping at minimal RCE proof rather than deploying a persistent webshell, and a full report template. Use for any download, file viewer, image loader, export/import, file-preview, or archive-upload feature, or writing up a path traversal/LFI finding for HackerOne, Bugcrowd, or YesWeHack.
---

# Path Traversal / LFI Testing

Same loop as the other skills in this set: **map → test → validate → report**. Path traversal is OWASP's textbook example of a vulnerability that's trivial in concept and still constantly found in production — CVE-2021-41773 (a path-normalization regression in Apache 2.4.49) proved that even mature, heavily-audited web servers reintroduce this class. The skill here isn't really "know that `../` exists" — it's knowing the encoding/normalization edge cases well enough to push past a filter that looks complete, and knowing how to escalate a "boring" file-read into something a program actually pays well for.

## Core principle: prove the boundary was actually crossed

A traversal payload returning *some* file content is not the finding — the finding is that you read a file *outside the directory the application intended you to access*, which you confirm by reading something that couldn't possibly live inside the intended base directory (`/etc/passwd`, `win.ini`, a config file with a path you specified yourself). Equally, when escalating toward RCE, the goal is minimal proof that code execution is achievable (one benign command's output, via whatever chain gets you there) — not deploying a persistent webshell or going further than confirming the chain works. This mirrors the same responsible-scope principle used in this skill set's command-injection skill, for the same reasons: it's both what programs expect and the more convincing report.

## Before you start: map file-handling functionality

```
1. List every feature that touches a filename/path supplied (even
   indirectly) by the client: file download, file/image preview,
   document viewers, export/import, backup/restore, template
   rendering, archive upload (zip/tar/jar — see Zip Slip below),
   log viewers, avatar/attachment serving.
2. Identify the parameter: filename, file, path, filepath, document,
   resource, template, page, lang, include — and check JSON bodies
   and headers, not just query strings (some frameworks build paths
   from a header like X-Forwarded-Host or a locale cookie).
3. Identify the stack: PHP (LFI via include()/require() is a distinct,
   higher-value variant — see escalation section), Java/Spring,
   Node.js, Python — each has different normalization behavior and
   different escalation paths.
```

## The methodology

| Phase | Goal | Reference |
|---|---|---|
| 1. Discovery | Map file-handling features and parameters | This file, above |
| 2. Basic & encoded traversal | `../`, absolute paths, nested sequences, encoding bypass | `references/detection-and-encoding-bypass.md` |
| 3. Platform-specific tricks | Windows/IIS, ASP.NET, Java/Tomcat, Nginx quirks | `references/windows-and-platform-quirks.md` |
| 4. Escalation | Zip Slip, PHP wrappers, log/session poisoning, LFI-to-RCE | `references/escalation-to-rce.md` |
| 5. Validate | Run every candidate through the gate below | This file, "Validation gate" |
| 6. Report | Confirmed file disclosure or minimal RCE proof | This file + `references/payloads-and-reporting.md` |

## Validation gate — run this before calling anything a finding

Every "no" means: keep digging, downgrade to "suspected," or drop it.

1. **Did you recover actual file content that couldn't exist inside the intended directory?** A different status code or error message is a lead; the literal contents of `/etc/passwd` (matching the real Unix passwd format) or `win.ini` is a finding.
2. **Did you rule out the file being accessible anyway through a legitimate path?** If the "sensitive" file you read is also served as a normal static asset, you haven't crossed a boundary — confirm the file is genuinely outside what the feature is supposed to expose.
3. **For filter-bypass claims: does the bypassed payload actually return file content, not just get accepted without erroring?** Passing a WAF/filter is not the finding by itself — see the chaining table below.
4. **For write-based findings (Zip Slip, archive extraction): did you confirm the file actually landed at the target path** — by re-requesting it, observing its effect, or another independent check — not just that the upload/extraction request returned success?
5. **If escalating to RCE, did you stop at minimal proof?** One benign command's output via the chain (log poisoning, a PHP wrapper, a poisoned session file) is sufficient and is the responsible stopping point — see the core principle above.
6. **Is it a duplicate, and is severity calibrated to what you actually read?** A generic static file vs. `/etc/shadow`/a `.env` with live credentials are very different severities even though both are "path traversal" — be specific in the report about exactly what was exposed.

### Patterns that usually die at this gate (don't submit these alone)

- A traversal-looking payload that returns a 200 with no actual sensitive content, or content that matches what a normal, legitimate request would also return
- A filter/WAF bypass confirmed (the payload wasn't rejected) with no file content recovered behind it
- Claiming "LFI to RCE possible" because PHP wrappers exist in principle, with no actual chain (log poisoning, a wrapper-based webshell, etc.) demonstrated end-to-end
- An archive upload accepted with traversal-looking entry names, with no confirmation the extracted file actually landed outside the intended directory
- Reading a file that requires no traversal at all (already directly reachable) and mislabeling it as path traversal

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| Read-only traversal restricted to text-like files in the web root | A writable, includable file (a log file, a PHP session file) on the same filesystem | LFI escalated to RCE via log/session poisoning — see `escalation-to-rce.md` |
| Traversal blocked by a string filter (`../` stripped) | Encoding (URL/double/Unicode), nested sequences, or a server-specific quirk (`..;/` on Tomcat) | Confirmed bypass restoring full read/write impact |
| Archive (zip/tar/jar) upload accepted with no path validation | A crafted entry name escaping the extraction directory (Zip Slip) into an auto-run location (web root, Startup folder, cron dir) | Direct RCE on extraction, often with zero further requests needed |
| File read confirmed, but limited to a directory with nothing obviously sensitive | A `.env`/config/source file revealing database or cloud credentials, confirmed real (format/structure matches a genuine secret, not a placeholder) | Credential exposure — report the exposure, don't use the credentials further |
| PHP `include()`-based LFI with no apparent way to write a file at all | `php://filter` base64-read to dump source first, then a PHP filter chain or `php://input`/`expect://` wrapper to get code execution without ever writing to disk | Full RCE from a "read-only-looking" LFI |

## Writing it up

**Title formula:** `Path traversal in [exact endpoint] allows [actor] to [impact]`
Good: `Path traversal in GET /download?file= allows authenticated user to read arbitrary files outside the uploads directory, including application .env containing database credentials`
Bad: `Directory traversal found`

Full payload cheat-sheet, sensitive-file target list, severity guidance, and report template: `references/payloads-and-reporting.md`.

## Reference files in this skill

- `references/detection-and-encoding-bypass.md` — basic/nested/absolute traversal, null-byte and extension-validation bypass, URL/double/Unicode encoding bypass
- `references/windows-and-platform-quirks.md` — backslash/forward-slash mixing, UNC shares, 8.3 short names, NTFS alternate data streams, Tomcat/Nginx/ASP.NET-specific quirks
- `references/escalation-to-rce.md` — Zip Slip archive-extraction traversal, PHP stream wrappers, log/session poisoning, LFI-to-RCE chains
- `references/payloads-and-reporting.md` — payload cheat-sheet, sensitive-file targets, CVSS quick-reference, full report template
