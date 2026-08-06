# VOGG SECURITY SPECIFICATION

**Date:** July 2026  
**Version:** 1.0  
**Status:** ✅ Final  
**Purpose:** Complete security architecture and requirements  
**Classification:** Internal Use Only  

---

## PART 1: SECURITY PRINCIPLES

### Defense in Depth

Multiple layers of security (assume breach mentality):

```
Layer 1: Network Security (WAF, DDoS protection)
    ↓
Layer 2: Transport Security (TLS, HTTPS)
    ↓
Layer 3: Application Security (Auth, validation, RBAC)
    ↓
Layer 4: Data Security (Encryption at rest)
    ↓
Layer 5: Access Control (IAM, audit logging)
```

### Least Privilege

- Users have **minimum required** permissions
- Default deny (whitelist approach)
- Role-based access control (RBAC)
- Time-limited tokens

### Zero Trust

- Verify **every** request
- Never trust IP addresses or networks
- Authenticate and authorize always
- Assume network is compromised

---

## PART 2: AUTHENTICATION

### User Registration Security

```
Register Flow:
1. User submits email + password
2. Validate email format
3. Validate password strength:
   - Minimum 12 characters
   - Uppercase + lowercase + number + special char
   - Cannot contain email address
4. Hash password: bcrypt with 12 rounds
   - Cost factor: 12 (provides sufficient security)
   - Salt: auto-generated per password
5. Store hash (never plaintext)
6. Send verification email with token
7. Token expires in 24 hours
```

### Password Security

