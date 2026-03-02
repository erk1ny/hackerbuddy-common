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
│   ├── httpx/
│   │   ├── tool.yaml
│   │   └── knowledge.md
│   └── sqlmap/
│       ├── tool.yaml
│       └── knowledge.md
└── skills/
    ├── check-security-headers/
    │   ├── skill.yaml
    │   └── skill.md
    ├── check-cors/
    │   ├── skill.yaml
    │   └── skill.md
    ├── check-framing/
    │   ├── skill.yaml
    │   └── skill.md
    ├── detect-sdks/
    │   ├── skill.yaml
    │   └── skill.md
    ├── tech-fingerprint/
    │   ├── skill.yaml
    │   └── skill.md
    ├── content-discovery/
    │   ├── skill.yaml
    │   └── skill.md
    ├── check-xss-reflected/
    │   ├── skill.yaml
    │   └── skill.md
    ├── check-sqli-error/
    │   ├── skill.yaml
    │   └── skill.md
    ├── check-open-redirect/
    │   ├── skill.yaml
    │   └── skill.md
    ├── check-idor/
    │   ├── skill.yaml
    │   └── skill.md
    ├── check-ssrf/
    │   ├── skill.yaml
    │   └── skill.md
    ├── check-jwt/
    │   ├── skill.yaml
    │   └── skill.md
    ├── web-app-baseline/
    │   ├── skill.yaml
    │   └── skill.md
    └── injection-scan/
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

## How tools work at runtime

Agents use external CLI tools via an MCP tool lifecycle:

1. **`start_tool(tool_name, args)`** — starts the tool as a background subprocess. If the tool has `scope_extraction`, the agent's args are checked against the project's scope (hostname extracted via `flag` + `pattern` regex). Tools without `scope_extraction` reject any URL in args (local-only). Returns a `proc_id` for polling.
2. **`read_tool_output(proc_id, max_lines=200, timeout_sec=30)`** — reads new output lines incrementally (offset-tracked). Output is wrapped in `<tool-output>` tags as prompt injection defense. Returns `{lines, output, running}`.
3. **`stop_tool(proc_id)`** — sends SIGTERM, waits 5s, then SIGKILL. Cleans up tracking.

**Calibration loop pattern (mandatory for all tool usage):**
1. Start tool with initial arguments
2. Read first ~100-200 lines of output
3. Analyze: WAF detected? Uniform response sizes? Connection errors? Rate limiting?
4. If bad: `stop_tool`, adjust parameters (rate, headers, wordlist, filters), `start_tool` again
5. If good: continue reading to completion
6. **Never** let a tool run unmonitored without checking the first output batch

Tool output is persisted under `.hackerbuddy/cache/tool-runs/{proc_id}/` with `stdout.log`, `stderr.log`, `pid`, and `meta.json`.

---

## How skills work at runtime

Skills are prompt-based instructions that agents follow step by step:

1. **`list_skills(tags?, skill_type?)`** — lists all valid skills, filterable by tags and type
2. **`run_skill(skill_id, inputs?)`** — resolves all `@skill-name` and `@tool:name` references in the skill's markdown and returns fully expanded instructions. Inputs are prepended as a structured block.
3. **`report_skill_result(skill_id, host, outcome, summary, ...)`** — agents MUST call this once per skill per host after completing. Outcomes: `found`, `not_found`, `partial`, `variant`, `error`. Results are stored in `skill_results.jsonl` and tracked by the coverage matrix.

**Reference operators in skill.md:**
- `@tool:name` — replaced at resolution time with the tool's full `knowledge.md` content. The agent receives tool documentation inline exactly where the reference appears. Both atomic and full skills can use this.
- `@skill-name` — replaced at resolution time with the referenced atomic skill's full `skill.md` instructions. Only full skills can use this, and every reference must appear in `depends_on`.

**Skill inputs** allow parameterization:
```yaml
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL to test"
  - name: wordlist
    type: string
    required: false
    default: "/usr/share/seclists/Discovery/Web-Content/common.txt"
    description: "Path to wordlist"
```

When `run_skill` is called with inputs, they're prepended as an `## Inputs` section before the instructions.

**Intensity scores (1-5)** control work queue lane concurrency:
- 1-2 (light): many concurrent agents (e.g., 5)
- 3 (medium): moderate agents (e.g., 3)
- 4-5 (heavy): few agents (e.g., 1-2)

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

Agents interact with tools via a start/read/stop lifecycle. The `knowledge.md` content is injected into skill instructions wherever `@tool:name` appears, giving agents calibration patterns and decision logic inline.

**tool.yaml schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Must match directory name (lowercase-kebab-case) |
| `description` | string | Yes | What the tool does |
| `binary` | string | Yes | Command name (must be in PATH) |
| `categories` | list[string] | Yes | Tool categories |
| `intensity` | int (1-5) | Yes | Work queue lane weight |
| `install` | string | Yes | Install instructions (shown when binary missing) |
| `scope_extraction` | object | No | `{flag, pattern}` — extracts target hostname from CLI args |
| `output_flags` | string | No | Flags for structured output format |
| `default_timeout` | int | No | Timeout in seconds (default 300) |

### Skills

Each skill is a directory under `skills/` containing:

- `skill.yaml` — skill metadata and configuration
- `skill.md` — markdown instructions that agents follow step by step

**Two types:**
- **Atomic** — single focused checks. Cannot reference other skills.
- **Full** — multi-phase procedures that compose atomic skills via `@skill-name` references.

**skill.yaml schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Must match directory name (lowercase-kebab-case) |
| `description` | string | Yes | What the skill checks |
| `type` | string | Yes | `"atomic"` or `"full"` |
| `tags` | list[string] | Yes | For filtering/discovery |
| `mode` | string | Yes | Primary agent mode (discover, research, exploit, verify, report) |
| `version` | string | No | Per-skill version tracking (default "0.0.0") |
| `tools` | list[string] | No | Required tool names (cross-validated against tools/) |
| `depends_on` | list[string] | No | Atomic skill names (must be empty for atomic type) |
| `intensity` | int (1-5) | No | Work queue lane weight (default 3) |
| `inputs` | list[object] | No | Input parameters (each: name, type, required, default, description) |

**Reference syntax in skill.md:**
- `@tool:name` — replaced with the tool's `knowledge.md` content at resolution time
- `@skill-name` — replaced with the atomic skill's `skill.md` instructions (full skills only, must match `depends_on`)

**Mandatory reporting:** agents must call `report_skill_result` after every skill with one of: `found`, `not_found`, `partial`, `variant`, `error`.

## How to add a new tool

1. Create `tools/<name>/` directory (lowercase-kebab-case)
2. Add `tool.yaml` with all required fields
3. Add `knowledge.md` with: what it does, core flags, calibration patterns, output format
4. Bump `manifest.json` version
5. Open a PR

## How to add a new skill

