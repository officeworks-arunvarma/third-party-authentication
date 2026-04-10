# Functional Descriptions - Officeworks Third-Party Authentication System

## System Purpose

The Officeworks Third-Party Authentication System is an OAuth-like authentication platform that enables third-party and non-core applications to authenticate Officeworks customers. It provides a secure mechanism for partner applications to issue, validate, and manage authentication tokens without directly handling customer credentials.

The system sits between a partner application and Officeworks' core identity platform. Rather than giving partners direct access to the user database or requiring them to build their own login screens from scratch, this system acts as a trusted intermediary: it handles the credential check, creates a short-lived proof of authentication, and hands that proof back to the partner application so it can establish a session on its own domain.

**Status**: DEPRECATED - Scheduled for decommissioning January 2026. No new integrations should be built against this system. See the Deprecation section at the end of this document for migration guidance.

---

## Core Functions

### 1. Trusted Party Registration & Management

**Functional Area**: Core Backend Service - Administration Routes

**Purpose**: Manage third-party applications that are authorised to integrate with the authentication system.

Before any partner application can use the authentication system, it must be registered. Registration is a one-time administrative act carried out by the Web Wizards team. It creates a relationship of trust between Officeworks and that partner, and produces the credentials (an API key and a secret key) that the partner will use in every subsequent interaction.

Think of this like issuing a contractor's access card: the card (API key) identifies who is knocking, and the PIN (secret key) proves the person holding the card is genuinely who they claim to be.

---

#### 1.1 Register Trusted Party

**What it does**: Creates a new trusted party record in the system and returns the credentials that the partner application will need to operate.

**Who can do this**: Only the Web Wizards team, using a protected admin key. This operation is not exposed to partner developers.

**What you provide**:
- A display name for the third party (e.g. "ACCIS Loyalty Platform")
- The callback URL — the web address on the partner's server that the system will redirect to after a successful authentication. This address is registered in advance as a security measure; the system will refuse to redirect anywhere else.
- A nonce — a one-time security value that further validates the request

**What you get back**: A unique Party ID, a public API key (used to identify the partner in all requests), and a secret key (used only server-side to sign requests — it must never be shared publicly or stored in client-side code).

**Why this matters**: Centralised registration means Officeworks retains full control over which applications can trigger authentication flows on its behalf. If a partner application is decommissioned, compromised, or changes ownership, access can be immediately revoked without touching any customer accounts.

**Business scenario**: The team responsible for the in-store kiosk ordering tool needs to add a "sign in or create account" feature. Web Wizards registers the kiosk application as a trusted party, receives an API key and secret, and hands those credentials to the kiosk development team. From that point on, the kiosk can initiate authenticated sessions without any direct access to Officeworks customer data.

---

#### 1.2 Retrieve Trusted Party Configuration

**What it does**: Looks up and returns the full configuration record for a registered trusted party.

**Who can do this**: Web Wizards team only, using the admin key.

**How to look it up**: Either by the Party ID (the numeric identifier assigned at registration) or by the public API key (useful when you have a key but need to find out which application it belongs to).

**Why this matters**: If a partner reports that their integration has stopped working, the first diagnostic step is confirming that their credentials still exist and that their registered callback URL matches what they are actually using. This lookup provides that information instantly. It is also used to audit which applications currently hold active credentials.

**Edge cases**: If the Party ID or API key does not exist in the system, the lookup returns an empty result rather than an error, so the caller must check whether a record was actually returned. A blank result effectively means the credentials are invalid or were never created.

---

#### 1.3 Update Trusted Party

**What it does**: Modifies an existing trusted party's configuration — their callback URL, display name, or security nonce — without revoking their credentials.

**Who can do this**: Web Wizards team only.

**When this is needed**:
- A partner migrates their platform to a new domain and their callback URL changes
- A partner application is rebranded and the display name needs to reflect the new name
- A security review requires rotating the nonce value

**What changes immediately**: Updates take effect for all new token requests as soon as the record is saved. Tokens that were already issued before the update are not affected.

**Why this matters**: Partner applications do not always control their domain names — they may be part of a larger platform migration. Allowing the callback URL to be updated without forcing a full re-registration avoids unnecessary disruption to active integrations.

**Edge case**: If the new callback URL is unreachable or malformed, the system will save it without verifying that it works. The error would only surface later when the system attempts to redirect a user to it. For this reason, Web Wizards should confirm the new URL is live before updating it.

---

#### 1.4 Delete Trusted Party

**What it does**: Permanently removes a trusted party from the system, preventing any further token issuance under their credentials.

**Who can do this**: Web Wizards team only.

**Immediate effect**: As soon as the record is deleted, the partner's API key is invalid. Any attempt by their application to request a guest token, trigger a login, or register a user will be rejected with an "Invalid API key" response.

**What it does not do**: Tokens that were already issued and are not yet expired remain valid until they naturally expire (within 8 hours). Deletion does not forcibly log out users who are mid-session. If immediate session termination is required for all active users, that would need to be coordinated separately.

**Why this matters**: Partner relationships end. Applications get retired. Credentials that are no longer actively maintained are a security liability — they represent an open door into the authentication system even if nobody is using them. Regular cleanup of unused registrations reduces the attack surface.

**Business scenario**: A third-party integration pilot is run for three months, then cancelled. The partner's registration is deleted. Even if the partner still has their API key and secret key on file, they can no longer use them to authenticate Officeworks customers.

---

### 2. Guest Authentication

**Functional Area**: Core Backend Service - Guest Token Route

