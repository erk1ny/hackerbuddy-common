# Detect SDKs

## Objective
Identify JavaScript SDKs and third-party libraries loaded on a target page, including their security-relevant behaviors (cross-origin communication, iframe injection, data collection).

## Procedure

### Step 1: Fetch and parse
Use `execute_request` to GET the target URL. Parse the HTML response for:
- `<script src="...">` tags — external JavaScript files
- Inline `<script>` blocks — SDK initialization code, config objects, API keys
- `<link>` and `<iframe>` tags — third-party stylesheets and embeds

### Step 2: Identify SDKs by URL pattern
For each external script URL, identify the SDK:

| Pattern | SDK |
|---------|-----|
| `gtag.js`, `analytics.js`, `ga.js` | Google Analytics |
| `gtm.js` | Google Tag Manager |
| `fbevents.js`, `fbsdk.js` | Facebook Pixel / SDK |
| `widget.js` (intercom) | Intercom |
| `js.stripe.com` | Stripe.js |
| `cdn.segment.com` | Segment |
| `cdn.hotjar.com` | Hotjar |
| `sentry.io`, `browser.sentry-cdn.com` | Sentry |
| `cdn.optimizely.com` | Optimizely |
| `js.hs-scripts.com` | HubSpot |
| `static.cloudflareinsights.com` | Cloudflare Web Analytics |

Note the version if visible in the URL, filename, or query parameter.

### Step 3: Analyze security-relevant behaviors
Fetch 2-3 key JavaScript files with `execute_request` and look for:
- **API keys / tokens** — hardcoded keys in SDK init calls (e.g., `Stripe('pk_live_...')`, `analytics.init('...')`)
- **postMessage handlers** — `window.addEventListener('message', ...)` patterns. Check if origin is validated.
- **iframe creation** — `document.createElement('iframe')` or `innerHTML` with `<iframe>`. Check cross-origin sources.
- **Cookie access** — `document.cookie` reads/writes. Check for sensitive data in SDK-managed cookies.
- **Cross-origin fetch/XHR** — requests to third-party domains. Note what data is sent.

### Step 4: Check for known vulnerable versions
If version numbers are identified, cross-reference with known vulnerabilities:
- jQuery < 3.5.0 — prototype pollution, XSS via `.html()`
- Angular.js < 1.6.9 — sandbox escape (CSP bypass via template injection)
- Lodash < 4.17.21 — prototype pollution
- moment.js — ReDoS patterns (if user input reaches `.format()`)

## Expected outcomes
- **found**: SDKs identified with details — names, versions, sources, security-relevant behaviors
- **not_found**: No third-party SDKs detected (server-rendered with no JavaScript dependencies)
- **variant**: SDKs found but loaded conditionally (A/B test, geo-gated, auth-gated)

## Output
For each identified SDK:
- Name and version (if detectable)
- Source URL (CDN vs self-hosted)
- Loading method (async, defer, inline, dynamic injection)
- Security observations: exposed API keys, postMessage without origin check, cookie access patterns
- Known CVEs for detected versions (if any)
