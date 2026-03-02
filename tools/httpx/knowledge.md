# httpx — HTTP Probing Toolkit

## What it does
httpx probes URLs and reports detailed HTTP response metadata: status codes, content types, titles, technologies, TLS info, CDN detection, and more. Much faster than manual `execute_request` for bulk probing.

## Core flags

| Flag | Purpose |
|------|---------|
| `-u URL` | Single target URL |
| `-l FILE` | File with list of URLs (one per line) |
| `-sc` | Show status code |
| `-cl` | Show content length |
| `-ct` | Show content type |
| `-title` | Show page title |
| `-server` | Show server header |
| `-td` | Show technology detection (Wappalyzer-based) |
| `-favicon` | Show favicon hash (useful for fingerprinting) |
| `-hash md5` | Show response body hash |
| `-location` | Show redirect location |
| `-method METHOD` | HTTP method (default: GET) |
| `-H "Header: Value"` | Custom header (repeatable) |
| `-fr` | Follow redirects |
| `-j` | JSON Lines output |
| `-o FILE` | Output file |
| `-rl N` | Rate limit in requests/second |
| `-t N` | Concurrent threads (default: 50) |
| `-timeout N` | HTTP timeout in seconds |
| `-mc CODES` | Match status codes |
| `-fc CODES` | Filter status codes |
| `-tls-grab` | Show TLS certificate info |
| `-cdn` | Detect CDN provider |
| `-web-server` | Detect web server software |
| `-ip` | Show resolved IP address |

## Calibration loop

httpx is fast and lightweight, but still follow the loop:

### Step 1: Start with metadata flags
For recon, combine multiple flags: `-sc -cl -title -td -server -favicon -j`
This gives a rich metadata profile per URL without additional requests.

### Step 2: Read initial output
Check for connection patterns. If many URLs timeout or error, the target may have rate limiting. Reduce `-rl` and `-t`.

### Step 3: Analyze technology results
The `-td` flag uses Wappalyzer signatures to detect:
- Web frameworks (React, Angular, Django, Rails, Spring)
- CMS platforms (WordPress, Drupal, Joomla)
- Server software (nginx, Apache, IIS, Cloudflare)
- JavaScript libraries (jQuery, Bootstrap, Lodash)
- Analytics (Google Analytics, Hotjar)
- CDN providers (Cloudflare, Akamai, Fastly)

This directly feeds into skill selection — knowing the tech stack determines which vulnerability patterns are relevant.

## Usage patterns

### Single URL full probe
```
httpx -u https://target.com -sc -cl -title -td -server -favicon -tls-grab -cdn -j
```

### Bulk alive checking
```
httpx -l urls.txt -sc -title -mc 200,301,302 -j -o alive.json
```

### Technology fingerprinting across subdomains
```
httpx -l subdomains.txt -td -server -sc -title -fr -j -o tech.json
```

## Output format
With `-j`, each line is a JSON object:
- `url` — final URL (after redirects if `-fr`)
- `status_code` — HTTP status
- `content_length` — response size
- `title` — page title
- `webserver` — server header value
- `technologies` — array of detected technologies
- `favicon_hash` — favicon mmh3 hash
- `tls` — TLS certificate details (if `-tls-grab`)
- `cdn` — CDN provider name (if `-cdn`)
- `a` — resolved IP addresses
