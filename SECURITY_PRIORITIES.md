# Security Priorities & Action Items

## 🚨 CRITICAL - Fix Immediately

### 1. Command Injection Vulnerability
**File:** `backend/src/ee/services/secret-scanning-v2/secret-scanning-v2-fns.ts`
**Lines:** 36, 50, 63

**Current Code:**
```typescript
const command = `git clone ${cloneUrl} ${repoPath} --bare`;
exec(command, (error) => { ... });
```

**Fix Required:**
```typescript
import { execFile } from 'child_process';
import { promisify } from 'util';

const execFileAsync = promisify(execFile);

// Instead of:
// const command = `git clone ${cloneUrl} ${repoPath} --bare`;
// exec(command, ...);

// Use:
await execFileAsync('git', ['clone', cloneUrl, repoPath, '--bare']);
```

**Impact:** Prevents arbitrary code execution via command injection

---

## ⚠️ HIGH PRIORITY - Address Within 1-2 Sprints

### 2. Fix Race Conditions in Critical Services

#### KMS Service
**File:** `backend/src/services/kms/kms-service.ts:1090`
- Using fixed UUID to avoid race conditions is a workaround, not a solution
- Implement proper database locking
- Add transaction isolation level checks

#### SSE Event Stream
**File:** `backend/src/ee/services/event/event-sse-stream.ts:119`
```typescript
const ping = async () => {
  if (!auth) return; // Avoid race condition if ping is called before open
  // ...
}
```
- Add proper connection state management
- Use mutex or semaphore for critical sections
- Ensure authentication is completed before allowing operations

### 3. Atomic Concurrency Control
**Files:** 
- `backend/src/services/secret-sync/secret-sync-queue.ts`
- `backend/src/services/pki-sync/pki-sync-queue.ts`

**Current Issue:**
```typescript
const currentCount = Number.parseInt(concurrencyCount || "0", 10);
const incrementedCount = Number.isNaN(currentCount) ? 1 : currentCount + 1;
await keyStore.setItemWithExpiry(..., incrementedCount);
```

**Fix Required:**
```typescript
// Use Redis INCR for atomic operations
const count = await redis.incr(`${prefix}:${connectionId}`);
await redis.expire(`${prefix}:${connectionId}`, ttlSeconds);
```

---

## 📋 MEDIUM PRIORITY - Address Within Quarter

### 4. Fix Secret Override Bug
**File:** `backend/src/services/secret-v2-bridge/secret-v2-bridge-dal.ts:451`

**Action Items:**
- [ ] Fix secret move operation to properly handle overrides
- [ ] Re-enable override counting
- [ ] Add integration tests for secret moves
- [ ] Verify data consistency across environments

### 5. Implement Certificate Validation
**File:** `backend/src/services/certificate-authority/internal/internal-certificate-authority-service.ts:1105`

**Action Items:**
- [ ] Validate that latest CA key-pair is used for signing
- [ ] Add key rotation handling
- [ ] Implement key usage auditing
- [ ] Add alerts for key mismatches

### 6. Fix Hardcoded Expiry Dates
**File:** `backend/src/services/certificate-authority/internal/internal-certificate-authority-service.ts:300`

**Current:**
```typescript
nextUpdate: new Date("2025/12/12"), // TODO: change
```

**Fix:**
```typescript
nextUpdate: new Date(Date.now() + CRL_UPDATE_INTERVAL_MS)
```

### 7. Security TODOs Resolution

**File:** `backend/src/services/auth/auth-signup-service.ts:83,117`
- [ ] Fix user enumeration vulnerability
- [ ] Implement consistent error messages
- [ ] Review authentication flow for timing attacks

---

## 🔒 SECURITY HARDENING

### 8. Admin SSO Bypass Review
**File:** `backend/src/services/auth/auth-login-service.ts`

**Recommendations:**
- [ ] Document security implications of bypass feature
- [ ] Require MFA for bypass actions
- [ ] Implement time-limited bypass tokens
- [ ] Add comprehensive audit logging
- [ ] Review compliance requirements (SOC2, ISO 27001)
- [ ] Consider removal if not essential

### 9. Rate Limiting Implementation
**File:** `backend/src/services/secret-sync/github/github-sync-fns.ts:20`