1. Create `skills/<name>/` directory (lowercase-kebab-case)
2. Add `skill.yaml` with all required fields
3. Add `skill.md` with: objective, step-by-step procedure, expected outcomes, output format
4. Use `@tool:name` where the agent needs tool knowledge injected
5. If full type: add `depends_on` entries and use `@skill-name` references in skill.md
6. Bump `manifest.json` version
7. Open a PR

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

## Tools

### `tools/ffuf/tool.yaml`

```yaml
name: ffuf
description: Fast web fuzzer for content discovery, parameter enumeration, and endpoint mapping
binary: ffuf
categories: [fuzzing, discovery, enumeration]
intensity: 5
scope_extraction:
  flag: "-u"
  pattern: "https?://([^/:?#]+)"
output_flags: "-o {output_file} -of json"
default_timeout: 600
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

**WAF detection:** Many 403 responses at the start → target likely has a WAF blocking ffuf's User-Agent.
- Verify by using `execute_request` to fetch 2-3 of those 403 URLs with browser headers
- If they return non-403: add `-H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'`
- Also reduce threads: `-t 5`

**Uniform response sizes:** Many results share the same content length → default/error page noise.
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
```

---

### `tools/nuclei/tool.yaml`

```yaml
name: nuclei
description: Template-based vulnerability scanner with thousands of community detection signatures
binary: nuclei
categories: [scanning, vulnerability, detection]
intensity: 4
scope_extraction:
  flag: "-u"
  pattern: "https?://([^/:?#]+)"
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
```

---

### `tools/httpx/tool.yaml`

```yaml
name: httpx
description: HTTP probing toolkit for technology detection, status checking, and response analysis
binary: httpx
categories: [probing, fingerprinting, recon]
intensity: 2
scope_extraction:
  flag: "-u"
  pattern: "https?://([^/:?#]+)"
output_flags: "-j -o {output_file}"
default_timeout: 300
install: |
  httpx is a Go-based HTTP toolkit. Install from https://github.com/projectdiscovery/httpx
  - macOS: brew install httpx
  - Linux: go install github.com/projectdiscovery/httpx/cmd/httpx@latest
  - Or download from GitHub releases
```

---

### `tools/httpx/knowledge.md`

```markdown
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
```

---

### `tools/sqlmap/tool.yaml`

```yaml
name: sqlmap
description: Automated SQL injection detection and exploitation framework
binary: sqlmap
categories: [exploitation, injection, sqli]
intensity: 5
scope_extraction:
  flag: "-u"
  pattern: "https?://([^/:?#]+)"
output_flags: "--output-dir={output_file}"
default_timeout: 900
install: |
  sqlmap is a Python-based SQL injection tool. Install from https://github.com/sqlmapproject/sqlmap
  - macOS: brew install sqlmap
  - Linux: apt install sqlmap or pip install sqlmap
  - Or clone: git clone https://github.com/sqlmapproject/sqlmap.git
```

---

### `tools/sqlmap/knowledge.md`

```markdown
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
- "parameter appears to be injectable" → genuine finding, continue
- "all tested parameters do not appear to be injectable" → try higher `--level`
- Connection errors or WAF blocks → add `--random-agent --delay 2 --tamper space2comment`
- "testing connection to the target URL" hangs → target is slow, increase `--timeout`

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
```

---

## Atomic Skills

### `skills/check-security-headers/skill.yaml`

```yaml
name: check-security-headers
version: "1.0.0"
description: Audit HTTP security headers for missing or misconfigured defenses
type: atomic
tags: [headers, defense, hardening, owasp, misconfig]
tools: []
mode: discover
depends_on: []
intensity: 1
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL to check security headers on"
```

---

### `skills/check-security-headers/skill.md`

```markdown
# Check Security Headers

## Objective
Audit the target's HTTP response headers for missing or misconfigured security defenses that could enable attacks like XSS, clickjacking, MIME sniffing, or information disclosure.

## Procedure

### Step 1: Fetch the target
Use `execute_request` with a GET to the target URL. Also make a second request to a known-HTML page (like the root `/`) if the target URL serves non-HTML content, since security headers often vary by content type.

### Step 2: Check each security header

**Strict-Transport-Security (HSTS):**
- Present? → check `max-age` (should be >= 31536000 / 1 year)
- Has `includeSubDomains`? (recommended)
- Has `preload`? (strongest, requires HSTS preload list submission)
- Missing entirely on HTTPS site? → browsers allow HTTP downgrade

**Content-Security-Policy (CSP):**
- Present? → check for unsafe directives:
  - `unsafe-inline` in `script-src` → XSS not mitigated by CSP
  - `unsafe-eval` in `script-src` → allows `eval()` based XSS
  - `*` or overly broad wildcards in `script-src` → CDN-based bypass potential
  - Missing `frame-ancestors` → clickjacking not covered by CSP
  - Missing `object-src` → Flash/plugin based attacks
- `Content-Security-Policy-Report-Only` alone (no enforcing CSP) → monitoring only, no protection

**X-Content-Type-Options:**
- Should be `nosniff` — prevents MIME sniffing attacks
- Missing → browser may interpret uploaded files as executable content

**Referrer-Policy:**
- `no-referrer` or `strict-origin-when-cross-origin` → good
- `unsafe-url` → leaks full URL (including query params) in Referer header
- Missing → browser default (varies, often leaks origin)

**Permissions-Policy (formerly Feature-Policy):**
- Controls access to browser features: camera, microphone, geolocation, payment
- Missing → all features available by default to embedded content

**X-Frame-Options:**
- `DENY` or `SAMEORIGIN` → clickjacking defense
- `ALLOW-FROM` → deprecated, modern browsers ignore it
- Missing → check CSP `frame-ancestors` as alternative

**Cache-Control / Pragma:**
- Sensitive pages (login, account, admin) should have: `no-store, no-cache, must-revalidate`
- Missing on sensitive pages → responses may be cached by proxies or browsers

**Server / X-Powered-By:**
- Present with version info → information disclosure (e.g., `Apache/2.4.41`, `PHP/7.4.3`)
- Helps attackers identify exact software versions for known CVE exploitation

### Step 3: Test header consistency
Fetch 2-3 different page types (HTML page, API endpoint, static asset) to check if security headers are applied consistently or only on certain routes.

## Expected outcomes
- **found**: One or more security headers are missing or misconfigured in a way that enables specific attacks
- **not_found**: All critical security headers are present and properly configured
- **partial**: Some headers present but others missing, or headers applied inconsistently across routes

## Output
For each header checked, report:
- Header name
- Value found (or "missing")
- Security implication of the current state
- Specific attack enabled by the weakness (if any)
```

---

### `skills/check-cors/skill.yaml`

```yaml
name: check-cors
version: "1.0.0"
description: Test for CORS misconfiguration that could allow unauthorized cross-origin data access
type: atomic
tags: [cors, headers, cross-origin, owasp, misconfig]
tools: []
mode: discover
depends_on: []
intensity: 1
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL to test CORS configuration on"
```

---

### `skills/check-cors/skill.md`

```markdown
# Check CORS

## Objective
Test whether the target's CORS (Cross-Origin Resource Sharing) configuration allows unauthorized cross-origin access to sensitive data or actions.