**Purpose**: Enable third-party applications to establish a meaningful session for anonymous users — users who have not logged in and have not created an account — without requiring any personal information from them.

A "guest" in this context is a real user interacting with a partner application who has chosen not to identify themselves. The system creates a lightweight, temporary identity for that session so that the partner application can track the session, offer a shopping basket, or enable browsing features, all without forcing the user to register.

---

#### 2.1 Request Guest Token

**What it does**: Issues a guest authentication token to a registered partner application on behalf of an anonymous user.

**What the partner provides**: Their API key, identifying which registered application is making the request.

**What the system does**:
1. Confirms the API key belongs to a registered, active trusted party
2. Requests a guest identity from the core authentication service
3. Creates a session record — a pairing of the guest identity and the partner application — and stores it securely
4. Generates two tokens: a One-Time Token (OTT) and an Officeworks Token (OWT). These are described in detail in Section 5.

**What is returned**: Both tokens, plus the identifier that has been assigned to this anonymous user for the duration of their session.

**Why this matters**: Forcing every user to register before they can interact with an application is a significant conversion barrier. Guest access allows a user to browse, add items to a basket, or explore features before committing to an account. If and when they do register or log in later in the same session, the session can be upgraded to an authenticated one.

**Business scenario**: A customer visits the Officeworks website through a partner landing page. They do not want to log in yet. The partner application requests a guest token, receives it, and uses it to maintain a shopping basket for the user across page loads. The customer browses, selects items, then proceeds to checkout where they are prompted to sign in — at which point the session transitions to a full authenticated session.

**Edge cases**:
- If the API key is not recognised, the request is rejected immediately and no guest identity is created.
- If the upstream guest token service is temporarily unavailable, the request fails. The partner application should handle this gracefully (for example, by retrying after a short delay or displaying a friendly message rather than a raw error).
- Guest tokens are subject to the same 8-hour expiry as standard session tokens. A user who leaves a guest session open for more than 8 hours will need to start a new session.

---

### 3. Account Registration

**Functional Area**: Core Backend Service - Registration Routes

**Purpose**: Enable third-party applications to create new Officeworks customer accounts — either personal or business — from within their own interface, without directing the user away to the Officeworks website.

When a partner application is trusted to register users, Officeworks effectively delegates the account creation experience to that partner. The partner collects the user's details, the system validates them, creates the account in Officeworks' identity platform, and immediately logs the user in — all in one flow.

---

#### 3.1 Register Personal Account

**What it does**: Creates a new personal Officeworks customer account and simultaneously logs the new user in, returning session tokens ready for use.

**What the partner provides**: The user's email address, chosen password, first name, last name, and phone number, along with their API key.

**Validation performed**:
- The email address must be in a valid format
- The password must meet the strength requirements configured for the system
- The email address must not already be registered in Officeworks' identity platform (if it is, registration is rejected and the user should be directed to log in instead)

**What happens on success**: The account is created in Officeworks' identity platform (AWS Cognito). A confirmation email is sent to the user. The user is immediately logged in — no separate login step is required. Session tokens are returned to the partner application.

**Why this matters**: A registration flow that requires the user to leave the partner application, complete a separate Officeworks registration, and then return is a poor experience and leads to abandonment. Embedded registration keeps the user within the partner's interface throughout, which improves conversion and reduces drop-off.

**Business scenario**: A corporate procurement portal integrated with Officeworks wants to let new employees create personal accounts as part of their onboarding flow. The portal presents a registration form, collects the required fields, and submits them. The system creates the account and returns session tokens. The employee is now logged in and can immediately begin placing orders, without visiting the Officeworks website at all.

**Edge cases and failure scenarios**:
- If the email address is already registered, the system signals that the account already exists. The partner application should surface a message like "An account with this email already exists — please sign in instead."
- If the password does not meet the strength requirements, the system returns a clear error indicating this. The partner should ideally validate password strength on the client side as well to catch this before submission.
- If the Officeworks identity platform is temporarily unavailable, the registration attempt fails. The system does not create a partial account; the failure is clean and the user can retry.
- Confirmation emails are sent by the Officeworks platform automatically. The partner application does not need to handle this, but should be aware that users may need to check their inbox to complete email verification for certain features.

---

#### 3.2 Register Business Account

**What it does**: Creates a new Officeworks business customer account with associated company details, and simultaneously logs the new user in.

**What the partner provides**: The company name, Australian Business Number (ABN), the business email address, a password, and the primary contact name.

**ABN validation**: The system applies the Australian Government's published ABN checksum algorithm. This is not just a format check — it mathematically verifies that the 11-digit number is a plausible, legitimately structured ABN. A number that does not pass this check is rejected before any account creation is attempted. Note that passing this check does not confirm the business is currently registered with the ABR (Australian Business Register); it only confirms the number is structurally valid.

**What a business account unlocks**: Business accounts have access to Officeworks' business-specific features, including business pricing, thirty-day account terms (where applicable), and B2B purchasing workflows. The `userType` of `BUSINESS` on the returned session token signals to any downstream system that this user should receive the business experience.

**Why this matters**: B2B customer acquisition through partner channels requires the partner to be able to onboard business customers directly. Without embedded business registration, a procurement platform or reseller portal would have to redirect every new business customer to the Officeworks website, fragmenting the onboarding journey.

**Business scenario**: A facilities management company uses a third-party spend management platform that integrates with Officeworks. A new subsidiary company needs to be set up with an Officeworks business account. The spend management platform presents a business registration form, the user enters their ABN and company details, and the system creates a business account immediately. The user is logged in with a business-type session and can begin purchasing on business terms.

