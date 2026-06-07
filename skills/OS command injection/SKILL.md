---
name: os-command-injection
description: Detect and exploit operating system command injection vulnerabilities including blind and OAST-based command execution.
---

# OS Command Injection Skill

Use this skill when:

- User input reaches system commands
- File operations occur
- Network utilities are invoked
- Legacy integrations exist
- Monitoring or diagnostic features exist

## Objectives

1. Identify command execution sinks
2. Detect command injection
3. Identify execution context
4. Verify code execution
5. Escalate to blind/OAST techniques
6. Assess infrastructure impact

## High Value Findings

- Remote Command Execution
- Arbitrary OS Command Execution
- Server Compromise
- Credential Exposure
- Lateral Movement

Refer to:

- methodology.md
- detection.md
- exploitation.md
- blind-oast.md