## Procedure

### Step 1: Baseline request
Use `execute_request` with a normal GET to the target URL. Note whether any `Access-Control-*` headers appear in the response without an `Origin` header in the request.

### Step 2: Test origin reflection
Send requests with different `Origin` header values and observe `Access-Control-Allow-Origin` (ACAO) in the response:

1. **Arbitrary origin:** `Origin: https://evil.com`
   - If ACAO reflects `https://evil.com` → attacker-controlled site can read responses
2. **Null origin:** `Origin: null`
   - If ACAO reflects `null` → sandboxed iframes and data: URLs can read responses
3. **Subdomain:** `Origin: https://subdomain.target.com`
   - If ACAO reflects → any subdomain XSS becomes a full account takeover vector
4. **Prefix/suffix match bypass:**
   - `Origin: https://target.com.evil.com` (suffix confusion)
   - `Origin: https://eviltarget.com` (prefix confusion)
   - If ACAO reflects either → regex-based validation is broken

### Step 3: Check credentials
For each reflected origin, check if `Access-Control-Allow-Credentials: true` is also present.
- ACAO reflects attacker origin + credentials: true = **critical** — attacker can steal authenticated data
- ACAO reflects attacker origin but no credentials header = lower severity — only public data accessible

### Step 4: Preflight behavior
Send an OPTIONS request with:
- `Origin: https://evil.com`
- `Access-Control-Request-Method: PUT`
- `Access-Control-Request-Headers: X-Custom-Header`

Check if `Access-Control-Allow-Methods` includes dangerous methods (PUT, DELETE, PATCH) and if `Access-Control-Allow-Headers` is overly permissive.

### Step 5: Test API endpoints specifically
If the target has API endpoints (found via traffic or content discovery), test CORS on those separately — API endpoints often have different CORS configs than the main site.

## Expected outcomes
- **found**: CORS reflects arbitrary/null origins with credentials, enabling cross-origin data theft
- **not_found**: CORS properly restricts origins or is not configured (no ACAO header)
- **partial**: CORS reflects some origins but without credentials, or only reflects specific subdomains
- **variant**: CORS is restrictive on the main domain but misconfigured on specific API endpoints

## Output
- Origin values tested and ACAO responses for each
- Whether credentials are allowed for reflected origins
- Specific endpoints with different CORS behavior
- Attack scenario: what an attacker could steal/do with the misconfiguration
```

---

### `skills/check-framing/skill.yaml`

```yaml
name: check-framing
version: "1.0.0"
description: Test if a target allows being embedded in an iframe (clickjacking defense)
type: atomic
tags: [clickjacking, headers, defense, framing]
tools: []
mode: discover
depends_on: []
intensity: 1
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL to check for framing defenses"
```

---

### `skills/check-framing/skill.md`

```markdown
# Check Framing

## Objective
Determine if a target URL allows being embedded in an iframe, enabling clickjacking attacks.

## Procedure

### Step 1: Fetch the target
Use `execute_request` to GET the target URL.

### Step 2: Check framing headers
Examine the response for two independent framing defenses:

**X-Frame-Options (XFO):**
- `DENY` → fully blocked from framing (strongest)
- `SAMEORIGIN` → only same-origin framing allowed
- `ALLOW-FROM uri` → deprecated, modern browsers (Chrome, Firefox) ignore this entirely
- Missing → no XFO protection

**Content-Security-Policy frame-ancestors:**
- `frame-ancestors 'none'` → equivalent to DENY
- `frame-ancestors 'self'` → equivalent to SAMEORIGIN
- `frame-ancestors https://trusted.com` → specific origins allowed
- Missing → no CSP framing protection

### Step 3: Check for inconsistency
- XFO present but no CSP `frame-ancestors` → partial protection (CSP is the modern standard)
- CSP `frame-ancestors` present but no XFO → good for modern browsers, older browsers unprotected
- Both present but conflicting → CSP takes precedence in modern browsers
- `ALLOW-FROM` used without CSP `frame-ancestors` → effectively unprotected in modern browsers

### Step 4: Test state-changing pages
If the target URL is a landing page, also check pages with state-changing actions (forms, buttons) — clickjacking is most dangerous on pages where a user click triggers an action (change password, delete account, transfer funds). Use `get_traffic` to find such pages.

## Expected outcomes
- **found**: Page can be framed (no XFO, no frame-ancestors, or only deprecated ALLOW-FROM)
- **not_found**: Page properly blocks framing via DENY/SAMEORIGIN + frame-ancestors
- **partial**: Inconsistent protection (e.g., XFO present but CSP frame-ancestors missing, or only protecting some pages)

## Output
- X-Frame-Options header value (or "missing")
- CSP frame-ancestors directive value (or "missing")
- Whether state-changing pages are specifically protected
- Clickjacking feasibility assessment
```

---

### `skills/detect-sdks/skill.yaml`

```yaml
name: detect-sdks
version: "1.0.0"
description: Identify JavaScript SDKs, third-party libraries, and their security-relevant behaviors
type: atomic
tags: [recon, javascript, sdk, fingerprinting, supply-chain]
tools: []
mode: discover
depends_on: []
intensity: 2
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL to scan for JavaScript SDKs"
```

---

### `skills/detect-sdks/skill.md`

```markdown
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
```

---

### `skills/tech-fingerprint/skill.yaml`

```yaml
name: tech-fingerprint
version: "1.0.0"
description: Identify the target's full technology stack using HTTP probing and response analysis
type: atomic
tags: [recon, fingerprinting, technology, enumeration]
tools: [httpx]
mode: discover
depends_on: []
intensity: 2
inputs:
  - name: target_url
    type: string
    required: true
    description: "Base URL to fingerprint"
```

---

### `skills/tech-fingerprint/skill.md`

```markdown
# Tech Fingerprint

## Objective
Identify the target's full technology stack — web server, framework, language, CMS, CDN, WAF — to inform which vulnerability patterns are relevant.

## Procedure

### Step 1: HTTP probing
Use @tool:httpx to probe the target URL with metadata extraction flags. Use `start_tool` to run httpx against the target with `-sc -cl -title -td -server -favicon -tls-grab -cdn -j`, then read output with `read_tool_output`.

The `-td` flag performs Wappalyzer-based technology detection covering frameworks, CMS, libraries, analytics, CDN, and server software.

### Step 2: Manual header analysis
Use `execute_request` to fetch the target URL and inspect response headers for technology indicators not caught by automated detection:

- `Server` header — web server and version (e.g., `nginx/1.24.0`, `Apache/2.4.57`)
- `X-Powered-By` — language/framework (e.g., `PHP/8.2.0`, `Express`, `ASP.NET`)
- `X-AspNet-Version` / `X-AspNetMvc-Version` — ASP.NET version disclosure
- `X-Generator` — CMS name (e.g., `WordPress 6.4`, `Drupal 10`)
- `X-Drupal-Cache` / `X-Drupal-Dynamic-Cache` — Drupal indicators
- `X-WordPress-*` — WordPress indicators
- `Set-Cookie` names — `PHPSESSID` (PHP), `JSESSIONID` (Java), `ASP.NET_SessionId` (.NET), `connect.sid` (Express), `csrftoken` (Django)