**Edge cases and failure scenarios**:
- If the ABN fails the checksum validation, the system rejects the request with an explanation. The partner should display a clear message: "Please check your ABN — the number entered does not appear to be valid."
- If an account already exists for the provided email address, the system returns an "account exists" error. The partner should redirect the user to the login flow.
- The ABN itself is not unique across accounts; the uniqueness constraint is on email address. Two different contacts within the same company would register separately using the same ABN but different email addresses.

---

### 4. User Login

**Functional Area**: Core Backend Service - Login Route

**Purpose**: Authenticate an existing Officeworks customer — verifying their identity against stored credentials — and issue a session token that the partner application can use to identify and serve that user.

---

#### 4.1 Login with Credentials

**What it does**: Accepts a registered user's email address and password, verifies them against Officeworks' identity platform, and returns session tokens if the credentials are valid.

**What the partner provides**: The user's email address, their password, and the partner's API key.

**What the system checks**:
1. That the API key belongs to a registered, active trusted party
2. That the email address is in a valid format
3. That a password has been provided
4. Against Officeworks' identity platform: that an account exists for that email, that the password is correct, that the account has been email-confirmed, that the account is not suspended, and whether multi-factor authentication is configured

**On a successful login**: The system determines whether the account is a personal or business account, creates a session record, generates both a One-Time Token and an Officeworks Token, stores the session, and returns the tokens to the partner application. A record of the login event is written to the audit log.

**Why this matters**: Centralising credential verification in a dedicated authentication service — rather than having each partner application call Cognito directly — means Officeworks can enforce consistent security policy (rate limiting, lockout thresholds, MFA requirements) across all partner integrations simultaneously. A change to the security policy only needs to happen in one place.

**Business scenario**: A customer opens a retail partner's app on their phone and taps "Sign in with your Officeworks account." They enter their email and password. The partner's app sends these credentials to the authentication system, which verifies them and returns session tokens. The user is now signed in; the partner's app can display their name, preferences, and account status.

**Failure scenarios explained in plain terms**:

| What went wrong | What the user experiences (suggested messaging) |
|----------------|------------------------------------------------|
| Email address not found in the system | "We couldn't find an account with that email address." |
| Password does not match | "Incorrect password. Please try again." (Note: both this and the above return the same generic response from the system, to prevent confirming whether an email is registered.) |
| Account not yet confirmed | "Please check your email and click the confirmation link before signing in." |
| Account has been suspended | "Your account has been suspended. Please contact Officeworks support." |
| Multi-factor authentication is required | The system returns a challenge response containing the MFA prompt. The partner application must present this to the user and collect their MFA code before the login can complete. |
| Too many failed login attempts | The account is temporarily locked to protect against brute-force attacks. The user should wait before trying again or use the password reset flow. The partner application should surface a message like "Too many sign-in attempts. Please wait a few minutes and try again." |

**Important note on error messaging**: The system intentionally returns the same generic "Invalid credentials" response for both "email not found" and "wrong password" scenarios. This is a deliberate security measure: confirming which of the two is wrong would help an attacker enumerate valid email addresses. Partner applications should not attempt to provide more specific feedback than this.

---

### 5. Token Validation

**Functional Area**: Core Backend Service - Token Validation Routes

**Purpose**: Provide partner applications with a reliable way to check whether a session token is still valid before acting on it, and to securely exchange a short-lived one-time token for a durable session token.

**Background — understanding the two token types**:

The system uses two types of tokens that work together:

- **Officeworks Token (OWT)**: This is the main session credential. It is a cryptographically signed token that contains the user's identifier and account type. It is valid for 8 hours. It is stored in an HTTP-only cookie on the user's browser, which means client-side JavaScript cannot read it — only the server can.

- **One-Time Token (OTT)**: This is a short-lived, single-use token generated at the moment of login, registration, or guest access. Its purpose is to bridge the gap between the authentication service and the partner's own backend. Once the partner's server has received and exchanged the OTT for an OWT, the OTT is marked as used and can never be redeemed again.

The reason for this two-token design is security: the OWT (the durable credential) is only ever transmitted via a secure server-to-server exchange that requires the partner's secret key. It is never passed through the browser URL or accessible to client-side JavaScript.

---

#### 5.1 Validate Token

**What it does**: Checks whether a given session token (OWT) is currently valid — that it is genuine, has not been tampered with, and has not expired — and returns confirmation along with the user's identity.

**What the partner provides**: The session token to be validated.

**What the system checks**:
1. That the token's cryptographic signature is valid (i.e., it was genuinely issued by this system and has not been modified)
2. That the token has not passed its expiry time
3. If a One-Time Token is being validated: that it has not already been used

**What is returned**: A clear yes/no answer (valid or not), and if valid: the user's ID, their account type (Guest, Personal, or Business), when the token was issued, when it expires, and how many seconds of validity remain.

**Why this matters**: Partner applications should not blindly trust a token that arrives with a request — they need to confirm it is still valid before taking any action that depends on the user's identity. Calling this endpoint before serving sensitive content or processing a transaction is good practice and protects against session-hijacking scenarios where a stale token might be replayed.

**Business scenario**: A partner's server receives a request to display a user's order history. Before returning any data, the server calls the validate endpoint with the session token from the user's cookie. The response confirms the token is valid, has 3 hours of remaining life, and belongs to a business-type user. The server proceeds to return the appropriate order history for that user type.

