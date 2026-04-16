# Security Review Report - Happy Platform MCP Server

**Review Date:** April 15, 2026  
**Reviewed By:** CISO Security Audit  
**Project:** `happy-platform-mcp` (ServiceNow MCP Server)  
**Version:** 3.0.1  
**Risk Rating:** 🔴 **CRITICAL**

---

## Executive Summary

This security assessment identified **8 critical** and **6 high-severity** vulnerabilities in the Happy Platform MCP Server codebase. The most severe finding is **exposed production credentials** committed to the repository, which requires immediate remediation.

---

## Critical Findings (Immediate Action Required)

### 1. 🔴 CRITICAL: Hardcoded Credentials Committed to Repository

**Location:** [config/servicenow-instances.json](config/servicenow-instances.json)

**Details:**
```json
{
  "password": "oAkDp6J6=!Nj",
  "username": "admin"
}
```

**Impact:** Production ServiceNow credentials are exposed in the source code. If this repository is public or shared, attackers have full admin access to the ServiceNow instance.

**Remediation:**
1. **IMMEDIATELY** rotate the compromised password
2. Remove the credentials file from the repository and all commit history
3. Add `config/servicenow-instances.json` to `.gitignore` (already present but file was committed)
4. Use environment variables or a secrets manager (e.g., HashiCorp Vault, AWS Secrets Manager)
5. Run `git filter-branch` or `git-filter-repo` to purge from history

---

### 2. 🔴 CRITICAL: Known Vulnerable Dependencies

**Source:** `npm audit`

| Package | Severity | Vulnerability |
|---------|----------|---------------|
| `axios <=1.14.0` | **CRITICAL** | SSRF via NO_PROXY Bypass (GHSA-3p68-rc4w-qgx5), Cloud Metadata Exfiltration (GHSA-fvcv-3m26-pcqx) |
| `hono <=4.12.11` | Moderate | Path traversal, IP restriction bypass, cookie handling issues |
| `follow-redirects <=1.15.11` | Moderate | Authentication header leakage on cross-domain redirects |
| `@hono/node-server <1.19.13` | Moderate | Middleware bypass via repeated slashes |

**Remediation:**
```bash
npm audit fix
npm update axios@latest
```

---

### 3. 🔴 CRITICAL: Arbitrary Script Execution Without Validation

