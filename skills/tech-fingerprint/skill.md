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
- `/wp-admin/`, `/wp-includes/` — WordPress
- `/admin/config/`, `/user/login` — Drupal
- `/administrator/` — Joomla
- `/__debug__/`, `/admin/` (Django-style) — Django
- `/elmah.axd`, `/trace.axd` — ASP.NET debugging
- `/actuator/`, `/actuator/health` — Spring Boot
- `/rails/info/routes` — Ruby on Rails (dev mode)
- `/server-info`, `/server-status` — Apache mod_status
- `/swagger.json`, `/openapi.json`, `/api-docs` — API documentation

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
