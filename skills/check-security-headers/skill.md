# Check Security Headers

## Objective
Audit the target's HTTP response headers for missing or misconfigured security defenses that could enable attacks like XSS, clickjacking, MIME sniffing, or information disclosure.

## Procedure

### Step 1: Fetch the target
Use `execute_request` with a GET to the target URL. Also make a second request to a known-HTML page (like the root `/`) if the target URL serves non-HTML content, since security headers often vary by content type.

### Step 2: Check each security header

**Strict-Transport-Security (HSTS):**
- Present? Check `max-age` (should be >= 31536000 / 1 year)
- Has `includeSubDomains`? (recommended)
- Has `preload`? (strongest, requires HSTS preload list submission)
- Missing entirely on HTTPS site? Browsers allow HTTP downgrade

**Content-Security-Policy (CSP):**
- Present? Check for unsafe directives:
  - `unsafe-inline` in `script-src` — XSS not mitigated by CSP
  - `unsafe-eval` in `script-src` — allows `eval()` based XSS
  - `*` or overly broad wildcards in `script-src` — CDN-based bypass potential
  - Missing `frame-ancestors` — clickjacking not covered by CSP
  - Missing `object-src` — Flash/plugin based attacks
- `Content-Security-Policy-Report-Only` alone (no enforcing CSP) — monitoring only, no protection

**X-Content-Type-Options:**
- Should be `nosniff` — prevents MIME sniffing attacks
- Missing — browser may interpret uploaded files as executable content

**Referrer-Policy:**
- `no-referrer` or `strict-origin-when-cross-origin` — good
- `unsafe-url` — leaks full URL (including query params) in Referer header
- Missing — browser default (varies, often leaks origin)

**Permissions-Policy (formerly Feature-Policy):**
- Controls access to browser features: camera, microphone, geolocation, payment
- Missing — all features available by default to embedded content

**X-Frame-Options:**
- `DENY` or `SAMEORIGIN` — clickjacking defense
- `ALLOW-FROM` — deprecated, modern browsers ignore it
- Missing — check CSP `frame-ancestors` as alternative

**Cache-Control / Pragma:**
- Sensitive pages (login, account, admin) should have: `no-store, no-cache, must-revalidate`
- Missing on sensitive pages — responses may be cached by proxies or browsers

**Server / X-Powered-By:**
- Present with version info — information disclosure (e.g., `Apache/2.4.41`, `PHP/7.4.3`)
- Helps attackers identify exact software versions for known CVE exploitation

### Step 3: Test header consistency
Fetch 2-3 different page types (HTML page, API endpoint, static asset) to check if security headers are applied consistently or only on certain routes.

## Expected outcomes
- **found**: One or more security headers are missing or misconfigured in a way that enables specific attacks
- **not_found**: All critical security headers are present and properly configured
- **partial**: Some headers present but others missing, or headers applied inconsistently across routes

## Output
For each header checked, report:
- Header name
- Value found (or "missing")
- Security implication of the current state
- Specific attack enabled by the weakness (if any)