### Step 3: Path-based fingerprinting
Use `execute_request` to probe technology-specific paths:
- `/wp-admin/`, `/wp-includes/` → WordPress
- `/admin/config/`, `/user/login` → Drupal
- `/administrator/` → Joomla
- `/__debug__/`, `/admin/` (Django-style) → Django
- `/elmah.axd`, `/trace.axd` → ASP.NET debugging
- `/actuator/`, `/actuator/health` → Spring Boot
- `/rails/info/routes` → Ruby on Rails (dev mode)
- `/server-info`, `/server-status` → Apache mod_status
- `/swagger.json`, `/openapi.json`, `/api-docs` → API documentation

### Step 4: Error page analysis
Request a URL guaranteed to 404 (e.g., `/nonexistent-path-12345`). The error page often reveals:
- Framework default error templates (Django, Rails, Spring, Express)
- Server software in error HTML
- Stack traces (if debug mode is enabled — this is an information disclosure finding)

### Step 5: JavaScript and meta analysis
From the HTML response:
- `<meta name="generator">` — CMS generator tag
- JavaScript framework globals: React (`__REACT_DEVTOOLS_GLOBAL_HOOK__`), Vue (`__VUE__`), Angular (`ng-version`)
- Build tool artifacts: webpack chunk names, Vite module scripts

## Expected outcomes
- **found**: Technology stack identified with specific versions and components
- **not_found**: Target successfully obscures its technology stack (rare)
- **partial**: Some technologies identified but versions are hidden

## Output
- Web server software and version
- Programming language/framework and version
- CMS or application platform (if applicable)
- CDN/WAF provider
- Notable version-specific observations (known CVEs, end-of-life software, debug mode enabled)
```

---

### `skills/content-discovery/skill.yaml`

```yaml
name: content-discovery
version: "1.0.0"
description: Discover hidden paths, endpoints, backup files, and API routes on a web target
type: atomic
tags: [recon, discovery, enumeration, fuzzing]
tools: [ffuf]
mode: discover
depends_on: []
intensity: 4
inputs:
  - name: target_url
    type: string
    required: true
    description: "Base URL to discover content on"
  - name: wordlist
    type: string
    required: false
    default: "/usr/share/seclists/Discovery/Web-Content/common.txt"
    description: "Path to wordlist for fuzzing"
```

---

### `skills/content-discovery/skill.md`

```markdown
# Content Discovery

## Objective
Discover hidden paths, endpoints, API routes, backup files, and configuration files on the target.

## Procedure

### Phase 1: Passive discovery
1. Use `execute_request` to fetch the target root URL. Note the response size, status, and headers.
2. Fetch `/robots.txt` — lists disallowed paths (directories the site owner wants hidden from crawlers)
3. Fetch `/sitemap.xml` — lists known pages (may reveal internal sections)
4. Check `/.well-known/` directory — standardized paths for security.txt, openid-configuration, etc.
5. Use `get_traffic` to see what paths are already captured from proxy traffic

### Phase 2: Baseline calibration
1. Fetch a URL guaranteed to not exist (e.g., `/this-path-does-not-exist-abc123`) to establish the 404 baseline: status code, response size, key response text
2. This baseline determines what filters to apply during fuzzing

### Phase 3: Directory fuzzing
Use @tool:ffuf for content discovery. The tool knowledge above covers calibration patterns, WAF detection, and adaptive strategy.

1. Use `start_tool` to run ffuf against the target with the wordlist
2. Read first output batch with `read_tool_output` (200 lines)
3. Apply calibration from Phase 2: if many results match the 404 baseline size, add `-fs <size>`
4. If calibrated poorly, `stop_tool` and restart with adjusted parameters
5. Once calibrated, read remaining output to completion

### Phase 4: Extension fuzzing
For each interesting directory found in Phase 3, use @tool:ffuf with extension lists:
- Code files: `.php`, `.asp`, `.aspx`, `.jsp`, `.py`, `.rb`
- Data files: `.json`, `.xml`, `.csv`, `.sql`, `.yaml`, `.env`
- Backup files: `.bak`, `.old`, `.orig`, `.save`, `.swp`, `.tmp`
- Backup patterns: `file.ext.bak`, `file.ext~`, `.file.ext`, `file.ext.1`
- Version control: `/.git/HEAD`, `/.svn/entries`, `/.hg/`, `/.env`

### Phase 5: API endpoint discovery
If API endpoints were found (paths starting with `/api/`, `/v1/`, `/graphql`, etc.):
1. Use @tool:ffuf with API-specific wordlists
2. Try versioning patterns: `/api/v1/`, `/api/v2/`, `/v1/`, `/v2/`
3. Check for documentation: `/swagger-ui/`, `/docs`, `/redoc`, `/swagger.json`, `/openapi.json`, `/graphql` (introspection)
4. Test HTTP methods on discovered endpoints using `execute_request` — try GET, POST, PUT, DELETE, PATCH, OPTIONS
5. OPTIONS response reveals supported methods and CORS configuration

## Expected outcomes
- **found**: Hidden paths, backup files, API endpoints, debug pages, or configuration files discovered
- **not_found**: No additional content beyond what's publicly linked
- **partial**: Some paths found but fuzzing was limited by WAF, rate limiting, or incomplete wordlist coverage

## Output
- Discovered paths with status codes, response sizes, and content descriptions
- API endpoints with supported HTTP methods
- Backup/config files found (with severity assessment — `.env` files are critical)
- Access-controlled paths (401/403) worth further investigation
- Version control artifacts found (`.git/HEAD` is critical — may expose source code)
```

---

### `skills/check-xss-reflected/skill.yaml`

```yaml
name: check-xss-reflected
version: "1.0.0"
description: Test for reflected XSS by injecting payloads into parameters and checking for unencoded reflection
type: atomic
tags: [xss, injection, owasp, cwe-79]
tools: []
mode: discover
depends_on: []
intensity: 2
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL with parameters to test for reflected XSS"
```

---

### `skills/check-xss-reflected/skill.md`

```markdown
# Check Reflected XSS

## Objective
Test whether user-controlled input is reflected in the response without proper encoding, enabling cross-site scripting.

## Procedure

### Step 1: Identify reflection points
Use `get_traffic` and `get_request_details` to find the target's parameters. Identify all user-controlled input that could appear in the response:
- URL query parameters (e.g., `?search=TERM`, `?q=TERM`, `?name=VALUE`)
- Form fields (POST body parameters)
- URL path segments (e.g., `/user/NAME/profile`)
- HTTP headers reflected in error pages (Referer, User-Agent)

### Step 2: Reflection canary
For each parameter, inject a unique non-malicious canary string (e.g., `hb7xss9test`) via `execute_request` and check if it appears verbatim in the response body. This confirms the parameter's value is reflected.

