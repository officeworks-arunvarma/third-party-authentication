# Functional Descriptions - Officeworks Third-Party Authentication System

## System Purpose

The Officeworks Third-Party Authentication System is an OAuth-like authentication platform that enables third-party and non-core applications to authenticate Officeworks customers. It provides a secure mechanism for partner applications to issue, validate, and manage authentication tokens without directly handling customer credentials.

**Status**: DEPRECATED - Scheduled for decommissioning January 2026

---

## Core Functions

### 1. Trusted Party Registration & Management

**Functional Area**: `trustedauth-service` - Routes `/auth/tp`

**Purpose**: Manage third-party applications that integrate with the authentication system.

**Functions**:

#### 1.1 Register Trusted Party
- **Operation**: `POST /auth/tp` or `POST /auth/tp/:tpId`
- **Required Admin Key**: `X-OW-ADMIN-KEY` header
- **Input Parameters**:
  - `name` (string): Display name of the third-party
  - `callbackUrl` (string): OAuth callback endpoint
  - `nonce` (string): Security nonce
  - `tpId` (optional): Specific party ID (auto-generated if omitted)
  
- **Output**:
  ```json
  {
    "partyId": "90003",
    "name": "accis",
    "apiKey": "Izj5SZEe8b7L4vxG01N0",
    "secretKey": "OIHMFfAv24sInyNd6EOdzrVTRMxOtct8QXSOUV18",
    "callbackUrl": "http://www.owt.com",
    "nonce": "393939393939",
    "tpClientId": "90003"
  }
  ```

- **Side Effects**:
  - Creates record in `TrustedParty_Api` DynamoDB table
  - Generates unique `apiKey` (20 chars)
  - Generates `secretKey` for HMAC-SHA512 signing (38 chars)
  - Increments party counter

- **Use Case**: Web Wizards team registers a new third-party customer who wants to integrate authentication

---

#### 1.2 Retrieve Trusted Party Configuration
- **Operation**: `GET /auth/tp/:tpId` or `GET /auth/tp/apiKey/:apiKey`
- **Required Admin Key**: `X-OW-ADMIN-KEY` header
- **Input Parameters**:
  - `tpId`: Party identifier OR
  - `apiKey`: Party's public API key

- **Output**: Same as registration response

- **Lookup Process**:
  - By ID: Direct DynamoDB GET on primary key
  - By apiKey: Query GSI `ApiKey-index`

- **Use Case**: Verify that a third-party's credentials are correct; retrieve callback URL for security validation

---

#### 1.3 Update Trusted Party
- **Operation**: `PUT /auth/tp/:tpId`
- **Required Admin Key**: `X-OW-ADMIN-KEY` header
- **Input Parameters**:
  - `callbackUrl` (query param, optional): New callback endpoint
  - `name` (query param, optional): Updated name
  - `nonce` (query param, optional): Updated nonce

- **Output**: Updated party object

- **Side Effects**:
  - Updates DynamoDB record
  - Changes take effect immediately for new tokens

- **Use Case**: Third-party changes their callback URL after domain migration

---

#### 1.4 Delete Trusted Party
- **Operation**: `DELETE /auth/tp/:tpId`
- **Required Admin Key**: `X-OW-ADMIN-KEY` header
- **Output**: `{deleted: true}`

- **Side Effects**:
  - Removes from `TrustedParty_Api` table
  - Existing tokens remain valid until expiry
  - New token requests fail with "Invalid apiKey"

- **Use Case**: Revoke access for a third-party that no longer needs integration

---

### 2. Guest Authentication

**Functional Area**: `trustedauth-service` - Route `PUT /auth/token/guest`

**Purpose**: Enable third-party applications to authenticate anonymous/guest users without account creation.

**Functions**:

#### 2.1 Request Guest Token
- **Operation**: `PUT /auth/token/guest`
- **Input Parameters**:
  - `apiKey` (query or body): Third-party's API key
  - `signature` (header): HMAC-SHA512 signature (optional for guest flow)
  - `nonce` (header): Random nonce

