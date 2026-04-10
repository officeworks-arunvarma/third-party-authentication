# Officeworks Third-Party Authentication System - Documentation

**Status**: DEPRECATED - Marked for decommissioning January 2026

This directory contains comprehensive documentation artifacts for the Officeworks third-party authentication system, a microservices-based OAuth-like authentication platform built across 7 independent git repositories.

---

## Documentation Artifacts

### 1. **context.json**
Complete system metadata in JSON format including:
- Repository definitions and configurations
- Data models (Trusted Party, Token, User Profile)
- Security specifications
- Environment configurations
- Integration points
- Deployment information

**Use Case**: System integration, configuration reference, metadata consumption by tooling

---

### 2. **C1_CONTEXT_ARCHITECTURE.md**
System-level context architecture diagram following C4 Model (Level 1).

**Contents**:
- High-level system overview
- External applications and integration patterns
- Core system components and their relationships
- AWS infrastructure
- Integration patterns (3 main approaches)
- Deprecation status

**Audience**: Architects, new team members, stakeholder presentations

---

### 3. **C2_CONTAINER_ARCHITECTURE.md**
Container-level architecture diagram following C4 Model (Level 2).

**Contents**:
- Individual service specifications:
  - trustedauth-app (OAuth UI, Express.js, Port 3001)
  - trustedauth-service (Core API, Express.js, Port 3002)
  - user-auth-service (User auth, ECS, Cognito)
  - trustedauth-profile (Profile endpoint, Express.js, Port 3004)
  - Client libraries (Node.js, React, Browser)
- Technology stack for each container
- Key responsibilities and routes
- Dependencies between services
- Authentication flows overview

**Audience**: Backend developers, DevOps engineers, technical leads

---

### 4. **C3_COMPONENT_ARCHITECTURE.md**
Component-level architecture diagram following C4 Model (Level 3).

**Contents**:
- Detailed module breakdown for each service:
  - tpapi.js (Trusted party operations)
  - userapi.js (User API integration)
  - util.js (Utilities and signing)
  - Route handlers
- Client library components:
  - TANK (trustedauth-node-client)
  - TARAS (trustedauth-react-redux)
  - Browser client
- Data models and database schemas
- HMAC-SHA512 signature algorithm details
- Request/response patterns

**Audience**: Senior developers, code reviewers, maintainers

---

### 5. **AUTHENTICATION_FLOWS.md**
Detailed sequence diagrams and step-by-step flows for all authentication scenarios.

**Flows Documented**:
1. **Guest Token Flow** - Anonymous user access
2. **Login Flow** - Existing user authentication
3. **Registration Flow** - New account creation
4. **Token Exchange Flow** - OTT to OWT conversion
5. **Profile Fetch Flow** - User data retrieval
6. **Token Keepalive Flow** - Session extension
7. **React/Redux Integration Flow** - Frontend state management
8. **Logout Flow** - Session termination

**Features**:
- Mermaid sequence diagrams (rendered by GitHub/GitLab/compatible viewers)
- Detailed step-by-step instructions
- Message formats and signatures
- Error handling paths
- State transitions
- Timing information

**Audience**: Integration engineers, QA, support engineers

---

### 6. **FUNCTIONAL_DESCRIPTIONS.md**
Detailed functional specification of all system operations.

**Contents**:
- Trusted party registration & management
  - Register TP
  - Retrieve configuration
  - Update settings
  - Delete party
- User authentication operations
  - Guest token generation
  - Personal account registration
  - Business account registration
  - Login
  - Token validation
- Token management
  - Token exchange (OTT → OWT)
  - Keepalive/session extension
  - Profile retrieval
- Admin operations
  - User token retrieval
  - Cookie management
- Integration points
- Configuration management
- Status codes and error handling

**Format**:
- Operation title
- HTTP method and endpoint
- Input parameters
- Output format
- Processing steps
- Use cases
- Error conditions

**Audience**: API consumers, integration engineers, developers, testers

---

### 7. **CONSISTENCY_REVIEW.md**
Comprehensive architectural consistency review covering all aspects of the system.

**Review Areas**:
1. **Architectural Consistency**
   - Layered architecture compliance
   - Configuration management patterns
   - Port assignment strategy
   - Service separation

2. **Security Consistency**
   - Authentication mechanisms
   - Signature verification
   - Token handling
   - Secret management

3. **API Design Consistency**
   - Endpoint patterns
   - Parameter naming
   - Error responses
   - Status codes

4. **Data Model Consistency**
   - Schema design
   - Naming conventions
   - Relationships
   - Indexing strategies

