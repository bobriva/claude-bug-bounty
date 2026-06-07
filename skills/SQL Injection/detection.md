# Detection Techniques

## Syntax Testing

'
''
"
`

Indicators:

- SQL errors
- HTTP 500
- Different responses

---

## Boolean-Based

TRUE:

' OR 1=1--

FALSE:

' OR 1=2--

Look for:

- Different page content
- Different row counts
- Different response length

---

## ORDER BY Testing

ORDER BY 1
ORDER BY 2
ORDER BY 3

Determine column count.

---

## UNION Testing

' UNION SELECT NULL--

Increase NULL count until valid.

---

## Time-Based

SLEEP(5)
WAITFOR DELAY
pg_sleep(5)

Observe delays.

---

## OAST Testing

Trigger:

- DNS
- HTTP
- SMB

Look for external interactions.