**Edge cases**:
- A token that is structurally valid but was issued by a different system (or a forgery) will fail the signature check and be reported as invalid.
- A token that was valid yesterday but has since expired is reported as invalid. The remaining time in the response will be zero or negative.
- If the token string is malformed or truncated, the validation returns invalid rather than raising an error, so the partner application can handle it gracefully.

---

#### 5.2 Exchange One-Time Token for Session Token

**What it does**: Accepts a One-Time Token from a partner's backend server, confirms it is genuine and unused, and returns the corresponding session token (OWT) so the partner can establish a cookie-based session for the user.

**What the partner provides**: The OTT (received from the authentication callback), the partner's API key, and a cryptographic signature proving the request is genuinely from the partner's own server.

**Why the signature is required here**: This is the most security-sensitive exchange in the entire flow. The OTT is the bridge between a successful authentication event and the partner's session. If an attacker could intercept an OTT and exchange it themselves, they could steal a user's session. Requiring the HMAC signature — which can only be generated by someone who holds the partner's secret key — means this exchange can only be completed by the partner's own server. Client-side code (JavaScript running in the browser) cannot perform this exchange, because the secret key must never be present in client-side code.

**The single-use guarantee**: The moment an OTT is exchanged for an OWT, the OTT is permanently marked as used. Any subsequent attempt to exchange the same OTT — whether by an attacker or an accidental retry — will be rejected. This is true even if the OWT has not yet expired.

**Why this matters**: This token exchange mechanism is what allows the authentication system to work across different domains. The partner's UI can live on a completely separate domain from the Officeworks authentication service, and the secure handoff happens server-to-server without ever exposing credentials in a browser URL or accessible JavaScript.

**Business scenario**: A user completes a login flow within the partner's embedded authentication widget. The authentication service calls the partner's registered callback URL with the OTT appended. The partner's server receives this callback, immediately calls the exchange endpoint (with its secret-signed request), receives the OWT in return, and sets it as an HTTP-only cookie on the user's browser. The OTT has now been consumed and is worthless to anyone who might have seen it in a server log.

**Failure scenarios**:
- If the OTT has already been used, the exchange is rejected. This would typically indicate a retry that should not happen; if it is happening regularly, it suggests an integration issue in the partner's callback handler.
- If the OTT has expired (which happens within minutes of issuance if not exchanged), the exchange is rejected. The user would need to start the authentication flow again.
- If the signature does not match — meaning the request was not signed with the correct secret key — the exchange is rejected. This protects against both accidental misconfiguration and deliberate interception attempts.

---

### 6. Session Keepalive

**Functional Area**: Core Backend Service - Keepalive Route

**Purpose**: Extend an active user's session so that users engaged in long working sessions are not unexpectedly logged out.

---

#### 6.1 Refresh Session

**What it does**: Resets the expiry clock on a user's existing session, extending it by a fresh 8-hour window from the current moment.

**What the partner provides**: The user's current session token.

**What the system does**: Confirms the token is still valid (an expired token cannot be refreshed — the user must log in again), then updates the expiry time in the session store and returns an updated token.

**Why this matters**: Sessions expire after 8 hours by design — this limits how long a stolen or abandoned session token can be misused. However, 8 hours is not long enough for users who spend an extended working day in an application without stopping. Keepalive allows the application to silently renew the session in the background as long as the user is genuinely active, without requiring them to log in again. The key is "genuinely active" — the partner application is expected to call keepalive only when there is evidence of real user activity, not simply on a timer regardless of usage.

**Typical usage**: The partner application monitors user activity (mouse movement, keystrokes, page navigation). If activity is detected within a given window — typically every 30 minutes — the application calls the keepalive endpoint. If the user has been idle for an extended period, the keepalive is not called, and the session will expire naturally, requiring the user to log in again when they return.

**Business scenario**: A procurement officer is building a large purchase order in a partner's platform. The task takes 2.5 hours. Without keepalive, their session would expire mid-task, losing their work. The partner application detects ongoing activity and calls the keepalive endpoint periodically. The session stays alive throughout the task. When the procurement officer steps away for lunch and comes back 3 hours later, the session has expired and they need to log in again — which is the intended behaviour.

**Edge cases**:
- A token that has already expired cannot be refreshed. Calling keepalive with an expired token will return an error. The user must re-authenticate.
- Keepalive does not change the user's identity, account type, or any other session attribute — it only extends the expiry window.

---

### 7. User Profile Retrieval

**Functional Area**: Profile Service - Profile Route

**Purpose**: Return detailed profile information for the currently authenticated user so that partner applications can personalise their experience and make decisions based on account type and status.

---

#### 7.1 Get User Profile

**What it does**: Validates the user's session token, extracts their identity, retrieves their full profile from Officeworks' customer system, and returns that profile to the requesting application.

**What the partner provides**: The user's session token, either as an HTTP cookie or as a request header.

**What is returned**:
- Name (first, last, and display name)
- Email address
- Phone and mobile numbers
- Account type (Personal or Business)
- Business Partner codes — internal identifiers used in Officeworks' SAP-based systems for B2B operations
- Whether the account has a thirty-day trading account (a credit arrangement that business customers can hold)

**Why this matters**: The profile data enables partner applications to deliver a personalised, context-aware experience without maintaining their own copy of customer data. Rather than asking users to re-enter information the system already holds, the partner can retrieve it in real time. The account type and thirty-day account flag are particularly important for B2B partner applications, as they determine which pricing tiers, payment options, and purchasing limits apply to that user.