- **Processing Steps**:
  1. Validate `apiKey` - lookup in DynamoDB
  2. Call user-auth-service: `GET /auth/guest`
  3. Receive `userToken` from upstream
  4. Create JWT payload:
     ```json
     {
       "user": {
         "id": "guest-token-abc123",
         "type": "GUEST"
       },
       "iat": 1696000000,
       "exp": 1696028800
     }
     ```
  5. Generate One-Time Token (OTT) - 32-char random string
  6. Store in DynamoDB `TrustedParty_Tokens`
  7. Return OTT + OWT + auth tokens

- **Output**:
  ```json
  {
    "owt": "eyJhbGciOiJIUzUxMiIsInR5cCI6IkpXVCJ9...",
    "ott": "abc123def456ghi789jkl012mno345pqr",
    "authTokens": {
      "wc_auth": "...",
      "wc_session": "..."
    },
    "user": {
      "id": "guest-token-abc123",
      "type": "GUEST"
    }
  }
  ```

- **HTTP Status**: 201 Created

- **Use Case**: Third-party app displays "Continue as Guest" option; user clicks button without logging in

---

### 3. Account Registration

**Functional Area**: `trustedauth-service` - Routes `/auth/register` and `/auth/register/business`

**Purpose**: Enable third-party applications to register new user accounts (personal or business) without exposing account creation to the internet.

**Functions**:

#### 3.1 Register Personal Account
- **Operation**: `PUT /auth/register`
- **Input Parameters**:
  ```json
  {
    "apiKey": "Izj5SZEe8b7L4vxG01N0",
    "email": "user@example.com",
    "password": "SecurePassword123!",
    "firstName": "John",
    "lastName": "Doe",
    "phone": "0401111222"
  }
  ```

- **Processing Steps**:
  1. Validate `apiKey`
  2. Validate email format (RFC 5322)
  3. Validate password strength (configurable requirements)
  4. Call user-auth-service:
     ```
     POST /auth/register/personal
     Body: {email, password, firstName, lastName, phone}
     ```
  5. Receive `userToken` from Cognito
  6. Create JWT with `PERSONAL` type
  7. Generate OTT + OWT
  8. Store in DynamoDB
  9. Return tokens

- **Output**: Same as guest token (with PERSONAL user type)

- **HTTP Status**: 201 Created

- **Side Effects**:
  - Creates user in AWS Cognito user pool
  - Sends confirmation email
  - User account ready for immediate login

- **Use Case**: Third-party app's "Sign Up" form; user completes registration and is immediately logged in

---

#### 3.2 Register Business Account
- **Operation**: `PUT /auth/register/business`
- **Input Parameters**:
  ```json
  {
    "apiKey": "Izj5SZEe8b7L4vxG01N0",
    "companyName": "ACME Corp",
    "abn": "12345678901",
    "email": "business@acmecorp.com",
    "password": "SecurePassword123!",
    "contactName": "John Doe"
  }
  ```

- **Processing Steps**:
  1. Validate `apiKey`
  2. Validate ABN format:
     - Must be exactly 11 digits
     - Must pass Australian Business Number checksum algorithm
  3. Validate email and password
  4. Call user-auth-service:
     ```
     POST /auth/register/business
     Body: {companyName, abn, email, password, contactName}
     ```
  5. Cognito creates account with `userType: BUSINESS`
  6. Store business details linked to account
  7. Generate JWT with `BUSINESS` type
  8. Return OTT + OWT + auth tokens

- **Output**: Same format, with `user.type = "BUSINESS"`

- **HTTP Status**: 201 Created

- **Validation**: ABN checksum algorithm
  ```
  weights = [10, 1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
  sum = weight[0]*(digit[0]-1) + weight[1]*digit[1] + ... + weight[10]*digit[10]
  Valid if sum % 89 == 0
  ```

- **Use Case**: Third-party's "Business Registration"; B2B customer creates account; gains access to business features

---

### 4. User Login

**Functional Area**: `trustedauth-service` - Route `POST /auth/login`

**Purpose**: Authenticate existing users with email/password credentials.

**Functions**:

#### 4.1 Login with Credentials
- **Operation**: `POST /auth/login`
- **Input Parameters**:
  ```json
  {
    "apiKey": "Izj5SZEe8b7L4vxG01N0",
    "email": "user@example.com",
    "password": "SecurePassword123!"
  }
  ```

