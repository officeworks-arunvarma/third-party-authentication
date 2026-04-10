# Repository Index - Officeworks Third-Party Authentication System

Complete mapping of all 7 repositories, their structure, key files, and documentation references.

---

## Repository Overview Table

| Repo | Type | Port | Language | Purpose | Status |
|------|------|------|----------|---------|--------|
| trustedauth-service | Backend | 3002 | Node.js | Core token API | Active |
| trustedauth-app | Backend | 3001 | Node.js | OAuth UI | Active |
| user-auth-service | Backend | 3003 | Node.js+TS | User auth | Active |
| trustedauth-profile | Backend | 3004 | Node.js | Profile endpoint | Active |
| trustedauth-node-client | Library | - | Node.js | Server SDK (TANK) | Active |
| trustedauth-react-redux | Library | - | React/JS | UI components (TARAS) | Active |
| trustedauth-client | Library | - | JavaScript | Browser SDK | Active |

---

## 1. trustedauth-service

**Type**: Express.js Backend Service
**Port**: 3002
**Language**: Node.js (ES6+)
**Purpose**: Core authentication backend providing token generation, validation, and trusted party management

### Directory Structure
```
trustedauth-service/
├── app.js                    # Express server setup
├── config.js                 # Environment-based configuration (local/test/master)
├── package.json              # Dependencies and scripts
├── package-lock.json
├── routes/
│   ├── auth.js              # Customer authentication routes
│   ├── auth.spec.js         # Auth tests
│   ├── tpAdmin.js           # Trusted party admin routes
│   ├── userAdmin.js         # User admin routes
│   ├── status.js            # Health check
│   ├── spec.js              # API specification
│   └── testRoutes.js        # Test utilities
├── lib/
│   ├── tpapi.js             # Trusted party CRUD and token management
│   ├── tpapi.spec.js        # tpapi tests
│   ├── userapi.js           # User auth service HTTP client
│   ├── userapi.spec.js      # userapi tests
│   ├── util.js              # Utilities (hashing, signing, validation)
│   ├── util.spec.js         # util tests
│   ├── logger.js            # Winston logger
│   ├── logger.spec.js       # Logger tests
│   └── __handlers__/        # Mock servers for testing
│       ├── userapiMockServer.js
│       └── utilMockServer.js
├── test/
│   ├── testRoutes.js
│   └── testUtil.js
└── unit-test-output/        # Generated test reports

```

### Key Files Deep Dive

#### app.js
- Express server initialization
- Middleware setup (body-parser, cookie-parser)
- Route registration
- Request logging with ID tracking
- Port: configurable via `PORT` env var (default 3002)

#### config.js
- Environment-specific configuration object
- Three environments: local, test, master
- Settings:
  - DynamoDB table names
  - Endpoint URLs
  - Token expiry (8h)
  - Logging levels
  - Public/admin keys

#### routes/auth.js
Main authentication endpoints:
- `PUT /auth/token/guest` - Generate guest token
- `POST /auth/login` - User login
- `PUT /auth/register` - Personal account registration
- `PUT /auth/register/business` - Business account registration
- `POST /auth/token/validate` - Validate token
- `GET /auth/token` - Exchange OTT for OWT (with HMAC)
- `POST /auth/keepalive` - Extend session
- `PUT /auth/token/cookies` - Legacy cookie-based token

#### lib/tpapi.js
**Trusted Party API operations**
- `findById(tpId)` - Get party by ID
- `findByApiKey(apiKey)` - Get party by API key
- `findByPartyToken(partyToken)` - Get party by token
- `registerParty(tpId?, name, ...)` - Create new TP
- `updateParty(tpId, {...})` - Update TP config
- `deleteParty(tpId)` - Delete TP
- `genToken(trustedParty, userToken, payload)` - Generate OTT + OWT
- `createOrUpdateUserToken(userToken, userId)` - Map user tokens
- `validateSignature(signature, payload, secret)` - HMAC-SHA512 verification

DynamoDB Tables:
- `TrustedParty_Api` (test/master) - Party configurations
- `TrustedParty_Tokens` (test/master) - Issued tokens
- `TrustedParty_UserToken` (test/master) - User token mappings

