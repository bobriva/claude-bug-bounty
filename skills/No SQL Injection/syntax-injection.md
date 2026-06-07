# Syntax Injection

## Goal

Break query syntax and inject custom logic.

---

## Discovery

Inject:

'

"

`

{

}

---

## Boolean Conditions

False:

' && 0 && 'x

True:

' && 1 && 'x

---

## Always True

'||'1'=='1

---

## Null Byte

%00

Goal:

Terminate query logic.

---

## Indicators

Different results

More records returned

Authentication bypass

Hidden content exposure