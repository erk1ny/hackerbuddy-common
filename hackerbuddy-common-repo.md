# hackerbuddy-common Repository — Full Specification

This document contains everything needed to create the `erk1ny/hackerbuddy-common` GitHub repository from scratch. It includes the repo structure, all file schemas, full file contents for every file, CI validation, and naming conventions.

## Create the repo

```bash
gh repo create erk1ny/hackerbuddy-common --public --license MIT \
  --description "Community tools and skills for Hackerbuddy"
```

---

## Full repo structure

```
hackerbuddy-common/
├── manifest.json
├── README.md
├── .github/
│   └── workflows/
│       └── validate.yml
├── tools/
│   ├── ffuf/
│   │   ├── tool.yaml
│   │   └── knowledge.md
│   ├── nuclei/
│   │   ├── tool.yaml
│   │   └── knowledge.md
│   └── browser/
│       ├── tool.yaml
│       └── knowledge.md
└── skills/
    ├── check-framing/
    │   ├── skill.yaml
    │   └── skill.md
    ├── detect-sdks/
    │   ├── skill.yaml
    │   └── skill.md
    └── content-discovery/
        ├── skill.yaml
        └── skill.md
```

---

## Naming conventions

- Directory names: **lowercase-kebab-case**
- The `name` field in `tool.yaml` / `skill.yaml` **MUST** match the directory name exactly
- Enforced by CI

---

## Version systems

Two independent version systems:

1. **`manifest.json` version** — the only thing the client updater checks. Bump on every push to main that changes content. Triggers a full tarball download on the client side.
2. **`version` field in each `skill.yaml`** — per-skill metadata for tracking individual changes. Not used by the updater. Useful for users to see what changed in a skill they've overridden.

**Version bumping convention:**
- Patch (0.1.0 -> 0.1.1): knowledge.md / skill.md wording updates, no schema changes
- Minor (0.1.0 -> 0.2.0): new tools or skills added
- Major (0.x -> 1.0.0): breaking changes to yaml schemas

No automated version bump — edit `manifest.json` manually before merging to main.

---

## How the client fetches this repo

The client updater uses these two URLs:

```
Manifest:  https://raw.githubusercontent.com/erk1ny/hackerbuddy-common/main/manifest.json
Tarball:   https://github.com/erk1ny/hackerbuddy-common/archive/refs/heads/main.tar.gz
```

The tarball extracts to a root directory named `hackerbuddy-common-main`. The updater looks for `tools/` and `skills/` inside that root.

**Update flow:**
1. Fetch remote `manifest.json`, compare version with local
2. If version changed OR community directories are empty: download full tarball
3. Atomic swap: backup existing `community/` dirs, move new content in, clean up backups
4. Never touches `user/` directories

**Client-side install location:**
```
~/.config/hackerbuddy/          # Linux (macOS: ~/Library/Application Support/hackerbuddy)
  tools/
    community/                  # Overwritten by updater
    user/                       # Never touched by updater
  skills/
    community/                  # Overwritten by updater
    user/                       # Never touched by updater
  manifest.json                 # Local version tracker
```

---

## File contents

### `manifest.json`

```json
{"version": "0.1.0"}
```

---

### `README.md`

```markdown
# hackerbuddy-common

Community tools and skills for [Hackerbuddy](https://github.com/erk1ny/hackerbuddy) — an AI-native terminal environment for web vulnerability research.

## What's in this repo

- **Tools** (`tools/`) — registered external CLI binaries + knowledge documents that teach agents how to use them effectively
- **Skills** (`skills/`) — prompt-based markdown instructions that encode vulnerability patterns and research procedures

Hackerbuddy downloads this repo automatically on launch and installs tools and skills to `~/.config/hackerbuddy/{tools,skills}/community/`.

## File formats

### Tools

Each tool is a directory under `tools/` containing:

- `tool.yaml` — tool metadata and configuration
- `knowledge.md` — natural language document teaching the agent how to use the tool

**tool.yaml schema:**

```yaml
name: ffuf                          # Must match directory name (lowercase-kebab-case)
description: Fast web fuzzer        # What the tool does
binary: ffuf                        # Command name (must be in PATH)
categories: [fuzzing, discovery]    # Tool categories
intensity: 5                        # 1-5 for work queue lane assignment
scope_extraction:                   # How to extract target hostname from args
  flag: "-u"                        # CLI flag containing target URL
  pattern: "https?://([^/:]+)"      # Regex to extract hostname
