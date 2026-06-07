# HTTP/2 Request Smuggling

Targets

HTTP/2 Frontend

HTTP/1 Backend

---

H2.CL

Injected Content-Length.

Backend trusts Content-Length.

---

H2.TE

Injected Transfer-Encoding.

Backend trusts chunked encoding.

---

Indicators

Downgrading

ALPN Support

Hidden HTTP/2

---

Tools

Burp Repeater

HTTP Request Smuggler