#### lib/userapi.js
**HTTP client for upstream user-auth-service**
- `guestToken()` - Request guest token
- `authTokens(userToken)` - Fetch auth cookies
- `login(credentials)` - Validate credentials
- Error handling and retry logic

#### lib/util.js
**Utility functions**
- `hash(str)` - SHA512 hashing
- `uuid4()` - UUID generation
- `genRandomStr(len)` - Random string generation
- `isValidAbn(abn)` - Australian Business Number validation
- `getSignaturePayLoadFromRequest(req)` - Extract signature params
- `validateSignature(signature, payload, secret)` - HMAC-SHA512 validation
- `fetch(options)` - HTTP client wrapper

### Key Dependencies
- express (4.17.1)
- jsonwebtoken (7.2.1)
- @aws-sdk/lib-dynamodb
- @aws-sdk/client-dynamodb
- winston (logger)
- crypto (Node.js built-in)

### Testing
- Framework: Mocha + Chai + Sinon
- Test files: `*.spec.js` alongside source
- Run: `npm test`
- Coverage: `npm run coverage`
- Output: `unit-test-output/junit.xml`

### Configuration
**Environment variables**:
```
NODE_ENV=local|test|master  # Config environment
PORT=3002                    # Server port
LOG_LEVEL=debug|info|warn    # Logging level
AWS_REGION=ap-southeast-2    # AWS region
```

**Endpoints** (based on NODE_ENV):
- Local: https://api-test.officeworks.com.au/...
- Test: https://api-test.officeworks.com.au/...
- Master: https://api.officeworks.com.au/...

### Documentation References
- C2_CONTAINER_ARCHITECTURE.md - Container specification
- C3_COMPONENT_ARCHITECTURE.md - Module details
- FUNCTIONAL_DESCRIPTIONS.md - API operations
- AUTHENTICATION_FLOWS.md - Flow sequences
- context.json - Configuration metadata

---

## 2. trustedauth-app

**Type**: Express.js Frontend Service
**Port**: 3001
**Language**: Node.js (ES6+) with Handlebars templates
**Purpose**: OAuth-like authorization server with UI (login, register, guest forms)

### Directory Structure
```
trustedauth-app/
├── app.js                    # Express server
├── package.json
├── routes/
│   ├── auth.js              # Main authorization routes
│   ├── login.js             # Login form handler
│   ├── register.js          # Registration form handler
│   └── guest.js             # Guest flow handler
├── views/                    # Handlebars templates
│   ├── layout.hbs
│   ├── login.hbs
│   ├── register.hbs
│   ├── guest.hbs
│   └── error.hbs
├── public/                   # Static assets
│   ├── css/
│   ├── js/
│   └── images/
├── middleware/               # Custom middleware
└── config/
    ├── local.js
    ├── test.js
    └── master.js
```

### Key Routes
- `GET /auth/authorise` - OAuth authorization endpoint
  - Query params: `apiKey`, `target` (login|register|guest), `cb` (callback)
  - Renders appropriate form
- `GET /auth/login` - Login form
- `GET /auth/register` - Registration form
- `GET /auth/guest` - Guest confirmation form
- `POST /auth/login` - Process login
- `POST /auth/register` - Process registration
- `POST /auth/guest` - Process guest request

### Responsibilities
1. Render authentication UI (Handlebars templates)
2. Validate user input
3. Call trustedauth-service for token generation
4. Handle OAuth callback with OTT parameter
5. Support multiple environments (local, test, master)
6. Handle form submissions and redirects

### Form-Based Flow
1. Third-party redirects user to `/auth/authorise?apiKey=XXX&target=login`
2. trustedauth-app renders login.hbs
3. User enters credentials
4. Form POSTs to `/auth/login`
5. App validates against user-auth-service
6. App calls trustedauth-service for token generation
7. User redirected to callback URL with OTT

### Key Dependencies
- express
- express-handlebars (templating)
- body-parser
- cookie-parser
- jsonwebtoken
- Request library (HTTP client)

### Testing
- Minimal/no visible tests
- Should add integration tests for UI flows

