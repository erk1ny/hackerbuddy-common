# Web App Baseline Security Assessment

## Objective
Perform a baseline security assessment of the target's defensive posture by auditing security headers, CORS configuration, and framing protections across the application.

## Procedure

### Phase 1: Identify key pages
Use `get_traffic` to find representative pages across the application:
- Landing/home page
- Login page (if exists)
- Authenticated page (if traffic shows logged-in requests)
- API endpoints
- Static asset pages
Collect 3-5 distinct URLs that represent different parts of the application.

### Phase 2: Security headers audit
For each collected URL, run:
@check-security-headers

Note: headers often differ between HTML pages, API responses, and static assets. Testing multiple URL types reveals inconsistencies.

### Phase 3: CORS configuration audit
For the target URL and any API endpoints found in traffic, run:
@check-cors

Pay special attention to API endpoints — they are the most common location for CORS misconfigurations because they need cross-origin access for legitimate SPAs.

### Phase 4: Framing protection audit
For the main application URL and any state-changing pages (forms, account pages), run:
@check-framing

### Phase 5: Cross-cutting analysis
After running all three checks, analyze patterns:
- Are security headers consistent across all pages or only on some routes?
- Do API endpoints have different (often weaker) security than HTML pages?
- Is the framing defense comprehensive or limited to specific pages?
- Are there CSP inconsistencies between pages (one has strict CSP, another has none)?

## Expected outcomes
- **found**: Significant security header gaps, CORS misconfigurations, or missing framing defenses that enable specific attacks
- **not_found**: Application has comprehensive security headers, strict CORS, and proper framing defenses across all routes
- **partial**: Some defenses are in place but inconsistently applied, or specific routes have weaker posture

## Output
- Per-page security header summary
- CORS configuration assessment
- Framing defense status
- Consistency analysis across different route types
- Prioritized list of missing defenses with attack scenarios each enables