**Business scenario — personalisation**: A partner's homepage displays "Welcome back, John" and pre-populates the user's delivery address from their profile, removing friction from repeat purchases.

**Business scenario — B2B decision making**: A corporate procurement platform checks the `userType` and `hasThirtyDayAccount` fields after login. If both indicate a business account with credit terms, the platform enables a "Pay on account" option at checkout. If neither is present, that option is not offered.

**Business scenario — account management**: An internal customer service tool retrieves the `custBP` and `orgBP` fields to look up the customer's record in the Officeworks ERP system, so the agent can see full order history and account standing.

**Edge cases**:
- If the session token has expired, the profile endpoint returns an authentication failure. The partner application should treat this as a signal to redirect the user to a login screen.
- If the upstream customer profile service is temporarily unavailable, the profile endpoint returns a service unavailable response. The partner application should handle this gracefully — for example, by displaying cached profile data (if available) or a message asking the user to try again shortly. It should not log the user out.
- Certain profile fields may be absent for some users. For example, a purely personal account will not have `orgBP` or `hasThirtyDayAccount` populated. Partner applications must handle optional fields defensively.

---

### 8. Admin Operations

**Functional Area**: Core Backend Service - Admin Routes

**Purpose**: Provide internal tooling with the ability to retrieve the underlying authentication credentials associated with a user's session, for use cases where downstream systems need to act on behalf of an authenticated user.

These operations are protected by both an agent token and the admin key. They are intended for internal Officeworks tools and automated processes — not for use by external partner applications.

---

#### 8.1 Get User Token from Session

**What it does**: Given an active session token, retrieves the underlying upstream user token — the credential that internal Officeworks systems use to identify the user when calling core APIs.

**Why it exists**: The session token (OWT) that partner applications handle is a wrapper — it is issued by this system and understood by this system. But the core Officeworks services (order management, loyalty, etc.) work with a different token format issued by the central authentication service. This endpoint bridges the two, allowing an internal tool that has received a session token to obtain the credential it needs to call those core services.

**Who uses this**: Internal Officeworks tooling, customer service agents operating tools that require acting on behalf of a customer, and automated processes that need to chain authenticated calls across services.

**Why this matters**: Without this bridge, every internal tool would need to implement its own authentication flow. Centralising the translation in a single endpoint means the internal tooling ecosystem can remain aligned with however the core authentication service evolves, without each tool needing to be updated individually.

---

#### 8.2 Get WC Cookies from Session

**What it does**: Retrieves the set of web client cookies associated with an authenticated session. These cookies are used by legacy Officeworks web systems built on the WooCommerce platform.

**Why it exists**: Some Officeworks systems pre-date the current authentication architecture and expect authentication to be presented as specific cookie values that the older platform issued. This endpoint allows integrations with those legacy systems to work alongside the modern token-based flow by retrieving the legacy credentials when needed.

**Who uses this**: Integrations that must interact with legacy WooCommerce-based Officeworks systems. This is expected to be a transitional use case; as legacy systems are modernised, the need for this endpoint diminishes.

---

## Cross-Cutting Concerns

### Logging and Audit Trail

Every meaningful action in the system — from a login attempt to a token exchange to a trusted party being deleted — is written to a structured log. These logs serve two distinct purposes: operational monitoring (is the system working?) and security auditing (who did what, and when?).

**What is recorded for every request**: A unique request identifier (so individual requests can be traced end-to-end across services), the type of operation, whether it succeeded or failed, and how long it took to complete. For authentication events specifically, the outcome is always recorded — successful logins, failed login attempts, token issuances, and token validations are all captured.

**Why this matters for the business**:

For security and compliance, the audit trail is the evidence layer. If a question arises about whether a particular user's account was accessed without authorisation, or whether a partner application behaved unexpectedly, the logs provide the forensic record. Without this trail, it would be impossible to reconstruct what happened after the fact.

For operations, the logs are the first diagnostic tool when something goes wrong. If a partner reports that their users cannot log in, the operations team can search the logs by the partner's API key or by the affected user's email address to see exactly what the system received, what it checked, and where it failed — without needing to reproduce the issue.

For pattern detection, monitoring tools can look for anomalies in the log data: a sudden spike in failed login attempts (indicating a credential-stuffing attack), an unusual number of guest tokens being requested (potentially indicating abuse), or a sharp increase in token validation failures (suggesting an integration has broken after a release).

**Sensitive data handling**: Request logs are sanitised before writing. Passwords are never logged. API keys in headers are truncated or masked. The logs are safe to forward to centralised monitoring infrastructure without exposing user credentials.

**Log verbosity by environment**: In development and test environments, the logs are verbose — capturing every processing step in detail to help developers trace problems. In production, the log level is reduced to capture meaningful events without generating excessive noise or storage costs. This distinction is important: a detailed log trace that is invaluable for debugging in test would generate enormous storage volumes in a production system handling real traffic.

---

### Security

The security model for this system is built around the principle that customer credentials — passwords, session tokens, and signing secrets — should be handled as few times as possible and stored as safely as possible, with multiple independent controls preventing misuse.

**Why the HMAC signature requirement matters**: Every time a partner application makes a request that modifies state (issuing a token, exchanging an OTT), that request must be accompanied by a cryptographic signature generated using the partner's secret key. This signature proves two things simultaneously: that the request came from the registered partner (not an impersonator), and that the request content has not been altered in transit. An attacker who intercepts a valid request cannot replay it or modify it without invalidating the signature. The secret key itself is never transmitted in any request — it is used locally to generate the signature, which is what travels over the network.