### Configuration
**Environment-specific**:
- Port: 3001 (configurable)
- Endpoints: Points to trustedauth-service (3002)
- User auth endpoints: Points to user-auth-service (3003)

### Documentation References
- C2_CONTAINER_ARCHITECTURE.md - Container specification
- AUTHENTICATION_FLOWS.md - Login/Register/Guest flows
- FUNCTIONAL_DESCRIPTIONS.md - Form operations

---

## 3. user-auth-service

**Type**: Express.js Backend Service (TypeScript)
**Port**: 3003
**Language**: Node.js/TypeScript
**Purpose**: Core user authentication with AWS Cognito integration

### Directory Structure
```
user-auth-service/
├── app/
│   ├── package.json
│   ├── package-lock.json
│   ├── .eslintrc.json       # ESLint configuration
│   ├── tsconfig.json        # TypeScript configuration
│   ├── src/
│   │   ├── index.js         # Entry point
│   │   ├── run.js           # Service startup
│   │   ├── config.js        # Configuration
│   │   ├── routes/          # HTTP route handlers
│   │   │   ├── auth.ts
│   │   │   ├── user.ts
│   │   │   ├── health.ts
│   │   │   └── profile.ts
│   │   ├── services/        # Business logic
│   │   │   ├── cognitoService.ts
│   │   │   ├── authService.ts
│   │   │   ├── profileService.ts
│   │   │   ├── tokenService.ts
│   │   │   └── emailService.ts
│   │   ├── dal/             # Data access layer
│   │   │   ├── dynamo.ts
│   │   │   ├── postgres.ts
│   │   │   ├── userRepository.ts
│   │   │   └── tokenRepository.ts
│   │   ├── middleware/      # Express middleware
│   │   │   ├── auth.ts
│   │   │   ├── error.ts
│   │   │   └── logging.ts
│   │   ├── models/          # TypeScript interfaces
│   │   │   ├── User.ts
│   │   │   ├── Token.ts
│   │   │   └── Profile.ts
│   │   └── utils/           # Helper functions
│   ├── tests/               # Test suite
│   │   ├── component/       # Component/integration tests
│   │   ├── unit/            # Unit tests
│   │   └── fixtures/        # Test data
│   └── unit-test-output/    # Generated test reports
├── swagger.yaml             # OpenAPI specification
├── cloudformation/          # Infrastructure as Code
│   ├── template.yaml
│   ├── parameters.json
│   └── outputs.json
└── README.md
```

### Key Files

#### src/index.js
- Entry point
- Loads logger and config
- Calls run.startService()
- Supports development mode with longjohn stack traces

#### src/config.js
- Environment-based configuration
- CloudFormation parameter support
- Database connection strings
- Cognito user pool ID
- JWT secrets

#### src/services/cognitoService.ts
**AWS Cognito Integration**
- User pool management
- Credential validation (sign-in)
- User creation
- MFA handling
- Password reset
- Session management

#### src/dal/
**Data Access Layer**
- DynamoDB operations (dynamo.ts)
- PostgreSQL operations (postgres.ts)
- User repository queries
- Token persistence

#### src/middleware/auth.ts
**Authentication Middleware**
- JWT validation
- Token extraction
- User context injection
- Authorization checks

### Key Dependencies
- typescript
- express
- jsonwebtoken
- @aws-sdk/client-cognito-identity-provider
- @aws-sdk/client-dynamodb
- pg-promise (PostgreSQL)
- winston (logging)
- joi (validation)
- eslint (linting)

### Testing
- Framework: Mocha + Chai
- Type Checking: TypeScript + tsc
- Linting: ESLint
- Test Output: JUnit XML to `unit-test-output/junit.xml`

**npm scripts**:
```bash
npm start              # Start service
npm test               # Run linting + tests
npm run unit-test      # Tests only
npm run lint           # ESLint
npm run lint:fix       # Auto-fix
npm run lint:tsc       # TypeScript type check
npm run watch          # Auto-reload with supervisor
```

### Infrastructure
- Deployment: AWS ECS (Elastic Container Service)
- Infrastructure: CloudFormation templates
- Database: PostgreSQL + DynamoDB
- Identity: AWS Cognito User Pools
- Environment: CloudFormation parameters

