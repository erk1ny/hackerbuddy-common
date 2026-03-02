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
