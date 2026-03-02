# ffuf — Fast Web Fuzzer

## What it does
ffuf replaces the keyword FUZZ in URLs, headers, or POST data with wordlist entries and reports interesting responses. It supports multiple FUZZ keywords for clusterbomb/pitchfork attack modes.

## Core flags

| Flag | Purpose |
|------|---------|
| `-u URL` | Target URL with FUZZ keyword (e.g., `https://target/FUZZ`) |
| `-w WORDLIST` | Path to wordlist file. For multiple keywords: `-w list1:KEYWORD1 -w list2:KEYWORD2` |
| `-H "Header: Value"` | Custom header (repeatable) |
| `-X METHOD` | HTTP method (default: GET) |
| `-d DATA` | POST data body (use with `-X POST`) |
| `-b "Cookie=Value"` | Cookie data |
| `-t THREADS` | Concurrent threads (default: 40) |
| `-mc CODES` | Match HTTP status codes (default: 200,204,301,302,307,401,403,405,500) |
| `-fc CODES` | Filter (exclude) HTTP status codes |
| `-fs SIZE` | Filter by response size |
| `-fw WORDS` | Filter by word count |
| `-fl LINES` | Filter by line count |
| `-fr REGEX` | Filter by response body regex |
| `-recursion` | Enable recursive fuzzing |
| `-recursion-depth N` | Max recursion depth |
| `-e EXTENSIONS` | Extension list (e.g., `.php,.html,.js`) |
| `-o FILE -of json` | JSON output file |
| `-rate N` | Max requests per second |
| `-timeout N` | HTTP request timeout in seconds |
| `-ac` | Auto-calibrate filtering (experimental) |
| `-mode MODE` | Multi-wordlist mode: `clusterbomb` (all combos) or `pitchfork` (paired) |

## Calibration loop (MANDATORY)

Never let ffuf run to completion without checking initial output first.

### Step 1: Start conservatively
Use `start_tool` with reasonable defaults. For unknown targets, start with `-rate 50 -t 10`.

### Step 2: Read first output batch
Use `read_tool_output` to get the first 100-200 lines. Look for:

**WAF detection:** Many 403 responses at the start — target likely has a WAF blocking ffuf's User-Agent.
- Verify by using `execute_request` to fetch 2-3 of those 403 URLs with browser headers
- If they return non-403: add `-H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'`
- Also reduce threads: `-t 5`

**Uniform response sizes:** Many results share the same content length — default/error page noise.
- Use `execute_request` to inspect a few of these responses
- Add `-fs <size>` to filter. Check if `-fw` (word count) or `-fl` (line count) gives cleaner results
- For wildcard DNS: filter the wildcard response size

**Connection errors / timeouts:** Rate too aggressive.
- Stop and restart with lower `-rate` and fewer `-t` threads

**No results at all:** Wordlist mismatch or overly restrictive filters.
- Try a different wordlist or remove `-mc`/`-fc` filters
- Try `-mc all -fc 404` to see everything except 404s

### Step 3: Adjust or continue
If output looks clean, continue reading to completion. If not, `stop_tool` and restart with adjusted parameters.

## Common wordlists
- `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` — general content discovery
- `/usr/share/seclists/Discovery/Web-Content/common.txt` — quick common paths (~4600 entries)
- `/usr/share/seclists/Discovery/Web-Content/raft-large-files.txt` — file names
- `/usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt` — API endpoint discovery
- `/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt` — path traversal payloads
- `/usr/share/seclists/Fuzzing/SQLi/Generic-SQLi.txt` — SQL injection payloads
- `/usr/share/seclists/Fuzzing/XSS/XSS-BruteLogic.txt` — XSS payloads

## Usage patterns

### Directory discovery
```
ffuf -u https://target.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -mc all -fc 404
```

### Parameter fuzzing (query string)
```
ffuf -u "https://target.com/page?FUZZ=test" -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -fs <baseline-size>
```

### Parameter value fuzzing (POST)
```
ffuf -u https://target.com/login -X POST -d "username=admin&password=FUZZ" -w passwords.txt -fc 401
```

### Subdirectory + extension fuzzing
```
ffuf -u https://target.com/admin/FUZZ -w common.txt -e .php,.bak,.old,.txt,.conf
```

### Multi-keyword (e.g., user enumeration)
```
ffuf -u https://target.com/api/users/USERID -w ids.txt:USERID -mc all -fc 404
```

## Output format
With `-of json`, the output JSON has a `results` array. Each result:
- `input.FUZZ` — the wordlist entry
- `status` — HTTP status code
- `length` — response content length
- `words` — word count
- `lines` — line count
- `url` — full URL tested
- `redirectlocation` — redirect target (if 3xx)
- `duration` — request duration in ms