### Configuration
**Environment variables**:
```
NODE_ENV=development|test|production
PORT=3003
LOG_LEVEL=debug|info|warn|error
COGNITO_USER_POOL_ID=...
COGNITO_CLIENT_ID=...
DATABASE_URL=postgresql://...
DYNAMODB_TABLE=...
```

### OpenAPI/Swagger
- Comprehensive API specification in swagger.yaml
- Defines all routes, parameters, responses
- Models and schemas documented

### Documentation References
- C2_CONTAINER_ARCHITECTURE.md - Container spec
- C3_COMPONENT_ARCHITECTURE.md - Component details
- FUNCTIONAL_DESCRIPTIONS.md - API operations
- AUTHENTICATION_FLOWS.md - Auth sequences

---

## 4. trustedauth-profile

**Type**: Express.js Backend Service
**Port**: 3004
**Language**: Node.js (ES6+)
**Purpose**: Customer profile endpoint for authenticated users

### Directory Structure
```
trustedauth-profile/
├── app.js                    # Express server
├── config.js                 # Configuration
├── package.json
├── routes/
│   ├── customer.js          # Profile routes
│   └── health.js            # Health check
├── middleware/
│   ├── auth.js              # Token validation
│   └── logger.js            # Request logging
├── services/
│   ├── profileService.js    # Business logic
│   └── userProfileClient.js # Upstream API client
└── test/
    └── test.js              # Integration tests
```

### Key Routes
- `GET /auth/customer/profile` - Get user profile
  - Input: OWT cookie or `x-owt` header
  - Output: User profile object
  - Validation: JWT signature check
- `GET /status` - Health check

### Responsibilities
1. Extract OWT from cookie or header
2. Validate JWT signature
3. Check token expiry
4. Extract userId from token
5. Call upstream API for profile data
6. Transform and return profile

### Profile Response
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

### Key Dependencies
- express
- body-parser
- cookie-parser
- request (HTTP client)
- winston (logger)
- jsonwebtoken

### Configuration
**config.js**:
- Local, test, master environments
- DynamoDB table names
- Upstream API endpoints
- Token validation keys

### Testing
- Integration tests in test/test.js
- Mocha test framework
- Run: `npm test`

### Documentation References
- C2_CONTAINER_ARCHITECTURE.md - Container spec
- FUNCTIONAL_DESCRIPTIONS.md - Profile retrieval function

---

## 5. trustedauth-node-client (TANK)

**Type**: NPM Package (Node.js Client Library)
**Language**: Node.js (ES6+)
**Purpose**: Server-side client for backend integrations

### Directory Structure
```
trustedauth-node-client/
├── index.js                  # Main export
├── package.json
├── package-lock.json
├── README.md
├── src/
│   └── auth/
│       ├── client.js        # Main Client class
│       ├── client.spec.js   # Client tests
│       └── expressHelper.js # Express middleware factory
├── test/
│   └── auth/
│       └── client.spec.js   # Integration tests
└── dist/                     # Compiled output (if using Babel)
```

### Main Class: Client

**Constructor**:
```javascript
new Client(apiKey, serverUrl, internalServerUrl, signatureSecret)
```

