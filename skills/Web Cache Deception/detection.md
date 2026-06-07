# Detection Techniques

## Static Extension Testing

Examples:

/profile/test.css
/profile/test.js
/profile/test.ico
/profile/test.exe

Questions:

- Does origin ignore suffix?
- Does cache store response?

---

## Delimiter Testing

Characters:

;
.
,
:
|
!
$
&
+
%

Examples:

/profile;test.css
/profile,test.css
/profile:test.css

---

## Encoded Delimiter Testing

Examples:

/profile%23test.css
/profile%3Ftest.css
/profile%00test.css
/profile%09test.css
/profile%0Atest.css

---

## Static Directory Testing

Common prefixes:

/static/
/assets/
/images/
/js/
/css/
/content/

Path Traversal Examples:

/static/..%2fprofile
/assets/..%2faccount

---

## Exact Filename Rules

Common targets:

favicon.ico
robots.txt
index.html

Examples:

/profile%2f%2e%2e%2ffavicon.ico
/profile%2f%2e%2e%2frobots.txt
