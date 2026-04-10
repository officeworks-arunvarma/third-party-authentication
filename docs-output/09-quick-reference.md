# Quick Reference Guide - Officeworks Third-Party Authentication

Fast lookup for common tasks, endpoints, and concepts.

---

## System Basics

**What is it?** OAuth-like authentication system for third-party apps
**Status**: DEPRECATED (Jan 2026 decommissioning)
**Architecture**: 7 microservices + 3 client libraries
**Stack**: Node.js, Express, AWS (Cognito, DynamoDB, Beanstalk)

---

## Ports & Endpoints

| Service | Port | URL | Purpose |
|---------|------|-----|---------|
| trustedauth-app | 3001 | https://www.officeworks.com.au/auth/* | OAuth UI |
| trustedauth-service | 3002 | https://www.officeworks.com.au/auth/* | Core API |
| user-auth-service | 3003 | (internal) | User auth |
| trustedauth-profile | 3004 | /auth/customer/profile | User profile |

---

## Core Concepts

### Tokens
| Token | Lifetime | Format | Use |
|-------|----------|--------|-----|
| **OTT** | Minutes | String | One-time exchange for OWT |
| **OWT** | 8 hours | JWT | Authenticated session |
| **JWT** | 8 hours | JSON | Payload structure |

### Keys (for each Trusted Party)
| Key | Visibility | Use |
|-----|-----------|-----|
| **apiKey** | Public | Identify the party |
| **secretKey** | Private | HMAC-SHA512 signing |
| **nonce** | Incremental | Additional security |

### User Types
- **GUEST** - Anonymous user
- **PERSONAL** - Individual customer
- **BUSINESS** - B2B customer

---

## API Endpoints Quick Reference

### Authentication Routes
```
PUT  /auth/token/guest              Generate guest token
POST /auth/login                    Login user
PUT  /auth/register                 Register personal account
PUT  /auth/register/business        Register business account
POST /auth/token/validate           Validate token
GET  /auth/token?ott=XXX            Exchange OTT for OWT
POST /auth/keepalive                Extend session
```

### Trusted Party Admin Routes
```
POST   /auth/tp                     Create trusted party
POST   /auth/tp/:tpId               Create with specific ID
GET    /auth/tp/:tpId               Get party config
GET    /auth/tp/apiKey/:apiKey      Get party by API key
PUT    /auth/tp/:tpId               Update party
DELETE /auth/tp/:tpId               Delete party
```

### Profile Routes
```
GET /auth/customer/profile          Get user profile (with OWT)
```

---

## Common Request Patterns

### 1. Create Trusted Party (Admin)
```
POST /auth/tp
Headers: X-OW-ADMIN-KEY: <admin_key>
Body: {
  "name": "My App",
  "callbackUrl": "https://myapp.com/callback",
  "nonce": "random-nonce"
}
Response: {
  "partyId": "123",
  "apiKey": "ABC123...",
  "secretKey": "XYZ789...",
  "callbackUrl": "https://myapp.com/callback",
  "nonce": "random-nonce"
}
```

### 2. Get User Profile (Signed Request)
```
GET /auth/customer/profile
Headers:
  Cookie: owt=<JWT_TOKEN>
  OR
  x-owt: <JWT_TOKEN>

Response: {
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

### 3. Guest Token (Browser Flow)
```
1. User clicks "Continue as Guest"
2. Browser redirects:
   GET /auth/authorise?apiKey=ABC123&target=guest&cb=<CALLBACK>

3. User confirms, form submits:
   PUT /auth/token/guest
   Body: {apiKey, signature, nonce}

4. Receive OTT in redirect:
   GET <CALLBACK>?ott=XXX

5. Backend exchanges OTT for OWT:
   GET /auth/token?ott=XXX
   (with HMAC signature)

6. Response:
   {owt: "JWT_TOKEN..."}

7. Set cookie:
   Set-Cookie: owt=JWT_TOKEN; httpOnly; secure
```

### 4. HMAC-SHA512 Signature

**Generate (Node.js)**:
```javascript
const crypto = require('crypto');
const signStr = `ow-api-key=${apiKey}&ow-nonce=${nonce}`;
const signature = crypto
  .createHmac('sha512', secretKey)
  .update(signStr)
  .digest('hex');
```

**Headers Required**:
```
x-ow-signature: <hex_signature>
x-ow-nonce: <nonce_value>
x-ow-signing-string: <data_signed>
```

---

## Integration Patterns

### 1. Browser Only (No Backend)
```html
<script src="https://s3-ap-southeast-2.amazonaws.com/trustedauth-client/authclient.min.js"></script>

<ow-auth apikey="ABC123" 
         onlogin="handleLogin" 
         target="login" 
         mode="production">
  Login
</ow-auth>

<script>
function handleLogin(result) {
  // Backend sets OWT cookie
  window.location.href = '/dashboard';
}
</script>
```

### 2. Node.js Backend (TANK)
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
  res.cookie('owt', token.owt, {httpOnly: true, secure: true});
});

// Get user profile
client.getProfile(owt).then(profile => {
  res.json(profile);
});

// Middleware
app.use(client.expressMiddleware(['/api', '/admin']));
```

### 3. React (TARAS)
```javascript
import {OWAuth, OWAuthReducer} from '@ow/trustedauth-react-redux';

const store = createStore(
  combineReducers({owauth: OWAuthReducer}),
  applyMiddleware(OWAuthMiddleware)
);

export default (
  <OWAuth 
    apiKey="ABC123" 
    serverHostname="https://www.officeworks.com.au"
    mode="production"
    view="login"
  />
);
```

