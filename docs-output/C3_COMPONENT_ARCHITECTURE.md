# C3 Component Architecture - Officeworks Third-Party Authentication System

## Overview

This document describes the detailed components within each container, including modules, classes, functions, and their interactions.

---

## 1. TrustedAuth Service Components

### trustedauth-service (Port 3002)

#### Module: tpapi.js
**Purpose**: Trusted Party API operations and token management

```mermaid
graph TB
    subgraph tpapi["tpapi.js Module"]
        subgraph db["Database Layer"]
            transformTP["transformTrustedParty(dbItem)<br/>→ TrustedParty object"]
            transformTPT["transformTrustedPartyToken(dbItem)<br/>→ TrustedPartyToken object"]
            buildRecord["buildRecord(dbPromise, getDataFn, transformFn)<br/>Promise wrapper for DynamoDB"]
        end
        
        subgraph query["Query Operations"]
            findById["findById(tpId)<br/>→ Promise&lt;TrustedParty&gt;"]
            findByApiKey["findByApiKey(apiKey)<br/>→ Promise&lt;TrustedParty&gt;"]
            findByPartyToken["findByPartyToken(partyToken)<br/>→ Promise&lt;TrustedPartyToken&gt;"]
            findUserTokenByUserId["findUserTokenByUserId(userId)<br/>→ Promise&lt;UserToken&gt;"]
        end
        
        subgraph create["Create/Update Operations"]
            registerParty["registerParty(tpId?, name, apiKey, secretKey)<br/>→ Promise&lt;TrustedParty&gt;"]
            updateParty["updateParty(tpId, config)<br/>→ Promise&lt;TrustedParty&gt;"]
            deleteParty["deleteParty(tpId)<br/>→ Promise&lt;{deleted: true}&gt;"]
        end
        
        subgraph token["Token Operations"]
            createOrUpdateUserToken["createOrUpdateUserToken(userToken, userId)<br/>→ Promise&lt;UserToken&gt;"]
            genToken["genToken(trustedParty, userToken, payload)<br/>→ Promise&lt;{owt, ott, userToken}&gt;"]
            validateSignature["validateSignature(signature, payload, secret)<br/>→ boolean"]
            validateToken["validateToken(token)<br/>→ Promise&lt;{userId, userType, issued, expires}&gt;"]
        end
        
        AWSSdk["AWS SDK v3 (@aws-sdk/lib-dynamodb)<br/>DynamoDBDocument client<br/>Region: ap-southeast-2"]
    end
    
    findById --> buildRecord
    findByApiKey --> buildRecord
    findByPartyToken --> buildRecord
    findUserTokenByUserId --> buildRecord
    registerParty --> buildRecord
    updateParty --> buildRecord
    deleteParty --> buildRecord
    createOrUpdateUserToken --> buildRecord
    genToken --> buildRecord
    validateSignature --> buildRecord
    validateToken --> buildRecord
    buildRecord --> AWSSdk
```

#### Module: userapi.js
**Purpose**: Upstream user authentication service integration

```mermaid
graph TB
    subgraph http["HTTP Client Operations"]
        guestToken["guestToken()<br/>GET endpoint.guest<br/>→ Promise&lt;userToken&gt;"]
        authTokens["authTokens(userToken)<br/>GET usercookies<br/>→ Promise&lt;{wc_auth, wc_session}&gt;"]
        login["login(email, password)<br/>POST login<br/>→ Promise&lt;userToken&gt;"]
        register["register(type, formData)<br/>POST register.personal|business<br/>→ Promise&lt;userToken&gt;"]
        fetchAuthTokens["fetchAuthTokens(userToken)<br/>GET fetchAuthTokens<br/>→ Promise&lt;{authToken}&gt;"]
    end
    
    subgraph config["Configuration"]
        endpoints["Endpoints from config.js:<br/>login, guest, register.personal,<br/>register.business, usercookies"]
        timeout["Request Timeout:<br/>config.requestTimeout: 20000ms"]
        errorHandling["Error Handling:<br/>Network errors, HTTP 4xx/5xx"]
    end
    
    deps["Dependencies: request library<br/>util.fetch() wrapper"]
    
    guestToken --> endpoints
    authTokens --> endpoints
    login --> endpoints
    register --> endpoints
    fetchAuthTokens --> endpoints
    endpoints --> timeout
    endpoints --> errorHandling
    http --> deps
```