- **Validation Steps**:
  1. Validate `apiKey`
  2. Validate email format
  3. Check password not empty
  4. Call user-auth-service:
     ```
     POST /auth/auth
     Body: {email, password}
     ```

- **Cognito Validation**:
  - Lookup user in pool
  - Hash provided password
  - Compare with stored hash
  - Check if user confirmed email
  - Verify account not suspended
  - Check if MFA enabled and validate code

- **On Success**:
  1. Receive `userToken` from Cognito
  2. Fetch auth tokens (cookies)
  3. Determine user type (PERSONAL or BUSINESS)
  4. Create JWT with user type
  5. Generate OTT + OWT
  6. Store in DynamoDB
  7. Return tokens
  8. **Log successful login** (audit trail)

- **Output**:
  ```json
  {
    "owt": "JWT_TOKEN",
    "ott": "ONE_TIME_TOKEN",
    "authTokens": {...},
    "user": {
      "id": "customer-123",
      "type": "PERSONAL"
    }
  }
  ```

- **HTTP Status**: 200 OK

- **On Failure** (common scenarios):

  | Scenario | Status | Response |
  |----------|--------|----------|
  | Email not found | 401 | `{err: "Invalid credentials"}` |
  | Password incorrect | 401 | `{err: "Invalid credentials"}` |
  | Email not confirmed | 403 | `{err: "Email not confirmed"}` |
  | Account suspended | 403 | `{err: "Account suspended"}` |
  | MFA required | 422 | `{err: "MFA required", mfaChallenge: {...}}` |
  | Too many attempts | 429 | `{err: "Too many login attempts"}` |

- **Use Case**: User enters credentials in third-party app's login form; system verifies against Cognito; returns tokens for session establishment

---

### 5. Token Validation

**Functional Area**: `trustedauth-service` - Route `POST /auth/token/validate`

**Purpose**: Validate that a token is currently valid and not expired.

**Functions**:

#### 5.1 Validate Token
- **Operation**: `POST /auth/token/validate`
- **Input Parameters**:
  - `x-owt` header: Officeworks Token to validate
  - OR `x-ott` header: One-Time Token to validate

- **Processing**:
  1. Extract token from headers
  2. Determine token type (OWT or OTT)
  3. Parse JWT structure
  4. Verify signature with public key
  5. Check `exp` claim vs current time
  6. If OTT: lookup in DynamoDB and verify not used
  7. Return validation result

- **Output**:
  ```json
  {
    "valid": true,
    "userId": "customer-123",
    "userType": "PERSONAL",
    "issuedAt": 1696000000,
    "expiresAt": 1696028800,
    "remainingSeconds": 28000
  }
  ```

- **HTTP Status**: 200 OK (always; success/failure in response body)

- **Use Cases**:
  - Third-party app periodically validates user's token
  - Service-to-service token validation
  - Before granting sensitive operations

---

#### 5.2 Exchange One-Time Token for OWT
- **Operation**: `GET /auth/token`
- **Input Parameters**:
  - `ott` (query): One-time token to exchange
  - `apiKey` (query): Trusted party's API key
  - HMAC-SHA512 signature headers

- **Processing**:
  1. Extract `ott` and `apiKey`
  2. Validate HMAC signature
  3. Lookup party in DynamoDB
  4. Lookup OTT in `TrustedParty_Tokens`
  5. Verify OTT:
     - Not expired
     - Not previously used
     - Matches requesting party
  6. Mark OTT as used (set flag in DynamoDB)
  7. Retrieve corresponding OWT
  8. Return OWT

- **Output**:
  ```json
  {
    "owt": "JWT_TOKEN"
  }
  ```

- **HTTP Status**: 200 OK on success, 401 on failure

- **Security Notes**:
  - OTT single-use only
  - Signature required (prevents token stealing)
  - Request must come from party's server (signature requires secret)
  - Not accessible from client-side (requires secret)

- **Use Case**: Third-party backend receives OTT from callback URL; exchanges it for OWT to set as HTTP-only cookie

---

### 6. Session Keepalive

**Functional Area**: `trustedauth-service` - Route `POST /auth/keepalive`

**Purpose**: Extend user session when the user is actively using the application.

**Functions**:

#### 6.1 Refresh Session
- **Operation**: `POST /auth/keepalive`
- **Input Parameters**:
  - `x-owt` header: Current token
  - Optional body parameters

