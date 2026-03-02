# Check Framing

## Objective
Determine if a target URL allows being embedded in an iframe.

## Procedure
1. Use `execute_request` to fetch the target URL with a standard GET
2. Check response headers:
   - `X-Frame-Options`: DENY (blocked), SAMEORIGIN (same-origin only),
     ALLOW-FROM (deprecated -- modern browsers ignore this)
   - `Content-Security-Policy`: look for `frame-ancestors` directive
3. If no framing headers present, the page can be framed by anyone
4. Document: framing status, specific header values, any CSP details

## Expected outcomes
- **found**: Page can be framed (no X-Frame-Options, no frame-ancestors, or misconfigured)
- **not_found**: Page properly blocks framing (DENY, SAMEORIGIN, or strict frame-ancestors)
- **partial**: Mixed signals (e.g., X-Frame-Options set but CSP missing frame-ancestors, or deprecated ALLOW-FROM used)

## Output
- Framing allowed: yes / no / partial (same-origin only)
- Headers found and their values
- Notes on deprecated or misconfigured headers