#### Module: util.js
**Purpose**: Utility functions for cryptography, validation, and helpers

```mermaid
graph TB
    subgraph crypto["Cryptographic Functions"]
        hash["hash(str)<br/>SHA512 hash<br/>crypto.createHash('sha512')<br/>→ hexadecimal"]
        getSigPayload["getSignaturePayLoadFromRequest(req)<br/>Extract: x-owt, x-ott, x-ow-signature,<br/>x-ow-nonce, x-ow-signing-string, apiKey<br/>→ {ows, nonce, apiKey, signatureString}"]
    end
    
    subgraph idGen["ID Generation Functions"]
        uuid4["uuid4()<br/>Generate UUID v4<br/>→ string"]
        genRandomStr["genRandomStr(len)<br/>Generate random alphanumeric<br/>→ string"]
    end
    
    subgraph validate["Validation Functions"]
        isValidAbn["isValidAbn(abn)<br/>Validate ABN: 11 digits<br/>Checksum algorithm<br/>Weights: [10,1,3,5,7,9,11,13,15,17,19]<br/>→ boolean"]
        isNumeric["isNumeric(n)<br/>Check if numeric<br/>→ boolean"]
        digits["digits(num)<br/>Convert to digit array<br/>→ [1,2,3,...] for 123"]
    end
    
    subgraph httpUtil["HTTP Utilities"]
        fetch["fetch(options, respHandler, errHandler)<br/>Promise-wrapped HTTP request<br/>- Set timeout from config<br/>- Handle 200/201 success codes<br/>→ Promise&lt;response&gt;"]
    end
    
    deps["Dependencies:<br/>crypto, uuid/v4,<br/>randomstring, request"]
    
    crypto --> deps
    idGen --> deps
    validate --> deps
    httpUtil --> deps
```

#### Module: routes/auth.js
**Purpose**: Customer authentication endpoints

```mermaid
graph TB
    subgraph middleware["Middleware"]
        apiKeyVal["API Key Validation<br/>- Check query params or body<br/>- Lookup trusted party by apiKey<br/>- Attach party to req.party<br/>- 400 if missing, 401 if invalid"]
    end
    
    subgraph handlers["Route Handlers"]
        guestToken["guestToken()<br/>PUT /auth/token/guest<br/>→ {owt, ott, authTokens, user} 201"]
        register["register()<br/>PUT /auth/register<br/>→ {owt, ott, authTokens, user} 201"]
        regBusiness["registerBusiness()<br/>PUT /auth/register/business<br/>→ {owt, ott, authTokens, user} 201"]
        login["login()<br/>POST /auth/login<br/>→ {owt, ott, authTokens, user} 200"]
        validateToken["validateToken()<br/>POST /auth/token/validate<br/>→ {valid, user, expires}"]
        fetchToken["fetchToken()<br/>GET /auth/token<br/>→ {owt: JWT}"]
        keepAlive["keepAlive()<br/>POST /auth/keepalive<br/>→ {owt, expiresIn}"]
        tokenForCookies["tokenForCookies()<br/>PUT /auth/token/cookies<br/>→ {owt, user} Legacy"]
    end
    
    respFormat["Response Format<br/>buildResponseObject()<br/>{owt, ott, authTokens,<br/>user: {id, type}}<br/>type: GUEST|PERSONAL|BUSINESS"]
    
    handlers --> respFormat
    middleware --> handlers
```

#### Module: routes/tpAdmin.js
**Purpose**: Trusted Party management endpoints (admin only)