**Why the nonce prevents replay attacks**: Even with a valid signature, an attacker who captures a request could theoretically replay it (send it again) to trigger the same action twice. The nonce — a random value included in the request and tracked by the system — prevents this. Once a request with a given nonce has been processed, the system will reject any further request carrying the same nonce, even if the signature is valid.

**Why HTTP-only cookies protect against browser-based attacks**: Session tokens (OWTs) are stored in HTTP-only cookies. This means the browser will send them automatically with every relevant request, but JavaScript running on the page cannot read them. This is a direct defence against cross-site scripting (XSS) attacks: even if an attacker manages to inject malicious JavaScript into a partner's page, that script cannot read or steal the session token. The token remains safe in a place that only the server can access.

**Why single-use tokens limit the blast radius of interception**: The One-Time Token is designed to be interceptable without being dangerous. Even if an attacker sees the OTT (for example, in a server access log where the callback URL was recorded), it is useless to them: it can only be exchanged for an OWT by the registered partner's server using the HMAC signature. And once exchanged, it is immediately invalidated. The window of vulnerability for a stolen OTT is the seconds between issuance and exchange — deliberately kept as short as possible.

**Why rate limiting protects customer accounts**: The system imposes limits on how many failed login attempts can occur against a single account within a given time window. After that threshold is crossed, the account is temporarily locked. This is a defence against automated credential-stuffing and brute-force attacks, where attackers use large lists of stolen email/password combinations to try to access accounts. A real user who genuinely forgot their password will typically fail a small number of times; an automated attack will fail at a much higher rate, which triggers the lockout.

**What this means for partners**: Partners who build retry logic into their login flows must be careful not to automatically retry failed logins — doing so on behalf of a user who enters the wrong password could trigger the lockout threshold faster than expected, affecting the user's ability to log in.

---

### Error Handling

When something goes wrong, the system always returns a structured error response that includes a human-readable description of the problem and the unique request identifier. The request identifier is important: it allows the operations team to locate the exact log entries for a failing request without the partner needing to describe what happened. Partners should surface the request ID in any error reports or support tickets.

**How errors are categorised for integration purposes** (described by meaning, not code):

- **Bad request**: The data provided was incomplete or incorrectly formatted. The partner application's input validation should prevent most of these from reaching production.
- **Authentication failed**: The credentials provided (API key, user password, or token) were not valid. The response will not reveal whether the specific field was wrong, to prevent information leakage.
- **Forbidden**: The request was valid and the credentials were recognised, but this operation is not permitted. This occurs for admin operations attempted without admin credentials, or for operations that have been restricted for this partner.
- **MFA required**: The login was otherwise valid but the account has multi-factor authentication enabled. The system returns a challenge that the partner application must present to the user before the login can be completed.
- **Rate limit exceeded**: Too many requests of this type have been made in a short period. The partner application should back off and retry after a delay.
- **Server error**: Something unexpected went wrong within the system. These are monitored automatically and investigated by the operations team.
- **Service unavailable**: A downstream service that this system depends on (such as the Officeworks identity platform or the profile API) is temporarily unreachable. These are transient and the operation can typically be retried after a short delay.

---

## Integration Methods

Partners integrating with this authentication system have three distinct options for how they connect. The right choice depends on the partner's technical architecture, the nature of their application, and their security requirements. Each method represents a different balance of flexibility, security, and development effort.

### Option A: Browser Client Library (authclient.js)

This is a pre-built JavaScript library that partners include in their web pages. It provides a ready-made authentication UI — login, registration, and guest access — delivered in an iframe embedded within the partner's page. The library handles all communication with the authentication service internally, using a browser security mechanism (window.postMessage) to pass results back to the host page.

**How it works in practice**: The partner adds a single HTML tag (`<ow-auth>`) to their page and includes the library script. The authentication widget appears within their page. When a user completes a login or registration, the library fires an event in the host page with the resulting tokens, which the partner's own JavaScript can then handle (typically by sending them to the partner's backend server to establish a session).

**When to choose this**: This option is best suited for partner applications that are primarily browser-based, have limited backend development capacity, and want to get authentication working quickly without writing complex integration code. It is also appropriate when the partner wants the Officeworks-designed login UI rather than building their own.

**Limitations and trade-offs**: Because this approach runs in the browser, it cannot use HMAC request signing (the secret key must never appear in client-side code). This means it relies on the browser's same-origin security model rather than cryptographic signing. It is less suitable for scenarios where strict server-side control over the authentication flow is required, and it is not appropriate for automated or server-to-server flows where there is no browser involved.

**Typical user**: A content management or marketing platform that wants to offer "Sign in with your Officeworks account" as a feature but has a small development team and no dedicated backend service.

---

### Option B: Node.js Server-Side Client (TANK)

This is an installable software package (`@ow/trustedauth-node-client`) for partner applications that have a Node.js backend server. It provides a programmatic interface for performing all authentication operations: exchanging tokens, validating sessions, retrieving profiles, and signing requests correctly.

**How it works in practice**: The partner's backend server installs the package and configures it with their API key and secret key. When the partner's application needs to perform an authentication action — for example, exchanging an OTT received at a callback URL for an OWT — it calls the appropriate method on the client. The client handles the HMAC signing, the HTTP communication with the authentication service, and error handling. The partner's server then sets the resulting OWT as an HTTP-only cookie in the user's browser.

