# OS Command Injection Checklist

## Discovery

[ ] Network tools
[ ] File processing
[ ] PDF generation
[ ] Backup systems
[ ] Monitoring systems

## Detection

[ ] &
[ ] &&
[ ] |
[ ] ||
[ ] ;
[ ] newline

## Reflection

[ ] whoami
[ ] hostname
[ ] id

## Blind

[ ] Time delay
[ ] File write
[ ] DNS callback

## OAST

[ ] DNS interaction
[ ] HTTP interaction
[ ] SMB interaction

## Impact

[ ] Arbitrary command execution
[ ] Credential access
[ ] System compromise