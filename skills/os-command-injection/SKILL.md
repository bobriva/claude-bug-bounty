---
name: os-command-injection
description: Detect and prove OS command injection — separator-based syntax injection, reflection testing, filter/WAF bypass (IFS substitution, quote insertion, encoding, wildcards), argument/flag injection into wrapped CLI tools (git, ffmpeg, wkhtmltopdf, ImageMagick), and blind/OAST detection via time delays, file-write validation, and DNS/HTTP out-of-band callbacks — for authorized bug bounty and pentest work. Covers Linux/Windows command separators and discovery commands, async/background-process injection that produces no visible response signal, Burp Collaborator/interactsh-based data exfiltration with DNS-label chunking, and a validation gate plus report template focused on minimal, non-destructive proof of execution. Use for any feature touching network tools (ping/DNS/traceroute), file conversion (PDF/image/video), backup or monitoring utilities, or legacy system integrations — or writing up a command injection finding for HackerOne, Bugcrowd, or YesWeHack.
---

# OS Command Injection Testing

Same loop as the other skills in this set: **map → test → validate → report**. Command injection is one of the few bug classes where the finding itself (arbitrary code execution) is unambiguous once proven — the work is almost entirely in *proving it cleanly and safely*, especially when the application gives no visible signal at all (asynchronous/background execution), which is common and where most testers give up too early.

## Core principle: prove execution, don't perform a compromise

The goal of testing is a minimal, reproducible, non-destructive proof that attacker-controlled input reaches a system shell — not establishing a shell, dropping a backdoor, or pivoting into the internal network. A `whoami`/`sleep`/OAST-callback proof is both sufficient for almost every program's standards and the responsible stopping point: it demonstrates full impact (arbitrary command execution implies an attacker *could* do anything that user can do) without you actually doing any of it. Going further than that — interactive shells, file deletion, lateral movement, installing anything — is outside what this skill covers and outside what most programs want from a report; if you need to demonstrate broader impact, describe it ("this user can read X, write to the web root, reach internal host Y") rather than actually carrying it out.

## Before you start: scope and safety

```
1. Confirm program rules on command injection PoCs specifically —
   many explicitly require stopping at non-destructive proof (id,
   whoami, a sleep delay, an OAST callback) and immediately reporting
   rather than further exploitation.
2. Never run destructive commands (rm, file overwrites outside a
   path you created yourself, service restarts/stops) even to "prove
   impact further" — a clean read-only proof is more convincing than
   a destructive one, and destructive actions can violate program
   rules or cause real, unintended harm.
3. Identify the likely execution context first: synchronous (the
   HTTP response directly reflects command output or waits for it,
   so direct/time-based testing works) vs. asynchronous (the command
   runs in a background job/queue/cron with no connection to your
   HTTP response, so only OAST callbacks will reveal it — see
   `blind-and-oast-techniques.md`).
```

## The methodology

| Phase | Goal | Reference |
|---|---|---|
| 1. Discovery | Find functionality likely to shell out: network tools, file/image/PDF conversion, backups, monitoring, legacy integrations | This file, below |
| 2. Reflection & separator testing | Confirm injection with separators and output reflection | `references/detection-and-separators.md` |
| 3. Filter/WAF bypass | If basic separators are blocked or stripped | `references/filter-and-waf-bypass.md` |
| 4. Blind & OAST | No visible signal — time delay, file write, DNS/HTTP callback | `references/blind-and-oast-techniques.md` |
| 5. Validate | Run every candidate through the gate below | This file, "Validation gate" |
| 6. Report | Minimal, non-destructive, fast to triage | This file + `references/payloads-and-reporting.md` |

## Validation gate — run this before calling anything a finding

Every "no" means: keep digging, downgrade to "suspected," or drop it.