The package also provides an Express middleware component. When added to a route handler, this middleware automatically validates the OWT cookie on every incoming request to that route, and either proceeds (if the token is valid) or redirects to a login flow (if it is not). This saves the partner from writing repetitive token-checking code across every protected endpoint.

**When to choose this**: This option is appropriate for partner applications with a Node.js backend that want full server-side control over the authentication flow. It is the most secure option because all sensitive operations — token exchanges, signed requests — happen on the server and the secret key is never exposed to the browser. It is also the right choice for automated workflows where there is no human user interacting with a browser (for example, a service-to-service integration that needs to operate as an authenticated user).

**Limitations and trade-offs**: Requires Node.js on the partner's backend. Partners using other server technologies (Java, Python, .NET, etc.) would need to implement the HMAC signing and token exchange logic themselves using this document as a reference.

**Typical user**: A React single-page application backed by a Node.js API server, where the server handles the secure token exchange and sets cookies, and the React frontend uses those cookies for subsequent authenticated requests.

---

### Option C: React and Redux Component Library (TARAS)

This is an installable software package (`@ow/trustedauth-react-redux`) for partner applications built with the React framework and using Redux for state management. It provides pre-built React components for login and registration forms, a Redux reducer to manage authentication state within the application, and Redux middleware that intercepts actions requiring authentication and triggers the appropriate flow automatically.

**How it works in practice**: The partner integrates the library's components into their React application and registers the provided Redux middleware and reducer. From that point, when the application dispatches an action that requires an authenticated user — such as "add item to order" or "view account details" — the middleware automatically checks whether a valid session exists. If not, it triggers the login flow (showing a modal or redirecting to a login screen), waits for the user to authenticate, and then completes the original action. The partner does not need to write this authentication-gating logic themselves.

**When to choose this**: This option is purpose-built for React/Redux applications and provides the highest degree of native integration with that architecture. It is the right choice when the partner's application is already using React and Redux, and when the partner wants authentication state to be a first-class part of their application's data model (rather than being handled separately). It delivers the smoothest user experience for React applications because authentication flows trigger contextually in response to user actions rather than requiring explicit redirects.

**Limitations and trade-offs**: This option is only relevant for React applications using Redux. A React application without Redux, or an application using a different framework or state management approach, cannot use this library. It also imposes a particular architecture on how the partner manages authentication state — partners with an existing, established Redux store architecture should review whether the library's reducer and middleware compose cleanly with their existing setup before committing to this approach.

**Typical user**: A full-featured single-page application built by a partner with a dedicated front-end engineering team, already invested in the React/Redux stack, that wants authentication to feel fully native rather than bolted on.

---

### Choosing Between the Three Methods

| Consideration | Browser Library | Node.js Client | React/Redux Library |
|---|---|---|---|
| Backend required? | No | Yes (Node.js) | Optional (depends on token exchange approach) |
| Secret key ever in browser? | No | No | No |
| Request signing (HMAC)? | No | Yes | Depends on integration |
| React/Redux assumed? | No | No | Yes |
| Quickest to integrate | Yes | Moderate | Moderate |
| Most control over flow | Limited | Full | Full within React |
| Automated/server-to-server? | Not suitable | Yes | Not suitable |

The most important principle across all three methods: the secret key must remain on the server. Whichever integration method is used, any operation that requires HMAC signing must be performed server-side. If a partner application has no backend server, it is limited to operations that do not require signing — primarily the browser library flow, which handles security through the browser's same-origin policy instead.

---

## Configuration Management

The authentication system runs in three distinct environments, each serving a different purpose in the development and operational lifecycle. Understanding the differences between environments is important for partners and internal teams who need to test integrations, investigate issues, or plan deployments.

### Local Environment

The local environment is used exclusively by developers working on the authentication system itself. Each developer runs their own copy of the service on their own machine. The configuration in this environment points to test-grade data storage and test-grade API endpoints — it never touches production data. Logging is set to maximum verbosity so that developers can trace every step of a request in detail, which makes debugging straightforward.

The key characteristic of local is that it is isolated and disposable. A developer can create and delete test records freely without any risk of affecting anyone else. The DynamoDB tables used here (`TrustedParty_Api_test`, `TrustedParty_Tokens_test`, etc.) are shared test tables, not production tables, so registered trusted parties and issued tokens in this environment are purely for development use.

### Test Environment (Pre-Production)

The test environment is a shared, persistent environment that mirrors the production setup as closely as possible. It is used for integration testing — by the Web Wizards team when validating new features, by partner development teams when building and testing their integrations, and by QA when running acceptance tests before any change is released to production.

Test uses the same database table names as local (the `_test` suffix variants) and points to the test-grade Officeworks API endpoints (`api-test.officeworks.com.au`). This means it can simulate the full end-to-end flow — including account creation in the Cognito test user pool and profile retrieval from the test customer API — without any of those actions affecting real customer records.

Logging in test is also set to maximum verbosity, for the same reason as local: problems are easier to diagnose when the full detail is available. Partners who encounter unexpected behaviour in their test integration should be able to work with the Web Wizards team to pull the relevant logs from this environment.

### Master (Production) Environment

The master environment is the live, customer-facing deployment. All configuration here points to production-grade infrastructure: production DynamoDB tables (without the `_test` suffix), the production Officeworks API (`api.officeworks.com.au`), and the production Cognito user pool.

In production, logging is set to a reduced verbosity level — capturing meaningful events (successful logins, failures, token issuances, admin operations) without writing exhaustive detail for every processing step. This is a deliberate trade-off: verbose logging in production would generate large volumes of data and could inadvertently capture sensitive values in edge cases. The reduced level provides the audit trail and alerting capability the operations team needs without unnecessary data collection.

