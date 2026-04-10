# Authentication Flows - Officeworks Third-Party Authentication System

## Overview

This document describes the detailed flows for each authentication scenario supported by the system, including message sequences, state transitions, and error handling.

---

## 1. Guest Token Flow

### Sequence Diagram

```
Third-Party App          Browser           TrustedAuth App      TrustedAuth Service    User Auth Service
      │                    │                     │                      │                    │
      ├─ 1. User clicks    │                     │                      │                    │
      │    "Continue as    │                     │                      │                    │
      │    Guest"          │                     │                      │                    │
      │                    │                     │                      │                    │
      ├─ 2. Redirect to   │                     │                      │                    │
      │    /auth/authorise │                     │                      │                    │
      │    ?apiKey=XXX     │                     │                      │                    │
      │    &target=guest   │                     │                      │                    │
      │                    │                     │                      │                    │
      │                    ├─ 3. GET /auth/authorise ────────────────────────────────────>│
      │                    │                     │                      │                    │
      │                    │    4. Render guest │                      │                    │
      │                    │    form, click     │                      │                    │
      │                    │    "Continue"      │                      │                    │
      │                    │                    │                      │                    │
      │                    │    5. PUT /auth/token/guest ────────────────────────────────>│
      │                    │       {apiKey, ...}│                      │                    │
      │                    │                    │     6. Call         │                    │
      │                    │                    │    guestToken()     │                    │
      │                    │                    │                     ├───────────────────>│
      │                    │                    │                     │  GET /auth/guest   │
      │                    │                    │                     │  Returns: userToken│
      │                    │                    │<────────────────────┤                    │
      │                    │                    │     7. userToken    │                    │
      │                    │                    │                     │                    │
      │                    │                    │   8. Create JWT     │                    │
      │                    │                    │   payload:          │                    │
      │                    │                    │   {user: {          │                    │
      │                    │                    │     id: userToken,  │                    │
      │                    │                    │     type: GUEST     │                    │
      │                    │                    │   }}                │                    │
      │                    │                    │                     │                    │
      │                    │                    │   9. Generate OTT   │                    │
      │                    │                    │      & OWT           │                    │
      │                    │                    │   10. Store in DB   │                    │
      │                    │                    │                     │                    │
      │                    │<─ 11. Return {ott, owt, ...} ─────────────────────────────────│
      │                    │                    │                     │                    │
      │<─ 12. Redirect to callback ───────────────────────────────────────────────────────│
      │    ?ott=XXX        │                    │                     │                    │
      │                    │                    │                     │                    │
      ├─ 13. POST /callback?ott=XXX             │                     │                    │
      │    (Server-side)   │                    │                     │                    │
      │                    │                    │                     │                    │
      │ 14. Use TANK Client to exchange OTT    │                     │                    │
      ├────────────────────────────────────────>│ 15. GET /auth/token ────────────────────>│
      │                    │                    │     With HMAC       │                    │
      │                    │                    │     signature       │                    │
      │                    │                    │                     │                    │
      │                    │                    │   16. Validate      │                    │
      │                    │                    │   signature & OTT   │                    │
      │                    │                    │                     │                    │
      │<─ 17. Return {owt} ───────────────────────────────────────────────────────────────│
      │                    │                    │                     │                    │
      │ 18. Set OWT cookie │                    │                     │                    │
      │     (httpOnly,     │                    │                     │                    │
      │     secure)        │                    │                     │                    │
      │                    │                    │                     │                    │
      │ 19. Redirect to    │                    │                     │                    │
      │     success page   │                    │                     │                    │
      │                    │                    │                     │                    │
      ▼                    ▼                    ▼                      ▼                    ▼
```

### Detailed Steps

1. **Initiation**
   - Third-party app displays "Continue as Guest" option
   - User clicks button
   - App redirects to: `GET /auth/authorise?apiKey=XXX&target=guest&cb=CALLBACK_URL`