If the canary does not appear: the parameter is not reflected — skip it.

### Step 3: Context analysis
For each reflected parameter, determine the reflection context by examining the HTML around the canary:

**HTML body context:** `<p>Your search: hb7xss9test</p>`
- Test: `<img src=x onerror=alert(1)>`
- Encoding needed to block: `<` and `>` must be HTML-encoded

**HTML attribute context:** `<input value="hb7xss9test">`
- Test: `" onfocus=alert(1) autofocus="`
- Encoding needed: `"` must be HTML-encoded

**JavaScript context:** `var x = "hb7xss9test";`
- Test: `";alert(1)//`
- Encoding needed: `"`, `\`, and `;` must be escaped

**URL context:** `<a href="https://target.com/redirect?url=hb7xss9test">`
- Test: `javascript:alert(1)`
- Encoding needed: `javascript:` protocol must be blocked

### Step 4: Payload testing
Based on the context, craft specific payloads using `execute_request`:

**Basic payloads (try first):**
- `<script>alert(1)</script>`
- `<img src=x onerror=alert(1)>`
- `<svg onload=alert(1)>`
- `"><script>alert(1)</script>`

**If basic payloads are filtered, try bypasses:**
- Case variation: `<ScRiPt>alert(1)</ScRiPt>`
- Tag alternatives: `<details open ontoggle=alert(1)>`, `<marquee onstart=alert(1)>`
- Encoding: `%3Csvg%20onload%3Dalert(1)%3E`
- Double encoding: `%253Csvg%2520onload%253Dalert(1)%253E`
- Null bytes: `<scr%00ipt>alert(1)</script>`
- Attribute context escapes: `' onfocus='alert(1)` (for single-quoted attributes)

### Step 5: Filter analysis
If payloads are being sanitized:
1. Determine what's being filtered: specific tags? Event handlers? Angle brackets? Quotes?
2. Try payloads that avoid filtered patterns
3. Test if the filter is applied consistently on all parameters or just some
4. Check if POST parameters have different filtering than GET parameters

## Expected outcomes
- **found**: Parameter reflects input without encoding, allowing script execution (include exact payload and response showing execution context)
- **not_found**: All reflected input is properly encoded or sanitized for its context
- **partial**: Reflection exists but WAF or server-side filter blocks all tested payloads (filter may be bypassable with more research)
- **variant**: Input is reflected and partially encoded but in a context where encoding is insufficient (e.g., JavaScript string context with only HTML encoding)

## Output
- Each tested parameter and its reflection context
- Successful payload (if found) with exact request/response evidence
- Filter behavior observed (what was blocked, what was passed)
- Request IDs from traffic for evidence
```

---

### `skills/check-sqli-error/skill.yaml`

```yaml
name: check-sqli-error
version: "1.0.0"
description: Test for error-based SQL injection by injecting SQL syntax and observing error responses
type: atomic
tags: [sqli, injection, owasp, cwe-89, database]
tools: []
mode: discover
depends_on: []
intensity: 2
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL with parameters to test for SQL injection"
```

---

### `skills/check-sqli-error/skill.md`

```markdown
# Check Error-Based SQL Injection

## Objective
Test whether user-controlled input reaches a SQL query without proper parameterization, by injecting SQL syntax and observing error messages or behavioral differences.

## Procedure

### Step 1: Identify injectable parameters
Use `get_traffic` to find endpoints with parameters that likely interact with a database:
- Search/filter parameters (`?search=`, `?q=`, `?id=`, `?category=`)
- Authentication endpoints (login forms, password reset)
- API endpoints that return structured data from a data store
- Sorting/ordering parameters (`?sort=`, `?order=`, `?orderby=`)

### Step 2: Baseline the normal response
Use `execute_request` with the original parameter value. Record:
- Response status code
- Response body size
- Key content indicators (e.g., number of results, specific text)

### Step 3: SQL syntax injection
For each parameter, inject SQL syntax characters and compare responses to baseline:

**Single quote test:** Replace value with `value'`
- SQL error in response → error-based SQLi confirmed
- Different response (size, content, status) → potential blind SQLi
- Same response → parameter may not reach SQL

**Double quote test:** `value"`
- Tests for double-quoted SQL contexts (MySQL with ANSI_QUOTES, PostgreSQL)

**Comment test:** `value'--` and `value'/*`
- If `value'` causes an error but `value'--` returns normally → confirms SQL injection (the comment neutralizes the syntax error)

**Boolean tests (blind):**
- `value' AND '1'='1` vs `value' AND '1'='2`
- `value' OR '1'='1` vs `value' OR '1'='2`
- If true-condition matches baseline and false-condition differs → blind SQLi

**Numeric parameter tests:** (for `?id=1` style params)
- `1 AND 1=1` vs `1 AND 1=2`
- `1-0` (should equal `1`) vs `2-1` (should also equal `1`)
- Arithmetic equivalence confirms the value is interpreted as SQL

### Step 4: Error fingerprinting
If error messages appear, identify the database engine:
- `You have an error in your SQL syntax` → MySQL
- `ERROR: syntax error at or near` → PostgreSQL
- `Unclosed quotation mark` / `Incorrect syntax near` → MSSQL
- `ORA-01756` / `ORA-00933` → Oracle
- `SQLITE_ERROR` / `near "...": syntax error` → SQLite

### Step 5: Determine injection type
Based on observations:
- **Error-based:** database errors visible in response → can extract data via error messages
- **Boolean-blind:** no errors, but response content differs between true/false conditions
- **Time-blind:** no visible difference, but `value' AND SLEEP(5)--` (MySQL) or `value'; WAITFOR DELAY '0:0:5'--` (MSSQL) causes delayed response
- Test time-blind only if error-based and boolean-blind show no results

## Expected outcomes
- **found**: SQL injection confirmed — error messages contain SQL syntax errors, or boolean/time-based behavioral differences prove query manipulation. Include exact payloads and responses.
- **not_found**: All parameters properly parameterized — no SQL errors, no behavioral differences with injected syntax
- **partial**: Suspicious behavior (error messages or slight response differences) but unable to confirm exploitation. May need deeper testing with sqlmap.
- **variant**: Input reaches SQL but is filtered/escaped in a non-standard way (e.g., backslash escaping instead of parameterization)

## Output
- Parameters tested and injection payloads used
- Error messages observed (exact text)
- Database engine identified
- Injection type confirmed (error-based, boolean-blind, time-blind)
- Request IDs for evidence
```

---

### `skills/check-open-redirect/skill.yaml`

```yaml
name: check-open-redirect
version: "1.0.0"
description: Test for open redirect vulnerabilities in URL redirect parameters
type: atomic
tags: [redirect, owasp, phishing, cwe-601]
tools: []
mode: discover
depends_on: []
intensity: 2
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL with redirect parameter to test"
```

---

### `skills/check-open-redirect/skill.md`

```markdown
# Check Open Redirect

## Objective
Test whether the target accepts arbitrary URLs in redirect parameters, allowing phishing attacks that abuse the target's trusted domain.

## Procedure

