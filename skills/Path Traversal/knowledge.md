# Path Traversal Knowledge

## Definition

Path Traversal (Directory Traversal) occurs when user-controlled input is used in filesystem operations without proper validation, allowing access outside the intended directory.

## Common Root Causes

- Unsanitized filenames
- Weak path validation
- Improper normalization
- URL decoding inconsistencies

## Typical Impact

- Arbitrary file read
- Source code disclosure
- Credential exposure
- Sensitive configuration leakage
- Potential remote code execution chains

## Common Sensitive Files

Linux:
- /etc/passwd
- /etc/shadow
- /proc/self/environ

Windows:
- win.ini
- hosts
- web.config

Application:
- .env
- config.php
- application.properties
- settings.py