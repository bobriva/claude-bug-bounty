# SQL Injection Methodology

## Phase 1 - Identify Inputs

Test:

- Query parameters
- POST body
- JSON body
- XML body
- Cookies
- Headers
- Search fields
- Login forms
- API endpoints

---

## Phase 2 - Error Detection

Inject:

'
''
"
`

Observe:

- SQL syntax errors
- Database exceptions
- Stack traces
- Different HTTP responses

---

## Phase 3 - Boolean Testing

TRUE:

' OR 1=1--

FALSE:

' OR 1=2--

Compare:

- Response size
- Content changes
- Status codes

---

## Phase 4 - Time Testing

Inject delays.

Observe:

- Consistent response delay
- Repeatability

---

## Phase 5 - Data Extraction

Prioritize:

- Current user
- Database version
- Table names
- Column names
- Sensitive records

---

## Phase 6 - Impact Validation

Verify:

- Data exposure
- Authentication bypass
- Data modification
- Potential RCE