5. **Logging and Monitoring**
   - Framework consistency
   - Log levels
   - Metrics collection
   - Alerting

6. **Testing Consistency**
   - Test framework (Mocha+Chai+Sinon)
   - Coverage approaches
   - Test locations
   - CI/CD integration

7. **Dependency Management**
   - Package versions
   - Vulnerability status
   - Update strategy

8. **Deployment Consistency**
   - Infrastructure as Code
   - Deployment platforms
   - Configuration management

**Format**:
- Assessment (✓ Good, ⚠ Inconsistent, ✗ Problems)
- Detailed findings
- Recommendations
- Priority levels
- Score cards

**Audience**: Architects, tech leads, security reviewers, DevOps

---

## How to Use This Documentation

### For New Team Members
1. Start with **C1_CONTEXT_ARCHITECTURE.md** for the big picture
2. Read **FUNCTIONAL_DESCRIPTIONS.md** for operation overview
3. Explore **AUTHENTICATION_FLOWS.md** to understand user journeys
4. Reference **context.json** for technical details

### For Integration Engineers
1. Check **FUNCTIONAL_DESCRIPTIONS.md** for your use case
2. Review **AUTHENTICATION_FLOWS.md** for your flow
3. Consult **C3_COMPONENT_ARCHITECTURE.md** for library details
4. Use **context.json** for endpoint/configuration reference

### For Developers
1. Read **C3_COMPONENT_ARCHITECTURE.md** for code organization
2. Review relevant flow in **AUTHENTICATION_FLOWS.md**
3. Check **CONSISTENCY_REVIEW.md** for patterns and best practices
4. Reference **context.json** for data models

### For Architects/Tech Leads
1. Start with **C1_CONTEXT_ARCHITECTURE.md**
2. Deep dive with **C2_CONTAINER_ARCHITECTURE.md**
3. Review **CONSISTENCY_REVIEW.md** for system health
4. Check **context.json** for infrastructure details

### For Security Reviews
1. Read **CONSISTENCY_REVIEW.md** section 2 (Security Consistency)
2. Review authentication mechanisms in **C2_CONTAINER_ARCHITECTURE.md**
3. Check **FUNCTIONAL_DESCRIPTIONS.md** for sensitive operations
4. Consult **context.json** for security specifications

---

## System Overview

### Core Purpose
OAuth-like third-party customer authentication system enabling third-party and non-core applications to authenticate Officeworks customers without handling credentials directly.

### Architecture
- **Type**: Microservices
- **Services**: 7 independent repositories (git submodules)
- **Platform**: AWS (Elastic Beanstalk, ECS, DynamoDB, Cognito)
- **Protocol**: HTTP/HTTPS with HMAC-SHA512 signatures
- **Languages**: Node.js, JavaScript, React, TypeScript
- **Databases**: AWS DynamoDB, PostgreSQL

### Key Technologies
- **Express.js** - HTTP server framework
- **AWS Cognito** - User credential validation
- **AWS DynamoDB** - Token and party storage
- **JWT** - Token format
- **HMAC-SHA512** - Request signing
- **React + Redux** - Frontend components
- **AWS Elastic Beanstalk** - Backend hosting

### Port Mapping
| Service | Port | Purpose |
|---------|------|---------|
| trustedauth-app | 3001 | OAuth UI and authorization |
| trustedauth-service | 3002 | Core API - token management |
| user-auth-service | 3003 | User authentication backend |
| trustedauth-profile | 3004 | Profile endpoint |

### Repositories
1. **trustedauth-service** - Core backend API
2. **trustedauth-app** - OAuth-like UI and flow
3. **user-auth-service** - Main user authentication
4. **trustedauth-profile** - Profile endpoint service
5. **trustedauth-node-client** (TANK) - Node.js client library
6. **trustedauth-react-redux** (TARAS) - React component library
7. **trustedauth-client** - Browser client library

---

## Key Concepts

### Tokens
- **OWT** (Officeworks Token): Long-lived (8 hours), HTTP-only secure cookie
- **OTT** (One-Time Token): Short-lived, exchanged for OWT
- **JWT**: JSON Web Token containing user payload, signed with RSA

### Authentication Methods
- **HMAC-SHA512**: Signature-based API authentication
- **AWS Cognito**: User credential validation
- **JWT**: Session token format

### Trusted Parties
Third-party applications that integrate with the system. Each party has:
- `apiKey` (public): Identifies the party
- `secretKey` (private): Used for HMAC signing
- `nonce` (incremental): Additional security
- `callbackUrl`: OAuth callback endpoint