```mermaid
graph TB
    subgraph middleware["Middleware"]
        adminKeyVal["Admin Key Validation<br/>- Require X-OW-ADMIN-KEY header<br/>- Compare against config.adminHeader<br/>- Return 401 if missing or invalid"]
    end
    
    subgraph handlers["Route Handlers"]
        createWithId["POST /:tpId<br/>Create party with specified ID<br/>Body: {name, callbackUrl, nonce}<br/>→ TrustedParty"]
        createAutoId["POST /<br/>Create party with auto-generated ID<br/>Body: {name, callbackUrl, nonce}<br/>→ TrustedParty"]
        updateParty["PUT /:tpId<br/>Update trusted party<br/>Query: callbackUrl, name, nonce<br/>→ Updated TrustedParty"]
        deleteParty["DELETE /:tpId<br/>Delete trusted party<br/>→ {deleted: true}"]
        getById["GET /:tpId<br/>Get party by ID<br/>→ TrustedParty"]
        getByKey["GET /apiKey/:apiKey<br/>Get party by API key<br/>→ TrustedParty"]
    end
    
    respFormat["Response Format<br/>{partyId, name, apiKey, secretKey,<br/>callbackUrl, nonce, tpClientId}"]
    
    handlers --> respFormat
    middleware --> handlers
```

#### Module: routes/userAdmin.js
**Purpose**: User authentication admin endpoints

```mermaid
graph TB
    subgraph middleware["Middleware"]
        adminKeyVal["Admin Key Validation<br/>- Require: X-OW-AGENTTOKEN, X-OW-ADMIN-KEY<br/>- Optional: X-OW-SIGNATURE, X-OW-NONCE<br/>- Return 401 if missing"]
    end
    
    subgraph handlers["Route Handlers"]
        getToken["GET /token<br/>Require: OWT cookie or header<br/>1. Validate OWT token<br/>2. Lookup user token mapping<br/>3. Return API token for OWT<br/>→ {userToken: string}"]
        getCookies["GET /cookies<br/>Require: OWT cookie or header<br/>1. Validate OWT token<br/>2. Fetch WC cookies<br/>3. Return legacy cookie format<br/>→ {wc_auth, wc_session, ...}"]
    end
    
    usage["Used by: Internal admin operations<br/>Legacy integrations"]
    
    handlers --> usage
    middleware --> handlers
```

---

### trustedauth-app (Port 3001)

```mermaid
graph TB
    subgraph views["Views - Handlebars Templates"]
        login["login.hbs<br/>Login form with<br/>email/password"]
        register["register.hbs<br/>Registration form<br/>Personal: email, password,<br/>firstName, lastName<br/>Business: companyName, abn, etc"]
        guest["guest.hbs<br/>Continue as Guest button"]
        authorize["authorize.hbs<br/>OAuth authorization<br/>confirmation"]
    end
    
    subgraph routes["Routes"]
        loginGet["GET /auth/login<br/>Render login form<br/>Params: apiKey, cb, target"]
        registerGet["GET /auth/register<br/>Render registration form<br/>Params: apiKey, cb"]
        loginPost["POST /auth/login<br/>Process form<br/>Call trustedauth-service<br/>Redirect callback?ott=XXX"]
        registerPost["POST /auth/register<br/>Process registration<br/>Auto-login + redirect"]
        guestGet["GET /auth/guest<br/>Generate guest token<br/>Call trustedauth-service<br/>Redirect callback?ott=XXX"]
        authorise["GET /auth/authorise<br/>Main OAuth endpoint<br/>Route to login/register/guest<br/>Params: apiKey, cb, target, guest"]
    end
    
    subgraph client["Client-side JavaScript"]
        owauth["lib/client/owauth.js<br/>Communication with parent window<br/>window.postMessage for iframe<br/>Pass login/register/guest events"]
        js["public/js/<br/>jQuery manipulation<br/>Form validation"]
    end
    
    config["Configuration<br/>Support: local/test/master<br/>environments"]
    
    views --> routes
    routes --> client
    client --> config
```

---

## 3. TrustedAuth Profile Components

### trustedauth-profile (Port 3004)

