# Check Framing

## Objective
Determine if a target URL allows being embedded in an iframe, enabling clickjacking attacks.

## Procedure

### Step 1: Fetch the target
Use `execute_request` to GET the target URL.

### Step 2: Check framing headers
Examine the response for two independent framing defenses:

**X-Frame-Options (XFO):**
- `DENY` — fully blocked from framing (strongest)
- `SAMEORIGIN` — only same-origin framing allowed
- `ALLOW-FROM uri` — deprecated, modern browsers (Chrome, Firefox) ignore this entirely
- Missing — no XFO protection

**Content-Security-Policy frame-ancestors:**
- `frame-ancestors 'none'` — equivalent to DENY
- `frame-ancestors 'self'` — equivalent to SAMEORIGIN
- `frame-ancestors https://trusted.com` — specific origins allowed
- Missing — no CSP framing protection

### Step 3: Check for inconsistency
- XFO present but no CSP `frame-ancestors` — partial protection (CSP is the modern standard)
- CSP `frame-ancestors` present but no XFO — good for modern browsers, older browsers unprotected
- Both present but conflicting — CSP takes precedence in modern browsers
- `ALLOW-FROM` used without CSP `frame-ancestors` — effectively unprotected in modern browsers

### Step 4: Test state-changing pages
If the target URL is a landing page, also check pages with state-changing actions (forms, buttons) — clickjacking is most dangerous on pages where a user click triggers an action (change password, delete account, transfer funds). Use `get_traffic` to find such pages.

## Expected outcomes
- **found**: Page can be framed (no XFO, no frame-ancestors, or only deprecated ALLOW-FROM)
- **not_found**: Page properly blocks framing via DENY/SAMEORIGIN + frame-ancestors
- **partial**: Inconsistent protection (e.g., XFO present but CSP frame-ancestors missing, or only protecting some pages)

## Output
- X-Frame-Options header value (or "missing")
- CSP frame-ancestors directive value (or "missing")
- Whether state-changing pages are specifically protected
- Clickjacking feasibility assessment