---

## HTTP Status Codes

| Code | Meaning | When |
|------|---------|------|
| 200 | Success | Valid request, token issued |
| 400 | Bad request | Missing/invalid parameters |
| 401 | Unauthorized | Invalid signature, expired token |
| 403 | Forbidden | Invalid admin key, access denied |
| 404 | Not found | Party/token not found |
| 500 | Server error | Internal error |
| 503 | Unavailable | Service down |

---

## Configuration Reference

### Environment Variables
```
NODE_ENV=local|test|master       # Config environment
PORT=3001-3004                   # Server port
LOG_LEVEL=debug|info|warn        # Logging level
AWS_REGION=ap-southeast-2        # AWS region
```

### DynamoDB Tables (by environment)
```
Development/Test:
  TrustedParty_Api_test
  TrustedParty_Tokens_test
  TrustedParty_UserToken_test

Production:
  TrustedParty_Api
  TrustedParty_Tokens
  TrustedParty_UserToken
```

### Endpoints (by environment)
```
Local/Test:
  API: https://api-test.officeworks.com.au
  Auth: https://ofwtest.officeworks.com.au

Production:
  API: https://api.officeworks.com.au
  Auth: https://www.officeworks.com.au
```

---

## Common Commands

### Start Services
```bash
# Terminal 1
cd trustedauth-service && npm start

# Terminal 2
cd trustedauth-app && npm start

# Terminal 3
cd user-auth-service/app && npm start

# Terminal 4
cd trustedauth-profile && npm start
```

### Run Tests
```bash
cd <service> && npm test
```

### Build Libraries
```bash
# TANK
cd trustedauth-node-client && npm run build

# TARAS
cd trustedauth-react-redux && npm run build

# Browser client
cd trustedauth-client && uglifyjs authclient.js > authclient.min.js
```

### Update Submodules
```bash
git pull --recurse-submodules
# or
git submodule foreach git pull origin main
```

---

## Error Codes & Messages

| Error | Cause | Fix |
|-------|-------|-----|
| Invalid apiKey | Party not found | Verify apiKey in config |
| Invalid signature | HMAC mismatch | Check secretKey and algorithm |
| Token expired | Token lifetime exceeded | Request new token |
| Invalid nonce | Nonce mismatch | Ensure nonce is unique |
| User not found | Login failed | Verify credentials |
| Service unavailable | Downstream service down | Check service status |

---

## Data Models Summary

### Trusted Party
```json
{
  "partyId": "string (unique ID)",
  "name": "string",
  "apiKey": "string (public)",
  "secretKey": "string (private)",
  "callbackUrl": "string (OAuth callback)",
  "nonce": "string (security)",
  "tpClientId": "string (defaults to partyId)"
}
```

### Token
```json
{
  "userId": "string",
  "partyId": "string",
  "partyToken": "string",
  "userToken": "string",
  "issueTime": "number (timestamp)",
  "expiryTime": "number (timestamp)",
  "oneTimeToken": "string"
}
```

### User Profile
```json
{
  "userId": "string",
  "userName": "string",
  "userType": "GUEST|PERSONAL|BUSINESS",
  "email": "string",
  "firstName": "string",
  "lastName": "string",
  "phone": "string",
  "mobile": "string",
  "custBP": "string",
  "orgBP": "string",
  "hasThirtyDayAccount": "boolean"
}
```

---

## Security Quick Tips

1. **Never expose secretKey** - Keep private, use only on backend
2. **Always use httpOnly cookies** for OWT tokens
3. **Validate signatures** - Always check HMAC on requests
4. **Use HTTPS** - Never send tokens over HTTP
5. **Check token expiry** - Tokens expire after 8 hours
6. **Regenerate nonces** - Each request should have unique nonce

---

## Deprecation Notice

**This system is scheduled for decommissioning January 2026.**

- No new integrations
- Critical bug fixes only
- Plan migration to OAuth 2.0
- Contact Web Wizards team

---

## Documentation Map

| Need | Document |
|------|----------|
| System overview | C1_CONTEXT_ARCHITECTURE.md |
| Service details | C2_CONTAINER_ARCHITECTURE.md |
| Code structure | C3_COMPONENT_ARCHITECTURE.md |
| How flows work | AUTHENTICATION_FLOWS.md |
| API operations | FUNCTIONAL_DESCRIPTIONS.md |
| System health | CONSISTENCY_REVIEW.md |
| Metadata | context.json |
| Repository guide | REPOSITORY_INDEX.md |
| This file | QUICK_REFERENCE.md |

---

## Support & Resources

### Internal Wiki
- Beanstalk Overview: https://officeworks.atlassian.net/wiki/spaces/Digital/pages/3690528925/Beanstalk+Overview
- Trusted Auth Design: https://officeworks.atlassian.net/wiki/spaces/Digital/pages/3736961282/Trusted+Auth+Beanstalk

### Team
- **Support**: Web Wizards team
- **Deprecation questions**: Web Wizards team
- **Integration help**: Web Wizards team

### Repository
- **Code**: 7 git submodules
- **Clone**: `git clone --recurse-submodules <url>`
- **Update**: `git pull --recurse-submodules`

---

**Last Updated**: April 10, 2026
**Quick Ref Version**: 1.0