```mermaid
graph TB
    subgraph routes["Route Handlers"]
        profile["GET /auth/customer/profile<br/>1. Extract OWT from headers (x-owt)<br/>2. Validate signature via tpapi<br/>3. Call userProfileClient<br/>4. Return profile data"]
        status["GET /status<br/>Health check endpoint"]
    end
    
    subgraph module["Module: userProfileClient.js"]
        getProfile["getProfile(userToken)<br/>HTTP GET to upstream API<br/>Endpoint: config.endpoints.userinfo<br/>→ Promise&lt;UserProfile&gt;"]
        config["Configuration from config.js<br/>Upstream endpoint settings"]
    end
    
    response["Response Format<br/>{userId, userName, userType, email,<br/>firstName, lastName, phone, mobile,<br/>custBP, orgBP}"]
    
    profile --> getProfile
    getProfile --> response
    status --> response
    getProfile --> config
```

---

## 4. User Auth Service Components

### user-auth-service (AWS ECS)

```mermaid
graph TB
    subgraph spec["OpenAPI/Swagger Specification"]
        swagger["app/spec/swagger.yaml<br/>Main API specification"]
        paths["app/spec/paths.yaml<br/>POST /register/personal<br/>POST /register/business<br/>POST /auth<br/>GET /auth/guest<br/>GET /account<br/>POST /tokens"]
        definitions["app/spec/definitions.yaml<br/>User, Credentials, AuthToken,<br/>UserProfile, BusinessUser"]
        parameters["app/spec/parameters.yaml<br/>Common parameter definitions"]
    end
    
    subgraph appCode["Application Code - app/src/"]
        routes["routes/<br/>registerPersonal()<br/>registerBusiness()<br/>login()<br/>getGuest()<br/>getProfile()<br/>getAuthTokens()"]
        services["services/<br/>CognitoUserPoolService<br/>UserProfileService<br/>SessionService"]
    end
    
    subgraph infra["Infrastructure as Code"]
        cognito["cognito-user-pool.yaml<br/>Password policies, MFA,<br/>email verification,<br/>user attributes"]
        cognitoAccess["cognito-access.yaml<br/>JWT claim mappings,<br/>token expiry,<br/>callback URLs"]
        config["[environment]/config.yaml<br/>[environment]/ecs.yaml<br/>[environment]/task-role.yaml"]
    end
    
    testing["Testing<br/>tests/component/<br/>Verify authentication flows"]
    
    spec --> appCode
    appCode --> infra
    infra --> testing
```

---

## 5. Client Library Components

### trustedauth-node-client (TANK)

```mermaid
graph TB
    subgraph client["class Client"]
        constructor["Constructor(apiKey, serverUrl,<br/>internalServerUrl, secret)"]
        
        subgraph public["Public Methods"]
            getProfile["getProfile(owt)<br/>→ Promise&lt;UserProfile&gt;<br/>Calls signedReq()"]
            exchangeToken["exchangeToken(ott)<br/>→ Promise&lt;{owt}&gt;<br/>Exchange OTT for OWT"]
            validateToken["validateToken(owt)<br/>→ Promise&lt;ValidationResult&gt;<br/>Internal endpoint only"]
            middleware["expressMiddleware(whitelist)<br/>→ Express middleware<br/>Validate owt cookie on paths"]
        end
        
        subgraph private["Private Methods"]
            signedReq["signedReq(method, uri, params, headers)<br/>1. Generate random nonce<br/>2. Create signing string<br/>3. HMAC-SHA512(signing string, secret)<br/>4. Add headers to request<br/>5. Send HTTP request<br/>→ Promise&lt;response&gt;"]
            req["req(method, uri, headers, internal)<br/>Unsigned HTTP request wrapper"]
        end
    end
    
    subgraph express["Express Helper<br/>src/auth/expressHelper.js"]
        factory["Express middleware factory<br/>- Attach to app.use()<br/>- Validate owt cookie<br/>- Pass auth context to next"]
    end
    
    deps["Dependencies<br/>- node-rest-client (HTTP)<br/>- crypto (HMAC-SHA512)<br/>- promise"]
    
    constructor --> public
    public --> private
    express --> factory
    factory --> deps
    private --> deps
```