2. **Authorization Server (trustedauth-app)**
   - Receives authorization request
   - Validates apiKey (looks up in DynamoDB)
   - Renders guest confirmation page
   - User confirms

3. **Guest Token Request**
   - Guest form POSTed to trustedauth-app
   - App calls trustedauth-service: `PUT /auth/token/guest`
   - Request includes apiKey and nonce

4. **Token Generation (trustedauth-service)**
   - Validates apiKey from request
   - Calls user-auth-service: `GET /auth/guest`
   - Receives user token from upstream
   - Creates JWT payload: `{ user: { id: userToken, type: "GUEST" } }`
   - Generates OWT (JWT token) with 8-hour expiry
   - Generates OTT (random 32-char string)
   - Stores in DynamoDB:
     ```
     TrustedParty_Tokens:
     {
       UserToken: userToken,
       PartyId: partyId,
       PartyToken: owt,
       OneTimeToken: ott,
       IssueTime: now,
       ExpiryTime: now + 8h,
       UserType: "GUEST"
     }
     ```

5. **Callback with OTT**
   - trustedauth-app redirects to callback URL
   - `GET CALLBACK_URL?ott=XXX&owt=YYY`
   - OWL included for browser-side consumption

6. **Token Exchange (Third-Party Backend)**
   - Third-party backend receives callback
   - Uses TANK client to exchange OTT for OWT
   - Calls: `GET /auth/token?ott=OTT_VALUE`
   - Includes HMAC-SHA512 signature in headers:
     - `x-ow-signature`: HMAC-SHA512(apiKey + nonce, secret)
     - `x-ow-nonce`: random value
     - `x-ow-signing-string`: data signed

7. **Token Validation & Return**
   - trustedauth-service receives exchange request
   - Validates HMAC signature using party's secret key
   - Looks up OTT in DynamoDB
   - Verifies OTT not expired and not used
   - Returns OWT: `{ owt: JWT_TOKEN }`

8. **Session Establishment**
   - Third-party backend sets HTTP-only secure cookie:
     ```
     Set-Cookie: owt=JWT_TOKEN; Path=/; HttpOnly; Secure; Max-Age=28800
     ```
   - Redirects user to success page

### Error Scenarios

| Scenario | Status | Response |
|----------|--------|----------|
| Invalid apiKey | 401 | `{err: "Invalid apikey"}` |
| Missing apiKey | 400 | `{err: "apiKey is a required parameter"}` |
| Invalid signature | 401 | `{err: "Invalid signature"}` |
| OTT not found | 401 | `{err: "Invalid token"}` |
| OTT expired | 401 | `{err: "Token expired"}` |
| User service down | 500+ | Upstream error propagated |

---

## 2. Personal Account Login Flow

### Sequence Diagram

