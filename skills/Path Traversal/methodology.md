# Path Traversal Methodology

## Phase 1 - Discovery

Identify:

- Download endpoints
- Image endpoints
- Document viewers
- Export functions
- Import functions
- Backup downloads

Examples:

/download
/file
/image
/export
/view

---

## Phase 2 - Input Identification

Look for:

filename=
file=
path=
filepath=
document=
template=
resource=

---

## Phase 3 - Traversal Testing

Test:

../
..\

Observe:

- Response changes
- File contents
- Error messages

---

## Phase 4 - Filter Bypass

Test:

- Absolute paths
- Nested traversal
- URL encoding
- Double encoding
- Null byte injection

---

## Phase 5 - Impact Validation

Attempt access to:

- Configuration files
- Source code
- Credentials
- System files