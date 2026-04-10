# C2 Container Architecture - Officeworks Third-Party Authentication System

## Overview

This document describes the individual containers (applications, services, and libraries) that comprise the third-party authentication system, their responsibilities, technologies, and interactions.

---

## Container Diagram

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         External Applications                                │
│                    (Third-Party Integration Clients)                          │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌────────────────────────┐   ┌────────────────────────┐   ┌─────────────┐  │
│  │   Browser Client       │   │   Node.js Backend      │   │ React Redux │  │
│  │ (trustedauth-client)   │   │   (TANK Client)        │   │   (TARAS)   │  │
│  │                        │   │                        │   │             │  │
│  │ - authclient.min.js   │   │ - @ow/trustedauth-     │   │ - OWAuth    │  │
│  │ - <ow-auth> element   │   │   node-client          │   │ - Middleware│  │
│  │ - iframe-based auth   │   │ - Client class         │   │ - Reducer   │  │
│  │ - postMessage comm    │   │ - expressMiddleware    │   │ - Components│  │
│  │ - CDN: S3             │   │ - Request signing      │   │             │  │
│  └───────────┬───────────┘   └───────────┬────────────┘   └──────┬──────┘  │
│              │                           │                       │          │
└──────────────┼───────────────────────────┼───────────────────────┼──────────┘
               │                           │                       │
               │ HTTP/HTTPS                │ HTTP/HTTPS            │
               │ Signed requests           │ with signatures       │
               │ (HMAC-SHA512)             │ (HMAC-SHA512)         │ HTTPS
               │                           │                       │
