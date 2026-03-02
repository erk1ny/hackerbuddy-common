# Detect SDKs

## Objective
Identify JavaScript SDKs and third-party libraries loaded on a target page.

## Procedure
1. Use `execute_request` to fetch the target URL
2. Parse the HTML response for:
   - `<script src="...">` tags (external JavaScript files)
   - Inline `<script>` blocks that initialize SDKs
   - `<link>` tags that load third-party stylesheets (often bundled with SDKs)
3. For each external script URL:
   - Identify the SDK by URL pattern (e.g., `sdk.js`, `analytics.js`, `gtm.js`)
   - Note the version if visible in the URL or filename
   - Check if the script is loaded from a CDN or self-hosted
4. Common SDKs to look for:
   - Analytics: Google Analytics (`gtag.js`, `analytics.js`), Segment, Mixpanel, Hotjar
   - Advertising: Google Ads, Facebook Pixel, Twitter Pixel
   - Chat/Support: Intercom, Drift, Zendesk, LiveChat
   - Social: Facebook SDK, Twitter widgets, LinkedIn Insight
   - Payments: Stripe.js, PayPal SDK, Square
   - A/B Testing: Optimizely, VWO, Google Optimize
   - Tag Managers: Google Tag Manager, Tealium, Adobe Launch
   - Error Tracking: Sentry, Bugsnag, Rollbar, LogRocket
5. Fetch key JavaScript files to inspect:
   - SDK initialization patterns (API keys, configuration objects)
   - iframe creation logic
   - postMessage usage
   - Cross-origin requests

## Expected outcomes
- **found**: SDKs identified with details (name, version, source, initialization patterns)
- **not_found**: No third-party SDKs detected (rare for modern web apps)

## Output
- List of identified SDKs with:
  - Name and version (if detectable)
  - Source URL
  - Loading method (async, defer, inline)
  - Notable behaviors (creates iframes, uses postMessage, sets cookies)
