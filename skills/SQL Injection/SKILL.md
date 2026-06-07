---
name: sql-injection
description: Detect, validate, and exploit SQL Injection vulnerabilities across all contexts including error-based, union-based, blind, time-based, OAST, and second-order SQLi.
---

# SQL Injection Skill

Use this skill when:

- User input reaches a database
- Error messages reference SQL
- Parameters affect returned records
- Login functionality exists
- Search, filters, sorting, or reporting features exist
- XML/JSON data is processed server-side

## Objectives

1. Detect SQL injection
2. Identify DBMS
3. Determine injection context
4. Extract evidence safely
5. Assess impact
6. Generate reproducible PoC

## Testing Priority

1. Error-based SQLi
2. Boolean-based SQLi
3. UNION-based SQLi
4. Time-based SQLi
5. OAST SQLi
6. Second-order SQLi

Always verify manually before reporting.

Refer to:

- methodology.md
- detection.md
- exploitation.md
- dbms-fingerprinting.md