### Step 1: Identify redirect parameters
Search captured traffic with `get_traffic` for redirect patterns:
- URL parameters: `?redirect=`, `?url=`, `?next=`, `?return=`, `?returnUrl=`, `?continue=`, `?dest=`, `?goto=`, `?target=`, `?rurl=`, `?callback=`
- Login/logout flows often redirect after auth: check post-login redirect behavior
- OAuth callbacks: `?redirect_uri=`

### Step 2: Test basic redirect
For each redirect parameter, use `execute_request` to inject an external URL and check the `Location` header in the response:

**Direct external URL:**
- `?redirect=https://evil.com`
- Check: does the response have a 3xx status with `Location: https://evil.com`?

**Protocol-relative URL:**
- `?redirect=//evil.com`
- Some filters check for `http://` or `https://` but miss `//` prefix

### Step 3: Filter bypass attempts
If direct URLs are blocked, try bypass techniques:

**URL encoding:**
- `?redirect=https%3A%2F%2Fevil.com`
- `?redirect=%2F%2Fevil.com`

**Backslash confusion:**
- `?redirect=https://evil.com\@target.com` — some parsers see target.com as the host
- `?redirect=\/\/evil.com` — backslash normalization differences

**Domain confusion:**
- `?redirect=https://target.com.evil.com` — subdomain of attacker domain
- `?redirect=https://target.com@evil.com` — userinfo component points to evil.com
- `?redirect=https://evil.com#target.com` — fragment confusion
- `?redirect=https://evil.com?.target.com` — query string confusion

**Path-based:**
- `?redirect=/\evil.com` — backslash treated as path separator
- `?redirect=////evil.com` — multiple slash normalization

**Data/JavaScript URI:**
- `?redirect=javascript:alert(1)` — if redirect is via client-side JS
- `?redirect=data:text/html,<script>alert(1)</script>`

### Step 4: Verify redirect chain
If the target returns a 3xx redirect, follow the chain with `execute_request` to confirm the final destination is the attacker-controlled URL. Some servers redirect through an intermediate page.

## Expected outcomes
- **found**: Target redirects to arbitrary external URL — include exact payload and Location header value
- **not_found**: Target properly validates redirect destinations (allowlist or same-origin check)
- **partial**: Redirect exists but is constrained (e.g., only allows same-domain paths, or adds a warning interstitial)
- **variant**: Redirect is via client-side JavaScript (meta refresh or `window.location`) rather than HTTP 3xx — still exploitable but different mechanism

## Output
- Redirect parameters found and their behavior
- Successful bypass payload (if found)
- Redirect type (server-side 3xx vs client-side JS)
- Request IDs for evidence
```

---

### `skills/check-idor/skill.yaml`

```yaml
name: check-idor
version: "1.0.0"
description: Test for insecure direct object reference by manipulating resource identifiers
type: atomic
tags: [idor, authorization, access-control, owasp, cwe-639]
tools: []
mode: discover
depends_on: []
intensity: 2
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL with resource identifier to test"
```

---

### `skills/check-idor/skill.md`

```markdown
# Check IDOR

## Objective
Test whether the target enforces authorization on object access — can one user's data be accessed by manipulating the resource identifier (numeric IDs, UUIDs, filenames)?

## Procedure

### Step 1: Identify object references
Use `get_traffic` to find endpoints that reference specific objects:
- Numeric IDs: `/api/users/123`, `/api/orders/456`, `?id=789`
- UUIDs: `/api/documents/a1b2c3d4-...`
- Filenames: `/uploads/report.pdf`, `/files/invoice_123.pdf`
- Encoded references: base64-encoded IDs, hashed references

### Step 2: Baseline request
Use `execute_request` to fetch the legitimate resource with the original identifier. Record:
- Response status code (should be 200)
- Response body content and size
- Any authorization headers used (cookies, Bearer tokens)

### Step 3: Horizontal access test
Modify the resource identifier to reference a different user's resource:

**Sequential ID manipulation:**
- If current ID is `123`, try `122`, `124`, `1`, `0`, `999999`
- If the response returns different user data with the same session → IDOR confirmed

**UUID guessing:**
- UUIDs are harder to guess, but check:
  - Does the API return UUID references for other users' objects in any response? (e.g., list endpoints, search results)
  - Are UUIDs sequential (v1 UUIDs contain timestamps) or truly random (v4)?

**Filename manipulation:**
- Change `/uploads/user123/report.pdf` to `/uploads/user124/report.pdf`
- Try path traversal: `/uploads/../user124/report.pdf`

### Step 4: Vertical access test (privilege escalation)
If different user roles exist:
- Access admin-only endpoints with a regular user session
- Change role-specific IDs (admin user ID, admin project ID)
- Access management API endpoints that should be restricted

### Step 5: HTTP method variation
Test if authorization is consistent across HTTP methods:
- If GET `/api/users/123` is blocked, try PUT, PATCH, DELETE on the same resource
- Some applications only check authorization on read operations but not write operations

### Step 6: Parameter pollution
Test if the server handles duplicate parameters inconsistently:
- `/api/users?id=123&id=456` — which ID does the server use?
- Mix body and URL parameters: PUT `/api/users/123` with body `{"id": 456}`

## Expected outcomes
- **found**: Resource belonging to another user/role is accessible by manipulating the identifier. Include exact request showing unauthorized access and the data returned.
- **not_found**: Server properly validates authorization for each resource — returns 403 or 404 for unauthorized IDs
- **partial**: Some endpoints are protected but others using the same ID scheme are not (inconsistent authorization)
- **variant**: Object references are predictable (sequential IDs) but authorization is currently enforced — the predictability is an enumeration risk even if IDOR is blocked

## Output
- Endpoints tested and identifier types (numeric, UUID, filename)
- Successful unauthorized access (if found) with request/response evidence
- Authorization model observations (consistent enforcement vs. gaps)
- Request IDs for evidence
```

---

### `skills/check-ssrf/skill.yaml`

```yaml
name: check-ssrf
version: "1.0.0"
description: Test for server-side request forgery by injecting internal/external URLs into server-fetched parameters
type: atomic
tags: [ssrf, injection, owasp, cwe-918, cloud]
tools: []
mode: discover
depends_on: []
intensity: 3
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL with parameter that triggers server-side fetching"
```

---

### `skills/check-ssrf/skill.md`

```markdown
# Check SSRF

## Objective
Test whether user-controlled URLs or hostnames are fetched server-side without proper validation, enabling access to internal services, cloud metadata, or other restricted resources.

## Procedure

### Step 1: Identify server-side fetch parameters
Use `get_traffic` to find endpoints where the server fetches a user-supplied URL:
- URL/webhook parameters: `?url=`, `?src=`, `?href=`, `?link=`, `?feed=`, `?callback=`
- Image/file import: upload-from-URL features, avatar-from-URL, PDF generation from URL
- Proxy/preview features: link previews, URL shorteners, screenshot services
- Webhook configuration endpoints
- XML/SVG processing (XXE-to-SSRF via DTD external entities)