output_flags: "-o {output_file} -of json"  # Flags for structured output
default_timeout: 300                # Timeout in seconds
install: |                          # Install instructions (shown when binary missing)
  brew install ffuf
```

Set `scope_extraction` to null/omit for tools that don't target URLs (local analysis only).

### Skills

Each skill is a directory under `skills/` containing:

- `skill.yaml` — skill metadata and configuration
- `skill.md` — markdown instructions that agents follow

**Two types:**
- **Atomic** — single focused checks. Cannot reference other skills.
- **Full** — multi-phase procedures that compose atomic skills via `@skill-name` references.

**skill.yaml schema:**

```yaml
name: check-framing                 # Must match directory name (lowercase-kebab-case)
version: "1.0.0"                    # Per-skill version tracking
description: Check for clickjacking # What the skill checks
type: atomic                        # "atomic" or "full"
tags: [clickjacking, headers]       # For filtering/discovery
tools: []                           # Required tool names (cross-validated)
mode: discover                      # Primary agent mode
depends_on: []                      # Atomic: must be empty. Full: list of atomic skill names
intensity: 3                        # 1-5 for work queue lane assignment
```

**Reference syntax in skill.md:**
- `@skill-name` — references another atomic skill (full skills only). Must match `depends_on`.
- `@tool:name` — references a tool's knowledge. Both types can use this.

## How to add a new tool

1. Create `tools/<name>/` directory (lowercase-kebab-case)
2. Add `tool.yaml` with all required fields
3. Add `knowledge.md` with usage patterns and decision logic
4. Bump `manifest.json` version
5. Open a PR

## How to add a new skill

1. Create `skills/<name>/` directory (lowercase-kebab-case)
2. Add `skill.yaml` with all required fields
3. Add `skill.md` with detection/research procedures
4. If full type: add `depends_on` entries and use `@skill-name` references in skill.md
5. Bump `manifest.json` version
6. Open a PR

## Testing locally

1. Copy your tool/skill directory to `~/.config/hackerbuddy/{tools,skills}/user/`
2. Run `hackerbuddy chat`
3. Startup validation catches structural errors (bad yaml, missing deps, type mismatches)
4. Use `list_skills` / `list_tools` MCP tools to confirm visibility
5. Run a test workflow to verify instructions make sense to agents
6. Move from `user/` to the community repo once validated
```

---

### `.github/workflows/validate.yml`