**Requirements:**
- Minimum 12 characters
- Must include:
  - Uppercase letter (A-Z)
  - Lowercase letter (a-z)
  - Number (0-9)
  - Special character (!@#$%^&*)
- Cannot contain email address
- Cannot be common passwords (check against list)

**Storage:**
```
Password Hash:
- Algorithm: bcrypt
- Cost factor: 12 rounds
- Salt: auto-generated
- Never store plaintext
- Never log password
```

**Password Reset:**
```
1. User requests password reset
2. System generates secure token (crypto.randomBytes(32))
3. Token hashed and stored (not plaintext)
4. Email sent with reset link
5. Token expires in 1 hour
6. User clicks link, validates token, enters new password
7. Validate new password (same rules as registration)
8. Update password hash
9. Invalidate all existing sessions
10. Require login with new password
```

### JWT Authentication

**Token Structure:**
```
Header: {
  "alg": "RS256",
  "typ": "JWT"
}

Payload: {
  "sub": "user_uuid",
  "email": "user@example.com",
  "roles": ["member", "admin"],
  "organizationId": "org_uuid",
  "iat": 1719399600,
  "exp": 1719400500
}

Signature: RS256(base64(header) + "." + base64(payload), privateKey)
```

**Token Lifecycle:**
```
Access Token:
- Expiry: 15 minutes
- Use: Authorization header
- Refresh: Via refresh token

Refresh Token:
- Expiry: 7 days
- Use: Refresh endpoint only
- Storage: HTTP-only, Secure cookies
- Cannot use for API access
```

**Token Validation:**
```
Every request:
1. Extract token from Authorization header
2. Verify signature (check it wasn't tampered)
3. Check expiry (not expired)
4. Check claims (sub, exp, iat valid)
5. Lookup user (still active, not suspended)
6. Allow request if all checks pass
7. Reject if any check fails
```

---

## PART 3: AUTHORIZATION (RBAC)

### Role Definitions

```
Role: superadmin
├─ Can create organizations
├─ Can delete any organization
├─ Can view audit logs
└─ Full platform access

Role: admin (per organization)
├─ Manage organization members
├─ Create and publish votes
├─ Close votes
├─ View organization audit
├─ Manage roles
└─ View results

Role: moderator (per organization)
├─ Create votes
├─ Publish votes
├─ View results
└─ Moderate discussions

Role: member (per organization)
├─ View organization
├─ Participate in votes
├─ View results
└─ View public data

Role: viewer (per organization)
├─ View organization (read-only)
├─ View closed votes only
└─ View public data
```

### Permission Matrix

```
                    superadmin  admin  moderator  member  viewer
Organization:create     ✓       -         -        -       -
Organization:delete     ✓       ✓         -        -       -
Organization:update     ✓       ✓         -        -       -
Member:add              ✓       ✓         -        -       -
Member:remove           ✓       ✓         -        -       -
Vote:create             ✓       ✓         ✓        ✓       -
Vote:publish            ✓       ✓         ✓        -       -
Vote:vote               ✓       ✓         ✓        ✓       -
Vote:results            ✓       ✓         ✓        ✓       ✓(closed)
Audit:view              ✓       ✓         -        -       -
```

### Authorization Enforcement

**Every Endpoint:**
```typescript
// Example: Create vote endpoint
app.post('/votes', [
  authMiddleware,           // Verify JWT
  rbacMiddleware('vote:create'),  // Check permission
  validateInput,            // Validate data
  (req, res) => {
    // Handle request
  }
]);
```

**Middleware Check:**
```typescript
function rbacMiddleware(requiredPermission) {
  return (req, res, next) => {
    // 1. Get user from JWT
    const user = req.user;
    
    // 2. Get user's roles in organization
    const roles = await getRoles(user.id, req.body.organizationId);
    
    // 3. Check if any role has permission
    const hasPermission = roles.some(role => 
      role.permissions.includes(requiredPermission)
    );
    
    // 4. Allow or deny
    if (hasPermission) {
      next();
    } else {
      res.status(403).json({ error: 'Forbidden' });
    }
  };
}
```

---

## PART 4: DATA SECURITY

### Encryption at Rest

**Database:**
```
PostgreSQL encryption:
- AWS RDS with default encryption
- AES-256 encryption
- Managed by AWS KMS (Key Management Service)
- Keys rotated automatically
- Encryption transparent to application
```

**Sensitive Fields (Future Phase 2):**
```
// Encrypted at application level
- Personal identification numbers
- Social security numbers
- Government IDs
- Financial account numbers

Implementation:
- TweetNaCl.js for encryption
- Separate encryption keys per organization
- Key derivation from master key
```

### Encryption in Transit

**HTTPS/TLS:**
```
All traffic encrypted:
- Protocol: TLS 1.3 minimum
- Cipher: High-strength ciphers only
- Certificate: Let's Encrypt or AWS ACM
- HSTS: Enabled (2 year max age)

Header: Strict-Transport-Security: max-age=63072000; includeSubDomains
```

**Certificate Pinning (Mobile - Phase 2):**
```
Mobile apps:
- Pin expected certificate
- Prevent man-in-the-middle attacks
- Certificate rotation strategy
```

---

## PART 5: SECRETS MANAGEMENT

### Secret Types & Storage

```
Database credentials:
- Stored in: AWS Secrets Manager
- Rotation: 90 days
- Access: Application IAM role only

API Keys:
- Stored in: AWS Secrets Manager
- Rotation: 90 days
- Scoped: Least privilege

JWT Secret:
- Stored in: AWS Secrets Manager
- Rotation: Quarterly
- Backup: 2 copies in separate regions

Email Password:
- Stored in: AWS Secrets Manager
- Rotation: 90 days
- Use OAuth when possible
```

### Secret Access Control

```
Development:
- .env.local (local only, git-ignored)
- Never commit secrets

Staging:
- AWS Secrets Manager
- Access via IAM role
- Audit all access

Production:
- AWS Secrets Manager
- Access via IAM role
- Audit all access
- Separate secrets per environment
```

### Never

```
❌ Commit secrets to Git
❌ Hardcode passwords
❌ Log secrets
❌ Send in URL parameters
❌ Include in error messages
❌ Expose in client-side code
```

---

## PART 6: AUDIT LOGGING

### What Gets Logged

```
Authentication:
- User registration
- Login success/failure
- Logout
- Token refresh
- Password change/reset

Data Changes:
- Entity create/update/delete
- Vote publish
- Vote close
- Member add/remove
- Role assignment

Sensitive Operations:
- Permission changes
- Organization suspension
- User suspension
- Audit log access

Failed Operations:
- Authorization failures
- Validation failures
- System errors
```

### Audit Log Structure

```sql
CREATE TABLE audit_logs (
  id BIGSERIAL PRIMARY KEY,
  timestamp TIMESTAMP NOT NULL,
  user_id UUID,
  action VARCHAR(50),           -- 'CREATE', 'UPDATE', 'DELETE'
  resource_type VARCHAR(50),    -- 'vote', 'user', 'organization'
  resource_id UUID,
  old_values JSONB,
  new_values JSONB,
  status VARCHAR(20),           -- 'success', 'failure'
  error_message TEXT,
  ip_address INET,
  user_agent TEXT
);

-- Immutable: Prevent UPDATE/DELETE
ALTER TABLE audit_logs DISABLE TRIGGER ALL;
REVOKE UPDATE, DELETE ON audit_logs FROM app_user;
```

### Audit Log Query Examples

```sql
-- Find all votes created by user
SELECT * FROM audit_logs
WHERE user_id = $1 AND action = 'CREATE' AND resource_type = 'vote';

-- Find all failed logins
SELECT * FROM audit_logs
WHERE action = 'LOGIN' AND status = 'failure'
ORDER BY timestamp DESC;

-- Find suspicious activity (many failures)
SELECT user_id, COUNT(*) as failures
FROM audit_logs
WHERE action = 'LOGIN' AND status = 'failure'
AND timestamp > now() - interval '1 hour'
GROUP BY user_id
HAVING COUNT(*) > 5;
```

---

## PART 7: INPUT VALIDATION & SANITIZATION

### Validation Strategy

```
All user input validated:
- Email: Format + domain verification
- Strings: Length + character set
- Numbers: Range + type
- JSON: Schema validation
- Files: Type + size
```

### XSS Prevention

```typescript
// ❌ Dangerous
res.json({ message: req.body.userInput });

// ✅ Safe (JSON escapes automatically)
res.json({ message: sanitize(req.body.userInput) });

// For HTML content (rare):
import DOMPurify from 'isomorphic-dompurify';
const safe = DOMPurify.sanitize(userHTML);
```

### SQL Injection Prevention

```typescript
// ❌ Dangerous
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ Safe (parameterized)
const user = await prisma.user.findUnique({
  where: { email: email }
});
```

### CSRF Prevention

```
CSRF Token:
- Generated: Per session
- Stored: Server-side
- Validated: On state-changing requests (POST, PUT, DELETE)
- Token: Single-use, expires in 1 hour

Implementation:
- Library: csurf or similar
- Header: X-CSRF-Token
- Validated before processing
```

---

## PART 8: RATE LIMITING

### Rate Limit Rules

```
Per User:
- 1,000 requests/minute
- 50,000 requests/day

Per API Key:
- 100,000 requests/minute
- 1,000,000 requests/day

Auth Endpoints:
- /auth/login: 5 attempts/hour per IP
- /auth/register: 5 registrations/hour per IP

Vote Endpoints:
- /votes/{id}/vote: 1 vote/minute per user per vote
```

### Implementation

```typescript
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 5,                     // 5 requests
  message: 'Too many login attempts, try again later',
  standardHeaders: true,      // Return info in RateLimit-* headers
  legacyHeaders: false,       // Disable X-RateLimit-* headers
  keyGenerator: (req) => {
    // Use IP for non-authenticated requests
    return req.ip;
  }
});

app.post('/auth/login', loginLimiter, loginHandler);
```

---

## PART 9: SECURITY HEADERS

### Response Headers

```
// Prevent clickjacking
X-Frame-Options: DENY

// Prevent MIME type sniffing
X-Content-Type-Options: nosniff

// Enable XSS filter
X-XSS-Protection: 1; mode=block

// Control referrer information
Referrer-Policy: strict-origin-when-cross-origin

// Content Security Policy
Content-Security-Policy: default-src 'self'; script-src 'self' 'trusted-domain'

// Force HTTPS
Strict-Transport-Security: max-age=31536000; includeSubDomains

// Prevent DNS prefetch
X-DNS-Prefetch-Control: off

// Don't cache sensitive data
Cache-Control: no-store, no-cache, must-revalidate
```

---

## PART 10: COMPLIANCE & STANDARDS

### GDPR Compliance

```
User Rights:
- Right to access: User can download their data
- Right to delete: "Forget me" endpoint
- Right to rectify: Update personal data
- Right to portability: Export data in standard format

Implementation:
- Data inventory: Document what data collected
- Retention policy: Delete after 2 years (if inactive)
- Consent: Explicit opt-in for non-essential processing
- Privacy Policy: Published and linked
- Data Processing Agreement: For integrations
```

### CCPA Compliance (US)

```
California Consumer Rights:
- Right to know: What data collected
- Right to delete: Request deletion
- Right to opt-out: Opt out of data selling
- Right to non-discrimination: No penalty for opting out

Implementation:
- Same as GDPR above
- Privacy Policy updated for CCPA
- Opt-out mechanism for data sales
```

### Secure Development Practices

```
Code Review:
- Security review mandatory
- OWASP Top 10 checked
- No hardcoded secrets
- No deprecated libraries

Testing:
- Security unit tests
- Input validation tests
- Authorization tests
- HTTPS/TLS tests

Scanning:
- SAST: Static code analysis
- DAST: Dynamic testing
- Dependency scanning: npm audit
- Container scanning: Image vulnerabilities
```

---

## PART 11: INCIDENT RESPONSE

### Security Incident Plan

```
1. Detect: Automated alerts + manual review
2. Assess: Severity (critical/high/medium/low)
3. Contain: Stop the bleeding
4. Investigate: Find root cause
5. Fix: Patch vulnerability
6. Notify: Customers within 24 hours
7. Review: Post-incident analysis
8. Prevent: Improve to prevent recurrence
```

### Breach Notification

```
Customer notification within 24 hours:
- What data was affected
- What we're doing about it
- What you should do
- Contact information
- Documentation of the incident

Regulatory notification (if required):
- GDPR: 72 hours to authorities
- CCPA: Immediately
- Industry-specific: As required
```

---

## PART 12: SECURITY TESTING

### Manual Testing Checklist

```
Authentication:
[ ] Can register with email
[ ] Email verification required
[ ] Cannot login without verification
[ ] Cannot login with wrong password
[ ] Tokens expire correctly
[ ] Refresh token works
[ ] Logout invalidates token

Authorization:
[ ] Cannot access org without membership
[ ] Cannot vote without permission
[ ] Cannot delete vote as non-admin
[ ] Cannot access other user's data

Input Validation:
[ ] Cannot submit empty email
[ ] Cannot submit weak password
[ ] Cannot submit too-long string
[ ] Cannot submit special characters in name
[ ] Cannot submit malicious script

Data Security:
[ ] Passwords not in logs
[ ] Secrets not in database
[ ] No sensitive data in URLs
[ ] No sensitive data in error messages
```

### Automated Testing

```
// Example: RBAC test
describe('Authorization', () => {
  it('should not allow non-admin to delete vote', async () => {
    const result = await deleteVote(voteId, memberUser);
    expect(result.status).toBe(403);
  });

  it('should allow admin to delete vote', async () => {
    const result = await deleteVote(voteId, adminUser);
    expect(result.status).toBe(200);
  });
});
```

---

## SUMMARY

**Security Requirements:**
- ✅ Authentication (JWT + bcrypt)
- ✅ Authorization (RBAC enforced)
- ✅ Encryption (TLS in transit, AES at rest)
- ✅ Audit logging (immutable)
- ✅ Input validation (all user input)
- ✅ Rate limiting (per user/API key)
- ✅ Secrets management (AWS Secrets Manager)
- ✅ Security headers (OWASP recommended)
- ✅ Compliance (GDPR/CCPA)
- ✅ Testing (security-focused)

---

**Security Specification Complete**  
**Status:** ✅ All requirements documented and verifiable