**Action Items:**
- [ ] Implement exponential backoff
- [ ] Monitor rate limit headers
- [ ] Add request queuing
- [ ] Implement circuit breaker pattern
- [ ] Add metrics for rate limit hits

---

## 🧹 TECHNICAL DEBT CLEANUP

### 10. Remove Deprecated Code
**Priority:** Low but important

**Action Items:**
- [ ] Create migration timeline for deprecated endpoints
- [ ] Notify users of deprecation schedule
- [ ] Remove V1 secret scanning code
- [ ] Remove all deprecated routers after migration period
- [ ] Update API documentation

**Files to Remove:**
- `deprecated-secret-router.ts`
- `deprecated-project-router.ts`
- `deprecated-group-project-router.ts`
- `deprecated-project-membership-router.ts`
- Secret Scanning V1 (`secret-scanning-queue.ts`)

---

## 📈 MONITORING & OBSERVABILITY

### 11. Add Security Monitoring
- [ ] Monitor for command injection attempts
- [ ] Alert on Admin SSO bypass usage
- [ ] Track race condition indicators
- [ ] Monitor concurrency limit violations
- [ ] Alert on legacy auth attempts

### 12. Implement Security Metrics
- [ ] Track authentication failures
- [ ] Monitor certificate validation failures
- [ ] Track API rate limit hits
- [ ] Monitor deprecated endpoint usage
- [ ] Alert on security-related errors

---

## 🔄 PROCESS IMPROVEMENTS

### 13. Security Review Process
- [ ] Implement mandatory security review for auth changes
- [ ] Add SAST (Static Application Security Testing) to CI/CD
- [ ] Regular dependency vulnerability scanning
- [ ] Quarterly security audit schedule
- [ ] Penetration testing for critical features

### 14. Code Quality Standards
- [ ] Enforce TypeScript strict mode
- [ ] Require error handling patterns
- [ ] Mandate input validation
- [ ] Document security considerations
- [ ] Code review checklist for security issues

---

## 📊 TESTING REQUIREMENTS

### Critical Tests Needed:
1. **Command Injection Tests**
   - Test with malicious input in clone URLs
   - Test with special characters in paths
   - Verify input sanitization

2. **Race Condition Tests**
   - Concurrent KMS operations
   - Parallel SSE connections
   - Concurrent secret sync operations

3. **Concurrency Control Tests**
   - Test beyond concurrency limits
   - Verify atomic operations
   - Test edge cases (NaN, undefined)

4. **Security Tests**
   - User enumeration attempts
   - Timing attack tests
   - Authentication bypass attempts
   - Permission escalation tests

---

## 🎯 SUCCESS METRICS

### Short Term (1 Month)
- [ ] Command injection vulnerability fixed and tested
- [ ] Race conditions in critical paths resolved
- [ ] Atomic concurrency control implemented
- [ ] Security monitoring in place

### Medium Term (3 Months)
- [ ] All high-priority security issues resolved
- [ ] Deprecated code removed
- [ ] Comprehensive test coverage for security issues
- [ ] Security documentation updated

### Long Term (6 Months)
- [ ] All TODOs with security implications resolved
- [ ] Regular security audit process established
- [ ] Zero critical vulnerabilities
- [ ] Security-first development culture

---

## 📝 INCIDENT RESPONSE PLAN

### If Command Injection is Exploited:
1. Immediately disable secret scanning features
2. Review audit logs for suspicious activity
3. Notify affected customers
4. Deploy emergency patch
5. Conduct security review
6. Update security documentation

### If Race Condition is Exploited:
1. Enable database query logging
2. Review transaction logs
3. Verify data integrity
4. Deploy fix with database migration if needed
5. Monitor for anomalies

---

## 👥 RESPONSIBILITY MATRIX

| Issue | Owner | Reviewer | Timeline |
|-------|-------|----------|----------|
| Command Injection | Security Team | Senior Dev | Immediate |
| Race Conditions | Backend Lead | Architect | 2 weeks |
| Concurrency Control | Backend Dev | DevOps | 2 weeks |
| Secret Override Bug | Product Team | QA Lead | 1 month |
| Certificate Validation | Security Team | PKI Expert | 1 month |
| Admin SSO Review | Product + Security | Compliance | 1 month |
| Deprecated Code | Tech Lead | Product | 3 months |

---

**Last Updated:** 2025-10-08
**Next Review:** 2025-11-08