```yaml
name: Validate

on:
  push:
    branches: [main]
  pull_request:

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install PyYAML
        run: pip install pyyaml

      - name: Validate manifest.json
        run: |
          python3 -c "
          import json, sys
          m = json.load(open('manifest.json'))
          assert 'version' in m, 'missing version'
          parts = m['version'].split('.')
          assert len(parts) == 3, f'not semver: {m[\"version\"]}'
          print(f'Version: {m[\"version\"]}')
          "

      - name: Validate tool definitions
        run: |
          python3 -c "
          import yaml, pathlib, sys
          errors = []
          required = {'name', 'description', 'binary', 'categories', 'intensity', 'install'}
          tools = set()
          for d in sorted(pathlib.Path('tools').iterdir()):
              if not d.is_dir(): continue
              tools.add(d.name)
              ty = d / 'tool.yaml'
              km = d / 'knowledge.md'
              if not ty.exists():
                  errors.append(f'{d.name}: missing tool.yaml')
                  continue
              if not km.exists():
                  errors.append(f'{d.name}: missing knowledge.md')
              data = yaml.safe_load(ty.read_text())
              missing = required - set(data.keys())
              if missing:
                  errors.append(f'{d.name}: missing fields {missing}')
              if data.get('name') != d.name:
                  errors.append(f'{d.name}: name \"{data.get(\"name\")}\" != dir name')
              intensity = data.get('intensity', 3)
              if not (1 <= intensity <= 5):
                  errors.append(f'{d.name}: intensity {intensity} not in 1-5')
          if errors:
              print('\n'.join(errors), file=sys.stderr)
              sys.exit(1)
          print(f'All tools valid ({len(tools)} total)')
          "

      - name: Validate skill definitions
        run: |
          python3 -c "
          import yaml, pathlib, re, sys
          errors = []
          required = {'name', 'description', 'type', 'tags', 'mode'}

          # Collect tool names for cross-repo validation
          tool_names = set()
          for d in pathlib.Path('tools').iterdir():
              if d.is_dir(): tool_names.add(d.name)

          skills = {}
          # First pass: load all skills
          for d in sorted(pathlib.Path('skills').iterdir()):
              if not d.is_dir(): continue
              sy = d / 'skill.yaml'
              sm = d / 'skill.md'
              if not sy.exists():
                  errors.append(f'{d.name}: missing skill.yaml')
                  continue
              if not sm.exists():
                  errors.append(f'{d.name}: missing skill.md')
              data = yaml.safe_load(sy.read_text())
              skills[d.name] = data
              missing = required - set(data.keys())
              if missing:
                  errors.append(f'{d.name}: missing fields {missing}')
              if data.get('name') != d.name:
                  errors.append(f'{d.name}: name \"{data.get(\"name\")}\" != dir name')
              if data.get('type') not in ('atomic', 'full'):
                  errors.append(f'{d.name}: type must be atomic or full')
              intensity = data.get('intensity', 3)
              if not (1 <= intensity <= 5):
                  errors.append(f'{d.name}: intensity {intensity} not in 1-5')
              # Cross-repo: tools referenced in skill.yaml must exist in tools/
              for tool_ref in data.get('tools', []):
                  if tool_ref not in tool_names:
                      errors.append(f'{d.name}: references tool \"{tool_ref}\" but tools/{tool_ref}/ does not exist')

          # Second pass: cross-validate dependencies and @references
          for name, data in skills.items():
              stype = data.get('type')
              deps = data.get('depends_on', [])
              if stype == 'atomic' and deps:
                  errors.append(f'{name}: atomic skill cannot have depends_on')
              if stype == 'full':
                  for dep in deps:
                      if dep not in skills:
                          errors.append(f'{name}: depends_on \"{dep}\" not found')
                      elif skills[dep].get('type') != 'atomic':
                          errors.append(f'{name}: depends_on \"{dep}\" is not atomic')
              # Check @skill-name refs in skill.md match depends_on
              sm = pathlib.Path('skills') / name / 'skill.md'
              if sm.exists():
                  content = sm.read_text()
                  # Strip fenced code blocks to avoid false positives from @mentions in code/URLs
                  stripped = re.sub(r'\x60\x60\x60.*?\x60\x60\x60', '', content, flags=re.DOTALL)
                  # Extract @tool:name refs (not skill refs)
                  tool_md_refs = set(re.findall(r'(?<!\w)@tool:([\w-]+)', stripped))
                  for tr in tool_md_refs:
                      if tr not in tool_names:
                          errors.append(f'{name}: @tool:{tr} in skill.md but tools/{tr}/ does not exist')
                  # Extract @skill-name refs: must be preceded by whitespace/SOL, not inside @tool:
                  all_at_refs = set(re.findall(r'(?:^|(?<=\s))@([\w-]+)', stripped, re.MULTILINE))
                  all_at_refs -= {'tool'}  # @tool:name prefix word
                  skill_refs = all_at_refs - tool_md_refs
                  for ref in skill_refs:
                      if ref in skills and ref not in deps:
                          errors.append(f'{name}: @{ref} in skill.md but not in depends_on')

          if errors:
              print('\n'.join(errors), file=sys.stderr)
              sys.exit(1)
          print(f'All skills valid ({len(skills)} total)')
          "
```

---

### `tools/ffuf/tool.yaml`

```yaml
name: ffuf
description: Fast web fuzzer for content and parameter discovery
binary: ffuf
categories: [fuzzing, discovery]
intensity: 5
scope_extraction:
  flag: "-u"
  pattern: "https?://([^/:]+)"
output_flags: "-o {output_file} -of json"
default_timeout: 300
install: |
  ffuf is a Go-based web fuzzer. Install from https://github.com/ffuf/ffuf
  - macOS: brew install ffuf
  - Linux: apt install ffuf or download from GitHub releases
  - Go: go install github.com/ffuf/ffuf/v2@latest
```

