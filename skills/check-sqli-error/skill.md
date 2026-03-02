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
- SQL error in response — error-based SQLi confirmed
- Different response (size, content, status) — potential blind SQLi
- Same response — parameter may not reach SQL

**Double quote test:** `value"`
- Tests for double-quoted SQL contexts (MySQL with ANSI_QUOTES, PostgreSQL)

**Comment test:** `value'--` and `value'/*`
- If `value'` causes an error but `value'--` returns normally — confirms SQL injection (the comment neutralizes the syntax error)

**Boolean tests (blind):**
- `value' AND '1'='1` vs `value' AND '1'='2`
- `value' OR '1'='1` vs `value' OR '1'='2`
- If true-condition matches baseline and false-condition differs — blind SQLi

**Numeric parameter tests:** (for `?id=1` style params)
- `1 AND 1=1` vs `1 AND 1=2`
- `1-0` (should equal `1`) vs `2-1` (should also equal `1`)
- Arithmetic equivalence confirms the value is interpreted as SQL

### Step 4: Error fingerprinting
If error messages appear, identify the database engine:
- `You have an error in your SQL syntax` — MySQL
- `ERROR: syntax error at or near` — PostgreSQL
- `Unclosed quotation mark` / `Incorrect syntax near` — MSSQL
- `ORA-01756` / `ORA-00933` — Oracle
- `SQLITE_ERROR` / `near "...": syntax error` — SQLite

### Step 5: Determine injection type
Based on observations:
- **Error-based:** database errors visible in response — can extract data via error messages
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
