# Windows & Platform-Specific Quirks

Path traversal exploitation differs noticeably by server/platform — a payload that fails cleanly against one stack often works unmodified against another, so always identify the backend technology before concluding a parameter isn't exploitable.

## Windows-specific tricks

**Backslash/forward-slash mixing** — Windows treats both as path separators, but an application's filter might only check for one:
```
..\../..\../windows/win.ini
..%5c..%5cwindows%5cwin.ini
```

**UNC paths** — if the application passes the path to a Windows API that resolves UNC syntax, you may be able to reach a network share or, via certain local tricks, re-reference the local filesystem in a way a simple prefix check doesn't anticipate:
```
\\localhost\c$\windows\win.ini
\\?\C:\windows\win.ini
```

**8.3 short filenames** — legacy short-name aliases (`PROGRA~1` for `Program Files`) sometimes bypass filters that match against the full, expected filename string:
```
C:\PROGRA~1\
```

**NTFS Alternate Data Streams (ADS)** — appending `::$DATA` after a filename on NTFS can, in some legacy IIS/ASP configurations, cause the server to return file content instead of executing it (since the stream isn't recognized as the executable content), useful for source-disclosure when direct access is blocked:
```
file.asp::$DATA
```
ADS-based path-validation bypass during *archive extraction* (rather than direct file access) has also been the root cause of real, recently-patched RCE vulnerabilities in mainstream archive tools — see `escalation-to-rce.md`'s Zip Slip section for that angle specifically.

## Server-specific quirks

**Tomcat / Java servlet containers** — a semicolon after a path segment is sometimes interpreted as the start of a (legacy, rarely-used) path parameter and stripped before routing logic re-evaluates the path, while the underlying file-serving code still resolves the full string including the segment after the semicolon:
```
..;/..;/..;/etc/passwd
/admin/..;/user/profile
```
This same `;`-based confusion has historically also been used to bypass path-based *access control* rules (not just traversal) on servlet containers — if you find it works for traversal, also test it against any path-restricted admin/internal route.

**Nginx misconfiguration** — a missing trailing slash on an `alias` directive (`location /files { alias /var/www/uploads; }` instead of `location /files/ { alias /var/www/uploads/; }`) lets a request for `/files../` resolve outside the intended alias target entirely, independent of any application-level filtering — this is a server-config bug, not an app bug, but worth checking for on any nginx-fronted target serving static files.

**ASP.NET cookieless session IDs in the URL path** — when cookieless sessions are enabled, ASP.NET inserts a session-ID-shaped path segment (`(S(xxxxxx))`) into the URL; some traversal filters that pattern-match the expected URL shape can be confused by this extra segment, and it's worth testing traversal payloads both with and without a dummy session-ID-shaped segment inserted.

## Header-based path/routing confusion

Some reverse-proxy/application setups trust a header to determine the "real" requested path, independent of the actual URL — if direct traversal in the URL path is blocked, test whether the same payload works via these headers instead, especially on internal/admin routes:
```
X-Original-URL: /admin/../../etc/passwd
X-Rewrite-URL: /admin/../../etc/passwd
```

## What to check first when moving between platforms

```
1. Fingerprint the stack (response headers, error page styling,
   default file extensions, framework-specific cookies).
2. Re-run the basic and encoded payloads from
   `detection-and-encoding-bypass.md` before reaching for anything
   in this file — platform quirks are a second-pass technique once
   generic approaches are confirmed blocked, not a replacement for them.
3. Note whether the server itself (nginx/IIS/Tomcat) or the
   application code is doing the path resolution — server-level
   misconfigurations (like the nginx alias example above) need a
   different fix recommendation in your report than an application
   sanitization bug.
```
