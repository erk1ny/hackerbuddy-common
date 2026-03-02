# ffuf — Fast Web Fuzzer

## What it does
ffuf is a fast web fuzzer written in Go. It replaces the keyword FUZZ in URLs, headers, or POST data with wordlist entries and reports interesting responses.

## Core flags

| Flag | Purpose |
|------|---------|
| `-u URL` | Target URL with FUZZ keyword (e.g., `https://target/FUZZ`) |
| `-w WORDLIST` | Path to wordlist file |
| `-H "Header: Value"` | Custom header (repeatable) |
| `-X METHOD` | HTTP method (default: GET) |
| `-d DATA` | POST data (use with `-X POST`) |
| `-t THREADS` | Number of concurrent threads (default: 40) |
| `-mc CODES` | Match HTTP status codes (default: 200,204,301,302,307,401,403,405,500) |
| `-fc CODES` | Filter (exclude) HTTP status codes |
| `-fs SIZE` | Filter by response size |
| `-fw WORDS` | Filter by word count |
| `-fl LINES` | Filter by line count |
| `-fr REGEX` | Filter by response body regex |
| `-recursion` | Enable recursive fuzzing |
| `-recursion-depth N` | Max recursion depth |
| `-e EXTENSIONS` | Extension list (e.g., `.php,.html,.js`) |
| `-o FILE -of json` | Output to JSON file |
| `-rate RATE` | Requests per second limit |
| `-timeout SECS` | HTTP request timeout |

## Calibration patterns

### WAF detection
If you see many 403 responses at the start, the target likely has a WAF blocking ffuf's default User-Agent. Test this:
1. Use `execute_request` to fetch 2-3 of those 403 URLs with browser-like headers
2. If they return non-403 responses, the WAF is blocking ffuf specifically
3. Add browser headers to ffuf: `-H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'`
4. Reduce thread count with `-t 10` to avoid rate limiting

### Content length filtering
If many results share the same response size:
1. Use `execute_request` to inspect a few of these responses
2. If they're all default/error pages, add `-fs <size>` to filter them out
3. Check if word count (`-fw`) or line count (`-fl`) gives cleaner filtering

### Directory detection
When fuzzing directories:
1. Compare 404 responses for `/existing-dir/nonexistent` vs `/nonexistent-dir/nonexistent`
2. Different response headers or sizes indicate the directory exists even with 404 status
3. Use `-fc 404` only if 404 responses are genuinely uniform

### Adaptive strategy
- Start with default filters, read first 200 lines of output
- If too much noise: identify the common response size/code and add appropriate filters
- If no results: try removing restrictive filters, check if the wordlist is appropriate
- For API fuzzing: use `-mc all -fc 404` to see everything except 404s
- For parameter fuzzing: use FUZZ in query string (`?param=FUZZ`) or POST body

## Common wordlists
- `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` — general content discovery
- `/usr/share/seclists/Discovery/Web-Content/common.txt` — quick common paths
- `/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt` — API endpoint discovery
- `/usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` — alternative location

## Output format
With `-of json`, output is a JSON object with a `results` array. Each result has:
- `input.FUZZ` — the wordlist entry
- `status` — HTTP status code
- `length` — response content length
- `words` — word count
- `lines` — line count
- `url` — full URL
- `redirectlocation` — redirect target (if 3xx)
