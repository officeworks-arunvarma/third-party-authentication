# C1 Context Architecture - Officeworks Third-Party Authentication System

## System Overview

The Officeworks Third-Party Authentication System is an OAuth-like authentication platform that enables third-party and non-core applications to authenticate customers. The system is built on a microservices architecture using HMAC-SHA512 signature authentication with AWS infrastructure.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          External Applications                               │
│                    (Third-Party Customers/Partners)                          │
│                                                                              │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐          │
│  │  App Instance 1  │  │  App Instance 2  │  │  App Instance N  │          │
│  │  (React/Node)    │  │  (Node.js)       │  │  (Any Stack)     │          │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘          │
│           │                      │                      │                    │
└───────────┼──────────────────────┼──────────────────────┼──────────────────┘
            │                      │                      │
            │ OAuth-like flow      │ Uses client library  │ Validates tokens
            │ (HMAC-SHA512)        │ (TANK or TARAS)      │ (signature-based)
            │                      │                      │
┌───────────▼──────────────────────▼──────────────────────▼──────────────────┐
│              Officeworks Third-Party Authentication System                  │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    trustedauth-app (Port 3001)                       │  │
│  │          OAuth-like UI and Authorization Flow Handler                │  │
│  │                                                                       │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │  │
│  │  │   Login UI   │  │ Register UI  │  │  Guest Flow  │               │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘               │  │
│  └──────────────┬───────────────────────────────────────────────────────┘  │
│                 │                                                           │
│  ┌──────────────▼───────────────────────────────────────────────────────┐  │
│  │                trustedauth-service (Port 3002)                       │  │
│  │          Core Backend API - Token Generation & Validation             │  │
│  │                                                                       │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐                  │  │
│  │  │  Trusted Party      │  │  User               │                  │  │
│  │  │  Admin Routes       │  │  Admin Routes       │                  │  │
│  │  │  /auth/tp/*         │  │  /auth/user/*       │                  │  │
│  │  └─────────────────────┘  └─────────────────────┘                  │  │
│  │                                                                       │  │
│  │  ┌─────────────────────┐  ┌─────────────────────┐                  │  │
│  │  │  Auth Routes        │  │  Token Generation   │                  │  │
│  │  │  /auth/login        │  │  & Validation       │                  │  │
│  │  │  /auth/register     │  │  /auth/token/*      │                  │  │
│  │  │  /auth/token/guest  │  │  /auth/keepalive    │                  │  │
│  │  └─────────────────────┘  └─────────────────────┘                  │  │
│  └──────────────┬───────────────────────────────────────────────────────┘  │
│                 │                                                           │
│  ┌──────────────▼───────────────────────────────────────────────────────┐  │
│  │                trustedauth-profile (Port 3004)                       │  │
│  │                  Customer Profile Service                            │  │
│  │                                                                       │  │
│  │  ┌──────────────────────────────────────────┐                       │  │
│  │  │  GET /auth/customer/profile (OWT token)  │                       │  │
│  │  │  Returns authenticated user profile      │                       │  │
│  │  └──────────────────────────────────────────┘                       │  │
│  └──────────────┬───────────────────────────────────────────────────────┘  │
│                 │                                                           │
│  ┌──────────────▼───────────────────────────────────────────────────────┐  │
│  │                   user-auth-service (AWS ECS)                        │  │
│  │            Main User Authentication & Authorization Service           │  │
│  │                                                                       │  │
│  │  ┌──────────────────────────────────────────┐                       │  │
│  │  │  AWS Cognito Integration                 │                       │  │
│  │  │  - User Pool Management                  │                       │  │
│  │  │  - Credential Validation                 │                       │  │
│  │  │  - Session Management                    │                       │  │
│  │  └──────────────────────────────────────────┘                       │  │
│  └──────────────┬───────────────────────────────────────────────────────┘  │
│                 │                                                           │
│  ┌──────────────▼───────────────────────────────────────────────────────┐  │
│  │               Client Libraries (NPM Packages)                        │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────┐  ┌──────────────────────────────┐      │  │
│  │  │ trustedauth-node-client │  │ trustedauth-react-redux      │      │  │
│  │  │ (TANK)                  │  │ (TARAS)                      │      │  │
│  │  │                         │  │                              │      │  │
│  │  │ Server-side integration │  │ React/Redux integration      │      │  │
│  │  │ - Client class          │  │ - OWAuth component           │      │  │
│  │  │ - Express middleware    │  │ - Redux middleware           │      │  │
│  │  │ - getProfile()          │  │ - Reducer                    │      │  │
│  │  │ - exchangeToken()       │  │ - Modal UI                   │      │  │
│  │  │ - validateToken()       │  │                              │      │  │
│  │  └─────────────────────────┘  └──────────────────────────────┘      │  │
│  │                                                                       │  │
│  │  ┌──────────────────────────────────────────────────────────┐       │  │
│  │  │ trustedauth-client (Browser Client)                      │       │  │
│  │  │ - authclient.min.js                                      │       │  │
│  │  │ - Custom <ow-auth> HTML element                          │       │  │
│  │  │ - Iframe-based authentication UI                         │       │  │
│  │  │ - window.postMessage communication                       │       │  │
│  │  └──────────────────────────────────────────────────────────┘       │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                 │                                        │
└─────────────────────────────────┼────────────────────────────────────────┘
                                  │
                    Signature-based request validation
                    DynamoDB operations
                    AWS service integration
                                  │
┌─────────────────────────────────▼────────────────────────────────────────┐
│                    AWS Infrastructure (ap-southeast-2)                    │
│                                                                            │
│  ┌──────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │  AWS DynamoDB                │  │  AWS Cognito                    │  │
│  │  ────────────────────────    │  │  ──────────────────────────────  │  │
│  │  - TrustedParty_Api          │  │  - User Pools                   │  │
│  │  - TrustedParty_Tokens       │  │  - User Credentials             │  │
│  │  - TrustedParty_UserToken    │  │  - Session Management           │  │
│  │  - Indexes for fast queries  │  │  - Multi-factor Authentication  │  │
│  └──────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                            │
│  ┌──────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │  AWS Elastic Beanstalk       │  │  AWS ECS                        │  │
│  │  ──────────────────────────  │  │  ──────────────────────────────  │  │
│  │  - Hosts trustedauth-service │  │  - Hosts user-auth-service      │  │
│  │  - Auto-scaling              │  │  - Container orchestration      │  │
│  │  - Load balancing            │  │  - Service discovery            │  │
│  └──────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                            │
│  ┌────────────────────────────────────────────────────────────────────┐  │
│  │  AWS S3                                                            │  │
│  │  ────────────────────────────────────────────────────────────────  │  │
│  │  - trustedauth-client CDN distribution                            │  │
│  │  - authclient.min.js serving                                      │  │
│  └────────────────────────────────────────────────────────────────────┘  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

## System Components

### 1. **External Applications Layer**
- Third-party applications integrating with Officeworks authentication
- Can be built with any technology stack (React, Node.js, etc.)
- Implement client libraries (TANK, TARAS, or browser client)

### 2. **TrustedAuth Application Layer**
- **trustedauth-app**: OAuth authorization server UI and flow orchestrator
- **trustedauth-service**: Core API backend handling token generation and validation
- **trustedauth-profile**: Profile endpoint serving authenticated user information

### 3. **Client Integration Layer**
- **trustedauth-node-client (TANK)**: Server-side Node.js integration
- **trustedauth-react-redux (TARAS)**: React/Redux component library
- **trustedauth-client**: Browser-side JavaScript library

### 4. **Core Authentication Service**
- **user-auth-service**: AWS Cognito integration and user management
- Credential validation and session management

### 5. **Data Persistence**
- AWS DynamoDB for trusted party configuration and token storage
- Tables: TrustedParty_Api, TrustedParty_Tokens, TrustedParty_UserToken

### 6. **Infrastructure**
- AWS Elastic Beanstalk for web services
- AWS ECS for containerized services
- AWS S3 for CDN distribution
- AWS CloudFormation for infrastructure as code

## Key Characteristics

| Aspect | Detail |
|--------|--------|
| **Authentication Method** | HMAC-SHA512 signature-based |
| **Authorization Pattern** | OAuth-like flow with OTT/OWT tokens |
| **Token Lifecycle** | OTT (short-lived) → OWT (8 hours, HTTP-only cookie) |
| **Data Storage** | AWS DynamoDB |
| **User Management** | AWS Cognito |
| **Deployment Target** | AWS (Elastic Beanstalk, ECS) |
| **Protocol** | HTTPS only |
| **Status** | **DEPRECATED** - Scheduled for decommissioning January 2026 |

## Request Flow with Authentication

```
Third-Party App
    │
    ├─ 1. Redirect to /auth/authorise?apiKey=XXX
    │
    ▼
TrustedAuth App
    │
    ├─ 2. Display login/register/guest UI
    │
    ▼
User Authentication
    │
    ├─ 3. Validate credentials with user-auth-service via Cognito
    │
    ▼
Token Generation (trustedauth-service)
    │
    ├─ 4. Generate JWT with user payload
    ├─ 5. Store in DynamoDB (TrustedParty_Tokens)
    ├─ 6. Generate One-Time Token (OTT)
    │
    ▼
Callback
    │
    ├─ 7. Redirect to third-party callback with OTT
    │
    ▼
Third-Party Backend (using TANK)
    │
    ├─ 8. Exchange OTT for OWT (POST /auth/token)
    ├─ 9. Sign request with HMAC-SHA512
    ├─ 10. Include apiKey, nonce, signature in headers
    │
    ▼
TrustedAuth Service
    │
    ├─ 11. Validate HMAC-SHA512 signature
    ├─ 12. Return OWT token
    │
    ▼
Third-Party App
    │
    ├─ 13. Set OWT as HTTP-only secure cookie
    ├─ 14. User is authenticated
    │
    ▼
Profile Requests
    │
    ├─ 15. GET /auth/customer/profile with OWT
    ├─ 16. Returns user profile from trustedauth-profile service
    │
    ▼
Subsequent Requests
    │
    ├─ 17. Requests include OWT cookie
    ├─ 18. Middleware/Express handler validates token
    ├─ 19. Optional: POST /auth/keepalive to extend session
```

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      AWS Region: ap-southeast-2             │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         Elastic Beanstalk Environment                  │ │
│  │                                                         │ │
│  │  ┌─────────────────┐  ┌─────────────────────────────┐ │ │
│  │  │ Load Balancer   │  │ Auto Scaling Group          │ │ │
│  │  └────────┬────────┘  │                             │ │ │
│  │           │           │ ┌───────────────────────┐  │ │ │
│  │           │           │ │ EC2 Instance (Node.js)│  │ │ │
│  │           └───────────┼─┤ trustedauth-app       │  │ │ │
│  │                       │ │ trustedauth-service   │  │ │ │
│  │                       │ │ trustedauth-profile   │  │ │ │
│  │                       │ └───────────────────────┘  │ │ │
│  │                       │                             │ │ │
│  │                       │ ┌───────────────────────┐  │ │ │
│  │                       │ │ EC2 Instance (Node.js)│  │ │ │
│  │                       │ │ trustedauth-app       │  │ │ │
│  │                       │ │ trustedauth-service   │  │ │ │
│  │                       │ │ trustedauth-profile   │  │ │ │
│  │                       │ └───────────────────────┘  │ │ │
│  │                       └─────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         ECS Cluster (user-auth-service)               │ │
│  │                                                         │ │
│  │  ┌─────────────────┐  ┌─────────────────────────────┐ │ │
│  │  │ Load Balancer   │  │ Task Definitions            │ │ │
│  │  └────────┬────────┘  │                             │ │ │
│  │           │           │ ┌───────────────────────┐  │ │ │
│  │           │           │ │ ECS Task (Container)  │  │ │ │
│  │           └───────────┼─┤ user-auth-service     │  │ │ │
│  │                       │ │ Port 3003             │  │ │ │
│  │                       │ └───────────────────────┘  │ │ │
│  │                       │                             │ │ │
│  │                       │ ┌───────────────────────┐  │ │ │
│  │                       │ │ ECS Task (Container)  │  │ │ │
│  │                       │ │ user-auth-service     │  │ │ │
│  │                       │ │ Port 3003             │  │ │ │
│  │                       │ └───────────────────────┘  │ │ │
│  │                       └─────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         Data Services                                 │ │
│  │                                                         │ │
│  │  ┌─────────────────────┐ ┌─────────────────────────┐  │ │
│  │  │  DynamoDB           │ │ Cognito User Pool       │  │ │
│  │  │                     │ │                         │  │ │
│  │  │ - TrustedParty_Api  │ │ - User credentials      │  │ │
│  │  │ - TrustedParty_Token│ │ - Session management    │  │ │
│  │  │ - UserToken         │ │ - MFA options           │  │ │
│  │  └─────────────────────┘ └─────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         Content Distribution                          │ │
│  │                                                         │ │
│  │  ┌─────────────────────────────────────────────────┐  │ │
│  │  │  S3 Bucket + CloudFront CDN                     │  │ │
│  │  │  - authclient.min.js                           │  │ │
│  │  │  - trustedauth-client distribution             │  │ │
│  │  │  - Endpoint: s3-ap-southeast-2.amazonaws.com   │  │ │
│  │  └─────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Integration Patterns

### Pattern 1: Browser-Based Integration
1. Load authclient.min.js from CDN
2. Add `<ow-auth>` custom element to page
3. User completes authentication in iframe
4. window.postMessage communicates result to parent
5. Third-party sets OWT cookie

### Pattern 2: Server-Side Integration (Node.js)
1. Implement using TANK (trustedauth-node-client)
2. Create AuthClient with apiKey, serverUrl, secret
3. Use exchangeToken(ott) to get OWT
4. Use getProfile(owt) to fetch user data
5. Apply expressMiddleware for protected routes

### Pattern 3: React/Redux Integration
1. Use TARAS (trustedauth-react-redux)
2. Inject OWAuth component at root level
3. Setup Redux middleware for auto-login
4. Dispatch actions for login/register
5. User profile automatically synced to Redux state

---

**Status Note**: This system is marked for deprecation and will be decommissioned in January 2026. No new systems should integrate with this service. Reach out to Web Wizards for alternative authentication solutions.