---

### `tools/ffuf/knowledge.md`

```markdown
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
```

---

### `tools/nuclei/tool.yaml`

```yaml
name: nuclei
description: Fast vulnerability scanner using community-maintained templates
binary: nuclei
categories: [scanning, vulnerability]
intensity: 4
scope_extraction:
  flag: "-u"
  pattern: "https?://([^/:]+)"
output_flags: "-j -o {output_file}"
default_timeout: 600
install: |
  nuclei is a Go-based vulnerability scanner. Install from https://github.com/projectdiscovery/nuclei
  - macOS: brew install nuclei
  - Linux: go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
  - Or download from GitHub releases
  After install, run: nuclei -update-templates
```

---

### `tools/nuclei/knowledge.md`

```markdown
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
```

---

### `tools/browser/tool.yaml`

```yaml
name: browser
description: Headless browser for JavaScript-rendered page analysis
binary: chromium
categories: [browser, rendering, dom]
intensity: 3
install: |
  Requires a Chromium-based browser.
  - Linux: apt install chromium-browser
  - macOS: brew install --cask chromium
  - Or use Google Chrome: the binary is usually at /usr/bin/google-chrome
```

Note: no `scope_extraction` — browser tool is used for local DOM analysis, not direct URL fuzzing. The agent controls what URLs to visit through its own logic.

---

### `tools/browser/knowledge.md`

```markdown
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
```

---

### `skills/check-framing/skill.yaml`

```yaml
name: check-framing
version: "1.0.0"
description: Check if a target URL allows being embedded in an iframe (clickjacking defense)
type: atomic
tags: [clickjacking, headers, defense, framing]
tools: []
mode: discover
depends_on: []
intensity: 1
```

---

### `skills/check-framing/skill.md`

```markdown
# Check Framing

## Objective
Determine if a target URL allows being embedded in an iframe.

## Procedure
1. Use `execute_request` to fetch the target URL with a standard GET
2. Check response headers:
   - `X-Frame-Options`: DENY (blocked), SAMEORIGIN (same-origin only),
     ALLOW-FROM (deprecated -- modern browsers ignore this)
   - `Content-Security-Policy`: look for `frame-ancestors` directive
3. If no framing headers present, the page can be framed by anyone
4. Document: framing status, specific header values, any CSP details

## Expected outcomes
- **found**: Page can be framed (no X-Frame-Options, no frame-ancestors, or misconfigured)
- **not_found**: Page properly blocks framing (DENY, SAMEORIGIN, or strict frame-ancestors)
- **partial**: Mixed signals (e.g., X-Frame-Options set but CSP missing frame-ancestors, or deprecated ALLOW-FROM used)

## Output
- Framing allowed: yes / no / partial (same-origin only)
- Headers found and their values
- Notes on deprecated or misconfigured headers
```

---

### `skills/detect-sdks/skill.yaml`

```yaml
name: detect-sdks
version: "1.0.0"
description: Identify JavaScript SDKs and third-party libraries loaded on a page
type: atomic
tags: [recon, javascript, sdk, fingerprinting]
tools: []
mode: discover
depends_on: []
intensity: 2
```

---

### `skills/detect-sdks/skill.md`

```markdown
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
```

---

### `skills/content-discovery/skill.yaml`

```yaml
name: content-discovery
version: "1.0.0"
description: Discover hidden paths, endpoints, and files on a web target
type: atomic
tags: [recon, discovery, enumeration]
tools: [ffuf]
mode: discover
depends_on: []
intensity: 4
```

---

### `skills/content-discovery/skill.md`