```
Third-Party App          Browser           TrustedAuth App      TrustedAuth Service    User Auth Service
      │                    │                     │                      │                    │
      ├─ 1. Redirect to   │                     │                      │                    │
      │    /auth/authorise │                     │                      │                    │
      │    ?apiKey=XXX     │                     │                      │                    │
      │    &target=login   │                     │                      │                    │
      │                    │                     │                      │                    │
      │                    ├─ 2. GET /auth/authorise ────────────────────────────────────>│
      │                    │                     │                      │                    │
      │                    │   3. Render login   │                      │                    │
      │                    │   form (email,      │                      │                    │
      │                    │   password fields)  │                      │                    │
      │                    │                    │                      │                    │
      │                    │   4. User enters   │                      │                    │
      │                    │   credentials,      │                      │                    │
      │                    │   clicks login      │                      │                    │
      │                    │                    │                      │                    │
      │                    │   5. POST /login   │                      │                    │
      │                    │   {email,          │                      │                    │
      │                    │    password}       │                      │                    │
      │                    │                    │                      │                    │
      │                    │                    ├─ 6. POST /login────>│                    │
      │                    │                    │     {email,password} │                    │
      │                    │                    │                     ├───────────────────>│
      │                    │                    │                     │ POST /auth        │
      │                    │                    │                     │ {email, password} │
      │                    │                    │                     │                   │
      │                    │                    │                     │ 7. Validate vs    │
      │                    │                    │                     │ Cognito           │
      │                    │                    │                     │ (Success)         │
      │                    │                    │                     │                   │
      │                    │                    │                     │ 8. Return userToken
      │                    │                    │<────────────────────┤                   │
      │                    │                    │   9. userToken      │                   │
      │                    │                    │                     │                   │
      │                    │                    │   10. Call getAuthTokens
      │                    │                    │       (fetch cookies)│                   │
      │                    │                    │                     ├─────────────────>│
      │                    │                    │                     │ GET /tokens      │
      │                    │                    │                     │                  │
      │                    │                    │                     │<─────────────────┤
      │                    │                    │<─────────────────────┤ {wc_auth, ...}  │
      │                    │                    │   11. authTokens    │                  │
      │                    │                    │                     │                  │
      │                    │                    │   12. Create JWT    │                  │
      │                    │                    │   payload:          │                  │
      │                    │                    │   {user: {          │                  │
      │                    │                    │     id: userToken,  │                  │
      │                    │                    │     type: PERSONAL  │                  │
      │                    │                    │   }}                │                  │
      │                    │                    │                     │                  │
      │                    │                    │   13. Generate OTT  │                  │
      │                    │                    │        & OWT         │                  │
      │                    │                    │   14. Store in DB   │                  │
      │                    │                    │                     │                  │
      │                    │<─ 15. Redirect to callback ────────────────────────────────│
      │                    │    ?ott=XXX        │                     │                  │
      │                    │                    │                     │                  │
      │<─ 16. Callback received                  │                     │                  │
      │                    │                    │                     │                  │
      │ 17. Exchange OTT   │                    │                     │                  │
      │    for OWT using   │                    │                     │                  │
      │    TANK client     │                    │                     │                  │
      │                    │                    │     18. GET /token  │                  │
      │                    │                    │     (with signature)│                  │
      │ (Same as guest flow steps 6-8)          │                     │                  │
      │                    │                    │                     │                  │
      │ 19. Set OWT cookie │                    │                     │                  │
      │     & redirect to  │                    │                     │                  │
      │     success        │                    │                     │                  │
      │                    │                    │                     │                  │
      ▼                    ▼                    ▼                      ▼                    ▼
```

### Detailed Steps

1. **Authorization Request**
   - User clicks "Login" on third-party app
   - Redirected to: `GET /auth/authorise?apiKey=XXX&target=login&cb=CALLBACK_URL`

2. **Login Form Rendering**
   - trustedauth-app receives request
   - Validates apiKey
   - Renders login form with email/password fields

3. **Credential Submission**
   - User enters email and password
   - Form POSTed to trustedauth-app: `POST /auth/login`
   - Body includes email and password

4. **Credential Validation**
   - trustedauth-app calls trustedauth-service: `POST /auth/login`
   - trustedauth-service calls user-auth-service with credentials
   - user-auth-service validates against AWS Cognito
   - If valid: returns user token
   - If invalid: returns 401 error

5. **Auth Token Fetching**
   - trustedauth-service calls user-auth-service: `GET /tokens?userToken=XXX`
   - Receives authentication cookies (wc_auth, wc_session, etc.)

6. **JWT Generation**
   - Creates JWT payload with user info:
     ```json
     {
       "user": {
         "id": "user-token-123",
         "type": "PERSONAL"
       },
       "iat": 1696000000,
       "exp": 1696028800
     }
     ```

7. **Token Storage**
   - Stores in DynamoDB:
     ```
     TrustedParty_Tokens:
     {
       UserToken: user-token,
       PartyId: party-id,
       PartyToken: jwt-owt,
       OneTimeToken: random-ott,
       IssueTime: current-timestamp,
       ExpiryTime: current-timestamp + 8h,
       UserType: PERSONAL
     }
     ```

8. **Callback Redirect**
   - Redirects to third-party callback URL
   - `GET CALLBACK_URL?ott=OTT_VALUE&owt=OWT_VALUE`