┌──────────────▼───────────────────────────▼───────────────────────▼──────────┐
│                    Officeworks TrustedAuth System                            │
│                                                                               │
│ ╔═════════════════════════════════════════════════════════════════════════╗ │
│ ║                      1. trustedauth-app                                  ║ │
│ ║                   (OAuth Authorization Server UI)                        ║ │
│ ║──────────────────────────────────────────────────────────────────────────║ │
│ ║ Technology: Express.js + Handlebars templates                            ║ │
│ ║ Port: 3001                                                               ║ │
│ ║ Host: AWS Elastic Beanstalk                                              ║ │
│ ║ Main Responsibility: OAuth-like authorization flow & user interface      ║ │
│ ║                                                                           ║ │
│ ║ Key Routes:                                                              ║ │
│ ║  - GET /auth/login              - Login form & flow                      ║ │
│ ║  - GET /auth/register           - Registration form                      ║ │
│ ║  - GET /auth/authorise          - OAuth authorization endpoint           ║ │
│ ║  - GET /auth/guest              - Guest token request flow               ║ │
│ ║                                                                           ║ │
│ ║ Dependencies:                                                            ║ │
│ ║  - User Auth Service (for user validation)                              ║ │
│ ║  - TrustedAuth Service (for token generation)                           ║ │
│ ║  - AWS Cognito (for credential validation)                             ║ │
│ ║                                                                           ║ │
│ ║ Responsibilities:                                                        ║ │
│ ║  1. Render authentication UI (login, register, guest)                   ║ │
│ ║  2. Validate user input                                                  ║ │
│ ║  3. Coordinate with user-auth-service for credential validation         ║ │
│ ║  4. Call trustedauth-service to generate tokens                         ║ │
│ ║  5. Handle OAuth callback redirect with OTT parameter                   ║ │
│ ║  6. Support multiple environment modes (local, test, master)            ║ │
│ ╚═════════════════════════════════════════════════════════════════════════╝ │
│                                                                               │
│ ╔═════════════════════════════════════════════════════════════════════════╗ │
│ ║                    2. trustedauth-service                                ║ │
│ ║              (Core API - Token Generation & Validation)                  ║ │
│ ║──────────────────────────────────────────────────────────────────────────║ │
│ ║ Technology: Express.js + Node.js                                         ║ │
│ ║ Port: 3002                                                               ║ │
│ ║ Host: AWS Elastic Beanstalk                                              ║ │
│ ║ Main Responsibility: Token lifecycle management, signature validation    ║ │
│ ║                                                                           ║ │
│ ║ Key Modules:                                                             ║ │
│ ║                                                                           ║ │
│ ║  [tpapi.js]                                                              ║ │
│ ║  ├─ DynamoDB operations for trusted parties                              ║ │
│ ║  ├─ Token generation (JWT with RSA signing)                             ║ │
│ ║  ├─ findById(tpId) - Lookup trusted party by ID                         ║ │
│ ║  ├─ findByApiKey(apiKey) - Lookup trusted party by API key              ║ │
│ ║  ├─ genToken(party, userId, payload) - Generate OTT+OWT                 ║ │
│ ║  ├─ createOrUpdateUserToken(userId, userToken)                          ║ │
│ ║  └─ validateSignature(payload, signature) - HMAC-SHA512 validation      ║ │
│ ║                                                                           ║ │
│ ║  [userapi.js]                                                            ║ │
│ ║  ├─ HTTP client for user-auth-service                                   ║ │
│ ║  ├─ guestToken() - Request guest token from upstream                    ║ │
│ ║  ├─ authTokens(userToken) - Fetch auth cookies/tokens                   ║ │
│ ║  └─ login(credentials) - Validate user credentials                      ║ │
│ ║                                                                           ║ │
│ ║  [util.js]                                                               ║ │
│ ║  ├─ hash(str) - SHA512 hashing                                          ║ │
│ ║  ├─ uuid4() - UUID generation                                           ║ │
│ ║  ├─ genRandomStr(len) - Random string generation                        ║ │
│ ║  ├─ isValidAbn(abn) - ABN validation (Australian Business Number)       ║ │
│ ║  ├─ getSignaturePayLoadFromRequest(req) - Extract signature params      ║ │
│ ║  └─ fetch(options) - HTTP client wrapper                                ║ │
│ ║                                                                           ║ │
│ ║ API Routes:                                                              ║ │
│ ║                                                                           ║ │
│ ║  Trusted Party Admin Routes (/auth/tp):                                  ║ │
│ ║   - POST /auth/tp/:tpId            - Create trusted party with ID       ║ │
│ ║   - POST /auth/tp/                 - Create with auto-generated ID      ║ │
│ ║   - PUT /auth/tp/:tpId             - Update trusted party               ║ │
│ ║   - DELETE /auth/tp/:tpId          - Delete trusted party               ║ │
│ ║   - GET /auth/tp/:tpId             - Get party by ID                    ║ │
│ ║   - GET /auth/tp/apiKey/:apiKey    - Get party by API key               ║ │
│ ║                                                                           ║ │
│ ║  User Admin Routes (/auth/user):                                         ║ │
│ ║   - GET /auth/user/token           - Get API token for OWT              ║ │
│ ║   - GET /auth/user/cookies         - Get WC cookies for OWT             ║ │
│ ║                                                                           ║ │
│ ║  Customer Auth Routes (/auth):                                           ║ │
│ ║   - PUT /auth/token/guest          - Generate guest token               ║ │
│ ║   - PUT /auth/register/business    - Register business account          ║ │
│ ║   - PUT /auth/register             - Register personal account          ║ │
│ ║   - POST /auth/login               - Login and issue token              ║ │
│ ║   - POST /auth/token/validate      - Validate given token               ║ │
│ ║   - GET /auth/token                - Get OWT from OTT                   ║ │
│ ║   - POST /auth/keepalive           - Keep user session alive            ║ │
│ ║   - PUT /auth/token/cookies        - Legacy cookie-based auth           ║ │
│ ║                                                                           ║ │
│ ║ Dependencies:                                                            ║ │
│ ║  - AWS DynamoDB (Tables: TrustedParty_Api, TrustedParty_Tokens,         ║ │
│ ║    TrustedParty_UserToken)                                              ║ │
│ ║  - User Auth Service (HTTP calls)                                       ║ │
│ ║  - Winston Logger                                                        ║ │
│ ║                                                                           ║ │
│ ║ Key Features:                                                            ║ │
│ ║  1. HMAC-SHA512 signature validation for all signed requests            ║ │
│ ║  2. JWT token generation with configurable expiry (default 8h)         ║ │
│ ║  3. One-Time Token (OTT) for secure token exchange                      ║ │
│ ║  4. Request logging with unique request IDs                            ║ │
│ ║  5. Nonce tracking for replay attack prevention                        ║ │
│ ║  6. Support for guest, personal, and business user types               ║ │
│ ╚═════════════════════════════════════════════════════════════════════════╝ │
│                                                                               │
│ ╔═════════════════════════════════════════════════════════════════════════╗ │
│ ║                   3. trustedauth-profile                                ║ │
│ ║            (Customer Profile Service)                                   ║ │
│ ║──────────────────────────────────────────────────────────────────────────║ │
│ ║ Technology: Express.js + Node.js                                         ║ │
│ ║ Port: 3004                                                               ║ │
│ ║ Host: AWS Elastic Beanstalk                                              ║ │
│ ║ Main Responsibility: Return authenticated customer profile data          ║ │
│ ║                                                                           ║ │
│ ║ Key Routes:                                                              ║ │
│ ║  - GET /auth/customer/profile     - Get profile (requires OWT token)   ║ │
│ ║  - GET /status                    - Health check                        ║ │
│ ║                                                                           ║ │
│ ║ Dependencies:                                                            ║ │
│ ║  - TrustedAuth Service (token validation)                               ║ │
│ ║  - User Profile Client (upstream API)                                   ║ │
│ ║                                                                           ║ │
│ ║ Responsibilities:                                                        ║ │
│ ║  1. Validate OWT token signature                                        ║ │
│ ║  2. Call upstream API to fetch user profile                            ║ │
│ ║  3. Return profile: userId, userName, email, phone, userType,          ║ │
│ ║     firstName, lastName, business details                              ║ │
│ ║  4. Cache responses to reduce upstream calls                           ║ │
│ ║                                                                           ║ │
│ ║ Response Example:                                                        ║ │
│ ║  {                                                                       ║ │
│ ║    "userId": "20786849",                                                ║ │
│ ║    "userName": "John Doe",                                              ║ │
│ ║    "userType": "BUSINESS",                                              ║ │
│ ║    "email": "john@example.com",                                         ║ │
│ ║    "firstName": "John",                                                 ║ │
│ ║    "lastName": "Doe",                                                   ║ │
│ ║    "phone": "0401111222",                                               ║ │
│ ║    "mobile": null,                                                      ║ │
│ ║    "custBP": "20786849",                                                ║ │
│ ║    "orgBP": "20786848"                                                  ║ │
│ ║  }                                                                       ║ │
│ ╚═════════════════════════════════════════════════════════════════════════╝ │
│                                                                               │
│ ╔═════════════════════════════════════════════════════════════════════════╗ │
│ ║                    4. user-auth-service                                 ║ │
│ ║         (Main User Authentication & Authorization Service)              ║ │
│ ║──────────────────────────────────────────────────────────────────────────║ │
│ ║ Technology: Node.js + Express + AWS Cognito                             ║ │
│ ║ Port: 3003                                                               ║ │
│ ║ Host: AWS ECS (Elastic Container Service)                               ║ │
│ ║ Main Responsibility: User credential validation, session management     ║ │
│ ║                                                                           ║ │
│ ║ Key Components:                                                          ║ │
│ ║  - AWS Cognito User Pool integration                                    ║ │
│ ║  - User credential validation                                           ║ │
│ ║  - Multi-factor authentication support                                  ║ │
│ ║  - Session token generation                                             ║ │
│ ║  - Account registration (personal & business)                           ║ │
│ ║  - Account recovery and password reset                                  ║ │
│ ║                                                                           ║ │
│ ║ Swagger/OpenAPI Specification:                                          ║ │
│ ║  - /app/spec/swagger.yaml          - Full API specification             ║ │
│ ║  - /app/spec/paths.yaml            - Endpoint definitions               ║ │
│ ║  - /app/spec/definitions.yaml      - Data models                        ║ │
│ ║  - /app/spec/parameters.yaml       - Parameter definitions              ║ │
│ ║                                                                           ║ │
│ ║ Infrastructure as Code:                                                 ║ │
│ ║  - CloudFormation templates for Cognito user pools                      ║ │
│ ║  - ECS task definitions                                                  ║ │
│ ║  - IAM role definitions for task execution                              ║ │
│ ║                                                                           ║ │
│ ║ Typical Usage:                                                           ║ │
│ ║  1. Third-party calls trustedauth-service with credentials              ║ │
│ ║  2. trustedauth-service forwards to user-auth-service                   ║ │
│ ║  3. user-auth-service validates against Cognito                         ║ │
│ ║  4. Returns user token and session information                          ║ │
│ ║                                                                           ║ │
│ ║ Responsibilities:                                                        ║ │
│ ║  1. Manage user credentials securely                                    ║ │
│ ║  2. Integrate with AWS Cognito                                          ║ │
│ ║  3. Provide user account lifecycle management                           ║ │
│ ║  4. Return authenticated user tokens                                    ║ │
│ ║  5. Maintain OpenAPI specification                                      ║ │
│ ╚═════════════════════════════════════════════════════════════════════════╝ │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## Client Libraries

