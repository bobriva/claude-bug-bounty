# Path Traversal Checklist

## Discovery

[ ] Download functionality
[ ] File viewer
[ ] Image loader
[ ] Export feature
[ ] Import feature

## Testing

[ ] ../
[ ] ..\
[ ] Absolute path
[ ] URL encoded traversal
[ ] Double encoded traversal
[ ] Nested traversal
[ ] Null byte bypass

## Linux Targets

[ ] /etc/passwd
[ ] /etc/shadow
[ ] /proc/self/environ

## Windows Targets

[ ] win.ini
[ ] hosts
[ ] web.config

## Application Files

[ ] .env
[ ] config.php
[ ] application.properties
[ ] settings.py

## Impact

[ ] Arbitrary file read
[ ] Source disclosure
[ ] Credential disclosure
[ ] RCE chain potential