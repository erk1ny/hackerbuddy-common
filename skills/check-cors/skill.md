# Check CORS

## Objective
Test whether the target's CORS (Cross-Origin Resource Sharing) configuration allows unauthorized cross-origin access to sensitive data or actions.

## Procedure

### Step 1: Baseline request
Use `execute_request` with a normal GET to the target URL. Note whether any `Access-Control-*` headers appear in the response without an `Origin` header in the request.

### Step 2: Test origin reflection
Send requests with different `Origin` header values and observe `Access-Control-Allow-Origin` (ACAO) in the response:

1. **Arbitrary origin:** `Origin: https://evil.com`
   - If ACAO reflects `https://evil.com` — attacker-controlled site can read responses
2. **Null origin:** `Origin: null`
   - If ACAO reflects `null` — sandboxed iframes and data: URLs can read responses
3. **Subdomain:** `Origin: https://subdomain.target.com`
   - If ACAO reflects — any subdomain XSS becomes a full account takeover vector
4. **Prefix/suffix match bypass:**
   - `Origin: https://target.com.evil.com` (suffix confusion)
   - `Origin: https://eviltarget.com` (prefix confusion)
   - If ACAO reflects either — regex-based validation is broken

### Step 3: Check credentials
For each reflected origin, check if `Access-Control-Allow-Credentials: true` is also present.
- ACAO reflects attacker origin + credentials: true = **critical** — attacker can steal authenticated data
- ACAO reflects attacker origin but no credentials header = lower severity — only public data accessible

### Step 4: Preflight behavior
Send an OPTIONS request with:
- `Origin: https://evil.com`
- `Access-Control-Request-Method: PUT`
- `Access-Control-Request-Headers: X-Custom-Header`

Check if `Access-Control-Allow-Methods` includes dangerous methods (PUT, DELETE, PATCH) and if `Access-Control-Allow-Headers` is overly permissive.

### Step 5: Test API endpoints specifically
If the target has API endpoints (found via traffic or content discovery), test CORS on those separately — API endpoints often have different CORS configs than the main site.

## Expected outcomes
- **found**: CORS reflects arbitrary/null origins with credentials, enabling cross-origin data theft
- **not_found**: CORS properly restricts origins or is not configured (no ACAO header)
- **partial**: CORS reflects some origins but without credentials, or only reflects specific subdomains
- **variant**: CORS is restrictive on the main domain but misconfigured on specific API endpoints

## Output
- Origin values tested and ACAO responses for each
- Whether credentials are allowed for reflected origins
- Specific endpoints with different CORS behavior
- Attack scenario: what an attacker could steal/do with the misconfiguration