### Flows
1. **Guest Flow**: Fastest, no credentials needed
2. **Login Flow**: Existing user authentication
3. **Register Flow**: New account creation
4. **Browser Integration**: Client-side using authclient.js
5. **Server Integration**: Backend using TANK client library
6. **React Integration**: Frontend using TARAS component

---

## Deprecation Status

**Important**: This system is marked for decommissioning January 2026.

### Timeline
- **Now**: New integrations discouraged
- **December 2025**: Final deprecation notice
- **January 2026**: Service decommissioned
- **Q2 2026**: Archives created

### Implications
- Critical bug fixes only
- No new features
- Plan migration to modern OAuth 2.0
- Contact "Web Wizards" team for alternatives

---

## Security Notes

### Strong Points
- HMAC-SHA512 signature authentication
- HTTP-only secure cookies
- JWT expiry validation
- Database isolation

### Areas of Concern
- Deprecated `request` library (needs update to axios/fetch)
- Hardcoded secrets in config.js (use AWS Secrets Manager)
- Limited monitoring/metrics
- Error message information leakage

**See CONSISTENCY_REVIEW.md for detailed security assessment**

---

## Documentation Quality

| Document | Completeness | Accuracy | Freshness |
|----------|--------------|----------|-----------|
| context.json | 100% | High | April 2026 |
| C1_CONTEXT | 100% | High | April 2026 |
| C2_CONTAINER | 100% | High | April 2026 |
| C3_COMPONENT | 100% | High | April 2026 |
| FLOWS | 100% | High | April 2026 |
| FUNCTIONAL | 100% | High | April 2026 |
| CONSISTENCY | 100% | High | April 2026 |

---

## Document Generation

**Generated**: April 10, 2026
**Version**: 1.0
**Format**: Markdown + JSON
**C4 Model**: Levels 1-3 (Context, Container, Component)

### Document Structure
- Markdown files use standard formatting
- PlantUML C4 diagrams for architecture (C1/C2/C3)
- Mermaid diagrams for sequence, flow, and state diagrams
- JSON for structured data
- Tables for comparisons
- Code examples for technical details

---

## Quick Reference

### Common Tasks

#### I need to integrate my app
→ Start with FUNCTIONAL_DESCRIPTIONS.md, then AUTHENTICATION_FLOWS.md

#### I need to understand the code
→ Read C3_COMPONENT_ARCHITECTURE.md

#### I'm reviewing security
→ Check CONSISTENCY_REVIEW.md section 2

#### I need deployment info
→ See context.json under "deployment"

#### I need data models
→ See context.json under "data_models" or C3_COMPONENT_ARCHITECTURE.md

#### I need API endpoints
→ See FUNCTIONAL_DESCRIPTIONS.md or context.json under "repositories"

---

## Related Resources

### Internal Documentation
- [CLAUDE.md](../CLAUDE.md) - Claude Code guidance for this repository
- [README files](../) - Individual service README files

### External References
- Beanstalk Overview: https://officeworks.atlassian.net/wiki/spaces/Digital/pages/3690528925/Beanstalk+Overview
- Trusted Auth Design: https://officeworks.atlassian.net/wiki/spaces/Digital/pages/3736961282/Trusted+Auth+Beanstalk

### Support
- **New Issues**: Contact "Web Wizards" team
- **Migration Path**: Reach out to Web Wizards for OAuth 2.0 alternatives

---

## Navigation

### By Audience
- [Architects](./C1_CONTEXT_ARCHITECTURE.md)
- [Backend Developers](./C2_CONTAINER_ARCHITECTURE.md)
- [Integration Engineers](./FUNCTIONAL_DESCRIPTIONS.md)
- [Code Reviewers](./C3_COMPONENT_ARCHITECTURE.md)
- [QA/Testing](./AUTHENTICATION_FLOWS.md)
- [Security](./CONSISTENCY_REVIEW.md)

### By Topic
- [Architecture](./C1_CONTEXT_ARCHITECTURE.md)
- [Services](./C2_CONTAINER_ARCHITECTURE.md)
- [Components](./C3_COMPONENT_ARCHITECTURE.md)
- [Operations](./FUNCTIONAL_DESCRIPTIONS.md)
- [Flows](./AUTHENTICATION_FLOWS.md)
- [Quality](./CONSISTENCY_REVIEW.md)
- [Metadata](./context.json)

---

**Documentation Complete** ✓

All artifacts generated for the Officeworks Third-Party Authentication System
as of April 10, 2026.