### 5. trustedauth-client (Browser Library)

```
┌────────────────────────────────────────────────────────────┐
│             trustedauth-client                              │
│         (Browser-Side JavaScript Library)                   │
├────────────────────────────────────────────────────────────┤
│ NPM Package: N/A (CDN-based)                               │
│ Format: authclient.min.js (minified)                       │
│ CDN: https://s3-ap-southeast-2.amazonaws.com/              │
│      trustedauth-client/authclient.min.js                  │
│ Runtime: Browser (requires DOM & window object)            │
│                                                             │
│ Key Functionality:                                          │
│  - Custom HTML5 element: <ow-auth>                         │
│  - Attributes: apikey, onlogin, mode, target, guest,       │
│    guestToken, debug, btnLabel                            │
│  - Modes: test, local, production (default)                │
│                                                             │
│ How It Works:                                              │
│  1. User adds <ow-auth apikey="XXX" /> to page            │
│  2. authclient.js creates iframe pointing to              │
│     TrustedAuth login page                                │
│  3. iframe loads with full authentication UI              │
│  4. User interacts within iframe                          │
│  5. iframe posts message back to parent via               │
│     window.postMessage                                    │
│  6. Parent app receives message with login status         │
│  7. OWT cookie is set by third-party domain               │
│                                                             │
│ Methods:                                                    │
│  - window.owauth.init()      - Initialize library         │
│  - window.owauth.status()    - Check login status         │
│  - window.owauth.authorise() - Trigger authorization      │
│                                                             │
│ Message Protocol (postMessage):                            │
│  Payload: {                                                 │
│    type: 'login' | 'logout' | 'register',                 │
│    guest: boolean,                                         │
│    token: string (OWT)                                    │
│  }                                                          │
│                                                             │
│ Security Features:                                         │
│  - Same-origin policy enforcement via postMessage         │
│  - HTTP-only cookie setting by parent domain              │
│  - Iframe sandbox restrictions                            │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 6. trustedauth-node-client (Server-Side Node.js Library)

```
┌────────────────────────────────────────────────────────────┐
│           trustedauth-node-client (TANK)                   │
│    (Server-Side Node.js Client Library)                    │
├────────────────────────────────────────────────────────────┤
│ NPM Package: @ow/trustedauth-node-client                  │
│ Repository: trustedauth-node-client                        │
│ Runtime: Node.js (10+)                                     │
│ Main Export: { Client }                                    │
│                                                             │
│ Constructor:                                               │
│  Client(apiKey, serverUrl, internalServerUrl,            │
│         signatureSecret)                                   │
│                                                             │
│  Parameters:                                               │
│  - apiKey: Public API key issued by TrustedAuth            │
│  - serverUrl: Public endpoint (https://...)               │
│  - internalServerUrl: Internal endpoint (http://...)       │
│  - signatureSecret: Secret for HMAC-SHA512 signing        │
│                                                             │
│ Key Methods:                                               │
│  ┌────────────────────────────────────────────┐           │
│  │ getProfile(customerToken)                  │           │
│  │ - Fetch authenticated user profile         │           │
│  │ - Parameter: OWT token                    │           │
│  │ - Returns: Promise<UserProfile>            │           │
│  │ - Usage: authClient.getProfile(owt)        │           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│  ┌────────────────────────────────────────────┐           │
│  │ exchangeToken(ott)                         │           │
│  │ - Exchange one-time token for OWT          │           │
│  │ - Parameter: OTT from callback              │           │
│  │ - Returns: Promise<{ owt: string }>         │           │
│  │ - Signs request with HMAC-SHA512            │           │
│  │ - Must be called on server (never client)   │           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│  ┌────────────────────────────────────────────┐           │
│  │ validateToken(customerToken)               │           │
│  │ - Validate OWT token                       │           │
│  │ - Internal endpoint only                   │           │
│  │ - Parameter: OWT token                    │           │
│  │ - Returns: Promise<ValidationResult>       │           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│  ┌────────────────────────────────────────────┐           │
│  │ expressMiddleware(whiteList)               │           │
│  │ - Express middleware for token validation   │           │
│  │ - Parameter: Array of URL paths to protect │           │
│  │ - Validates 'owt' cookie on matching URLs  │           │
│  │ - Returns: Middleware function              │           │
│  │ - Example: app.use(client.expressMiddleware │           │
│  │           (['/api/orders', '/api/profile']))│           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│  ┌────────────────────────────────────────────┐           │
│  │ signedReq(method, uri, params, headers)    │           │
│  │ - Internal: Send HMAC-SHA512 signed request│           │
│  │ - Auto-adds: x-ow-signature, x-ow-nonce    │           │
│  │ - Parameters signed with secret key        │           │
│  │ - Returns: Promise<Response>                │           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│ Request Signing Process:                                   │
│  1. Generate random nonce                                  │
│  2. Create signing string from parameters                  │
│  3. Compute HMAC-SHA512(signingString, secret)            │
│  4. Add headers: x-ow-signature, x-ow-nonce,              │
│     x-ow-signing-string                                   │
│  5. Send HTTP request with signed headers                 │
│  6. Server validates signature using same secret          │
│                                                             │
│ Usage Example:                                             │
│  const authClient = new Client(                            │
│    'api-key-123',                                          │
│    'https://ofwtest.officeworks.com.au',                  │
│    'http://internal-server.local',                        │
│    'secret-key-456'                                       │
│  );                                                         │
│                                                             │
│  authClient.exchangeToken(req.query.ott)                  │
│    .then(resp => {                                         │
│      res.cookie('owt', resp.owt, {                         │
│        httpOnly: true,                                     │
│        secure: true                                        │
│      });                                                    │
│      res.json({ success: true });                          │
│    });                                                      │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### 7. trustedauth-react-redux (React Library)

```
┌────────────────────────────────────────────────────────────┐
│       trustedauth-react-redux (TARAS)                      │
│    (React & Redux Integration Library)                     │
├────────────────────────────────────────────────────────────┤
│ NPM Package: @ow/trustedauth-react-redux                  │
│ Runtime: React + Redux                                     │
│                                                             │
│ Key Exports:                                               │
│  ┌────────────────────────────────────────────┐           │
│  │ OWAuth Component                           │           │
│  │ - React component for authentication UI    │           │
│  │ - Props:                                   │           │
│  │   - apiKey (required)                     │           │
│  │   - serverHostname (required)              │           │
│  │   - iframeOptions (optional)               │           │
│  │   - mode (optional: test/local/prod)      │           │
│  │   - width, height (optional)               │           │
│  │                                            │           │
│  │ - Renders authentication modal iframe      │           │
│  │ - Listens for login/register events        │           │
│  │ - Dispatches Redux actions on auth events  │           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│  ┌────────────────────────────────────────────┐           │
│  │ OWAuthMiddleware                           │           │
│  │ - Redux middleware for auth checks        │           │
│  │ - Signature: (isApplicable, profileURI,   │           │
│  │            autoLogin)                     │           │
│  │                                            │           │
│  │ - isApplicable: (action) => boolean       │           │
│  │   Function to determine if auth is needed │           │
│  │                                            │           │
│  │ - profileURI: string                      │           │
│  │   Endpoint to fetch user profile           │           │
│  │   Example: '/customer/profile'             │           │
│  │                                            │           │
│  │ - autoLogin: boolean                      │           │
│  │   Show login modal if not authenticated    │           │
│  │                                            │           │
│  │ Behavior:                                  │           │
│  │ 1. Intercepts actions matching isApplicable│           │
│  │ 2. Checks if user is authenticated         │           │
│  │ 3. If not: dispatch login action           │           │
│  │ 4. Wait for authentication to complete     │           │
│  │ 5. Fetch profile from profileURI           │           │
│  │ 6. Re-dispatch original action             │           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│  ┌────────────────────────────────────────────┐           │
│  │ OWAuthReducer                              │           │
│  │ - Redux reducer managing auth state        │           │
│  │ - State structure:                         │           │
│  │   {                                        │           │
│  │     isLoggedIn: boolean,                   │           │
│  │     userProfile: object | null,            │           │
│  │     showModal: boolean,                    │           │
│  │     error: string | null,                  │           │
│  │     isLoading: boolean                    │           │
│  │   }                                        │           │
│  │                                            │           │
│  │ - Register in Redux store with key 'owauth'│           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│  ┌────────────────────────────────────────────┐           │
│  │ Action Creators                            │           │
│  │ - fetchUserProfile(profileURI)             │           │
│  │ - setUserProfile(userProfile)              │           │
│  │ - hideModal()                              │           │
│  │ - retrieveToken()                          │           │
│  │ - setError(errMessage)                     │           │
│  │ - setLoginStatus(data)                     │           │
│  │ - showLogin()                              │           │
│  │ - showRegister()                           │           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│  ┌────────────────────────────────────────────┐           │
│  │ OWAuthPropType                             │           │
│  │ - PropTypes definition for owauth state    │           │
│  │ - Useful for component prop validation     │           │
│  └────────────────────────────────────────────┘           │
│                                                             │
│ Setup Example:                                             │
│  import { OWAuth, OWAuthMiddleware, OWAuthReducer }       │
│    from '@ow/trustedauth-react-redux';                    │
│                                                             │
│  const store = createStore(                                │
│    combineReducers({                                       │
│      owauth: OWAuthReducer,                               │
│      // ... other reducers                                 │
│    }),                                                      │
│    applyMiddleware(                                        │
│      OWAuthMiddleware(                                     │
│        action => action.type === 'FETCH_DATA',            │
│        '/api/profile',                                     │
│        true // autoLogin                                   │
│      )                                                      │
│    )                                                        │
│  );                                                         │
│                                                             │
│  export default function App() {                          │
│    return (                                                │
│      <Provider store={store}>                              │
│        <OWAuth                                             │
│          apiKey="YOUR_API_KEY"                             │
│          serverHostname="https://ofwtest.officeworks..."   │
│        />                                                   │
│        {/* ... rest of app ... */}                         │
│      </Provider>                                           │
│    );                                                      │
│  }                                                          │
│                                                             │
│ Key Features:                                              │
│  - Automatic authentication check on actions               │
│  - User profile fetching and state management              │
│  - Modal-based login/register UI                          │
│  - Multiple environment support                            │
│  - Redux integration for state management                  │
│  - Delayed action dispatch until auth complete             │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

---

## Container Dependencies

```
┌─────────────────────────────────────────────────────────┐
│                   Dependency Graph                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  trustedauth-app                                        │
│  ├─ → user-auth-service (credential validation)         │
│  ├─ → trustedauth-service (token generation)            │
│  └─ → AWS Cognito                                       │
│                                                          │
│  trustedauth-service                                    │
│  ├─ → user-auth-service (upstream auth)                 │
│  ├─ → AWS DynamoDB (token & party storage)              │
│  └─ → AWS Cognito (indirect via user-auth-service)      │
│                                                          │
│  trustedauth-profile                                    │
│  ├─ → trustedauth-service (signature validation)        │
│  ├─ → User Profile API (upstream profile data)          │
│  └─ → Winston Logger                                    │
│                                                          │
│  user-auth-service                                      │
│  ├─ → AWS Cognito (user management)                     │
│  ├─ → Officeworks API Gateway (upstream)                │
│  └─ → CloudFormation (infrastructure)                   │
│                                                          │
│  trustedauth-client (browser)                           │
│  ├─ → trustedauth-app (iframe)                          │
│  └─ → window.postMessage API                            │
│                                                          │
│  trustedauth-node-client (TANK)                         │
│  ├─ → trustedauth-service (HTTP API calls)              │
│  ├─ → trustedauth-profile (profile endpoint)            │
│  └─ → crypto (request signing)                          │
│                                                          │
│  trustedauth-react-redux (TARAS)                        │
│  ├─ → trustedauth-client (iframe management)            │
│  ├─ → Redux Store (state management)                    │
│  └─ → React (component framework)                       │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## Container Communication Patterns

### HTTP/HTTPS Communication

```
External App ──HTTPS request with signature──> TrustedAuth Service
                                                      │
                                              Validate signature (HMAC-SHA512)
                                                      │
                                             ┌────────▼──────────┐
                                             │ Signature valid?   │
                                             └────────┬───────────┘
                                                      │
                                    ┌─────────────────┴─────────────────┐
                                    │                                   │
                                  YES                                  NO
                                    │                                   │
                                    ▼                                   ▼
                         Process request          Return 401 Unauthorized
                         Update DynamoDB
                         Return tokens
```

### Message Passing (Browser)

```
Third-Party App
    │
    ├─ Load <ow-auth> element
    │
    ▼
authclient.js
    │
    ├─ Create iframe pointing to trustedauth-app
    │
    ▼
TrustedAuth App (in iframe)
    │
    ├─ User logs in
    │
    ▼
Parent window receives postMessage
    │
    ├─ window.addEventListener('message', handler)
    │
    ▼
Third-Party App handles login result
    │
    ├─ Backend sets OWT cookie
    ├─ Redirect or update UI
```

---

## Data Flow Examples

### Guest Token Flow

```
1. Third-party app requests guest token
   GET /auth/authorise?apiKey=XXX&target=guest

2. trustedauth-app renders guest form

3. User clicks "Continue as Guest"

4. trustedauth-app calls trustedauth-service
   PUT /auth/token/guest { apiKey, signature, nonce }

5. trustedauth-service:
   - Validates HMAC signature
   - Calls user-auth-service.guestToken()
   - Receives user token from upstream
   - Generates JWT(payload: { user: { id: token, type: GUEST } })
   - Creates OTT + OWT
   - Stores in DynamoDB
   - Returns OTT to iframe

6. trustedauth-app redirects to callback
   GET callback?ott=XXX

7. Third-party backend:
   - Calls trustedauth-service.exchangeToken(ott)
   - TANK client handles signing
   - Receives OWT
   - Sets OWT cookie (httpOnly, secure)

8. Third-party calls trustedauth-profile
   GET /auth/customer/profile with OWT cookie

9. trustedauth-profile validates signature
   Returns user profile

10. Third-party app now has authenticated user
```

---

**Status**: System is marked for deprecation (January 2026). No new integrations should be created.
