---
name: file-upload
description: Comprehensive file upload exploitation skill covering 3 main categories - Basic Vectors (extension/MIME/path), Advanced Vectors (polyglot/race/PUT), and Server Misconfiguration (Apache/.htaccess, IIS/web.config, XXE/SVG injection).
---

# File Upload Testing Skill

## When to Use This Skill

Use this skill when testing:
- Image uploads (avatar, profile picture, media)
- Document uploads (PDF, Word, Excel)
- Import features (bulk upload, file import)
- File attachments (comments, messages, support tickets)
- Media processing (thumbnails, conversions, resizing)
- URL-based file uploads
- Any application accepting file uploads

---

## Core Methodology (5 Phases)

See **methodology.md** for detailed 5-phase approach:

1. **Phase 1:** Discover upload features
2. **Phase 2:** Identify validation mechanisms
3. **Phase 3:** Bypass validation
4. **Phase 4:** Test execution
5. **Phase 5:** Assess impact

---

## The 3 Categories

### 📁 Category 1: BASIC VECTORS

**Focus:** Foundational file upload exploitation techniques

**Covers:**
- ✅ Extension bypass (double ext, case variation, null bytes)
- ✅ Content-Type/MIME spoofing
- ✅ Path traversal (directory navigation)
- ✅ Magic byte manipulation basics

**Applicable to:** ~50% of vulnerable applications

**Expected bounty:** $300-$1,500

**Files:**
- [1.1 Extension Bypass](BASIC_VECTORS/1.1_extension-bypass.md)
- [1.2 Content-Type Bypass](BASIC_VECTORS/1.2_content-type-bypass.md)
- [1.3 Path Traversal Upload](BASIC_VECTORS/1.3_path-traversal-upload.md)

**Quick Start:**
```
1. Try uploading .php file directly
2. If blocked, try: shell.php.jpg
3. If blocked, try: shell.phtml
4. If blocked, try: shell.pHp (case variation)
5. If blocked, try: shell.php%00.jpg (null byte)
6. If blocked, check Content-Type header
7. Use checklist.md to track progress
```

---

### 🎨 Category 2: ADVANCED VECTORS

**Focus:** Sophisticated bypass and exploitation techniques

**Covers:**
- ✅ Polyglot files (image + executable code)
- ✅ Race conditions (timing-based exploitation)
- ✅ PUT method uploads (direct HTTP upload)
- ✅ Magic byte manipulation (hex-level)
- ✅ Multi-layer validation bypass

**Applicable to:** ~30% of vulnerable applications

**Expected bounty:** $1,000-$3,000

**Files:**
- [2.1 Polyglot Advanced](ADVANCED_VECTORS/2.1_polyglot-advanced.md)
- [2.2 Race Condition](ADVANCED_VECTORS/2.2_race-condition.md)
- [2.3 PUT Upload](ADVANCED_VECTORS/2.3_put-upload.md)

**Quick Start:**
```
1. Create polyglot: GIF89a<?php system($_GET['c']); ?>
2. Upload as image file with PHP extension
3. Access file - code executes
4. If single-request, try race condition timing
5. If no upload form, try OPTIONS then PUT method
```

---

### ⚙️ Category 3: SERVER MISCONFIGURATION

**Focus:** Server-specific exploitation and configuration abuse

**Covers:**
- ✅ .htaccess upload abuse (Apache)
- ✅ web.config handler mapping (IIS)
- ✅ .user.ini auto execution (PHP-FPM)
- ✅ nginx.conf abuse (Nginx)
- ✅ SVG/XXE injection (XML processing)
- ✅ Stored XSS via file upload

**Applicable to:** ~20% of vulnerable applications

**Expected bounty:** $500-$5,000 (highly variable)

**Files:**
- [3.1 .htaccess & web.config](SERVER_MISCONFIG/3.1_htaccess-webconfig-enhanced.md)
- [3.2 XSS/XXE SVG Injection](SERVER_MISCONFIG/3.2_xss-xxe-svg-injection.md)

**Quick Start:**
```
1. Upload .htaccess: AddType application/x-httpd-php .evil
2. Upload shell.evil - now executes as PHP
3. Or upload SVG with XXE: <!ENTITY xxe SYSTEM "file:///etc/passwd">
4. Or test .user.ini with auto_prepend_file
```

---

## Quick Decision Tree

**Which category should I use?**