**Parameters**:
- `apiKey` (string) - Public API key for TP
- `serverUrl` (string) - External endpoint (https://www.officeworks.com.au)
- `internalServerUrl` (string) - Internal endpoint (for validation)
- `signatureSecret` (string) - Secret key for HMAC signing

**Public Methods**:

1. **getProfile(owt)**
   - Purpose: Fetch user profile
   - Input: OWT token
   - Output: Promise<UserProfile>
   - Calls: GET /auth/customer/profile (signed request)

2. **exchangeToken(ott)**
   - Purpose: Exchange OTT for OWT
   - Input: One-time token
   - Output: Promise<{owt}>
   - Calls: GET /auth/token (signed request)
   - Note: Must not expose OWT to client side

3. **validateToken(owt)**
   - Purpose: Validate token (internal only)
   - Input: OWT token
   - Output: Promise<ValidationResult>
   - Calls: POST /auth/token/validate (internal endpoint)

4. **expressMiddleware(whiteList)**
   - Purpose: Express middleware for auth validation
   - Input: Array of URL paths to protect
   - Output: Middleware function
   - Behavior: Validates owt cookie on matching paths

**Private Methods**:

1. **signedReq(method, uri, params, headers)**
   - Creates HMAC-SHA512 signed request
   - Generates nonce
   - Computes signature
   - Sends HTTP request
   - Returns Promise<response>

2. **req(method, uri, headers, internal)**
   - Unsigned HTTP request wrapper
   - Used for public endpoints

### HMAC-SHA512 Signing Process
1. Generate nonce (random string or timestamp)
2. Build signing string from parameters (alphabetical order)
3. Compute HMAC-SHA512(signingString, secretKey)
4. Add headers:
   - `x-ow-signature`: Hex-encoded HMAC
   - `x-ow-nonce`: The nonce
   - `x-ow-signing-string`: The data that was signed

### Express Helper

**expressMiddleware(params)**
- Creates middleware for TANK client
- Validates owt cookie
- Extracts user info from token
- Attaches to request object

### Dependencies
- node-rest-client (HTTP)
- promise (Promise polyfill)
- crypto (HMAC-SHA512)

### Testing
- Framework: Mocha + Chai + Sinon + Mockery
- Test file: test/auth/client.spec.js
- Coverage: `npm run coverage`
- Run: `npm test`

### Build
- Optional Babel transpilation
- Output: dist/ directory
- npm script: `npm run build`

### Usage Example
```javascript
const Client = require('@ow/trustedauth-node-client');

const client = new Client(
  'API_KEY',
  'https://www.officeworks.com.au',
  'http://internal:3002',
  'SECRET_KEY'
);

// Exchange OTT for OWT
client.exchangeToken(ott).then(token => {
  // Store token as httpOnly cookie
  res.cookie('owt', token.owt, {httpOnly: true, secure: true});
});

// Get user profile
client.getProfile(owt).then(profile => {
  console.log(profile.userName);
});

// Use middleware
app.use(client.expressMiddleware(['/api', '/admin']));
```

### Documentation References
- C3_COMPONENT_ARCHITECTURE.md - Component details
- README.md - Usage guide
- FUNCTIONAL_DESCRIPTIONS.md - Operations

---

## 6. trustedauth-react-redux (TARAS)

**Type**: NPM Package (React Component Library)
**Language**: React (JSX) + Redux
**Purpose**: React component library for frontend integrations

### Directory Structure
```
trustedauth-react-redux/
├── index.js                  # Main export
├── package.json
├── README.md
├── src/
│   ├── OWAuthMiddleware.js  # Redux middleware
│   ├── fetchAndCatch.js     # HTTP utility
│   └── components/
│       └── OWAuth/
│           ├── OWAuth.jsx   # Main component
│           ├── OWAuthReducer.js   # Redux reducer
│           ├── OWAuthAction.js    # Redux actions
│           └── OWAuthPropTypes.js # PropTypes
├── test/                     # Test suite
│   ├── components/
│   │   └── OWAuth/
│   │       ├── OWAuthReducer.spec.js
│   │       └── OWAuthAction.spec.js
│   ├── OWAuthMiddleware.spec.js
│   ├── fetchAndCatch.spec.js
│   └── dom-mock.js          # DOM mock for tests
├── dist/                     # Compiled output
└── .babelrc                  # Babel configuration
```

### Main Exports

1. **OWAuth Component**
   - React component for authentication UI
   - Renders button/modal for login/register
   - Communicates with authorization server
   - Integrates with Redux

2. **OWAuthMiddleware**
   - Redux middleware
   - Intercepts actions
   - Checks authentication
   - Fetches user profile
   - Supports auto-login

3. **OWAuthReducer**
   - Redux reducer for auth state
   - Manages login status
   - Stores user profile
   - Handles errors

4. **OWAuthAction**
   - Redux action creators
   - Profile fetching
   - Modal visibility
   - Error handling

### OWAuth Component Props
```javascript
<OWAuth
  apiKey={string}              // Third-party API key
  serverHostname={string}      // TrustedAuth endpoint
  mode={string}                // 'local' | 'test' | 'production'
  view={string}                // 'login' | 'register'
  iframeOptions={string}       // Additional URL params
  width={number}               // Modal width
  height={number}              // Modal height
  onCloseModal={function}      // Close callback
  retrieveToken={boolean}      // Silent token fetch
/>
```

### Redux State (owauth reducer)
```javascript
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

### Redux Actions
- `SET_AUTH_CONFIG` - Configure API
- `SET_USER_PROFILE` - Store user data
- `SET_LOGIN_STATUS` - Update login state
- `SHOW_LOGIN` - Display login modal
- `SHOW_REGISTER` - Display register modal
- `HIDE_MODAL` - Close modal
- `SET_ERROR` - Set error message
- `FETCH_USER_PROFILE` - Async profile fetch

### OWAuthMiddleware
**Purpose**: Intercept actions and enforce authentication

**Configuration**:
```javascript
const middleware = createOWAuthMiddleware({
  isApplicable: (action) => action.type === 'PROTECTED_ACTION',
  profileURI: '/api/user/profile',
  autoLogin: true
});
```

**Flow**:
1. Action dispatched
2. Check isApplicable
3. If not logged in:
   - Dispatch SHOW_LOGIN
   - Wait for authentication
4. Fetch user profile
5. Dispatch SET_USER_PROFILE
6. Re-dispatch original action

### HTTP Utilities

**fetchAndCatch(url, options)**
- Promise-based HTTP fetch
- Error handling
- Response validation
- Throws on 4xx/5xx errors

### Integration with trustedauth-client
- Renders `<ow-auth>` element within Modal/iframe
- Listens for authentication events
- Sets cookies
- Updates Redux state

### Dependencies
- react (15.x)
- redux
- react-redux
- react-bootstrap (Modal)
- react-iframe-resizer-super
- prop-types
- babel-core (dev)
- mocha + chai (test)

### Testing
- Framework: Mocha + Chai + Sinon
- DOM Mock: jsdom simulation
- Coverage: `npm run coverage`
- Run: `npm test`

### Build
- Babel transpilation (ES6 → ES5)
- Output: dist/
- npm script: `npm run build`

### Usage Example
```javascript
// Store setup
const store = createStore(
  combineReducers({
    owauth: OWAuthReducer
  }),
  applyMiddleware(OWAuthMiddleware)
);

// Component usage
<OWAuth
  apiKey="ABC123"
  serverHostname="https://www.officeworks.com.au"
  mode="production"
  view="login"
/>
```

### Documentation References
- C3_COMPONENT_ARCHITECTURE.md - Component details
- AUTHENTICATION_FLOWS.md - React/Redux flow
- README.md - Usage guide

---

## 7. trustedauth-client

**Type**: Browser JavaScript Library
**Language**: JavaScript (ES5)
**Purpose**: Client-side authentication without backend (iframe-based)

### Files
```
trustedauth-client/
├── authclient.js             # Source code
├── authclient.min.js         # Minified (deployed to S3)
├── auth-sample.html          # Example integration
├── package.json              # Metadata only
└── README.md
```

### Global API: window.owauth

**Methods**:

1. **init(params)**
   - Initialize authentication client
   - Parameters:
     - `apiKey` (string) - Third-party API key
     - `target` (string) - 'login' | 'register'
     - `debug` (boolean) - Enable debug logging

2. **status()**
   - Check current authentication status
   - Returns: Current auth state

3. **authorise()**
   - Trigger authorization flow
   - Opens iframe/popup
   - Initiates login/register

### Custom HTML Element: `<ow-auth>`

**Attributes**:
```html
<ow-auth
  apikey="API_KEY"           <!-- Third-party API key (required) -->
  onlogin="callbackFunc"     <!-- Callback on login success -->
  target="login|register"    <!-- Flow type (default: login) -->
  mode="local|test|prod"     <!-- Environment (default: prod) -->
  guest="true|false"         <!-- Show guest option (default: false) -->
  guestToken="true|false"    <!-- Use guest token flow -->
  debug="true|false"         <!-- Debug logging -->
>
  Login Button Text
</ow-auth>
```

**Domain Routing**:
- `mode="test"` → `https://ofwtest.officeworks.com.au`
- `mode="local"` → `http://localhost:3001`
- Default → `https://www.officeworks.com.au`

### Communication Pattern

1. **Initialization**
   - Script loaded: `authclient.min.js`
   - DOM ready event: DOMContentLoaded
   - Scan for `<ow-auth>` elements
   - Create iframe for each

2. **User Interaction**
   - User clicks button
   - Iframe loads authorization page
   - User logs in/registers in iframe

3. **Iframe Messaging**
   - Iframe posts message to parent: `window.postMessage()`
   - Parent receives via `window.addEventListener('message')`
   - Message format: `{type: 'auth', data: {ott, owt, ...}}`

4. **Callback**
   - Parent calls `onlogin` callback function
   - Passes authentication result
   - Application handles token storage

### Example Integration
```html
<html>
<head>
  <script src="https://s3-ap-southeast-2.amazonaws.com/trustedauth-client/authclient.min.js"></script>
</head>
<body>
  <ow-auth apikey='MY_API_KEY' 
           onlogin='handleLogin'
           target='login'
           mode='production'
           guest='true'>
    Login with Officeworks
  </ow-auth>

  <script>
    function handleLogin(result) {
      console.log('User logged in:', result.userId);
      // Store OWT as httpOnly cookie (backend operation)
      // Set user profile in UI
    }
  </script>
</body>
</html>
```

### Features

1. **Security**
   - PostMessage same-origin policy
   - No credentials exposed to parent window
   - HMAC signature validation by server

2. **Flexibility**
   - Works without backend logic
   - Multiple environments (test/prod)
   - Guest flow support
   - Customizable button text

3. **Simplicity**
   - Single script include
   - Just add `<ow-auth>` element
   - Browser handles rest

### Deployment

**Build**:
```bash
uglifyjs --compress --mangle --output authclient.min.js -- authclient.js
```

**Deploy to S3**:
```
s3://trustedauth-client/
  ├── authclient.min.js      # Minified script
  └── auth-sample.html       # Example usage
```

**CDN URL**:
```
https://s3-ap-southeast-2.amazonaws.com/trustedauth-client/authclient.min.js
```

### No Testing
- No visible test files
- Should add Selenium/Puppeteer tests

### Key Code Sections

**Polyfills**:
```javascript
Object.assign polyfill for older browsers
```

**Initialization**:
```javascript
DOMContentLoaded event handler
Scan for <ow-auth> elements
Extract attributes
Create iframe
Setup postMessage listener
```

**Domain Resolution**:
```javascript
const domain = 'https://www.officeworks.com.au'; // default
if (mode === 'test') domain = 'https://ofwtest.officeworks.com.au';
if (mode === 'local') domain = 'http://localhost:3001';
```

**Message Handling**:
```javascript
window.addEventListener('message', function(event) {
  if (event.origin !== expectedOrigin) return; // Security check
  const result = event.data; // Contains ott, owt, userId, etc.
  const callbackFunc = window[callbackName];
  callbackFunc(result);
});
```

### Documentation References
- C3_COMPONENT_ARCHITECTURE.md - Component details
- AUTHENTICATION_FLOWS.md - Browser integration flow
- auth-sample.html - Live example

---

## Cross-Repository Dependencies

### Service Dependencies
```
trustedauth-app (3001)
    ↓ calls
    └─→ user-auth-service (3003)
    └─→ trustedauth-service (3002)

trustedauth-service (3002)
    ↓ calls
    └─→ user-auth-service (3003) [via userapi.js]
    └─→ DynamoDB [via tpapi.js]

trustedauth-profile (3004)
    ↓ calls
    └─→ Upstream API (user profile)
    └─→ DynamoDB [optionally for caching]

user-auth-service (3003)
    ↓ calls
    └─→ AWS Cognito [credential validation]
    └─→ DynamoDB [token storage]
    └─→ PostgreSQL [user data]

Third-party Apps
    ↓ integrate using
    ├─→ trustedauth-client [browser]
    ├─→ trustedauth-node-client (TANK) [server]
    └─→ trustedauth-react-redux (TARAS) [React]
```

### Data Model Dependencies
```
TrustedParty (stored in TrustedParty_Api)
    ├─ Uses apiKey, secretKey for HMAC signing
    └─ Linked via PartyId to TrustedParty_Tokens

Token (stored in TrustedParty_Tokens)
    ├─ Contains JWT with user payload
    ├─ Linked to PartyId (which TP issued it)
    └─ Linked to UserId (which user)

UserToken (stored in TrustedParty_UserToken)
    └─ Maps upstream userToken to userId
```

### Configuration Dependencies
```
All services depend on:
├─ NODE_ENV (determines config.js section)
├─ AWS credentials (for DynamoDB/Cognito)
├─ Port settings
└─ Logging level
```

---

## Development Workflow

### Setup All Repositories
```bash
# Clone with submodules
git clone --recurse-submodules git@github.com:officeworks-arunvarma/third-party-authentication.git

# Or if already cloned
git submodule update --init --recursive
```

### Install Dependencies
```bash
# For each service
cd trustedauth-service && npm install
cd trustedauth-app && npm install
cd user-auth-service/app && npm install
cd trustedauth-profile && npm install
cd trustedauth-node-client && npm install
cd trustedauth-react-redux && npm install
```

### Run All Services Locally
```bash
# Terminal 1: trustedauth-service
cd trustedauth-service && npm start

# Terminal 2: trustedauth-app
cd trustedauth-app && npm start

# Terminal 3: user-auth-service
cd user-auth-service/app && npm start

# Terminal 4: trustedauth-profile
cd trustedauth-profile && npm start
```

### Run Tests
```bash
# Test all services
for dir in trustedauth-* user-auth-service; do
  (cd "$dir" && npm test)
done
```

### Building Libraries
```bash
# Build TANK (node-client)
cd trustedauth-node-client && npm run build

# Build TARAS (react-redux)
cd trustedauth-react-redux && npm run build

# Build browser client
cd trustedauth-client && uglifyjs authclient.js > authclient.min.js
```

---

## File Reference Guide

### Core Business Logic
- `trustedauth-service/lib/tpapi.js` - Trusted party operations
- `trustedauth-service/lib/userapi.js` - User auth integration
- `user-auth-service/app/src/services/` - Auth services

### Route Handlers
- `trustedauth-service/routes/auth.js` - Authentication endpoints
- `trustedauth-app/routes/auth.js` - OAuth UI endpoints
- `user-auth-service/app/src/routes/` - User auth routes
- `trustedauth-profile/routes/customer.js` - Profile endpoint

### Client Libraries
- `trustedauth-node-client/src/auth/client.js` - TANK main class
- `trustedauth-react-redux/src/components/OWAuth/` - TARAS components
- `trustedauth-client/authclient.js` - Browser client

### Configuration
- `trustedauth-service/config.js` - Service config
- `user-auth-service/app/src/config.js` - User auth config
- `user-auth-service/swagger.yaml` - API spec
- `user-auth-service/cloudformation/` - Infrastructure

### Testing
- `**/test/` or `**/*.spec.js` - Test files
- `trustedauth-react-redux/test/dom-mock.js` - DOM mock
- `user-auth-service/app/tests/component/` - Integration tests

---

## Quick Navigation

### Find Code By Function
- **Token generation** → `trustedauth-service/lib/tpapi.js:genToken()`
- **Signature validation** → `trustedauth-service/lib/util.js:validateSignature()`
- **User login** → `trustedauth-app/routes/auth.js` + `user-auth-service/app/src/services/`
- **Profile fetch** → `trustedauth-profile/routes/customer.js`
- **Client requests** → `trustedauth-node-client/src/auth/client.js`
- **React integration** → `trustedauth-react-redux/src/components/OWAuth/`
- **Browser init** → `trustedauth-client/authclient.js:DOMContentLoaded`

### Find Docs By Topic
- **Architecture** → C1, C2, C3 architecture docs
- **Flows** → AUTHENTICATION_FLOWS.md
- **Operations** → FUNCTIONAL_DESCRIPTIONS.md
- **Quality** → CONSISTENCY_REVIEW.md
- **Metadata** → context.json

---

**Last Updated**: April 10, 2026
**Status**: Complete
