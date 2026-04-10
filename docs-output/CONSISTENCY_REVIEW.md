# Consistency Review - Officeworks Third-Party Authentication System

## Overview

This document performs a comprehensive review of architectural consistency, design patterns, security practices, and integration patterns across all seven repositories in the third-party authentication system.

---

## 1. Architectural Consistency

### 1.1 Layered Architecture Compliance

**Assessment**: ✓ CONSISTENT

**Findings**:
- All services follow standard 3-layer architecture:
  1. Route handlers (Express/HTTP layer)
  2. Business logic (Service modules)
  3. Data access (DynamoDB/API clients)

**Details**:
```
trustedauth-service:
  ✓ routes/auth.js        → Route handlers
  ✓ lib/tpapi.js          → DB operations
  ✓ lib/userapi.js        → Upstream API client
  ✓ lib/util.js           → Utilities
  ✓ lib/logger.js         → Logging

trustedauth-profile:
  ✓ routes/customer.js    → Route handlers
  ✓ userProfileClient.js  → API client
  ✓ config.js             → Configuration

user-auth-service:
  ✓ app/src/routes/       → Route handlers
  ✓ app/src/services/     → Business logic
  ✓ CloudFormation        → Infrastructure
```

**Recommendation**: MAINTAIN - Current structure is sound

---

### 1.2 Configuration Management

**Assessment**: ✓ CONSISTENT with minor inconsistencies

**Findings**:

| Repository | Config Method | Pattern | Consistency |
|------------|----------------|---------|-------------|
| trustedauth-service | config.js | Environment-based object | ✓ Standard |
| trustedauth-profile | config.js | Environment-based object | ✓ Standard |
| trustedauth-app | Not documented | Likely hardcoded | ? Unclear |
| user-auth-service | CloudFormation + YAML | IaC approach | ✓ Modern |
| TANK | Constructor params | Runtime injection | ✓ Flexible |
| TARAS | Redux action dispatch | State management | ✓ Appropriate |
| Browser client | HTML attributes | Element-based | ✓ Unique |

**Issues Found**:
1. trustedauth-app configuration method not documented
2. Mixed configuration patterns (code-based vs IaC)
3. No centralized secrets management (hardcoded keys in config.js)

**Recommendations**:
1. Document trustedauth-app config approach
2. Migrate hardcoded secrets to AWS Secrets Manager
3. Use environment variables consistently across all services
4. Consider centralized config service

**Priority**: Medium - Works but not optimized

---

### 1.3 Port Assignment Consistency

**Assessment**: ✓ CONSISTENT

**Findings**:
```
trustedauth-app         → 3001  (Authorization UI)
trustedauth-service     → 3002  (API)
user-auth-service       → 3003  (Auth backend)
trustedauth-profile     → 3004  (Profile endpoint)
```

**Pattern**: Sequential ports (3001-3004) are:
- Easy to remember
- Avoid conflicts with common ports
- Clearly indicate service relationships
- Follow numerical ordering by function

**Recommendation**: MAINTAIN - Well-chosen pattern

---

## 2. Security Consistency

### 2.1 Authentication Mechanism

**Assessment**: ✓ CONSISTENT

**Mechanism**: HMAC-SHA512 signature-based authentication

**Implementation Consistency**:

| Component | Implementation | Algorithm | Consistency |
|-----------|----------------|-----------|-------------|
| trustedauth-service | lib/tpapi.js | HMAC-SHA512 | ✓ |
| TANK client | src/auth/client.js | HMAC-SHA512 | ✓ |
| Browser client | postMessage | N/A (iframe) | ✓ |
| TARAS | Redux state | N/A (wrapper) | ✓ |

**Verification**: All signed requests use identical algorithm
```javascript
crypto.createHmac('sha512', secret).update(data).digest('hex')
```

**Recommendation**: MAINTAIN - Consistent implementation

---

### 2.2 Token Management

**Assessment**: ✓ CONSISTENT

**Token Types**:
- **OWT** (Officeworks Token): JWT, 8-hour expiry, HTTP-only cookie
- **OTT** (One-Time Token): Random string, ~10 minute expiry, single-use

**Consistency**:

| Aspect | Implementation | Consistency |
|--------|----------------|-------------|
| OWT format | JWT | ✓ Standard |
| OWT expiry | 8 hours (configurable) | ✓ Uniform |
| OWT storage | HTTP-only cookie + header | ✓ Secure |
| OTT generation | Random string (32 chars) | ✓ Standard |
| OTT validation | Single-use, time-limited | ✓ Secure |
| OTT transport | Query parameter (in callback) | ⚠ HTTP GET only |