```
Upload vulnerable?
│
├─ YES, PHP executes directly
│  └─ Already RCE (job done!)
│
├─ Upload blocked/filtered
│  ├─ Try BASIC_VECTORS first
│  │  ├─ Extension bypass? → Try all 7 techniques
│  │  ├─ MIME check? → Spoof Content-Type
│  │  └─ Magic bytes? → GIF89a prefix
│  │
│  ├─ Still blocked?
│  │  └─ Try ADVANCED_VECTORS
│  │     ├─ Create polyglot
│  │     ├─ Try race condition
│  │     └─ Try PUT method
│  │
│  └─ Still nothing?
│     └─ Try SERVER_MISCONFIG
│        ├─ Upload .htaccess
│        ├─ Upload web.config
│        └─ Upload SVG/XXE payload
│
├─ No direct file access
│  ├─ Check includes
│  ├─ Check processing (ImageMagick?)
│  └─ Check auto-execution (.user.ini)
│
└─ No upload form exists
   └─ Try PUT method (OPTIONS request first)
```

---

## Reference Files

### knowledge.md
Categorized knowledge base by vulnerability type

### checklist.md
Comprehensive testing checklist, grouped by category

### payloads.md
Ready-to-use payload templates for all categories

---

## Common Outcomes

| Outcome | Impact | Bounty Range |
|---------|--------|-------------|
| RCE (Code Execution) | Critical | $1,000-$10,000 |
| Stored XSS | Medium | $500-$2,000 |
| File Disclosure | Medium | $300-$1,000 |
| Path Traversal | Medium | $500-$2,000 |
| XXE/SSRF | High | $1,000-$5,000 |

---

## Testing Workflow

```
1. Read methodology.md (understand 5 phases)
2. Pick relevant category (see above)
3. Follow quick start for that category
4. Use checklist.md to track attempts
5. Reference payloads.md for examples
6. Validate with knowledge.md
7. Document findings
8. Report bounty
```

---

## Tools Mentioned

- **Burp Suite** - Intercept, modify, repeat requests
- **ExifTool** - Manipulate image metadata
- **curl** - Command-line HTTP requests
- **ffmpeg** - Create polyglot files
- **ImageMagick** - Identify file types, create polyglots
- **Nuclei** - Automated exploitation scanning

---

## Organization Structure

```
file-upload-skill/
│
├── SKILL.md (this file - navigation)
├── methodology.md (5-phase reference)
│
├── BASIC_VECTORS/
│  ├── 1.1_extension-bypass.md
│  ├── 1.2_content-type-bypass.md
│  └── 1.3_path-traversal-upload.md
│
├── ADVANCED_VECTORS/
│  ├── 2.1_polyglot-advanced.md
│  ├── 2.2_race-condition.md
│  └── 2.3_put-upload.md
│
├── SERVER_MISCONFIG/
│  ├── 3.1_htaccess-webconfig-enhanced.md
│  └── 3.2_xss-xxe-svg-injection.md
│
├── REFERENCE/
│  ├── knowledge.md
│  ├── checklist.md
│  └── payloads.md
│
└── NAVIGATION/
   ├── INDEX.md (detailed navigation)
   ├── QUICK_START.md (quick reference)
   └── README.md (overview)
```

---

## Next Steps

1. **New to file uploads?** → Read methodology.md first
2. **Want quick wins?** → Start with BASIC_VECTORS
3. **Blocked everywhere?** → Try ADVANCED_VECTORS
4. **Nothing standard working?** → Try SERVER_MISCONFIG
5. **Need payloads?** → Check payloads.md
6. **Testing systematically?** → Use checklist.md

---

## Tips for Success

- ✅ Always start with BASIC_VECTORS (easiest wins)
- ✅ Use checklist.md to avoid missing techniques
- ✅ Reference payloads.md for syntax
- ✅ Combine multiple techniques (chain attacks)
- ✅ Test all file types (not just PHP)
- ✅ Check alternative execution paths
- ✅ Verify impact before reporting

---

## Common Mistakes to Avoid

- ❌ Giving up on first blocked upload
- ❌ Not trying all 7 extension bypass variations
- ❌ Forgetting to test Content-Type header
- ❌ Not checking alternative file access paths
- ❌ Assuming server isn't processing files
- ❌ Not combining multiple techniques

---

## Version

- Last Updated: June 2026
- Status: Reorganized & Enhanced (3-category structure)
- Files: 13 organized into 3 main categories
- Coverage: Basic → Advanced → Server-specific exploitation

---

*This skill provides comprehensive file upload exploitation methodology organized into 3 practical categories. Start with the category that best fits your situation.*