The most important distinction between master and the other environments is consequence: operations performed here affect real customer accounts. Trusted parties registered in production are real integrations with real API credentials. Tokens issued in production are valid for real users. Changes to production configuration should go through the change management process and be validated in test first.

### How Configuration Differences Affect Partners

Partners building an integration will work in the test environment first, using test API keys issued for that purpose. Their integration should be fully validated in test before any production credentials are issued or production registration is performed. The endpoints, table names, and credentials are different between test and production, which means a partner cannot accidentally call production systems while testing — and cannot use test credentials in production. This separation is intentional and protects both partners and customers during the integration development period.

---

## Deprecation and Migration

**This system is scheduled for decommissioning in January 2026.** New integrations should not be built against it. Existing integrations need to be migrated to a modern authentication solution before that date.

### Timeline

| Date | What happens |
|---|---|
| Now (April 2026) | System is beyond its planned decommissioning date. All integrations are operating on extended support only. Urgent migration required. |
| No new date confirmed | Final shutdown pending migration of remaining integrations. Contact Web Wizards for current status. |
| Post-shutdown | System infrastructure will be archived. All credentials will be permanently invalidated. |

Note: The system has passed its originally planned January 2026 decommission date. If your integration is still running against this system, it is operating outside its supported lifecycle and is at risk of sudden service disruption as infrastructure changes proceed. Contact the Web Wizards team immediately.

### What "Contact Web Wizards" Means in Practice

The Web Wizards team is the Officeworks platform team responsible for the authentication and identity infrastructure. They are the owners of both this legacy system and its modern replacement.

To initiate a migration, the right starting point is raising a request through the standard Officeworks IT service management channel, addressed to the Web Wizards team, and describing:
- Which application is currently using the legacy authentication system
- What authentication flows it uses (login, registration, guest, or a combination)
- The technical stack of the partner application (browser-only, Node.js backend, React/Redux, other)
- Any constraints on the migration timeline

The Web Wizards team will then arrange a scoping conversation to assess the migration effort and issue credentials for the replacement system.

### What a Migration Involves

A migration from this system to the modern OAuth 2.0 solution is not a simple "swap the endpoint" change — the underlying authentication model is different. The effort required depends on how deeply the legacy system is embedded in the partner application. Here is what partners should anticipate:

**Credential replacement**: The API key, secret key, and Party ID issued by this system are specific to it and have no meaning in the new system. The partner will receive a new client ID and client secret from the replacement system. These need to be updated in configuration and the old credentials securely disposed of.

**Token format changes**: The OWT and OTT token formats used by this system are proprietary to it. The modern system uses standard OAuth 2.0 access tokens and refresh tokens. Any code that inspects, parses, or manipulates the token payload will need to be updated to handle the new format.

**Flow changes**: The authentication flows in this system broadly follow an OAuth-like pattern but with custom endpoints and non-standard request/response shapes. The modern system follows the OAuth 2.0 specification more closely, which means the flow logic in the partner application will need to be updated. For partners using one of the provided client libraries (TANK or TARAS), there may be updated versions of those libraries that support the new system — Web Wizards can confirm this during the scoping conversation.

**Session management**: The keepalive mechanism in this system is specific to it. OAuth 2.0's equivalent is the refresh token flow. Partners who implement session extension will need to update their keepalive logic to use the new token refresh approach.

**Testing**: Before cutting over to the new system in production, partners should run full end-to-end tests of all authentication flows in the test environment for the new system. Web Wizards will provision test credentials for this purpose.

**Rollback planning**: The migration cutover should be planned so that a rollback is possible — for example, by deploying the updated integration to production on a feature flag that can be toggled, rather than replacing the old integration entirely in a single release. This provides a safety net if unexpected behaviour is discovered after go-live.

The overall migration effort for a typical partner integration is estimated at between two and five development days, depending on the complexity of the existing integration and how many flows are in use. Partners with custom implementations of HMAC signing (rather than using the provided client libraries) will tend toward the higher end of that range, as the signing mechanism in the new system differs.

---

## Functional Summary Table

| Function | Endpoint | Method | What you provide | What you receive |
|---|---|---|---|---|
| Register trusted party | /auth/tp/:id | POST | Party name, callback URL | API key, secret key |
| Get trusted party | /auth/tp/:id | GET | Party ID or API key | Party configuration |
| Update trusted party | /auth/tp/:id | PUT | Updated fields | Updated configuration |
| Delete trusted party | /auth/tp/:id | DELETE | Party ID | Confirmation |
| Guest token | /auth/token/guest | PUT | API key | Session tokens (OWT + OTT) |
| Register personal | /auth/register | PUT | User details + API key | Session tokens + user identity |
| Register business | /auth/register/business | PUT | Business details + API key | Session tokens + user identity |
| Login | /auth/login | POST | Email, password, API key | Session tokens + user identity |
| Validate token | /auth/token/validate | POST | Session token | Valid/invalid + token details |
| Exchange OTT | /auth/token | GET | OTT + signed API key | Session token (OWT) |
| Keepalive | /auth/keepalive | POST | Current session token | Extended session token |
| User profile | /auth/customer/profile | GET | Session token | Full user profile |

---

**Document Status**: Enhanced functional specification — all major functions documented with business context, use cases, and migration guidance.
**Last Updated**: April 2026
**Deprecation Status**: System is past its January 2026 decommission date. All active integrations should be migrated urgently. Contact Web Wizards.