- **Processing**:
  1. Validate OWT signature
  2. Check if still valid (not expired)
  3. Update DynamoDB expiry:
     ```
     UPDATE TrustedParty_Tokens
     SET ExpiryTime = now + 8 hours
     WHERE UserToken = userToken AND PartyId = partyId
     ```
  4. Generate new OWT (optional, can reuse same token)
  5. Return updated token

- **Output**:
  ```json
  {
    "owt": "JWT_TOKEN",
    "expiresIn": 28800,
    "expiresAt": 1696028800
  }
  ```

- **HTTP Status**: 200 OK

- **Behavior**:
  - Prevents session timeout for active users
  - Typical polling interval: every 30 minutes
  - Extends session by full 8-hour window

- **Use Case**: Third-party app has JavaScript that periodically calls `/keepalive` when user is active; keeps session alive during long browsing sessions

---

### 7. User Profile Retrieval

**Functional Area**: `trustedauth-profile` - Route `GET /auth/customer/profile`

**Purpose**: Retrieve authenticated user's profile information.

**Functions**:

#### 7.1 Get User Profile
- **Operation**: `GET /auth/customer/profile`
- **Input Parameters**:
  - Cookie: `owt=JWT_TOKEN`
  - OR Header: `x-owt: JWT_TOKEN`

- **Processing**:
  1. Extract OWT from cookie or header
  2. Validate JWT signature
  3. Check expiry
  4. Extract `userId` from JWT
  5. Call upstream profile API:
     ```
     GET /v1/customers/account?userId=customer-123
     ```
  6. Transform response to standard format
  7. Return profile

- **Output**:
  ```json
  {
    "userId": "20786849",
    "userName": "John Doe",
    "userType": "BUSINESS",
    "email": "john@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "phone": "0401111222",
    "mobile": "+61401111222",
    "custBP": "20786849",
    "orgBP": "20786848",
    "hasThirtyDayAccount": true
  }
  ```

- **HTTP Status**: 200 OK on success, 401 on invalid token

- **Caching** (optional):
  - Cache profile for 5 minutes
  - Reduce upstream API calls
  - Invalidate on token refresh

- **Use Cases**:
  - Third-party app displays "Logged in as: John Doe"
  - Load user preferences from profile
  - Restrict features based on `userType`
  - Verify user's business ID for B2B operations

---

### 8. Admin Operations

**Functional Area**: `trustedauth-service` - Route `/auth/user`

**Purpose**: Admin-only operations for token and credential management.

**Functions**:

#### 8.1 Get User Token from OWT
- **Operation**: `GET /auth/user/token`
- **Required Headers**:
  - `X-OW-AGENTTOKEN`: Agent credentials
  - `X-OW-ADMIN-KEY`: Admin API key
  - Optional: Signature headers

- **Input Parameters**:
  - Cookie: `owt=JWT_TOKEN`
  - OR Header: `x-owt: JWT_TOKEN`

- **Output**:
  ```json
  {
    "userToken": "upstream-user-token-abc123"
  }
  ```

- **Use Case**: Internal tool needs to call upstream APIs on behalf of authenticated user

---

#### 8.2 Get WC Cookies from OWT
- **Operation**: `GET /auth/user/cookies`
- **Required Headers**:
  - `X-OW-AGENTTOKEN`: Agent credentials
  - `X-OW-ADMIN-KEY`: Admin API key

- **Input Parameters**:
  - Cookie: `owt=JWT_TOKEN`

- **Output**:
  ```json
  {
    "wc_auth": "...",
    "wc_session": "...",
    "wc_other": "..."
  }
  ```

- **Use Case**: Legacy systems needing web client cookies for authenticated user

---

## Cross-Cutting Concerns

### Logging & Monitoring

**Implemented in**: All services via Winston logger

**Logged Events**:
- Request ID (unique per request)
- HTTP method and URL
- Query parameters
- Request headers (sanitized)
- Response status code
- Processing time
- Errors and stack traces
- Authentication events (success/failure)
- Token operations (generation, validation)

**Log Levels**:
- `debug`: Development, detailed tracing
- `info`: Production, high-level operations
- `warn`: Potential issues
- `error`: Failures, exceptions

