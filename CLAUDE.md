# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a monorepo containing Officeworks' third-party authentication system, which provides an OAuth-like mechanism for third-party and non-core applications to authenticate customers. The system is **marked for deprecation** (scheduled decommissioning January 2026) and should not be used by new systems.

### Key Architecture Points

**Trust Model**: The system uses API keys and secret keys for trusted parties (TPs) to interact with the authentication services. Communication between services is secured using HMAC-SHA512 signatures with nonces.

**Service Communication Flow**:
- External apps integrate using `trustedauth-node-client` (TANK) - a Node.js library providing unified access
- Frontend integrations use `trustedauth-client` (browser library) or `trustedauth-react-redux` (React component)
- Backend flows through either `trustedauth-app` (OAuth-like popup) or `trustedauth-service` (direct API)
- `trustedauth-profile` provides customer profile endpoints
- `user-auth-service` handles core user authentication with AWS Cognito integration

**Data Layer**: Services use DynamoDB (via AWS SDK) for persistent storage, with `pg-promise` used in `user-auth-service` for PostgreSQL.

## Monorepo Structure

This is **not** a unified monorepo using lerna/yarn workspaces. Each package is independent with its own git repository, managed as **git submodules** in the parent repo.

### Working with Submodules

**Clone with submodules:**
```bash
git clone --recurse-submodules git@github.com:officeworks-arunvarma/third-party-authentication.git
```

**Update submodules after initial clone:**
```bash
git submodule update --init --recursive
```

**Pull latest changes in all submodules:**
```bash
git pull --recurse-submodules
# or
git submodule foreach git pull origin main
```

**Push changes from a submodule back to its origin:**
```bash
cd <submodule-dir>
git push origin <branch>
cd ..
git add <submodule-dir>
git commit -m "Update <submodule-name> to latest"
git push origin main
```

### Packages

| Package | Type | Purpose | Tech |
|---------|------|---------|------|
| `trustedauth-service` | Backend | Core trusted party admin and user auth APIs | Express.js, Node.js, DynamoDB |
| `trustedauth-app` | Backend | OAuth-like authorization UI and flow | Express.js, Handlebars templates |
| `user-auth-service` | Backend | Main user authentication service | Express.js, TypeScript, Cognito, PostgreSQL |
| `trustedauth-profile` | Backend | Customer profile endpoint for trusted parties | Express.js |
| `trustedauth-node-client` | Library | Node.js client/middleware for trusted parties | Node.js, promise-based |
| `trustedauth-react-redux` | UI Library | React login component & Redux middleware | React 15, Redux, Babel |
| `trustedauth-client` | Browser Library | Client-side authentication library | Vanilla JS, iframe-based |

## Common Development Commands

### Per-Package Commands

**Node.js Services** (`trustedauth-service`, `trustedauth-app`, `trustedauth-profile`):
```bash
cd <package-dir>
npm install
npm start                    # Start the service (Express.js on port 3001-3002)
npm test                     # Run Mocha tests
npm run coverage             # Generate coverage report
```

**user-auth-service** (TypeScript):
```bash
cd user-auth-service/app
npm install
npm start                    # Start service (port 3000)
npm test                     # Run linting + unit tests
npm run unit-test            # Run unit tests only (with JUnit XML report)
npm run lint                 # Run ESLint
npm run lint:fix             # Auto-fix linting issues
npm run lint:tsc             # TypeScript type checking
npm run watch                # Run with supervisor and auto-reload
```

**React/Redux Package**:
```bash
cd trustedauth-react-redux
npm install
npm test                     # Run Mocha tests with Babel
npm run coverage             # Generate HTML coverage report
npm run build                # Test → Babel transpile to dist/
```

**Node.js Client Library**:
```bash
cd trustedauth-node-client
npm install
npm test                     # Run Mocha tests with Babel
npm run coverage             # Generate HTML coverage report
```

**Browser Client Library**:
```bash
cd trustedauth-client
# Build minified version
uglifyjs --compress --mangle --output authclient.min.js -- authclient.js
# Deploy authclient.min.js, auth-sample.html to S3 bucket
# https://console.aws.amazon.com/s3/buckets/trustedauth-client/?region=ap-southeast-2
```

### Shared Testing Patterns

- **Test Framework**: Mocha with Chai (assertions), Sinon (mocking), Mockery (module mocking)
- **Coverage Tool**: nyc with 80% minimum coverage threshold (in most packages)
- **Test Output**: JUnit XML reports generated to `unit-test-output/junit.xml`
- **Test Watchmode**: `test-only:watch` (trustedauth-service) or `npm run watch` (user-auth-service with supervisor)
- **Coverage Reports**: HTML reports in `unit-test-output/coverage/` or project root

**Running a single test file**:
```bash
# Most packages use mocha directly
npx mocha test/path/to/file.spec.js

# For packages using Babel (react-redux, node-client)
npx mocha --compilers js:babel-register test/path/to/file.spec.js
```

**Running tests across multiple packages**:
```bash
# Test all packages
for dir in trustedauth-* user-auth-service; do
  (cd "$dir" && npm test) || echo "Tests failed in $dir"
done
```

## Architecture Patterns

### Authentication Flow

1. **Trusted Party Registration**: Admins register TPs via `POST /auth/tp` (trustedauth-service)
   - TPs receive: apiKey (public), secretKey (private), nonce (incremental)
   - Used for HMAC-SHA512 signature generation

