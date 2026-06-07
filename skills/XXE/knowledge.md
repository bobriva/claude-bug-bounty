# XXE Knowledge

## Definition

XML External Entity Injection occurs when XML parsers resolve attacker-controlled external entities.

## Main Attack Classes

1. File Read

2. SSRF

3. Blind XXE

4. Error-Based XXE

5. XInclude

6. Upload-Based XXE

## Common Targets

SOAP

Legacy APIs

SVG Uploads

Office Documents

XML Integrations

## High Value Files

/etc/passwd

/etc/hostname

.env

web.config

Application configuration files

## Relationship to SSRF

XXE frequently becomes SSRF because external entities can point to URLs instead of local files.