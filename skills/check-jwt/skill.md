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
- If accepted — critical vulnerability (no signature verification)

**Algorithm downgrade (RS256 to HS256):**
- Change header from `RS256` to `HS256`
- Sign the token using the server's RSA public key as the HMAC secret
- If the server's JWT library uses the `alg` header to select verification, it will verify HMAC using the public key (which is publicly available)

### Step 3: Signature verification test
Modify a single character in the payload (e.g., change `"role":"user"` to `"role":"admin"`) but keep the original signature. Submit via `execute_request`:
- If the server accepts the modified token — signature is not verified (critical)
- If the server rejects it (401/403) — signature verification is working

### Step 4: Claim manipulation
If signature bypass is found, test what claims are trusted:
- Change `"sub"` to another user's ID — horizontal privilege escalation
- Change `"role": "user"` to `"role": "admin"` — vertical privilege escalation
- Change `"exp"` to a far-future timestamp — token never expires
- Add `"admin": true` or `"is_staff": true` — custom claim injection

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
