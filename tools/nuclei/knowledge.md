# nuclei — Template-Based Vulnerability Scanner

## What it does
nuclei scans targets using YAML-based templates that define detection logic for vulnerabilities, misconfigurations, and exposures. It ships with thousands of community-maintained templates.

## Core flags

| Flag | Purpose |
|------|---------|
| `-u URL` | Single target URL |
| `-l FILE` | File with list of target URLs |
| `-t TEMPLATE` | Path to specific template or directory |
| `-tags TAG` | Run templates with specific tags (e.g., `cve,xss,sqli`) |
| `-severity LEVEL` | Filter by severity: info, low, medium, high, critical |
| `-exclude-tags TAG` | Exclude templates with these tags |
| `-j` | JSON output |
| `-o FILE` | Output file |
| `-rl RATE` | Rate limit (requests/second) |
| `-c CONCURRENCY` | Number of concurrent templates (default: 25) |
| `-timeout SECS` | HTTP timeout per request |
| `-retries N` | Number of retries for failed requests |
| `-H "Header: Value"` | Custom header |
| `-stats` | Show scan statistics |
| `-silent` | Only show findings |

## Calibration patterns

### Template selection
- Don't run all templates blindly — start with relevant tags
- For web apps: `-tags cve,xss,sqli,ssrf,lfi,rce,exposure,misconfig`
- For APIs: `-tags api,graphql,swagger,exposure`
- For infrastructure: `-tags network,dns,ssl,misconfig`
- Check template count before running: `nuclei -tl -tags <tags>` to see how many will run

### Rate limiting
- Default rate is aggressive; reduce for production targets: `-rl 50`
- If getting blocked, reduce further and add delays: `-rl 10`
- Watch for connection errors in output — sign of rate limiting

### False positive handling
- nuclei templates vary in quality — some have high false positive rates
- For critical/high findings: always verify manually with `execute_request`
- Check the template source to understand what it actually detects
- Info-severity findings are often just fingerprinting, not vulnerabilities

### Scan strategy
1. Start with high/critical severity only: `-severity high,critical`
2. Read results, verify interesting findings with `execute_request`
3. If target is well-hardened, expand to medium severity
4. Use specific tags rather than broad scans for focused testing

## Output format
With `-j`, each line is a JSON object with:
- `template-id` — template name
- `info.name` — human-readable finding name
- `info.severity` — severity level
- `info.tags` — template tags
- `matched-at` — URL where finding was detected
- `matcher-name` — which matcher triggered
- `extracted-results` — any extracted data (versions, tokens, etc.)
