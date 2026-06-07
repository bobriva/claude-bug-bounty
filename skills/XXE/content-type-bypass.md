# XML Content-Type Bypass

## Goal

Convert non-XML endpoints into XML parsers.

---

## Original

application/x-www-form-urlencoded

---

## Test

text/xml

application/xml

---

## Replace

foo=bar

With

<?xml version="1.0"?>

<foo>bar</foo>

---

## Indicators

XML parsing errors

Different responses

Entity resolution