**Issues Found**:
1. OTT transported via query parameter (less secure than POST)
   - Could be logged in access logs
   - Could be stored in browser history
   - Should ideally use POST with body

**Recommendations**:
1. Consider POST-based token exchange
2. Use referrer-policy headers to hide OTT
3. Document security implications of current approach

**Priority**: Low - Current approach functional, but could be improved

---

### 2.3 Signature Validation

**Assessment**: ✓ CONSISTENT

**Pattern**: All server-side APIs validate signatures

```
Request Pattern:
┌─────────────────────────────────────┐
│ GET /auth/token?ott=OTT_VALUE      │
│ Headers:                            │
│   x-ow-signature: HMAC(...)        │
│   x-ow-nonce: random_value         │
│   x-ow-signing-string: data        │
└─────────────────────────────────────┘

Validation Steps (consistent across services):
1. Extract signature from header
2. Lookup party secret by apiKey
3. Recompute HMAC
4. Compare signatures
5. Check nonce (replay prevention)
```

**Missing Validation**: Guest flow doesn't require signature
- Intentional: Public endpoint, no sensitive data exposed
- Signature added by third-party server before exchange
- Design is correct

**Recommendation**: MAINTAIN - Well-designed pattern

---

### 2.4 Error Handling & Information Disclosure

**Assessment**: ⚠ INCONSISTENT - Security issue

**Findings**:

| Scenario | Response | Issue |
|----------|----------|-------|
| Invalid email format | `{err: "Invalid email"}` | ✓ Generic |
| Email not in Cognito | `{err: "Invalid credentials"}` | ✓ Generic |
| Invalid password | `{err: "Invalid credentials"}` | ✓ Generic |
| User not confirmed | `{err: "Email not confirmed"}` | ✗ Information leakage |
| Invalid apiKey | `{err: "Invalid apikey"}` | ✓ Generic |
| Invalid signature | `{err: "Invalid signature"}` | ✗ Could leak info |

**Security Issues**:
1. "Email not confirmed" reveals user exists (enumeration attack)
2. "Invalid signature" vs "Invalid apiKey" both disclose information

**Impact**: Medium - Could enable email enumeration

**Recommendations**:
1. Return generic "Authentication failed" for all credential errors
2. Log detailed failure reason server-side for auditing
3. Return same HTTP status (401) for all auth failures
4. Implement rate limiting per IP/apiKey

**Priority**: High - Should fix before using in production

---

### 2.5 HTTPS Enforcement

**Assessment**: ✓ CONSISTENT

**Findings**:
- All production endpoints use HTTPS
- Browser client enforces HTTPS (except localhost:3001 for dev)
- Config shows HTTPS URLs for test/master
- Local development allows HTTP

**Verification**: config.js domains:
```
local:   https://api-test.officeworks.com.au
test:    https://api-test.officeworks.com.au
master:  https://api.officeworks.com.au
```

**Recommendation**: MAINTAIN - Well-enforced

---

## 3. API Design Consistency

### 3.1 Endpoint Naming

**Assessment**: ✓ MOSTLY CONSISTENT

**Pattern**: RESTful with some variations

```
Consistent patterns:
  /auth/tp/*              → Trusted party CRUD
  /auth/user/*            → User admin operations
  /auth/login             → Login (POST, verb-based)
  /auth/register          → Register (PUT, verb-based)
  /auth/token/*           → Token operations
  /auth/customer/profile  → Profile (GET, noun-based)

Inconsistencies:
  POST /auth/login        → Should be PUT or POST (consistent)?
  PUT /auth/register      → Should be POST (create) not PUT?
  PUT /auth/token/guest   → Should be POST (create)?
```

**Assessment**: Minor inconsistency - not problematic

**Recommendations**:
1. Document HTTP method selection rationale
2. Consider standardizing on POST for creation
3. Not critical to change (breaking change risk)

**Priority**: Low - Works, but could be more uniform

---

### 3.2 Response Format

**Assessment**: ✓ CONSISTENT

**Standard Response Format**:
```json
{
  "field1": "value1",
  "field2": "value2"
  // ... specific to endpoint
}
```

**Error Response Format**:
```json
{
  "err": "Error message",
  "status": 400
}
```

