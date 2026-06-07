# OS Command Injection Methodology

## Phase 1 - Discovery

Identify functionality:

- Ping tools
- DNS lookup tools
- Network diagnostics
- File processing
- Image processing
- PDF generation
- Backup systems
- Monitoring systems

---

## Phase 2 - Reflection Testing

Attempt command separators.

Observe:

- Output reflection
- Errors
- Response differences

---

## Phase 3 - Environment Discovery

Identify:

- Operating system
- Current user
- Network configuration
- Running processes

---

## Phase 4 - Blind Testing

Use:

- Time delays
- File write techniques
- OAST callbacks

---

## Phase 5 - Impact Validation

Verify:

- Arbitrary command execution
- Sensitive data access
- System compromise