1. **Did you confirm execution, not just a plausible-looking response?** Reflected text that merely echoes part of your input proves nothing about code execution — you need a time delay reproduced across multiple trials with a control comparison, an OAST callback actually received on infrastructure you control, or command output that couldn't otherwise have appeared (e.g., the literal output of `id` in a format that matches a real Unix/Windows identity, not just your injected string echoed back unchanged).
2. **For async/no-response-signal cases: did the OAST callback actually arrive?** Configuring a payload and sending it is not proof; check your Collaborator/interactsh log and confirm an interaction landed, ideally with the expected encoded data inside it.
3. **Is the delay/signal clearly attributable to your payload?** Run both a "true" (delay-inducing) and a baseline (no-delay) request multiple times each and compare distributions — a single slow response proves nothing; network and backend jitter produce false positives constantly.
4. **Did you stay non-destructive?** Recon-only commands (`whoami`, `id`, `hostname`, `uname -a`, harmless file reads like `/etc/passwd` or `win.ini`) are sufficient proof. If your PoC required deleting, overwriting, or restarting anything, reconsider — both for program-rule compliance and for the strength of the report (destructive proof isn't more convincing, just riskier).
5. **Is the execution context actually attacker-reachable, not just locally reproducible by you with elevated access you already had?** Confirm the vulnerable parameter is genuinely reachable by an ordinary user of the application.
6. **Is it a duplicate?** Check disclosed reports, especially on well-known wrapped tools (wkhtmltopdf, ImageMagick) where the same underlying library bug gets rediscovered across many programs.

### Patterns that usually die at this gate (don't submit these alone)

- A single "slow" response with no repeated trials or baseline comparison
- Input reflected verbatim in an error message or output field, with no evidence it was ever passed to a shell
- A payload sent toward a suspected OAST sink with no confirmed callback received
- Filter bypass confirmed (a blocked character was evaded) with no actual execution proof behind it — bypassing input validation is not itself the finding, see `filter-and-waf-bypass.md`
- Claiming "RCE" based solely on argument/flag injection into a wrapped tool without confirming which specific flag was reachable and what it actually let you do (see the chaining table below)

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| No visible application response to an injected command | A DNS/HTTP OAST callback received with the command's actual output encoded inside it | Confirmed blind/async command execution despite zero visible signal |
| Shell metacharacters (`;`, `&&`, `|`) all blocked or stripped | Argument/flag injection into the *specific wrapped binary* being called (e.g., a dangerous flag the tool itself exposes) | Full file read/write or execution despite no shell injection being possible — this is exactly how a $12,000 GitLab report escalated a search parameter into RCE via Git's own command-line flags, and how a documented FFmpeg flag (`-dump_attachment`) turned a media-conversion endpoint into arbitrary file write |
| A character/keyword filter blocks common payloads | `${IFS}` / quote-insertion / case-inversion / base64-wrapped re-encoding bypassing that specific filter | Confirmed injection survives a defense that looked complete from the outside |
| Command execution confirmed only via timing | A file-write payload landing inside a web-accessible directory | A directly viewable, trivially reproducible proof for the report, instead of relying on timing alone |

Always check whether the underlying binary being invoked (not just the web app's own input handling) has its own dangerous flags before concluding "no shell metacharacters work, so this isn't exploitable" — argument injection into the wrapped tool is frequently the actual bug once direct shell injection is blocked.

## Writing it up

**Title formula:** `OS command injection in [exact endpoint] allows [actor] to execute arbitrary commands as [context]`
Good: `OS command injection in POST /api/diagnostics/ping allows authenticated user to execute arbitrary commands as the web server user, confirmed via OAST DNS callback`
Bad: `RCE found`

Full payload cheat-sheet, severity guidance, and report template: `references/payloads-and-reporting.md`.

## Reference files in this skill

- `references/detection-and-separators.md` — command separators by OS, reflection testing, non-destructive discovery commands
- `references/filter-and-waf-bypass.md` — IFS/quote/case/encoding bypass techniques, argument/flag injection into wrapped CLI tools
- `references/blind-and-oast-techniques.md` — time-based detection, file-write validation, DNS/HTTP OAST callbacks and chunked exfiltration
- `references/payloads-and-reporting.md` — payload cheat-sheet, CVSS quick-reference, full report template