### Step 2: Out-of-band detection
First confirm the server fetches URLs at all. Use a URL that you can monitor for incoming requests:
- Use `execute_request` to supply your controlled URL as the parameter value
- If you don't have an out-of-band server, use response-based detection (Step 3)

### Step 3: Response-based detection
If the server returns the fetched content or metadata:

**Localhost access:**
- `?url=http://127.0.0.1:80/` — does the response show the server's own homepage?
- `?url=http://localhost/server-status` — Apache status page
- `?url=http://[::1]/` — IPv6 localhost

**Cloud metadata (critical severity if accessible):**
- AWS: `?url=http://169.254.169.254/latest/meta-data/`
- GCP: `?url=http://metadata.google.internal/computeMetadata/v1/` (needs header `Metadata-Flavor: Google`)
- Azure: `?url=http://169.254.169.254/metadata/instance?api-version=2021-02-01` (needs header `Metadata: true`)

**Internal network probing:**
- `?url=http://192.168.1.1/` — common gateway
- `?url=http://10.0.0.1/` — internal network
- `?url=http://172.16.0.1/` — internal network

### Step 4: Filter bypass attempts
If direct internal URLs are blocked:

**URL encoding:** `http://127.0.0.1/` → `http://%31%32%37%2e%30%2e%30%2e%31/`

**Alternative IP representations:**
- Decimal: `http://2130706433/` (= 127.0.0.1)
- Hex: `http://0x7f000001/`
- Octal: `http://0177.0.0.1/`
- Mixed: `http://127.0.0.1.nip.io/` (DNS rebinding via wildcard DNS)

**Redirect-based bypass:**
- Host an external URL that 302-redirects to `http://169.254.169.254/`
- If the server follows redirects, the initial URL check passes but the final fetch hits metadata

**DNS rebinding:**
- Use a domain that alternates between external and internal IPs
- First resolution passes allowlist → second resolution (during fetch) hits internal IP