**Consistency**:
- All services use same format
- All errors include `err` field
- HTTP status code in body matches header
- Optional request ID for tracing

**Recommendation**: MAINTAIN - Good standard

---

### 3.3 Query Parameters vs Body Parameters

**Assessment**: ⚠ INCONSISTENT

**Findings**:

| Endpoint | Input Method | Data |
|----------|--------------|------|
| POST /login | Body | Credentials |
| PUT /register | Body | Registration |
| PUT /token/guest | Body | apiKey |
| GET /token | Query | ott, apiKey |
| PUT /auth/tp | Query | name, callbackUrl |
| GET /profile | Cookie | token |

**Inconsistency**: 
- Token endpoints use query parameters (HTTP GET)
- Other creation endpoints use body (HTTP POST/PUT)

**Issue**: HTTP GET should be idempotent and have no side effects
- Token exchange (GET /auth/token) creates state (marks OTT as used)
- Violates REST principles

**Recommendation**:
1. Change `GET /auth/token` to `POST /auth/token`
2. Move query parameters to body
3. Would improve semantic correctness

**Priority**: Medium - Currently works, but not best practice

---

## 4. Data Model Consistency

### 4.1 DynamoDB Schema

**Assessment**: ✓ CONSISTENT

**Tables**:
```
TrustedParty_Api
├─ Key: PartyId (HASH)
├─ GSI: ApiKey-index
├─ Attributes: Name, ApiKey, Secret, CallbackUrl, Nonce
└─ Pattern: Straightforward configuration storage

TrustedParty_Tokens
├─ Key: (UserToken, PartyId) - Composite
├─ Attributes: PartyToken, OneTimeToken, IssueTime, ExpiryTime
└─ Pattern: Token lifecycle storage

TrustedParty_UserToken
├─ Key: UserId (HASH)
├─ Attributes: UserToken
└─ Pattern: User → token mapping
```

**Naming Convention**: PascalCase for attribute names
- PartyId, ApiKey, SecretKey, CallbackUrl, OneTimeToken
- Consistent across all tables

**TTL Strategy**: 
- No DynamoDB TTL used
- Manual expiry via ExpiryTime attribute
- Requires cleanup job (not shown in code)

**Recommendation**: Consider enabling DynamoDB TTL on ExpiryTime for automatic cleanup

**Priority**: Low - Current approach works, improvement is optimization

---

### 4.2 User Profile Schema

**Assessment**: ✓ CONSISTENT

**Profile Fields** (standardized across system):
```json
{
  "userId": "string",
  "userName": "string",
  "userType": "GUEST | PERSONAL | BUSINESS",
  "email": "string",
  "firstName": "string",
  "lastName": "string",
  "phone": "string",
  "mobile": "string",
  "custBP": "string (business partner code)",
  "orgBP": "string"
}
```

**Consistency**: Same schema across:
- User registration response
- Profile endpoint response
- JWT payload (user.id, user.type fields)
- Redux state (TARAS)

**Recommendation**: MAINTAIN - Well-designed schema

---

## 5. Logging & Monitoring Consistency

### 5.1 Logging Framework

**Assessment**: ✓ CONSISTENT

**Framework**: Winston logger (all Node.js services)

**Configuration**:
```
local:   debug
test:    debug
master:  info
```

**Logged Data** (consistent across services):
- Request ID
- HTTP method and URL
- Timestamps
- Status codes
- Errors and stack traces
- Authentication events

**Recommendation**: MAINTAIN - Good standard

---

### 5.2 Monitoring Gaps

**Assessment**: ⚠ INCOMPLETE

**Monitoring Present**:
- ✓ Logging to stdout (CloudWatch)
- ✓ Error tracking (stack traces)
- ✓ Request IDs for tracing

**Monitoring Missing**:
- ✗ Metrics (request latency, error rate)
- ✗ Alerts (threshold-based)
- ✗ Dashboards
- ✗ Performance tracking
- ✗ Token generation/validation metrics

**Recommendation**: 
1. Add CloudWatch metrics for:
   - Token generation rate
   - Token validation success/failure
   - API response times
   - Error rates by type
2. Setup alarms for:
   - High error rate
   - Service unavailability
   - Slow response times

**Priority**: Medium - Useful for production operations

---

## 6. Testing Consistency

### 6.1 Test Coverage

**Assessment**: ⚠ INCONSISTENT