9. **OTT/OWT Exchange** (Same as guest flow)
   - Third-party backend uses TANK client
   - Calls trustedauth-service with OTT
   - HMAC signature validation
   - Returns OWT

10. **Session Establishment**
    - Sets HTTP-only secure cookie with OWT
    - User is logged in to third-party app

### Validation Checks

```
Login Validation Flow:
    │
    ├─ Email format valid?
    │  └─ If NO → Return 400 "Invalid email"
    │
    ├─ Password not empty?
    │  └─ If NO → Return 400 "Password required"
    │
    ├─ Credentials match Cognito?
    │  ├─ If NO → Return 401 "Invalid credentials"
    │  └─ If YES → Continue
    │
    ├─ User account active?
    │  └─ If NO → Return 403 "Account suspended"
    │
    ├─ MFA required?
    │  └─ If YES → Challenge user, verify MFA code
    │
    └─ Success → Generate tokens
```

---

## 3. Business Account Registration Flow

### Sequence Diagram (Summary)

```
Third-Party App          Browser           TrustedAuth App      TrustedAuth Service    User Auth Service
      │                    │                     │                      │                    │
      ├─ Redirect to /auth/authorise?target=register
      │
      │                    ├─ GET /auth/authorise ──>
      │
      │                    │   Render registration form
      │                    │   (companyName, ABN, email, password, etc.)
      │
      │                    │   User enters details
      │                    │
      │                    │   POST /register/business
      │                    │   {companyName, abn, email, password}
      │                    │
      │                    │        ├─ Validate ABN format ──>
      │                    │        │                        │
      │                    │        │ PUT /auth/register/business
      │                    │        │                        ├─ Validate ABN (11 digits, checksum)
      │                    │        │                        │
      │                    │        │                        ├─ POST /register/business
      │                    │        │                        ├────────────────────────>│
      │                    │        │                        │  {companyName, abn,    │
      │                    │        │                        │   email, password}     │
      │                    │        │                        │                        │
      │                    │        │                        │ (Account created in    │
      │                    │        │                        │  Cognito)              │
      │                    │        │                        │                        │
      │                    │        │<───────────────────────┤ Return userToken       │
      │                    │        │   userToken            │                        │
      │                    │        │                        │                        │
      │                    │        │ (Fetch auth tokens)    │                        │
      │                    │        │ (Generate JWT)         │                        │
      │                    │        │ (Store in DynamoDB)    │                        │
      │                    │        │ (Return OTT)           │                        │
      │                    │        │                        │                        │
      │                    │<────── Redirect with OTT ─────────────────────────────>│
      │
      │ (Continue with OTT/OWT exchange)
      │
      ▼
```

### Key Differences from Personal Registration

1. **ABN Validation**
   - Extract ABN from form
   - Validate format: 11 digits
   - Validate checksum algorithm
   - Return 400 if invalid

2. **Cognito User Type**
   - User created with `userType: BUSINESS`
   - Different attributes stored

3. **JWT Payload**
   ```json
   {
     "user": {
       "id": "user-token-456",
       "type": "BUSINESS"
     }
   }
   ```

4. **Account Features**
   - Business name required
   - ABN stored with account
   - Can manage multiple users
   - Business-specific features enabled

### ABN Validation Algorithm

```
isValidAbn(abn):
  1. Check if numeric
  2. Check length = 11
  3. Convert to digit array: [a, b, c, d, e, f, g, h, i, j, k]
  4. Define weights: [10, 1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
  5. Calculate sum:
     sum = 10*(a-1) + 1*b + 3*c + 5*d + 7*e + 9*f + 11*g + 13*h + 15*i + 17*j + 19*k
  6. Check sum % 89 == 0
  7. Return valid/invalid
```

---

## 4. Token Validation Flow

### Profile Fetch with Token Validation