**Protocol smuggling:**
- `gopher://127.0.0.1:6379/_SET%20key%20value` — Redis via gopher (if supported)
- `file:///etc/passwd` — local file read (if file:// protocol is supported)
- `dict://127.0.0.1:6379/info` — Redis info via dict protocol

### Step 5: Port scanning
If SSRF is confirmed, probe internal services:
- Common ports: 80, 443, 8080, 8443, 3000, 5000, 6379, 27017, 5432, 3306
- Measure response time differences to distinguish open (quick response) vs closed (timeout) ports

## Expected outcomes
- **found**: Server fetches user-supplied URL targeting internal resources or cloud metadata. Include exact request and evidence of internal content in response.
- **not_found**: Server validates URLs to prevent internal access (allowlist, no private IP ranges)
- **partial**: Server fetches URLs but blocks obvious internal targets. Filter may be bypassable with more research.
- **variant**: SSRF exists but is blind (no response content returned). Server makes the request but doesn't expose the response — still exploitable for cloud metadata credential theft.

## Output
- Parameters that trigger server-side fetch
- Internal resources accessible (if any)
- Cloud metadata accessible (if any — this is critical severity)
- Filter bypass techniques that worked
- Request IDs for evidence
```

---

### `skills/check-jwt/skill.yaml`

```yaml
name: check-jwt
version: "1.0.0"
description: Analyze JWT implementation for signature bypass, weak secrets, and algorithm confusion attacks
type: atomic
tags: [jwt, auth, token, owasp, cwe-347]
tools: []
mode: discover
depends_on: []
intensity: 2
inputs:
  - name: target_url
    type: string
    required: true
    description: "URL where JWT authentication is used"
```

---

### `skills/check-jwt/skill.md`

```markdown
# Check JWT

## Objective
Analyze the target's JWT (JSON Web Token) implementation for signature verification bypass, weak signing secrets, algorithm confusion, and claim manipulation vulnerabilities.

## Procedure

### Step 1: Obtain a JWT
Use `get_traffic` to find JWTs in captured traffic:
- `Authorization: Bearer eyJ...` headers
- Cookies containing JWT-format values (three base64url segments separated by dots)
- Response bodies returning tokens (login endpoints, OAuth flows)

Decode the JWT header and payload (base64url decode, no signature verification needed):
- Header: `{"alg": "HS256", "typ": "JWT"}`
- Payload: `{"sub": "user123", "role": "user", "exp": 1700000000, ...}`

### Step 2: Algorithm confusion attack
If the header uses an asymmetric algorithm (RS256, RS384, RS512, ES256, PS256):

**"none" algorithm:** Modify the header to `{"alg": "none"}`, remove the signature, and submit:
- Token becomes: `eyJ...header.eyJ...payload.`
- If accepted → critical vulnerability (no signature verification)

**Algorithm downgrade (RS256 → HS256):**
- Change header from `RS256` to `HS256`
- Sign the token using the server's RSA public key as the HMAC secret
- If the server's JWT library uses the `alg` header to select verification → it will verify HMAC using the public key (which is publicly available)

### Step 3: Signature verification test
Modify a single character in the payload (e.g., change `"role":"user"` to `"role":"admin"`) but keep the original signature. Submit via `execute_request`:
- If the server accepts the modified token → signature is not verified (critical)
- If the server rejects it (401/403) → signature verification is working

### Step 4: Claim manipulation
If signature bypass is found, test what claims are trusted:
- Change `"sub"` to another user's ID → horizontal privilege escalation
- Change `"role": "user"` to `"role": "admin"` → vertical privilege escalation
- Change `"exp"` to a far-future timestamp → token never expires
- Add `"admin": true` or `"is_staff": true` → custom claim injection

### Step 5: Token expiration and refresh
- Check `exp` claim: is token lifetime reasonable? (>24h is excessive for access tokens)
- Submit an expired token: does the server reject it? (Missing expiration check)
- Check for refresh token rotation: does using a refresh token invalidate the old one?
- Check if logout invalidates the JWT server-side (JWTs are typically stateless — logout may not actually revoke the token)

### Step 6: Information disclosure
Examine JWT payload for sensitive data:
- Internal user IDs, database references
- Role/permission details (helps plan privilege escalation)
- Internal service URLs or API keys
- PII (email, name) that shouldn't be in a client-visible token

## Expected outcomes
- **found**: JWT signature bypass, algorithm confusion, or claim manipulation allows unauthorized access. Include exact modified token and server response.
- **not_found**: JWT implementation properly verifies signatures, enforces algorithm, and validates claims
- **partial**: Signature is verified but other weaknesses exist (excessive lifetime, sensitive data in payload, no revocation)
- **variant**: JWT uses "none" algorithm by design (unsigned tokens for non-sensitive data) — assess if this is intentional

## Output
- JWT algorithm used and signature type
- Signature verification status (enforced or bypassed)
- Claims found in payload (roles, permissions, sensitive data)
- Successful claim manipulation (if any)
- Token lifecycle observations (expiration, refresh, revocation)
- Request IDs for evidence
```

---

## Full Skills

### `skills/web-app-baseline/skill.yaml`

```yaml
name: web-app-baseline
version: "1.0.0"
description: Baseline security assessment covering security headers, CORS, and framing defenses
type: full
tags: [baseline, headers, defense, audit, owasp]
tools: []
mode: discover
depends_on: [check-security-headers, check-cors, check-framing]
intensity: 1
inputs:
  - name: target_url
    type: string
    required: true
    description: "Base URL to assess"
```

---

### `skills/web-app-baseline/skill.md`

```markdown
# Web App Baseline Security Assessment

## Objective
Perform a baseline security assessment of the target's defensive posture by auditing security headers, CORS configuration, and framing protections across the application.

## Procedure

### Phase 1: Identify key pages
Use `get_traffic` to find representative pages across the application:
- Landing/home page
- Login page (if exists)
- Authenticated page (if traffic shows logged-in requests)
- API endpoints
- Static asset pages
Collect 3-5 distinct URLs that represent different parts of the application.

### Phase 2: Security headers audit
For each collected URL, run:
@check-security-headers

Note: headers often differ between HTML pages, API responses, and static assets. Testing multiple URL types reveals inconsistencies.

### Phase 3: CORS configuration audit
For the target URL and any API endpoints found in traffic, run:
@check-cors

Pay special attention to API endpoints — they are the most common location for CORS misconfigurations because they need cross-origin access for legitimate SPAs.

### Phase 4: Framing protection audit
For the main application URL and any state-changing pages (forms, account pages), run:
@check-framing

### Phase 5: Cross-cutting analysis
After running all three checks, analyze patterns:
- Are security headers consistent across all pages or only on some routes?
- Do API endpoints have different (often weaker) security than HTML pages?
- Is the framing defense comprehensive or limited to specific pages?
- Are there CSP inconsistencies between pages (one has strict CSP, another has none)?

## Expected outcomes
- **found**: Significant security header gaps, CORS misconfigurations, or missing framing defenses that enable specific attacks
- **not_found**: Application has comprehensive security headers, strict CORS, and proper framing defenses across all routes
- **partial**: Some defenses are in place but inconsistently applied, or specific routes have weaker posture

## Output
- Per-page security header summary
- CORS configuration assessment
- Framing defense status
- Consistency analysis across different route types
- Prioritized list of missing defenses with attack scenarios each enables
```

---

### `skills/injection-scan/skill.yaml`

```yaml
name: injection-scan
version: "1.0.0"
description: Multi-vector injection testing covering XSS, SQLi, and open redirect across discovered endpoints
type: full
tags: [injection, xss, sqli, redirect, owasp, comprehensive]
tools: [ffuf]
mode: discover
depends_on: [check-xss-reflected, check-sqli-error, check-open-redirect, content-discovery]
intensity: 4
inputs:
  - name: target_url
    type: string
    required: true
    description: "Base URL to scan for injection vulnerabilities"
```

---

### `skills/injection-scan/skill.md`

```markdown
# Injection Scan

## Objective
Perform comprehensive injection testing across the target's endpoints — discover hidden attack surface, then test each injection vector systematically.

## Procedure

### Phase 1: Endpoint discovery
First, expand the known attack surface by discovering hidden paths and API routes:
@content-discovery

Collect all endpoints from discovery results plus those already in captured traffic (`get_traffic`). Prioritize endpoints that accept user input (query parameters, POST bodies, URL path segments).

### Phase 2: Parameter inventory
For each discovered endpoint, catalog injectable parameters:
- Use `get_traffic` and `get_request_details` to identify parameters from existing traffic
- Use `execute_request` to probe endpoints discovered in Phase 1 that aren't in traffic yet
- Classify each parameter by likely backend interaction:
  - Database queries (search, filter, sort, ID lookups) → SQLi candidates
  - HTML output (search terms reflected in page, error messages) → XSS candidates
  - URL handling (redirects, callbacks, links) → open redirect candidates

### Phase 3: Reflected XSS testing
For each parameter that appears in HTML output, run:
@check-xss-reflected

Focus on parameters that reflect user input in the response body. Test all reflection contexts (HTML body, attribute, JavaScript, URL).

### Phase 4: SQL injection testing
For each parameter that likely interacts with a database, run:
@check-sqli-error

Focus on search parameters, filter parameters, and ID-based lookups. Start with error-based detection before attempting blind techniques.

### Phase 5: Open redirect testing
For each parameter that handles URLs or redirect logic, run:
@check-open-redirect

Include OAuth callback URLs, login/logout redirect parameters, and any parameter containing URL-like values.

### Phase 6: Cross-vector analysis
After testing all three injection types, analyze patterns:
- Are different parameters on the same endpoint vulnerable to different injection types?
- Does the application apply consistent input validation or is it per-parameter?
- Are API endpoints more permissive than HTML-serving endpoints?
- Do POST parameters have different filtering than GET parameters?

## Expected outcomes
- **found**: One or more injection vulnerabilities confirmed with evidence across the discovered attack surface
- **not_found**: All tested parameters properly validated — no injection vulnerabilities found
- **partial**: Some endpoints showed suspicious behavior but couldn't be confirmed. Specific parameters may need deeper testing (e.g., with sqlmap for blind SQLi).

## Output
- Complete parameter inventory with classification
- Per-vector test results (XSS, SQLi, redirect)
- Successful injections with exact payloads and request/response evidence
- Endpoints needing deeper investigation
- Coverage summary: how many parameters were tested vs total discovered
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
| `scope_extraction` | object | null | `{flag, pattern}` — how to extract target hostname from CLI args. `flag` is the CLI flag containing the URL, `pattern` is a regex with exactly one capture group for the hostname. |
| `output_flags` | string | "" | Flags to force structured output format. `{output_file}` is replaced with a temp path at runtime. |
| `default_timeout` | int | 300 | Timeout in seconds before auto-termination |

### skill.yaml — required fields (CI enforced)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Must match directory name exactly (lowercase-kebab-case) |
| `description` | string | Yes | What the skill checks |
| `type` | string | Yes | Must be `"atomic"` or `"full"` |
| `tags` | list[string] | Yes | For filtering/discovery |
| `mode` | string | Yes | Primary agent mode: `discover`, `research`, `exploit`, `verify`, or `report` |

### skill.yaml — optional fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `version` | string | "0.0.0" | Per-skill version tracking |
| `tools` | list[string] | [] | Required tool names (cross-validated against tools/) |
| `depends_on` | list[string] | [] | Atomic skill names (must be empty for atomic type) |
| `intensity` | int | 3 | 1-5 for work queue lane assignment |
| `inputs` | list[object] | [] | Input parameters. Each: `{name: str, type: str, required: bool, default: any, description: str}` |

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
- `@skill-name` references in skill.md must appear in `depends_on` (and vice versa — bidirectional consistency)
- `@tool:name` references in skill.md must have corresponding `tools/<name>/` directory
- `tools` list entries in skill.yaml must have corresponding `tools/<name>/` directory
- Code blocks (fenced with triple backticks) are stripped before scanning for `@` references (prevents false positives in examples)