```markdown
# Content Discovery

## Objective
Discover hidden paths, endpoints, API routes, backup files, and configuration files on the target.

## Procedure

### Phase 1: Baseline
1. Use `execute_request` to fetch the target root URL
2. Note the default 404 response: status code, body size, headers
3. Check `robots.txt` and `sitemap.xml` for disclosed paths
4. Check common paths manually: `/.well-known/`, `/api/`, `/admin/`, `/swagger.json`, `/openapi.json`

### Phase 2: Directory fuzzing with @tool:ffuf
1. Start with a common wordlist for general content discovery
2. Use `start_tool` to run ffuf against the target
3. Read the first chunk of output with `read_tool_output` (200 lines / 30s)
4. Calibrate based on initial results:
   - If many responses share the same size: add `-fs <size>` to filter default pages
   - If getting 403s: check for WAF (see ffuf knowledge doc), add browser headers
   - If no results: try different wordlists or extensions
5. If calibration is bad, `stop_tool` and restart with adjusted parameters
6. Once calibrated, read remaining output to completion

### Phase 3: Extension fuzzing
For interesting directories found in Phase 2:
1. Fuzz with common extensions: `.php`, `.html`, `.js`, `.json`, `.xml`, `.txt`, `.bak`, `.old`, `.conf`
2. Try backup patterns: `file.php.bak`, `file.php~`, `file.php.swp`, `.file.php`
3. Check for version control artifacts: `/.git/HEAD`, `/.svn/entries`, `/.hg/`

### Phase 4: API endpoint discovery
If API endpoints were found:
1. Fuzz API paths with API-specific wordlists
2. Try common API versioning: `/api/v1/`, `/api/v2/`, `/v1/`, `/v2/`
3. Check for documentation endpoints: `/swagger-ui/`, `/docs`, `/redoc`
4. Test HTTP methods on discovered endpoints (GET, POST, PUT, DELETE, PATCH, OPTIONS)

## Expected outcomes
- **found**: Hidden paths, backup files, API endpoints, or configuration files discovered
- **not_found**: No additional content beyond what's publicly linked
- **partial**: Some paths found but fuzzing was limited (WAF, rate limiting, incomplete wordlist)

## Output
- List of discovered paths with status codes, sizes, and descriptions
- API endpoints with supported methods
- Backup/config files found
- Any access control observations (paths returning 401/403 worth further investigation)
```

---

## Schema reference (for CI and client-side validation)

### tool.yaml — required fields (CI enforced)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Must match directory name exactly (lowercase-kebab-case) |
| `description` | string | Yes | What the tool does |
| `binary` | string | Yes | Command name that must be in PATH |
| `categories` | list[string] | Yes | Tool categories |
| `intensity` | int | Yes | 1-5 for work queue lane assignment |
| `install` | string | Yes | Natural language install instructions |

### tool.yaml — optional fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `scope_extraction` | object | null | `{flag, pattern}` — how to extract target hostname from CLI args |
| `output_flags` | string | "" | Flags to force structured output format |
| `default_timeout` | int | 300 | Timeout in seconds |

### skill.yaml — required fields (CI enforced)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Must match directory name exactly (lowercase-kebab-case) |
| `description` | string | Yes | What the skill checks |
| `type` | string | Yes | Must be "atomic" or "full" |
| `tags` | list[string] | Yes | For filtering/discovery |
| `mode` | string | Yes | Primary agent mode (discover, research, exploit, verify, report) |

### skill.yaml — optional fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `version` | string | "0.0.0" | Per-skill version tracking |
| `tools` | list[string] | [] | Required tool names (cross-validated against tools/) |
| `depends_on` | list[string] | [] | Atomic skill references (must be empty for atomic type) |
| `intensity` | int | 3 | 1-5 for work queue lane assignment |
| `inputs` | list[object] | [] | Input parameters (each: name, type, required, default, description) |

### Validation rules

**Tools:**
- Directory must contain `tool.yaml` (error) and `knowledge.md` (warning if missing)
- `name` must match directory name
- `intensity` must be 1-5

**Skills:**
- Directory must contain `skill.yaml` (error) and `skill.md` (warning if missing)
- `name` must match directory name
- `type` must be "atomic" or "full"
- `intensity` must be 1-5
- Atomic skills: `depends_on` must be empty, no `@skill-name` references in skill.md
- Full skills: `depends_on` targets must exist and must be atomic type
- `@skill-name` references in skill.md must appear in `depends_on` (and vice versa)
- `@tool:name` references in skill.md must have corresponding `tools/<name>/` directory
- `tools` list entries in skill.yaml must have corresponding `tools/<name>/` directory
- Code blocks (fenced with ```) are stripped before scanning for `@` references
