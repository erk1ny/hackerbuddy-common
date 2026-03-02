# browser — Headless Chromium for DOM Analysis

## What it does
Headless Chromium renders JavaScript-heavy pages and exposes the full DOM for analysis. Use it when you need to inspect client-side rendered content, iframe structures, postMessage handlers, or dynamically loaded resources that aren't visible in raw HTTP responses.

## When to use
- Target page uses JavaScript frameworks (React, Angular, Vue) — raw HTTP response is just a shell
- Need to inspect iframe nesting, cross-origin communication, or window.name values
- Need to capture network requests made by JavaScript (XHR, fetch, WebSocket)
- Need to execute JavaScript in the page context to test DOM-based vulnerabilities

## When NOT to use
- Simple header checks — use `execute_request` instead (faster, no browser overhead)
- Content discovery / fuzzing — use ffuf instead
- Static HTML analysis — use `execute_request` and parse the response

## Usage patterns
The agent does not call Chromium directly. Instead, it uses `execute_request` for most HTTP work and reasons about when browser rendering is needed. When browser analysis is required:

1. Note that browser rendering is needed in your findings
2. Use `execute_request` to fetch the page first and analyze the raw HTML
3. Identify JavaScript files and inline scripts that modify the DOM
4. Reason about what the rendered DOM would look like based on the scripts

## DOM analysis without a browser
For many checks, you can analyze JavaScript behavior without rendering:
1. Fetch the HTML with `execute_request`
2. Identify `<script>` tags and external JS files
3. Fetch and read the JS files
4. Look for: postMessage handlers, iframe creation, DOM manipulation patterns
5. Document findings based on static analysis of the JavaScript code

## Limitations
- Heavy resource usage — only use when static analysis is insufficient
- May trigger client-side WAF or bot detection
- Cannot interact with CAPTCHAs or complex authentication flows
