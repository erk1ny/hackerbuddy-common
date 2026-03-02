# Check Open Redirect

## Objective
Test whether the target accepts arbitrary URLs in redirect parameters, allowing phishing attacks that abuse the target's trusted domain.

## Procedure

### Step 1: Identify redirect parameters
Search captured traffic with `get_traffic` for redirect patterns:
- URL parameters: `?redirect=`, `?url=`, `?next=`, `?return=`, `?returnUrl=`, `?continue=`, `?dest=`, `?goto=`, `?target=`, `?rurl=`, `?callback=`
- Login/logout flows often redirect after auth: check post-login redirect behavior
- OAuth callbacks: `?redirect_uri=`

### Step 2: Test basic redirect
For each redirect parameter, use `execute_request` to inject an external URL and check the `Location` header in the response:

**Direct external URL:**
- `?redirect=https://evil.com`
- Check: does the response have a 3xx status with `Location: https://evil.com`?

**Protocol-relative URL:**
- `?redirect=//evil.com`
- Some filters check for `http://` or `https://` but miss `//` prefix

### Step 3: Filter bypass attempts
If direct URLs are blocked, try bypass techniques:

**URL encoding:**
- `?redirect=https%3A%2F%2Fevil.com`
- `?redirect=%2F%2Fevil.com`

**Backslash confusion:**
- `?redirect=https://evil.com\@target.com` — some parsers see target.com as the host
- `?redirect=\/\/evil.com` — backslash normalization differences

**Domain confusion:**
- `?redirect=https://target.com.evil.com` — subdomain of attacker domain
- `?redirect=https://target.com@evil.com` — userinfo component points to evil.com
- `?redirect=https://evil.com#target.com` — fragment confusion
- `?redirect=https://evil.com?.target.com` — query string confusion

**Path-based:**
- `?redirect=/\evil.com` — backslash treated as path separator
- `?redirect=////evil.com` — multiple slash normalization

**Data/JavaScript URI:**
- `?redirect=javascript:alert(1)` — if redirect is via client-side JS
- `?redirect=data:text/html,<script>alert(1)</script>`

### Step 4: Verify redirect chain
If the target returns a 3xx redirect, follow the chain with `execute_request` to confirm the final destination is the attacker-controlled URL. Some servers redirect through an intermediate page.

## Expected outcomes
- **found**: Target redirects to arbitrary external URL — include exact payload and Location header value
- **not_found**: Target properly validates redirect destinations (allowlist or same-origin check)
- **partial**: Redirect exists but is constrained (e.g., only allows same-domain paths, or adds a warning interstitial)
- **variant**: Redirect is via client-side JavaScript (meta refresh or `window.location`) rather than HTTP 3xx — still exploitable but different mechanism

## Output
- Redirect parameters found and their behavior
- Successful bypass payload (if found)
- Redirect type (server-side 3xx vs client-side JS)
- Request IDs for evidence