| Repository | Test Files | Coverage | Type |
|------------|-----------|----------|------|
| trustedauth-service | tpapi.spec.js, userapi.spec.js, util.spec.js, auth.spec.js | Good | Unit + integration |
| trustedauth-profile | test/test.js | Minimal | Integration |
| user-auth-service | tests/component/ | Component tests | Integration |
| TANK | test/auth/client.spec.js | Moderate | Unit |
| TARAS | test suite present | Moderate | Unit + integration |
| Browser client | None visible | 0% | None |
| trustedauth-app | None visible | 0% | None |

**Issues**:
1. Browser client (authclient.js) has no visible tests
2. trustedauth-app has no visible tests
3. Test coverage percentages not documented
4. Integration test coverage varies

**Recommendations**:
1. Add integration tests for end-to-end flows
2. Implement test coverage reporting
3. Set minimum coverage thresholds (80%+)
4. Add browser client tests (Selenium/Puppeteer)
5. Add UI tests for trustedauth-app

**Priority**: Medium - Important for quality

---

### 6.2 Test Framework Consistency

**Assessment**: ✓ CONSISTENT

**Framework**: Mocha + Chai + Sinon (consistent across all services)

**Test Script**: 
```json
"test": "mocha --opts ./mocha.opts"
```

**Recommendation**: MAINTAIN

---

## 7. Dependency Management

### 7.1 Package Versions

**Assessment**: ⚠ OUTDATED

**Critical Issues**:

| Package | Current | Latest | Status |
|---------|---------|--------|--------|
| express | 4.17.1 | 4.18+ | Outdated |
| jsonwebtoken | 7.2.1 | 9.x+ | Very old |
| request | 2.79.0 | DEPRECATED | **CRITICAL** |
| body-parser | 1.16.0 | 1.20+ | Very old |
| aws-sdk v2 | 3.x+ | v3 | ✓ Modern |

**Major Problem**: `request` library is deprecated
- No longer maintained
- Security vulnerabilities may not be fixed
- Should be replaced with `axios` or native `fetch`

**Impact**: HIGH - Security risk

**Recommendations**:
1. Replace `request` with `axios` or `node-fetch`
2. Update Express to 4.18+
3. Update jsonwebtoken to 9.x
4. Audit all dependencies for vulnerabilities
5. Use `npm audit` regularly
6. Update package-lock.json

**Priority**: CRITICAL - Should be done before further development

---

### 7.2 Dependency Consistency

**Assessment**: ✓ CONSISTENT patterns

**Observations**:
- All services use consistent dependencies
- No duplicate/conflicting versions
- AWS SDK v3 consistently used
- Promise-based APIs where available

**Recommendation**: MAINTAIN patterns, but update versions

---

## 8. Documentation Consistency

### 8.1 README Files

**Assessment**: ⚠ INCONSISTENT

| Repository | README | Quality | Content |
|------------|--------|---------|---------|
| trustedauth-service | Yes | Good | Routes documented |
| trustedauth-app | Yes | Minimal | Basic setup |
| trustedauth-profile | None | N/A | Missing |
| user-auth-service | Yes | Minimal | Brief description |
| TANK | Yes | Excellent | Usage examples |
| TARAS | Yes | Excellent | Detailed setup |
| Browser client | None | N/A | None |

**Issues**:
1. No README for trustedauth-profile
2. No README for browser client
3. Inconsistent documentation depth
4. No overall system documentation (now provided!)

**Recommendation**: 
1. Add README to missing repositories
2. Create master README explaining system architecture
3. Link all READMEs together
4. Standardize documentation structure

**Priority**: Medium - Improves onboarding

---

### 8.2 Code Comments

**Assessment**: ⚠ MODERATE

**Observation**:
- Good JSDoc comments in service modules (tpapi.js)
- Route handlers have some inline comments
- Utility functions well-documented
- Some complex flows lack explanation

**Recommendation**:
1. Add JSDoc to all public functions
2. Document complex algorithms (ABN validation, HMAC flow)
3. Add comments explaining DynamoDB operations
4. Document error handling rationale

**Priority**: Low - Code is readable, more comments would help

---

## 9. Deployment Consistency

### 9.1 Infrastructure Patterns

**Assessment**: ✓ MOSTLY CONSISTENT

| Service | Platform | Deployment | Pattern |
|---------|----------|-----------|---------|
| trustedauth-service | Elastic Beanstalk | .ebextensions | ✓ Standard |
| trustedauth-app | Elastic Beanstalk | .ebextensions | ✓ Standard |
| trustedauth-profile | Elastic Beanstalk | .ebextensions | ✓ Standard |
| user-auth-service | ECS | CloudFormation | ✓ Modern |

