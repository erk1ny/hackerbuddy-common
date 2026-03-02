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
- If the response returns different user data with the same session — IDOR confirmed

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
