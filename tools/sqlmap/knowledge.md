# sqlmap — SQL Injection Automation

## What it does
sqlmap automates the detection and exploitation of SQL injection flaws. It supports all major SQL injection techniques (boolean-blind, time-blind, error-based, UNION, stacked queries, out-of-band) and all major database engines (MySQL, PostgreSQL, MSSQL, Oracle, SQLite).

## Core flags

| Flag | Purpose |
|------|---------|
| `-u URL` | Target URL with injectable parameter (e.g., `https://target/page?id=1`) |
| `-r FILE` | Load HTTP request from file (captures exact headers/cookies) |
| `-p PARAM` | Specific parameter to test (skip auto-detection) |
| `--data DATA` | POST body data |
| `--cookie "C=V"` | Session cookies |
| `-H "Header: Val"` | Custom header (repeatable) |
| `--level N` | Test thoroughness 1-5 (default: 1, higher = more payloads + more injection points like cookies/headers) |
| `--risk N` | Risk level 1-3 (default: 1, higher = more aggressive payloads including OR-based + UPDATE/INSERT) |
| `--technique TECH` | Injection techniques: B(oolean), E(rror), U(nion), S(tacked), T(ime), Q(uery) |
| `--dbms ENGINE` | Force database engine (skip fingerprinting) |
| `--batch` | Never ask for user input (use defaults) |
| `--threads N` | Concurrent threads |
| `--delay N` | Delay between requests in seconds |
| `--timeout N` | HTTP timeout in seconds |
| `--tamper SCRIPT` | Payload tampering scripts (WAF bypass: space2comment, charencode, between) |
| `--random-agent` | Use random User-Agent |
| `--dbs` | Enumerate databases |
| `--tables -D DB` | Enumerate tables in database |
| `--columns -T TBL -D DB` | Enumerate columns |
| `--dump -T TBL -D DB` | Dump table data |
| `--forms` | Parse and test HTML forms |
| `--crawl N` | Crawl target and test found forms/params |
| `-v N` | Verbosity level 0-6 |

## Calibration loop (MANDATORY)

### Step 1: Start targeted
Always use `--batch` (non-interactive mode is required since agents can't interact with prompts).
Start with specific parameter and low level: `-p <param> --level 1 --risk 1 --batch --technique BET`
Skip UNION and stacked queries initially (slower, noisier).

### Step 2: Read initial output
Look for:
- "parameter appears to be injectable" — genuine finding, continue
- "all tested parameters do not appear to be injectable" — try higher `--level`
- Connection errors or WAF blocks — add `--random-agent --delay 2 --tamper space2comment`
- "testing connection to the target URL" hangs — target is slow, increase `--timeout`

### Step 3: Escalate carefully
If level 1 finds nothing:
1. Try `--level 2` (tests cookie parameters)
2. Try `--level 3` (tests User-Agent and Referer headers)
3. Try `--risk 2` (adds OR-based payloads)
4. Only use `--risk 3` if authorized — it uses INSERT/UPDATE payloads

### Step 4: Verify independently
After sqlmap reports an injection:
1. Note the exact payload and technique from output
2. Use `execute_request` to manually reproduce the injection
3. Confirm the database response differs from baseline
4. This independent verification is critical — sqlmap has false positives at higher levels

## WAF bypass tamper scripts
- `space2comment` — replaces spaces with `/**/` (MySQL/PostgreSQL)
- `charencode` — URL-encodes payload characters
- `between` — replaces `>` with `NOT BETWEEN 0 AND` (bypasses comparison filters)
- `randomcase` — randomizes keyword case
- Combine: `--tamper "space2comment,charencode"`

## Output
sqlmap writes detailed logs to its output directory. Key files:
- `log` — full session log with payloads and responses
- `target.txt` — target URL and parameters
- `session.sqlite` — session database (resume capability)
- Dump files under `dump/<db>/<table>.csv`