```
Third-Party App                trustedauth-profile          trustedauth-service
      │                                │                            │
      ├─ GET /auth/customer/profile  │                            │
      │   Cookie: owt=JWT_TOKEN       │                            │
      │                               │                            │
      │                               ├─ Extract OWT from cookie  │
      │                               │                            │
      │                               ├─ 1. Validate JWT signature│
      │                               ├───────────────────────────>│
      │                               │  GET /auth/token/validate │
      │                               │  Header: x-owt            │
      │                               │                            │
      │                               │<─ 2. Validation result    │
      │                               │  {valid: true, exp: ...}  │
      │                               │                            │
      │                               ├─ 3. Call upstream profile │
      │                               │    API                     │
      │                               │                            │
      │<─ 4. Return user profile      │                            │
      │  {userId, email, userName,     │                            │
      │   firstName, lastName, ...}    │                            │
      │                               │                            │
      ▼                               ▼                            ▼
```

### Validation Details

1. **Extract Token**
   - From cookie: `owt`
   - From header: `x-owt`
   - From query: Not allowed for security

2. **Parse JWT**
   - Extract header, payload, signature
   - Validate JWT structure

3. **Verify Signature**
   - Extract public key from trusted party
   - Compute HMAC-SHA512 on header.payload
   - Compare with provided signature

4. **Check Expiry**
   - Extract `exp` claim from payload
   - Compare with current timestamp
   - Return 401 if expired

5. **Return Validation Result**
   ```json
   {
     "valid": true,
     "userId": "customer-123",
     "userType": "PERSONAL",
     "exp": 1696028800
   }
   ```

### Keepalive Flow

```
Third-Party App          trustedauth-service
      │                        │
      ├─ POST /auth/keepalive  │
      │  Header: x-owt         │
      │                        │
      │                        ├─ Validate token
      │                        │
      │                        ├─ Update expiry in DynamoDB
      │                        │  Add 8 hours to current time
      │                        │
      │<─ Return new token     │
      │  {owt, expiresIn}      │
      │                        │
      ▼                        ▼
```

---

## 5. Request Signing Flow

### HMAC-SHA512 Signature Verification

```
Client Side:
┌─────────────────────────────────────────────────────────────┐
│ 1. Prepare request data                                      │
│    apiKey = "Izj5SZEe8b7L4vxG01N0"                         │
│    secret = "OIHMFfAv24sInyNd6EOdzrVTRMxOtct8QXSOUV18"     │
│                                                              │
│ 2. Generate random nonce                                     │
│    nonce = genRandomStr(32)                                  │
│                                                              │
│ 3. Create signing string                                     │
│    signingString = apiKey + nonce                           │
│                                                              │
│ 4. Compute HMAC-SHA512                                      │
│    signature = HMAC-SHA512(signingString, secret)           │
│                                                              │
│ 5. Add request headers                                       │
│    x-ow-signature: <signature>                             │
│    x-ow-nonce: <nonce>                                      │
│    x-ow-signing-string: <signingString>                    │
│                                                              │
│ 6. Send HTTP request with signed headers                    │
│    GET /auth/token?ott=OTT_VALUE                            │
│    Headers: x-ow-signature, x-ow-nonce, x-ow-signing-string
└─────────────────────────────────────────────────────────────┘

Server Side:
┌─────────────────────────────────────────────────────────────┐
│ 1. Extract signature data from request                       │
│    signature = headers['x-ow-signature']                     │
│    nonce = headers['x-ow-nonce']                             │
│    signingString = headers['x-ow-signing-string']           │
│    apiKey = query['apiKey']                                 │
│                                                              │
│ 2. Lookup secret for apiKey in DynamoDB                     │
│    TrustedParty_Api WHERE ApiKey = apiKey                  │
│    secret = party.Secret                                    │
│                                                              │
│ 3. Recompute HMAC-SHA512                                   │
│    computedSignature = HMAC-SHA512(signingString, secret)  │
│                                                              │
│ 4. Compare signatures                                        │
│    if (signature === computedSignature)                      │
│      Accept request                                          │
│    else                                                      │
│      Reject with 401 "Invalid signature"                    │
│                                                              │
│ 5. Additional checks                                         │
│    - Nonce not previously used (replay attack prevention)   │
│    - Request within acceptable time window                  │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. Error Handling Flows

### Authentication Failure Sequence

```
User submits credentials
        │
        ▼