2. **User Authentication Options**:
   - **OAuth-like flow**: Redirect through `trustedauth-app` → gets one-time token (OTT) → exchange for OWT (Officeworks Token)
   - **Direct API**: TPsend signed requests directly to `trustedauth-service`
   - **Guest tokens**: Can be issued for anonymous users

3. **Profile Access**: `trustedauth-profile` endpoint validates signature and returns customer details

4. **Session Validation**: Tokens validated via `/auth/token/validate`

### Signature Algorithm (HMAC-SHA512)

Parameters ordered alphabetically:
```
ow-api-key={apiKey}&ow-nonce={nonce}&owt={token}
```
Then HMAC-SHA512 with secretKey, hex-encoded.

Node.js example:
```javascript
const crypto = require('crypto');
const hash = crypto.createHmac('sha512', secretKey).update(signStr).digest('hex');
```

### Request Headers (Admin/Protected Endpoints)

```
X-OW-AGENTTOKEN: <user_token>
X-OW-SIGNATURE: <hmac_sha512_hex>
X-OW-NONCE: <timestamp_or_incrementing>
X-OW-ADMIN-KEY: <admin_key>
```

### Browser Integration (iframe-based)

The `authclient.min.js` (built from `trustedauth-client`) provides client-side auth without backend logic:

1. **HTML Setup**: Include `ow-auth` custom element with attributes:
   ```html
   <ow-auth apikey='API_KEY' onlogin='callbackFunction' 
            target='login|register' mode='local|test|production' guest='true'>
     Login Button Text
   </ow-auth>
   <script src="https://s3-ap-southeast-2.amazonaws.com/trustedauth-client/authclient.min.js"></script>
   ```

2. **Domain Routing** (based on mode attribute):
   - `test` → `https://ofwtest.officeworks.com.au`
   - `local` → `http://localhost:3003`
   - Default (production) → `https://www.officeworks.com.au`

3. **Communication**: Hidden iframe posts authentication status back to parent window via `window.postMessage`

4. **Available Functions** (attached to `window.owauth`):
   - `init(params)` - Initialize with apiKey, target, debug
   - `status()` - Check current auth status
   - `authorise()` - Trigger login/register popup

## Key Files to Know

### trustedauth-service
- `lib/tpapi.js` - Trusted party CRUD and admin routes
- `lib/userapi.js` - User authentication and token management
- `lib/util.js` - Signature validation, JWT handling, utility functions
- `routes/auth.js` - Customer login/register endpoints
- `routes/userAdmin.js` - Admin user management

### user-auth-service/app
- `src/index.js` - Express server setup
- `src/routes/` - Route handlers
- `src/services/` - Business logic (Cognito integration, profile service)
- `src/dal/` - Data access layer (DynamoDB, PostgreSQL)
- `src/config.js` - Environment-based configuration
- `tests/component/` - Component/integration tests

### trustedauth-node-client
- `index.js` - Exports Client class
- `src/client.js` - Core client with request/signature methods
- `src/middleware.js` - Express middleware for token validation

### trustedauth-client
- `authclient.js` - Source browser library (iframe-based postMessage)
- `authclient.min.js` - Minified version deployed to S3
- `auth-sample.html` - Example integration HTML

## Important Notes

### Deprecation Status
- **trustedauth-service** is marked for decommissioning January 2026
- New integrations should not use this system
- Contact "Web Wizards" team for legacy auth needs

### Port Mapping
- `trustedauth-service`: Default port 3002 (configurable via `PORT` env var)
- `trustedauth-app`: Default port 3001 (configurable)
- `user-auth-service`: Default port 3000
- `trustedauth-profile`: Default port (check app.js)
- `trustedauth-client` iframe endpoints: port 3003 (local mode), 3001 (test mode)

**Running multiple services locally**: Use different terminals or environment variables to override PORT. The `authclient.min.js` respects the `mode` attribute to route to the correct domain/port.

### Environment Variables

Key env vars across services:
- `NODE_ENV`: 'development' enables longjohn stack traces (user-auth-service)
- `PORT`: Server port override
- `LOG_LEVEL`: Winston logging level
- AWS credentials for DynamoDB/Cognito access
- Database connection strings for PostgreSQL

### Cookie Handling

- HTTP-only, secure cookies used for OWT (Officeworks Token)
- Cookie parser middleware required before auth middleware
- `cookie-parser` must be configured in trustedauth-node-client integration

### Testing Notes

- `user-auth-service` requires `npm-run-all` for running lint + tests in sequence
- Mock external services with `mockery` module
- Use `msw` (Mock Service Worker) for HTTP mocking
- Tests often verify signature validation and token expiry

### Linting & Type Checking

Only `user-auth-service` has ESLint + TypeScript:
- ESLint checks JS files in `src/`
- TypeScript types (`.d.ts` files) included in devDependencies
- Other packages do not enforce linting

## References

- **Internal Docs**: 
  - Beanstalk Overview: https://officeworks.atlassian.net/wiki/spaces/Digital/pages/3690528925/Beanstalk+Overview
  - Trusted Auth Design: https://officeworks.atlassian.net/wiki/spaces/Digital/pages/3736961282/Trusted+Auth+Beanstalk
