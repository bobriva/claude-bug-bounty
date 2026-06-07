# OS Command Injection Knowledge

## Definition

OS Command Injection occurs when user-controlled input is incorporated into system commands without proper validation.

## Common Sinks

- exec()
- system()
- shell_exec()
- popen()
- Runtime.exec()
- ProcessBuilder

## Common Impact

- Remote Command Execution
- Sensitive File Access
- Credential Theft
- Server Compromise
- Lateral Movement

## Blind Variants

- Time-based
- File-write based
- DNS OAST
- HTTP OAST

## OAST Importance

Many command injection vulnerabilities are invisible from application responses and require out-of-band detection techniques.