**Location:** [src/mcp-server-consolidated.js](src/mcp-server-consolidated.js#L561-L580)

**Vulnerable Code:**
```javascript
name: 'SN-Execute-Background-Script',
// ... no validation of script content
const result = await serviceNowClient.executeScriptViaTrigger(script, description, true);
```

**Impact:** Any user with access to the MCP server can execute arbitrary JavaScript code on the ServiceNow instance with the configured user's permissions (admin).

**Remediation:**
1. Implement script content validation/sanitization
2. Maintain an allowlist of permitted operations
3. Add role-based access control
4. Log all script executions for audit
5. Consider removing this capability entirely

---

### 4. 🔴 CRITICAL: No Authentication on HTTP Endpoints

**Location:** [src/server.js](src/server.js)

**Details:** The Express server exposes MCP endpoints without any authentication:
- `GET /mcp` - SSE connection
- `POST /mcp` - JSON-RPC messages
- `GET /health` - Health check (exposes instance URL)
- `GET /instances` - Lists all configured instances

**Impact:** Any network-accessible client can interact with the MCP server and execute operations against ServiceNow.

**Remediation:**
1. Implement API key authentication
2. Add JWT token validation
3. Use mTLS for client certificate verification
4. Restrict network access via firewall rules

---

## High Severity Findings

### 5. 🟠 HIGH: Missing HTTP Security Headers

**Location:** [src/server.js](src/server.js)

**Missing Headers:**
- `Strict-Transport-Security` (HSTS)
- `X-Content-Type-Options`
- `X-Frame-Options`
- `Content-Security-Policy`
- `X-XSS-Protection`

**Remediation:**
```javascript
import helmet from 'helmet';
app.use(helmet());
```

---

### 6. 🟠 HIGH: No CORS Protection

**Location:** [src/server.js](src/server.js)

**Details:** No CORS headers or origin validation. Any website can make requests to the server.

**Remediation:**
```javascript
import cors from 'cors';
app.use(cors({
  origin: ['https://trusted-domain.com'],
  credentials: true
}));
```

---

### 7. 🟠 HIGH: No Rate Limiting

**Location:** [src/server.js](src/server.js)

**Impact:** Server is vulnerable to DoS attacks and brute-force attempts.

**Remediation:**
```javascript
import rateLimit from 'express-rate-limit';
app.use(rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 100
}));
```

---

### 8. 🟠 HIGH: Insufficient Input Validation

**Locations:**
- [src/servicenow-client.js](src/servicenow-client.js#L70-90) - Table names not validated
- [src/mcp-server-consolidated.js](src/mcp-server-consolidated.js) - Query parameters not sanitized

**Details:**
- Table names passed directly to REST API without validation
- Query strings not sanitized before use
- `sys_id` format validated in some places but not consistently

**Remediation:**
1. Validate table names against allowlist
2. Sanitize all user-provided query parameters
3. Implement consistent `sys_id` format validation
4. Use parameterized queries where possible

---

### 9. 🟠 HIGH: Credentials in Authentication Header Logging

**Location:** [src/servicenow-client.js](src/servicenow-client.js#L41-43)

**Details:** Basic auth credentials stored in memory as base64 (easily decoded).

**Remediation:**
1. Use OAuth 2.0 tokens instead of basic auth
2. Implement credential rotation
3. Store credentials in secure memory regions

---

### 10. 🟠 HIGH: Session Management Weaknesses

**Location:** [src/server.js](src/server.js#L23)

```javascript
const sessions = {};  // In-memory session store
```

**Issues:**
- No session timeout/expiry
- No session limit per user
- Sessions persist until server restart
- No session validation

**Remediation:**
1. Implement session timeouts
2. Add session ID validation
3. Limit concurrent sessions
4. Use persistent session store with encryption

---

## Medium Severity Findings

### 11. 🟡 MEDIUM: Information Disclosure in Health Endpoint

**Location:** [src/server.js](src/server.js#L114-120)

```javascript
app.get('/health', (req, res) => {
  res.json({
    servicenow_instance: defaultInstance.url,  // Exposes internal URL
    instance_name: defaultInstance.name,
    // ...
  });
});
```

**Remediation:** Remove sensitive information from health check responses.

---

### 12. 🟡 MEDIUM: Verbose Error Messages

**Location:** Throughout the codebase

**Details:** Error messages may leak internal system information to attackers.

**Remediation:** Implement generic error responses for production.

---

### 13. 🟡 MEDIUM: Docker Security Configuration

**Location:** [Dockerfile](Dockerfile)

**Issues:**
- Runs as root (no `USER` directive)
- Copies entire config directory which may contain secrets

**Remediation:**
```dockerfile
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
```

---

## Security Architecture Recommendations

### Short-Term (0-30 days)
1. ✅ Rotate all compromised credentials immediately
2. ✅ Run `npm audit fix` to patch dependencies
3. ✅ Add authentication to all HTTP endpoints
4. ✅ Implement HTTPS/TLS

### Medium-Term (30-90 days)
1. Implement OAuth 2.0 for ServiceNow authentication
2. Add comprehensive input validation layer
3. Deploy Web Application Firewall (WAF)
4. Set up security monitoring and alerting

### Long-Term (90+ days)
1. Conduct penetration testing
2. Implement Security Information and Event Management (SIEM)
3. Establish secure software development lifecycle (SSDLC)
4. Regular security training for development team

---

## Compliance Concerns

| Standard | Status | Issue |
|----------|--------|-------|
| SOC 2 | ❌ Non-Compliant | Credentials in source code, no access controls |
| GDPR | ⚠️ At Risk | Potential for unauthorized data access |
| PCI-DSS | ❌ Non-Compliant | Missing encryption, authentication |
| ISO 27001 | ❌ Non-Compliant | Multiple control failures |

---

## Summary of Required Actions

| Priority | Finding | Action Required |
|----------|---------|-----------------|
| 🔴 P0 | Exposed Credentials | Rotate password NOW |
| 🔴 P0 | Dependency Vulnerabilities | Run `npm audit fix` |
| 🔴 P1 | No Authentication | Add API authentication |
| 🔴 P1 | Script Execution | Add validation layer |
| 🟠 P2 | Missing Security Headers | Add helmet middleware |
| 🟠 P2 | No Rate Limiting | Implement rate limiting |
| 🟠 P2 | CORS Missing | Configure CORS |
| 🟡 P3 | Docker Security | Run as non-root user |

---

## Appendix: Files Reviewed

1. `src/mcp-server-consolidated.js` - Main MCP server logic
2. `src/server.js` - Express HTTP server
3. `src/servicenow-client.js` - ServiceNow REST API client
4. `src/config-manager.js` - Configuration management
5. `src/natural-language.js` - NL query processing
6. `config/servicenow-instances.json` - Instance credentials
7. `Dockerfile` - Container configuration
8. `package.json` - Dependencies
9. `.gitignore` - Source control exclusions

---

**Report Classification:** CONFIDENTIAL  
**Distribution:** Security Team, Development Lead, IT Management
