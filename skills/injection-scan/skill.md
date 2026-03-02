# Injection Scan

## Objective
Perform comprehensive injection testing across the target's endpoints — discover hidden attack surface, then test each injection vector systematically.

## Procedure

### Phase 1: Endpoint discovery
First, expand the known attack surface by discovering hidden paths and API routes:
@content-discovery

Collect all endpoints from discovery results plus those already in captured traffic (`get_traffic`). Prioritize endpoints that accept user input (query parameters, POST bodies, URL path segments).

### Phase 2: Parameter inventory
For each discovered endpoint, catalog injectable parameters:
- Use `get_traffic` and `get_request_details` to identify parameters from existing traffic
- Use `execute_request` to probe endpoints discovered in Phase 1 that aren't in traffic yet
- Classify each parameter by likely backend interaction:
  - Database queries (search, filter, sort, ID lookups) — SQLi candidates
  - HTML output (search terms reflected in page, error messages) — XSS candidates
  - URL handling (redirects, callbacks, links) — open redirect candidates

### Phase 3: Reflected XSS testing
For each parameter that appears in HTML output, run:
@check-xss-reflected

Focus on parameters that reflect user input in the response body. Test all reflection contexts (HTML body, attribute, JavaScript, URL).

### Phase 4: SQL injection testing
For each parameter that likely interacts with a database, run:
@check-sqli-error

Focus on search parameters, filter parameters, and ID-based lookups. Start with error-based detection before attempting blind techniques.

### Phase 5: Open redirect testing
For each parameter that handles URLs or redirect logic, run:
@check-open-redirect

Include OAuth callback URLs, login/logout redirect parameters, and any parameter containing URL-like values.

### Phase 6: Cross-vector analysis
After testing all three injection types, analyze patterns:
- Are different parameters on the same endpoint vulnerable to different injection types?
- Does the application apply consistent input validation or is it per-parameter?
- Are API endpoints more permissive than HTML-serving endpoints?
- Do POST parameters have different filtering than GET parameters?

## Expected outcomes
- **found**: One or more injection vulnerabilities confirmed with evidence across the discovered attack surface
- **not_found**: All tested parameters properly validated — no injection vulnerabilities found
- **partial**: Some endpoints showed suspicious behavior but couldn't be confirmed. Specific parameters may need deeper testing (e.g., with sqlmap for blind SQLi).

## Output
- Complete parameter inventory with classification
- Per-vector test results (XSS, SQLi, redirect)
- Successful injections with exact payloads and request/response evidence
- Endpoints needing deeper investigation
- Coverage summary: how many parameters were tested vs total discovered
