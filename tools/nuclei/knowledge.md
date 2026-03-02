# nuclei — Template-Based Vulnerability Scanner

## What it does
nuclei scans targets using YAML-based templates that encode detection logic for CVEs, misconfigurations, exposures, and default credentials. Ships with 8000+ community templates.

## Core flags

| Flag | Purpose |
|------|---------|
| `-u URL` | Single target URL |
| `-l FILE` | File with list of target URLs |
| `-t TEMPLATE` | Path to specific template or template directory |
| `-tags TAG` | Run templates matching tags (comma-separated) |
| `-etags TAG` | Exclude templates matching tags |
| `-severity LEVEL` | Filter by: info, low, medium, high, critical (comma-separated) |
| `-j` | JSON Lines output (one JSON object per line) |
| `-o FILE` | Output file path |
| `-rl N` | Rate limit in requests/second |
| `-c N` | Number of concurrent templates (default: 25) |
| `-timeout N` | HTTP timeout per request in seconds |
| `-retries N` | Retry count for failed requests |
| `-H "Header: Value"` | Custom header (repeatable) |
| `-stats` | Show runtime statistics |
| `-silent` | Only output findings (no banner/progress) |
| `-nc` | No color output |
| `-tl` | List available templates (dry run, shows count) |

## Calibration loop (MANDATORY)

### Step 1: Scope the scan
Don't run all templates blindly. Pick relevant tags for the target:
- Web applications: `-tags cve,xss,sqli,ssrf,lfi,rce,exposure,misconfig`
- APIs: `-tags api,graphql,swagger,exposure`
- WordPress: `-tags wordpress`
- Generic recon: `-tags tech,fingerprint,favicon`
Check template count first: `nuclei -tl -tags <tags>` to see how many will run.

### Step 2: Start conservatively
Use `start_tool` with:
- `-severity high,critical` first (fewer templates, most impactful)
- `-rl 50` to avoid rate limiting
- `-silent -j` for clean parseable output

### Step 3: Read initial output
Use `read_tool_output` for the first batch. Look for:

**Connection errors / timeouts:** Target or WAF blocking. Reduce `-rl` and `-c`.

**Many info-severity results:** These are fingerprinting, not vulnerabilities. Useful for tech identification but don't count as findings. Filter with `-severity medium,high,critical` if too noisy.

**Interesting findings:** For any high/critical finding, immediately verify with `execute_request` — nuclei templates vary in false positive rates. Reproduce the exact request the template describes.

### Step 4: Expand if needed
After high/critical sweep is clean, expand to `-severity medium` or add more tag groups.

## False positive verification
Critical rule: **always verify nuclei findings manually.** For each reported vulnerability:
1. Read the finding's `matched-at` URL and `matcher-name`
2. Use `execute_request` to reproduce the detection condition
3. Check if the response actually contains the vulnerable pattern
4. Info-severity = fingerprinting (tech version, headers); not a vulnerability unless paired with a known CVE

## Output format
With `-j`, each line is a JSON object:
- `template-id` — template identifier (e.g., `cve-2021-44228-log4j-rce`)
- `info.name` — human-readable finding name
- `info.severity` — severity level
- `info.tags` — template tags array
- `info.description` — finding description
- `info.reference` — reference URLs (CVE links, advisories)
- `matched-at` — exact URL where finding was detected
- `matcher-name` — which detection matcher triggered
- `extracted-results` — any extracted data (versions, tokens, paths)
- `curl-command` — reproducible curl command
