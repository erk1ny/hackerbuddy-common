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