### trustedauth-react-redux (TARAS)

```mermaid
graph TB
    subgraph components["Components"]
        owauth["OWAuth Component<br/>Props: apiKey, serverHostname,<br/>width, height, iframeOptions, mode<br/>Render: Modal/iframe auth UI<br/>Events: Listen for auth results,<br/>dispatch Redux actions"]
        protected["Protected Routes<br/>Wrapper component for<br/>auth-required routes"]
    end
    
    subgraph redux["Redux Integration"]
        reducer["OWAuthReducer<br/>State: isLoggedIn, userProfile,<br/>showModal, error, config<br/>Actions: SET_AUTH_CONFIG,<br/>SET_USER_PROFILE, SET_LOGIN_STATUS,<br/>SHOW_LOGIN, SHOW_REGISTER,<br/>HIDE_MODAL, SET_ERROR"]
        
        middleware["OWAuthMiddleware<br/>Params: isApplicable, profileURI,<br/>autoLogin<br/>Process:<br/>1. Intercept actions<br/>2. Check isApplicable<br/>3. If not logged in: SHOW_LOGIN<br/>4. Fetch profile<br/>5. Dispatch SET_USER_PROFILE<br/>6. Re-dispatch original action"]
        
        actions["Action Creators<br/>fetchUserProfile(), setUserProfile(),<br/>hideModal(), retrieveToken(),<br/>setError(), setLoginStatus(),<br/>showLogin(), showRegister()"]
    end
    
    subgraph types["Type Definitions"]
        proptype["OWAuthPropType<br/>PropTypes for Redux<br/>owauth state"]
    end
    
    build["Build Process<br/>webpack configuration<br/>→ dist/index.js"]
    
    components --> redux
    redux --> types
    redux --> build
```

### trustedauth-client (Browser Library)

```mermaid
graph TB
    subgraph globalAPI["Global API: window.owauth"]
        init["init()<br/>Initialize auth client"]
        status["status()<br/>Check login status"]
        authorise["authorise()<br/>Trigger auth flow"]
    end
    
    subgraph element["Custom HTML Element: &lt;ow-auth&gt;"]
        attrs["Attributes<br/>apikey (required), onlogin,<br/>mode (test|local|prod),<br/>target (login|register|guest),<br/>guest, guestToken, debug, btnLabel"]
        
        behavior["Behavior<br/>1. DOMContentLoaded: find elements,<br/>extract attributes, create iframe<br/>2. User interaction: login in iframe<br/>3. postMessage: extract result,<br/>call onlogin, set OWT cookie"]
        
        msg["Message Format<br/>{type: login|logout|register,<br/>guest: bool, token: OWT}"]
    end
    
    subgraph comms["Communication & Routing"]
        iframeConstruct["URL Construction<br/>{domain}/auth/login?apiKey={key}&<br/>btnLabel={label}&target={target}"]
        
        domains["Domain Selection<br/>Production: officeworks.com.au<br/>Test: ofwtest.officeworks.com.au<br/>Local: localhost:3001"]
        
        events["Event Communication<br/>- window.addEventListener(message)<br/>- Validate event.origin<br/>- Call window[onlogin] callback"]
    end
    
    polyfills["Polyfills<br/>Object.assign (older browsers)"]
    
    globalAPI --> element
    element --> comms
    comms --> polyfills
    attrs --> behavior
    behavior --> msg
    iframeConstruct --> domains
    domains --> events
```

---

## 6. Database Schema (DynamoDB)

### Table: TrustedParty_Api

| Attribute | Type | Key | Notes |
|-----------|------|-----|-------|
| PartyId | String | HASH (PK) | Unique party identifier |
| Name | String | | Display name |
| ApiKey | String | GSI: ApiKey-index | Public API key |
| Secret | String | | Private HMAC secret |
| CallbackUrl | String | | OAuth callback endpoint |
| Nonce | String | | Security nonce |
| TpClientId | String | | Client identifier |
| CreatedAt | Number | | Epoch timestamp |