**Consistency**: 
- All Node.js services on AWS
- Consistent region (ap-southeast-2)
- Standard port assignments
- Environment-based configuration

**Difference**: user-auth-service uses ECS (containerized) vs Elastic Beanstalk (application platform)
- Not necessarily inconsistent - could be intentional
- Recommend documenting rationale

**Recommendation**:
1. Document why user-auth-service uses ECS vs Beanstalk
2. Consider standardizing if possible
3. Could move all to ECS for consistency

**Priority**: Low - Works as-is

---

### 9.2 Configuration Management in Deployment

**Assessment**: ✓ CONSISTENT

**Approach**: Environment variables + config files
```
config.js (local/test/master)
Environment variables (NODE_ENV, PORT, etc.)
CloudFormation parameters (user-auth-service)
```

**Recommendation**: MAINTAIN - Well-structured

---

## 10. Security Review Findings Summary

### Critical Issues
1. **Outdated `request` library** - MUST FIX
   - Deprecated since 2020
   - No security updates
   - Replace immediately

2. **Error message information leakage** - SHOULD FIX
   - Can enumerate users
   - Reveals account status
   - Return generic errors

### High Priority
3. **OTT via query parameter** - SHOULD IMPROVE
   - Less secure than POST body
   - Could be logged/cached
   - Consider POST-based exchange

4. **No metrics/monitoring** - SHOULD ADD
   - Hard to debug production issues
   - No visibility into system health
   - Add CloudWatch metrics

### Medium Priority
5. **Hardcoded secrets in config.js** - SHOULD IMPROVE
   - Visible in source code
   - Visible in git history
   - Use AWS Secrets Manager

6. **Missing tests** - SHOULD COMPLETE
   - Browser client untested
   - trustedauth-app untested
   - Add test coverage

### Low Priority
7. **Documentation** - COULD IMPROVE
   - Some components under-documented
   - README files missing
   - Architecture documentation lacking (now provided)

---

## 11. Consistency Score Card

| Category | Score | Status |
|----------|-------|--------|
| Architectural Consistency | 9/10 | Excellent |
| Security Implementation | 7/10 | Good (update needed) |
| API Design | 8/10 | Good (minor improvements) |
| Data Models | 9/10 | Excellent |
| Logging | 8/10 | Good |
| Testing | 6/10 | Needs improvement |
| Dependencies | 4/10 | Critical updates needed |
| Documentation | 6/10 | Needs improvement |
| Deployment | 8/10 | Good |
| **Overall** | **7/10** | **Good, needs updates** |

---

## 12. Recommendations Summary

### Immediate (Critical)
- [ ] Update `request` library to axios/fetch
- [ ] Fix information disclosure in error messages
- [ ] Run `npm audit` and fix vulnerabilities

### Short Term (High Priority)
- [ ] Add monitoring/metrics
- [ ] Move secrets to AWS Secrets Manager
- [ ] Add missing tests
- [ ] Update outdated dependencies

### Medium Term (Nice to Have)
- [ ] Change OTT exchange to POST
- [ ] Improve documentation
- [ ] Add more integration tests
- [ ] Standardize deployment platform

### Long Term (Planning)
- [ ] Plan migration to modern OAuth 2.0
- [ ] Consider deprecation timeline (already planned for Jan 2026)
- [ ] Document lessons learned for next-gen auth system

---

## Conclusion

The Officeworks Third-Party Authentication System demonstrates **good architectural consistency** with **sound design patterns**. However, it suffers from **outdated dependencies** (particularly the deprecated `request` library) and **missing monitoring/testing infrastructure**.

**Key Strengths**:
- Consistent layered architecture
- Uniform security implementation (HMAC-SHA512)
- Well-designed token lifecycle
- Clear separation of concerns

**Key Weaknesses**:
- Outdated npm packages
- Incomplete test coverage
- Missing monitoring
- Error message information leakage

**Overall Assessment**: System is functional and well-designed, but requires **immediate security updates** before handling sensitive authentication data in production.

**Deprecation Note**: As system is scheduled for January 2026 decommissioning, fix critical issues but avoid major architectural changes. Plan migration to modern OAuth 2.0 solution.

---

**Review Date**: April 2026
**Reviewed By**: Architecture & Security Review
**Status**: Complete
**Deprecation Status**: System decommissioning January 2026
