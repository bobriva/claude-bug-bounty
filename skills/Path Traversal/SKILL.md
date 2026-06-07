---
name: path-traversal
description: Detect and exploit path traversal vulnerabilities to access arbitrary files outside intended directories.
---

# Path Traversal Skill

Use this skill when:

- File download functionality exists
- File upload functionality exists
- Image loading endpoints exist
- Export/import features exist
- File preview features exist
- User-controlled filenames exist

## Objectives

1. Identify file access functionality
2. Detect traversal sequences
3. Bypass filtering mechanisms
4. Access sensitive files
5. Assess impact

## High Value Findings

- Arbitrary File Read
- Source Code Disclosure
- Credential Disclosure
- Configuration Disclosure
- Potential RCE Chaining

Refer to:

- methodology.md
- detection.md
- exploitation.md