**Example item:**
```json
{
  "PartyId": "90003",
  "Name": "accis",
  "ApiKey": "Izj5SZEe8b7L4vxG01N0",
  "Secret": "OIHMFfAv24sInyNd6EOdzrVTRMxOtct8QXSOUV18",
  "CallbackUrl": "http://www.owt.com",
  "Nonce": "393939393939",
  "TpClientId": "90003"
}
```

### Table: TrustedParty_Tokens

| Attribute | Type | Key | Notes |
|-----------|------|-----|-------|
| UserToken | String | HASH (PK) | User token from upstream |
| PartyId | String | RANGE (SK) | Trusted party ID |
| PartyToken | String | | JWT token (OWT) issued |
| OneTimeToken | String | | OTT for token exchange |
| UserId | String | | Upstream user ID |
| IssueTime | Number | | Epoch timestamp |
| ExpiryTime | Number | | TTL — 8 hours from issue |
| UserType | String | | `GUEST` \| `PERSONAL` \| `BUSINESS` |

**Example item:**
```json
{
  "UserToken": "user-token-123",
  "PartyId": "90003",
  "PartyToken": "JWT...",
  "OneTimeToken": "ott-abc-def",
  "UserId": "customer-123",
  "IssueTime": 1696000000,
  "ExpiryTime": 1696028800,
  "UserType": "PERSONAL"
}
```

### Table: TrustedParty_UserToken

| Attribute | Type | Key | Notes |
|-----------|------|-----|-------|
| UserId | String | HASH (PK) | User identifier |
| UserToken | String | | Maps to upstream user token |
| CreatedAt | Number | | Epoch timestamp |

---

## 7. Request/Response Signing

### HMAC-SHA512 Signature Algorithm

```mermaid
sequenceDiagram
    participant C as Client (TANK)
    participant S as trustedauth-service
    participant DB as DynamoDB

    Note over C: 1. Build signing string
    Note over C: signingString = "ow-api-key={apiKey}&ow-nonce={nonce}&owt={token}"
    Note over C: 2. Compute HMAC-SHA512(signingString, secretKey)
    Note over C: 3. Add headers
    C->>S: Request + {x-ow-signature, x-ow-nonce, x-ow-signing-string}
    S->>DB: Lookup party secret by apiKey
    DB-->>S: party.Secret
    Note over S: 4. Recompute HMAC-SHA512(signingString, party.Secret)
    Note over S: 5. Compare provided vs computed signature
    alt Signatures match
        S-->>C: 200 OK + response body
    else Signatures differ
        S-->>C: 401 Unauthorized
    end
```

**Implementation (Node.js):**
```javascript
const crypto = require('crypto');

function sign(dataToSign, secret) {
  return crypto
    .createHmac('sha512', secret)
    .update(dataToSign)
    .digest('hex');
}

// Signing string format (ordered params):
const signingString = `ow-api-key=${apiKey}&ow-nonce=${nonce}&owt=${token}`;
const signature = sign(signingString, secretKey);
```

**Protected Request Headers:**

| Header | Value | Required |
|--------|-------|----------|
| `X-OW-SIGNATURE` | Hex-encoded HMAC-SHA512 | Yes |
| `X-OW-NONCE` | Random string (per-request) | Yes |
| `X-OW-AGENTTOKEN` | Agent credentials (admin only) | Admin routes |
| `X-OW-ADMIN-KEY` | Admin API key | Admin routes |

---

## Summary Table

| Component | Technology | Responsibility | Port |
|-----------|-----------|-----------------|------|
| trustedauth-app | Express + Handlebars | OAuth UI & authorization | 3001/3003 |
| trustedauth-service | Express + Node.js | Token generation & validation | 3002 |
| user-auth-service | Node.js + TypeScript + Cognito | User credential validation | 3000 |
| trustedauth-profile | Express + Node.js | User profile endpoint | 3004 |
| trustedauth-node-client | NPM package | Server-side integration | — |
| trustedauth-react-redux | NPM package | React/Redux integration | — |
| trustedauth-client | Browser JS (CDN) | Client-side authentication | — |

---

**Deprecation Status**: System will be decommissioned January 2026.