Validate format
        │
   ┌────┴──────┐
   │            │
  YES          NO
   │            │
   ▼            ▼
Continue    Return 400
            "Invalid format"
   │
   ▼
Call Cognito
        │
   ┌────┴──────────────┐
   │                   │
Valid              Invalid
   │                   │
   ▼                   ▼
Continue           Return 401
                   "Invalid credentials"
                   │
                   └─ Log failed attempt
                   └─ Track for rate limiting
```

### Token Expiry Handling

```
Request with OWT cookie
        │
        ▼
Extract and parse JWT
        │
        ▼
Check expiry claim
        │
   ┌────┴──────────┐
   │               │
Valid           Expired
   │               │
   ▼               ▼
Continue       Return 401
               "Token expired"
               │
               └─ Client should:
                  1. Clear cookie
                  2. Redirect to login
                  3. Show "Session expired" message
```

### Signature Validation Failure

```
Request received
        │
        ▼
Extract signature, nonce, apiKey
        │
        ▼
Lookup party secret
        │
   ┌────┴──────────────┐
   │                   │
Found            Not found
   │                   │
   ▼                   ▼
Continue           Return 401
                   "Invalid apiKey"
   │
   ▼
Recompute HMAC-SHA512
        │
   ┌────┴──────────────┐
   │                   │
Match            No match
   │                   │
   ▼                   ▼
Accept            Return 401
request           "Invalid signature"
                  │
                  └─ Security log event
```

---

## 7. Browser-Based Client Flow (authclient.js)

### Step-by-Step Interaction

```
Third-Party App
    │
    ├─ 1. Load script
    │   <script src="authclient.min.js"></script>
    │
    ├─ 2. Add <ow-auth> element
    │   <ow-auth apikey="XXX" 
    │            onlogin="handleLogin"
    │            mode="test"
    │            target="login"></ow-auth>
    │
    └─ 3. on DOMContentLoaded
        │
        ├─ authclient.js initializes
        │
        ├─ Creates iframe
        │  <iframe src="https://ofwtest.officeworks.com.au/auth/login
        │               ?apiKey=XXX
        │               &target=login
        │               &btnLabel=Login
        │               &debug=false"
        │           id="ow-auth"></iframe>
        │
        ├─ Setup postMessage listener
        │  window.addEventListener('message', handler)
        │
        └─ Wait for user interaction
        
User logs in within iframe
        │
        ├─ iframe calls parent via postMessage
        │  window.parent.postMessage({
        │    type: 'login',
        │    guest: false,
        │    token: 'OWT_VALUE'
        │  }, 'https://third-party.com')
        │
        └─ Parent app receives message
        
Parent window message handler
        │
        ├─ Validate message origin
        │
        ├─ Extract token
        │
        ├─ Call onlogin callback
        │  window.handleLogin({type: 'login', token: 'OWT'})
        │
        ├─ Set cookie in parent domain
        │  document.cookie = "owt=OWT_VALUE; Path=/; HttpOnly; Secure"
        │  (Note: HttpOnly must be set server-side)
        │
        ├─ Hide iframe
        │
        └─ Redirect to authenticated page