**Audit Trail**:
- All trusted party operations logged
- All login attempts logged
- Token generation logged
- Admin operations logged

---

### Security Features

**HMAC-SHA512 Signature Validation**:
- All requests from trusted parties include signature
- Prevents token tampering
- Prevents man-in-the-middle attacks
- Replay attack prevention via nonce

**Nonce Management**:
- Random value generated per request
- Prevents replay of captured requests
- Stored briefly to detect duplicates

**Token Expiry**:
- OWT expires in 8 hours (configurable)
- OTT single-use, valid for ~10 minutes
- Expired tokens rejected with 401

**HTTP-Only Cookies**:
- OWT stored in HTTP-only cookie
- Prevents JavaScript access
- Requires `Secure` flag for HTTPS
- Prevents XSS token theft

**Rate Limiting**:
- Too many failed login attempts → 429
- Temporary account lockout
- Configurable threshold

---

### Error Handling

**Standard Error Response Format**:
```json
{
  "err": "Error description",
  "status": 401,
  "requestId": "req-12345"
}
```

**HTTP Status Codes**:
- `200 OK`: Successful operation
- `201 Created`: Resource created (guest/register/login)
- `400 Bad Request`: Invalid parameters
- `401 Unauthorized`: Authentication failed
- `403 Forbidden`: Operation not permitted
- `422 Unprocessable Entity`: MFA required or similar
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error
- `503 Service Unavailable`: Downstream service unavailable

---

## Integration Points

### External Services

| Service | Usage | When |
|---------|-------|------|
| AWS Cognito | User credential validation | Login, registration |
| Officeworks API Gateway | User profile, account info | Profile endpoint |
| AWS DynamoDB | Token & party storage | All operations |
| AWS SQS | Async operations (optional) | Email, logging |
| AWS CloudWatch | Monitoring, logs | Continuous |

### Third-Party Integration Methods

| Method | Use Case | Security |
|--------|----------|----------|
| Browser client (authclient.js) | Simple integration, no backend | postMessage same-origin policy |
| Node.js client (TANK) | Server-side integration | HMAC signature |
| React library (TARAS) | React apps | Redux state management |

---

## Configuration Management

**Environment-Specific Config** (`config.js`):
```
local, test, master
├─ Logging level
├─ DynamoDB table names
├─ Endpoint URLs
├─ Token expiry times
├─ Request timeout
├─ Public/admin keys
└─ Feature flags
```

**Runtime Configuration** (Environment variables):
```
NODE_ENV=master|test|local
PORT=3002 (or 3001, 3004)
AWS_REGION=ap-southeast-2
LOG_LEVEL=debug|info|warn|error
```

---

## Deprecation Timeline

| Date | Action |
|------|--------|
| Now | New integrations discouraged |
| Dec 2025 | Final deprecation notice |
| Jan 2026 | Service decommissioned |
| Q2 2026 | Archives created |

**Migration Path**: Contact Web Wizards team for modern OAuth 2.0 solution

---

## Functional Summary Table

| Function | Endpoint | Method | Input | Output | Status |
|----------|----------|--------|-------|--------|--------|
| Register TP | /auth/tp/:id | POST | party details | apiKey, secret | Active |
| Get TP | /auth/tp/:id | GET | - | party object | Active |
| Update TP | /auth/tp/:id | PUT | new details | party object | Active |
| Delete TP | /auth/tp/:id | DELETE | - | {deleted:true} | Active |
| Guest token | /auth/token/guest | PUT | apiKey | OWT, OTT | Active |
| Register | /auth/register | PUT | credentials | OWT, OTT | Active |
| Register Biz | /auth/register/business | PUT | biz details | OWT, OTT | Active |
| Login | /auth/login | POST | email, password | OWT, OTT | Active |
| Validate | /auth/token/validate | POST | token | valid:bool | Active |
| Exchange | /auth/token | GET | ott, signature | OWT | Active |
| Keepalive | /auth/keepalive | POST | OWT | new OWT | Active |
| Profile | /auth/customer/profile | GET | OWT | user profile | Active |

---

**Document Status**: Complete - All major functions documented
**Last Updated**: April 2026
**Deprecation Status**: System scheduled for January 2026 decommissioning