```

### Message Protocol

| Message Type | Payload | Meaning |
|--------------|---------|---------|
| `login` | `{type:'login', guest:false, token:'OWT'}` | User logged in |
| `register` | `{type:'register', guest:false, token:'OWT'}` | User registered |
| `guest` | `{type:'login', guest:true, token:'OWT'}` | User continued as guest |
| `logout` | `{type:'logout'}` | User logged out |
| `error` | `{type:'error', message:'...'}` | Authentication error |

---

## 8. React/Redux Integration Flow (TARAS)

### Component Initialization

```
App Component Renders
        │
        ├─ 1. Dispatch SET_AUTH_CONFIG
        │   {
        │     type: 'SET_AUTH_CONFIG',
        │     payload: {
        │       apiKey: 'XXX',
        │       serverHostname: 'https://ofwtest.officeworks.com.au'
        │     }
        │   }
        │
        ├─ 2. OWAuthReducer updates state
        │   owauth.config = {apiKey, serverHostname}
        │
        └─ 3. Render OWAuth component
           <OWAuth 
             apiKey="XXX"
             serverHostname="https://ofwtest.officeworks.com.au"
             width={600}
             height={400}
           />

OWAuth Component
        │
        ├─ Renders <iframe>
        │  Points to authserver/auth/login?apiKey=XXX
        │
        └─ Setup postMessage listener
           window.addEventListener('message', handleAuthMessage)

User Interacts
        │
        ├─ Logs in within iframe
        │
        ├─ iframe posts message
        │
        ├─ OWAuth receives message
        │
        ├─ Dispatch authentication action
        │  store.dispatch({
        │    type: 'SET_LOGIN_STATUS',
        │    payload: {authenticated: true, token: 'OWT'}
        │  })
        │
        └─ OWAuthReducer updates state
           owauth.isLoggedIn = true
           owauth.showModal = false

Redux Middleware (OWAuthMiddleware)
        │
        ├─ Intercepts actions
        │
        ├─ Checks if action matches isApplicable
        │
        ├─ If user not logged in and autoLogin=true:
        │  - Dispatch SHOW_LOGIN
        │  - Wait for authentication
        │
        ├─ Fetch user profile
        │  axios.get(profileURI)
        │
        ├─ Dispatch SET_USER_PROFILE
        │  {
        │    type: 'SET_USER_PROFILE',
        │    payload: {userId, email, firstName, lastName, ...}
        │  }
        │
        └─ Re-dispatch original action
```

### State Management

```
Redux State (owauth reducer):
{
  isLoggedIn: boolean,
  userProfile: {
    userId: string,
    userName: string,
    email: string,
    firstName: string,
    lastName: string,
    userType: string,
    custBP: string,
    orgBP: string
  } | null,
  showModal: boolean,
  error: string | null,
  isLoading: boolean,
  config: {
    apiKey: string,
    serverHostname: string
  }
}
```

---

## 9. Logout Flow

### Session Termination

```
User clicks logout
        │
        ├─ 1. Client-side: Clear cookies
        │   document.cookie = "owt=; expires=Thu, 01 Jan 1970 00:00:00 UTC"
        │
        ├─ 2. Optional: Server-side logout
        │   POST /auth/logout (if supported)
        │
        ├─ 3. Clear local storage/session storage
        │
        ├─ 4. Dispatch LOGOUT action (Redux)
        │   owauth.isLoggedIn = false
        │   owauth.userProfile = null
        │
        └─ 5. Redirect to login or home page

Backend considerations:
        │
        ├─ Delete session from DynamoDB
        │  (Optional - tokens are time-limited)
        │
        ├─ Blacklist token in cache
        │  (Optional - prevents replay)
        │
        └─ Log logout event
```

---

## Summary Table

| Flow | Entry Point | Exit Token | Time | Notes |
|------|------------|-----------|------|-------|
| Guest | `/auth/authorise?target=guest` | OWT via callback | Immediate | Fastest flow |
| Login | `/auth/authorise?target=login` | OWT via callback | Credential validation time | Standard flow |
| Register | `/auth/authorise?target=register` | OWT via callback | Account creation time | Requires form fields |
| Token Exchange | Client exchangeToken() | OWT from API | <100ms | Server-side only |
| Profile Fetch | GET /profile with OWT | User profile | <100ms | Requires valid OWT |
| Keepalive | POST /keepalive | Updated OWT | <50ms | Extends session |

---

**Deprecation Notice**: This system is marked for decommissioning January 2026. No new flows should be designed based on this architecture.
