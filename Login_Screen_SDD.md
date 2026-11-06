# Software Design Document (SDD)
## Login Screen — Web Application

**Version:** 1.4  
**Date:** 2026-04-30  
**Author:** Savitri Gokarnesan  
**Status:** Draft

---

## 1. Introduction

### 1.1 Purpose
This document describes the design of the Login Screen feature for a web application built with **NestJS** following **Hexagonal (Ports & Adapters) Clean Architecture**. It covers UI structure, backend architecture layers, data flow, validation rules, and security considerations.

### 1.2 Scope
This SDD covers the login screen only: username input, password input, "Remember Me" checkbox, and the submit button. It does not cover post-login flows, session management beyond token storage, or account recovery.

### 1.3 Technology Stack

| Concern | Technology |
|---------|-----------|
| API Entry Point | AWS API Gateway (HTTP API) |
| Backend Framework | NestJS (Node.js) |
| Architecture Pattern | Hexagonal / Ports & Adapters |
| Language | TypeScript |
| Authentication Provider | AWS Cognito (User Pools) |
| Token Authorizer | AWS API Gateway — Cognito JWT Authorizer (validates Access Token on protected routes) |
| Cognito SDK | `@aws-sdk/client-cognito-identity-provider` |
| Database | PostgreSQL |
| ORM | TypeORM |
| Validation | `class-validator`, `class-transformer`, NestJS `ValidationPipe` |
| Rate Limiting | AWS API Gateway throttling + `@nestjs/throttler` (NestJS layer) + Cognito built-in lockout |

### 1.4 Definitions
| Term | Meaning |
|------|---------|
| Port | An interface (inbound or outbound) that defines how the application core communicates with the outside world |
| Adapter | A concrete implementation of a port — either driving (inbound) or driven (outbound) |
| Use Case | Application service that orchestrates domain logic for a specific user action |
| API Gateway | AWS-managed API proxy that routes all traffic to NestJS; enforces the Cognito JWT Authorizer on protected routes |
| Cognito JWT Authorizer | API Gateway authorizer that validates the Cognito Access Token against the User Pool's JWKS before forwarding the request to NestJS |
| Cognito User Pool | AWS-managed directory that handles user storage, password policy, and token issuance |
| Access Token | Short-lived JWT (1 h) issued by Cognito; sent by the client in the `Authorization: Bearer` header; validated by API Gateway |
| ID Token | JWT containing user identity claims (email, `sub`, etc.) |
| Refresh Token | Long-lived opaque token used to obtain new Access/ID tokens without re-login; expires after **30 days** |
| `sub` | Cognito-assigned UUID that uniquely identifies a user; used as the foreign key in PostgreSQL |
| Remember Me | Persistent login preference; controls whether tokens are stored in `localStorage` vs `sessionStorage` |

---

## 2. System Overview

The login screen is a client-side HTML form. All browser traffic goes through **AWS API Gateway**, which routes requests to the NestJS backend. The login endpoint is a public route (no Cognito Authorizer); all other application routes are protected by the **Cognito JWT Authorizer** configured on API Gateway — NestJS never validates tokens itself.

On the backend, the HTTP controller (inbound adapter) delegates to a use case, which calls outbound ports via interfaces — never importing infrastructure directly. Authentication (credential verification and token issuance) is fully delegated to **AWS Cognito**. PostgreSQL stores application-level user profile data linked to the Cognito `sub`.

```
                  ┌─────────────────────────┐
                  │      AWS API Gateway     │
  Browser  ──────▶│                         │──────▶  NestJS
                  │  /login   → NO auth      │
                  │  /other   → Cognito JWT  │
                  │             Authorizer   │
                  └─────────────────────────┘
                              │
                              │ (validates JWT via JWKS)
                              ▼
                       AWS Cognito User Pool
```

```
┌─────────────────────────────────────────┐
│              Login Screen               │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │  Username                         │  │
│  └───────────────────────────────────┘  │
│  ┌───────────────────────────────────┐  │
│  │  Password                     👁  │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ☑ Remember Me                          │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │            Log In                 │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

---

## 3. Hexagonal Architecture Design

### 3.1 Layer Overview

```
  Browser
     │
     ▼
┌─────────────────────────────────────────────────────────────────────┐
│  AWS API Gateway                                                     │
│   POST /api/auth/login  → route: NO Cognito Authorizer  (public)    │
│   All other routes      → Cognito JWT Authorizer (validates token)  │
└─────────────────────────────────────────────────────────────────────┘
     │
     ▼
──────────────────────────────────────────────────────────────────────────
  PRESENTATION         APPLICATION CORE           INFRASTRUCTURE
  (Inbound Adapters)   (Ports & Use Cases)        (Outbound Adapters)
──────────────────────────────────────────────────────────────────────────

  AuthController ──▶  ILoginUseCase          ICognitoAuthPort ──▶ CognitoAuthAdapter
  (HTTP Adapter)       LoginUseCaseImpl ──▶   IUserRepository  ──▶ TypeOrmUserRepository
  LoginDto             LoginCommand                                  (PostgreSQL)
                       LoginResult
                       User (Domain Entity)
                       AuthenticationException
──────────────────────────────────────────────────────────────────────────
        │ drives (primary)                    driven by (secondary) │
```

> **API Gateway is the single entry point.** The login route has no authorizer — it is intentionally public. All other routes are protected by the Cognito JWT Authorizer at the API Gateway level; NestJS never validates tokens itself.

> **Cognito is the authority for authentication.** The `LoginUseCaseImpl` calls `ICognitoAuthPort` to delegate credential verification and token issuance entirely to AWS Cognito. `IUserRepository` (PostgreSQL) is available in the module for profile-enrichment use cases but is **not called during login**.

### 3.2 Layer Responsibilities

| Layer | Role | NestJS Artefacts |
|-------|------|-----------------|
| **Domain** | Core business entities; zero framework dependency | `User` entity, `AuthenticationException` |
| **Application** | Use cases; owns port interfaces | `ILoginUseCase`, `LoginUseCaseImpl`, `ICognitoAuthPort`, `IUserRepository`, `LoginCommand`, `LoginResult` |
| **Infrastructure** | Implements outbound ports | `CognitoAuthAdapter` (AWS SDK), `TypeOrmUserRepository` (PostgreSQL), `UserOrmEntity` |
| **Presentation** | Inbound HTTP adapter; maps HTTP ↔ use-case | `AuthController`, `LoginDto`, `AuthModule` |

### 3.3 Folder Structure

```
src/
└── auth/
    ├── domain/
    │   ├── entities/
    │   │   └── user.entity.ts                  # Pure domain model (no ORM decorators)
    │   └── exceptions/
    │       └── authentication.exception.ts
    │
    ├── application/
    │   ├── ports/
    │   │   ├── inbound/
    │   │   │   └── login.use-case.port.ts      # ILoginUseCase interface
    │   │   └── outbound/
    │   │       ├── cognito-auth.port.ts         # ICognitoAuthPort interface
    │   │       └── user.repository.port.ts      # IUserRepository interface
    │   ├── use-cases/
    │   │   └── login.use-case.ts               # LoginUseCaseImpl
    │   └── dtos/
    │       ├── login.command.ts                # Input to use case
    │       └── login.result.ts                 # Output from use case
    │
    ├── infrastructure/
    │   ├── cognito/
    │   │   └── cognito-auth.adapter.ts          # Implements ICognitoAuthPort (AWS SDK)
    │   └── persistence/
    │       ├── typeorm-user.repository.ts       # Implements IUserRepository (TypeORM)
    │       └── user.orm-entity.ts              # TypeORM @Entity (PostgreSQL)
    │
    ├── presentation/
    │   ├── auth.controller.ts                  # POST /api/auth/login
    │   └── dtos/
    │       └── login.dto.ts                    # HTTP request DTO (class-validator)
    │
    └── auth.module.ts                          # NestJS module wiring
```

### 3.4 Key Interfaces (Ports)

**Inbound port — `ILoginUseCase`**
```typescript
export interface ILoginUseCase {
  execute(command: LoginCommand): Promise<LoginResult>;
}
```

**Outbound port — `ICognitoAuthPort`**
```typescript
export interface CognitoTokens {
  accessToken: string;
  idToken: string;
  refreshToken: string;
  expiresIn: number;      // seconds until accessToken expires (typically 3600)
}

export interface ICognitoAuthPort {
  authenticate(username: string, password: string): Promise<CognitoTokens>;
}
```

**Outbound port — `IUserRepository`** *(PostgreSQL; not called during login)*
```typescript
export interface IUserRepository {
  findByCognitoSub(sub: string): Promise<User | null>;
}
```

### 3.5 Use Case Logic — `LoginUseCaseImpl`

```
execute(command: LoginCommand):
  1. tokens = await ICognitoAuthPort.authenticate(command.username, command.password)
       │
       ├── Cognito NotAuthorizedException  → throw AuthenticationException
       ├── Cognito UserNotFoundException   → throw AuthenticationException  (same message)
       └── Cognito UserNotConfirmedException → throw AuthenticationException
  2. return LoginResult {
       accessToken:  tokens.accessToken,
       idToken:      tokens.idToken,
       refreshToken: tokens.refreshToken,
       expiresIn:    tokens.expiresIn
     }
```

> All Cognito error variants are caught and re-thrown as a single `AuthenticationException("Invalid username or password.")` — prevents username enumeration via differing error messages.

### 3.6 Infrastructure — `CognitoAuthAdapter`

Uses `InitiateAuthCommand` from `@aws-sdk/client-cognito-identity-provider`:

```
AuthFlow: USER_PASSWORD_AUTH
AuthParameters:
  USERNAME: command.username
  PASSWORD: command.password
ClientId:   process.env.COGNITO_CLIENT_ID
```

> `USER_SRP_AUTH` (Secure Remote Password) can replace `USER_PASSWORD_AUTH` to avoid sending the plaintext password to the backend — evaluate with the security team.

### 3.7 Environment Configuration

| Variable | Where used | Description |
|----------|-----------|-------------|
| `COGNITO_USER_POOL_ID` | NestJS + API Gateway | Cognito User Pool ID (`us-east-1_XXXXXXXX`) |
| `COGNITO_CLIENT_ID` | NestJS (`CognitoAuthAdapter`) | App client ID (no client secret for public SPA clients) |
| `COGNITO_REGION` | NestJS + API Gateway | AWS region of the User Pool |
| `DATABASE_URL` | NestJS (TypeORM) | PostgreSQL connection string |
| `API_GATEWAY_URL` | Client (browser) | Base URL of the API Gateway endpoint; all API calls use this as the host |

> The Cognito JWT Authorizer on API Gateway is configured with:
> - **Issuer:** `https://cognito-idp.{COGNITO_REGION}.amazonaws.com/{COGNITO_USER_POOL_ID}`
> - **Audience:** `{COGNITO_CLIENT_ID}`
> - **Identity source:** `$request.header.Authorization` (Bearer token)

### 3.8 PostgreSQL — User Profile Table

The `users` table stores app-specific profile data. The `cognito_sub` column is the link between Cognito and PostgreSQL — it is the value of the `sub` claim in the Cognito ID Token.

```sql
CREATE TABLE users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cognito_sub VARCHAR(255) NOT NULL UNIQUE,  -- Cognito user UUID
  email       VARCHAR(255) NOT NULL UNIQUE,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 4. UI Component Design

### 4.1 UI Components

| Component | Type | Description |
|-----------|------|-------------|
| Username Field | `<input type="text">` | Accepts alphanumeric and email-format usernames |
| Password Field | `<input type="password">` | Masked input; optional show/hide toggle |
| Remember Me | `<input type="checkbox">` | Client-side only: controls whether Cognito tokens are stored in `localStorage` (persistent) or `sessionStorage` (session only) |
| Submit Button | `<button type="submit">` | Triggers client validation and API call |
| Error Banner | `<div>` (conditional) | Displays inline error on failed login |

### 4.2 Component States

| State | Description |
|-------|-------------|
| Default | Empty form, all fields enabled |
| Active/Focus | Field border highlighted on focus |
| Filled | Field contains user input |
| Loading | Submit button disabled; spinner shown while awaiting API response |
| Error | Inline error message displayed |
| Success | Redirect to `/dashboard` |

---

## 5. Data Flow

```
[Browser]
  │
  │  User submits form
  ▼
Client-Side Validation
  │
  ├── Fails → Display inline error, halt
  │
  ▼
POST {API_GATEWAY_URL}/api/auth/login  { username, password }
  │
[AWS API Gateway]
  │  Route: POST /api/auth/login
  │  Authorizer: NONE  (public route — no token required)
  │  Gateway throttle: applies if configured
  │  Forwards request to NestJS
  │
  ▼
[NestJS — Presentation Layer]
  │  ThrottlerGuard checks rate limit (5 req / 10 min per IP)
  │  AuthController receives request
  │  ValidationPipe transforms & validates LoginDto
  │
  ▼
[NestJS — Application Layer]
  │  LoginUseCaseImpl.execute(LoginCommand)
  │  ICognitoAuthPort.authenticate(username, password)
  │
  ▼
[AWS Cognito — User Pool]
  │  Verifies credentials
  │  Issues Access Token (1 h), ID Token (1 h), Refresh Token (30 d)
  │
  ├── NotAuthorizedException    → AuthenticationException → 401
  ├── UserNotFoundException     → AuthenticationException → 401  (same message)
  ├── UserNotConfirmedException → AuthenticationException → 401
  ├── ThrottlerException (NestJS) → 429 "Too many login attempts"
  ├── Unhandled AWS error       → 500 "Internal server error"
  │
  ▼
200 OK  { accessToken, idToken, refreshToken, expiresIn }
  │
[Browser]
  ├── rememberMe = true  → localStorage.setItem(...)   (tokens persist across sessions)
  ├── rememberMe = false → sessionStorage.setItem(...) (tokens cleared on tab close)
  │
  ▼
Redirect to /dashboard
  │
  (subsequent API calls)
  ▼
GET {API_GATEWAY_URL}/api/...
  Authorization: Bearer {accessToken}
  │
[AWS API Gateway]
  │  Cognito JWT Authorizer validates token against User Pool JWKS
  ├── Invalid / expired → 401 (request never reaches NestJS)
  │
  ▼
[NestJS] — receives request with decoded claims in context
```

---

## 6. Validation Rules

### 6.1 Client-Side (Pre-Submit)

| Field | Rule | Error Message |
|-------|------|---------------|
| Username | Required, non-empty | "Username is required" |
| Username | Max 255 characters | "Username is too long" |
| Password | Required, non-empty | "Password is required" |
| Password | Min 8 characters | "Password must be at least 8 characters" |

### 6.2 Server-Side — NestJS `ValidationPipe` on `LoginDto`

```typescript
export class LoginDto {
  @IsString()
  @IsNotEmpty()
  @MaxLength(255)
  username: string;

  @IsString()
  @IsNotEmpty()
  @MinLength(8)
  password: string;
}
```

The global `ValidationPipe` is configured with `whitelist: true` and `forbidNonWhitelisted: true` to strip unknown properties.

### 6.3 Rate Limiting

Three independent layers of rate limiting protect the login endpoint:

| Layer | Mechanism | Limit | Response |
|-------|-----------|-------|----------|
| AWS API Gateway | Gateway-level throttling | Configurable per stage/route | 429 (before reaching NestJS) |
| NestJS | `@nestjs/throttler` on `AuthController` | 5 req / 10 min per IP | 429 (before reaching Cognito) |
| AWS Cognito | Built-in User Pool account lockout | Configurable; email sent on lockout | `NotAuthorizedException` mapped to 401 |

---

## 7. API Contract

### 7.1 Request

```
POST /api/auth/login
Content-Type: application/json

{
  "username": "string",   // required, max 255 chars
  "password": "string"    // required, min 8 chars
}
```

### 7.2 Response — Success

```
HTTP 200 OK

{
  "accessToken":  "eyJhbGci...",   // Cognito Access Token  — use in Authorization header
  "idToken":      "eyJhbGci...",   // Cognito ID Token      — contains user identity claims (sub, email)
  "refreshToken": "eyJjdHki...",   // Cognito Refresh Token — long-lived; store per Remember Me preference
  "expiresIn":    3600             // seconds until accessToken expires
}
```

> **Remember Me is client-side only.** The backend always returns all three tokens. The frontend decides where to store them based on the checkbox state.

### 7.3 Response — Failure

```
HTTP 401 Unauthorized

{
  "statusCode": 401,
  "message": "Invalid username or password."
}
```

```
HTTP 429 Too Many Requests

{
  "statusCode": 429,
  "message": "Too many login attempts. Please try again later."
}
```

```
HTTP 400 Bad Request  (validation failure)

{
  "statusCode": 400,
  "message": ["password must be longer than or equal to 8 characters"],
  "error": "Bad Request"
}
```

---

## 8. Security Considerations

| Concern | Mitigation |
|---------|-----------|
| Password in transit | HTTPS enforced; password never logged server-side |
| Password at rest | Managed entirely by AWS Cognito (SRP hashing; NestJS never stores passwords) |
| Username enumeration | All Cognito error variants caught and re-thrown as `AuthenticationException("Invalid username or password.")` |
| Brute force (NestJS layer) | `@nestjs/throttler` — 5 req / 10 min per IP, blocks before reaching Cognito |
| Brute force (Cognito layer) | Cognito User Pool account lockout after repeated failures; user notified via **email** to unlock |
| XSS | Framework-level output escaping; no `innerHTML` with user data |
| CSRF | Tokens returned in response body and sent via `Authorization` header — not cookies, so no CSRF surface |
| Token storage | `sessionStorage` by default; `localStorage` only when "Remember Me" is checked |
| Token validation (downstream) | API Gateway Cognito JWT Authorizer validates the Access Token against the User Pool's JWKS before any protected request reaches NestJS — NestJS never validates tokens itself |
| Unauthenticated access to login route | `/api/auth/login` is explicitly configured with **no Cognito Authorizer** in API Gateway — it is the only public route |
| Cognito credentials in config | `COGNITO_USER_POOL_ID`, `COGNITO_CLIENT_ID`, `COGNITO_REGION` loaded from environment — never hardcoded |
| Autocomplete | `autocomplete="username"` and `autocomplete="current-password"` for password managers |
| Dependency injection | Ports injected via NestJS DI tokens — `CognitoAuthAdapter` never imported directly by use cases |

---

## 9. Accessibility

- All inputs have associated `<label>` elements.
- Error messages are linked to inputs via `aria-describedby`.
- Submit button is keyboard-focusable and activatable via `Enter`.
- Color is not the sole indicator of error state (icon + text alongside color).
- Meets WCAG 2.1 AA contrast ratio (4.5:1 minimum).

---

## 10. Non-Functional Requirements

| Requirement | Target |
|-------------|--------|
| API response time (p95) | < 300 ms under normal load |
| Page load time | < 1 second on a 4G connection |
| Accessibility | WCAG 2.1 AA |
| Browser support | Chrome, Firefox, Safari, Edge (latest 2 versions) |
| Mobile responsiveness | Functional on viewports ≥ 320 px wide |
| Testability | Each layer independently unit-testable; use cases tested with mock adapters |

---

## 11. Out of Scope

- Password reset / "Forgot Password" flow
- Multi-factor authentication (MFA)
- OAuth / SSO login (Passport strategies can be added later as additional adapters)
- New user registration
- Session expiry and automatic logout
- Refresh token rotation

---

## 12. Backend Implementation Standards

These standards are **non-negotiable** and apply to all code produced for this feature.

### 12.1 Code Style
- All code must pass the repository's **ESLint + Prettier** configuration without modification.
- No new formatting conventions may be introduced.

### 12.2 Architecture & SOLID Principles

| Layer | Sole responsibility |
|-------|-------------------|
| **Controller** (`AuthController`) | Transport only — validate HTTP input, map to command/DTO, call use case, map result to HTTP response |
| **Use Case / Service** (`LoginUseCaseImpl`) | Business logic only — no HTTP, no ORM entities, no AWS SDK types |
| **Repository / DAO** (`TypeOrmUserRepository`) | Persistence concerns only — no business logic |
| **Adapter** (`CognitoAuthAdapter`) | External integration only — translate between domain port and AWS SDK |
| **DTO / Mapper** | Boundary conversion only — no logic leaking into or out of layers |

- Prefer composition and small, single-purpose functions.
- Extract shared logic into well-owned utility modules; do not duplicate.
- Keep all auth feature code inside the `auth` module; do not leak internals across module boundaries.

### 12.3 NestJS Dependency Injection

- **Never** instantiate providers manually with `new` when NestJS DI can resolve them.
- Declare every provider in the correct module and export/import explicitly — no implicit global registration.
- Avoid circular dependencies; if unavoidable, use `forwardRef()` sparingly and document the reason in a comment.
- Provider scope defaults to **singleton**; use request-scoped only when strictly necessary and document why.
- Inject `ICognitoAuthPort` and `IUserRepository` via DI tokens (symbols), not concrete class references, to preserve the hexagonal boundary.

### 12.4 PostgreSQL / Persistence

- **Always** use TypeORM parameterized queries or QueryBuilder — no raw string interpolation into SQL.
- Wrap multiple writes in a **transaction** (TypeORM `DataSource.transaction()` or `EntityManager`).
- Avoid N+1 queries; use joins or `findBy` with `relations` as appropriate.
- Store and retrieve all timestamps as UTC; use `TIMESTAMPTZ` columns.
- Use the shared **connection pool** — never create a per-request `DataSource` instance.

### 12.5 Logging

- Log **once at the controller boundary** for request context (method, path, requestId/traceId).
- Log in services only for meaningful domain events (e.g., successful login, account locked).
- **Never log**: passwords, tokens, credentials, or any PII. Redact before logging.
- Include structured context fields: `requestId`, `traceId`, `userId` (when available).
- Use correct log levels:

| Level | When |
|-------|------|
| `debug` | Detailed diagnostics (disabled in production) |
| `info` | Normal business events (login succeeded) |
| `warn` | Recoverable anomalies (rate-limit approaching) |
| `error` | Failures — include stack and context, never sensitive data |

### 12.6 Error Handling

- Validate all inputs via `LoginDto` + `class-validator`; the global `ValidationPipe` must be enabled.
- Convert Cognito SDK errors into domain exceptions (`AuthenticationException`) inside `CognitoAuthAdapter`; the use case must never catch raw AWS SDK error types.
- Map domain exceptions to NestJS HTTP exceptions in the controller or via a global exception filter — never return raw errors to the client.
- Do **not** leak stack traces, SQL errors, AWS error codes, or secrets to HTTP responses.
- All `async` calls must be `await`-ed or explicitly handled — no unhandled promise rejections.

### 12.7 Security

- Follow the OWASP mindset on every input path: validate, sanitize, enforce authN/authZ.
- Never trust a client-supplied `userId` or `sub` for authorization decisions — always derive identity from the decoded Cognito JWT claim forwarded by API Gateway.
- All secrets (`COGNITO_USER_POOL_ID`, `COGNITO_CLIENT_ID`, `DATABASE_URL`, etc.) must be loaded via the repository's config service pattern (e.g., NestJS `ConfigService`) — never hardcoded or read directly from `process.env` in business code.
- Do not introduce new cryptographic primitives; use libraries already present in the repository.

### 12.8 Testing Requirements

| Test type | Scope | Requirement |
|-----------|-------|-------------|
| **Unit** | `LoginUseCaseImpl` | Mock `ICognitoAuthPort`; cover success, each Cognito error variant, and the single-message enumeration-prevention rule |
| **Unit** | `CognitoAuthAdapter` | Mock `CognitoIdentityProviderClient`; verify correct `InitiateAuthCommand` params and error mapping |
| **Unit** | `AuthController` | Mock `ILoginUseCase`; verify DTO validation, HTTP status codes, and response shape |
| **Integration / e2e** | `POST /api/auth/login` | Covers happy path and error paths across module/DB boundaries, per repository norms |

- Tests must be deterministic — no time-dependent or order-dependent behaviour.
- Follow existing test patterns and utilities in the repository.
- If a test is intentionally deferred, state why and what remains in a `// TODO(test):` comment.

### 12.9 Static Checks (run before each PR)

```bash
npm run lint        # ESLint + Prettier
npm run test        # Unit tests
npm run typecheck   # tsc --noEmit (if present in package.json)
```

All checks must pass green before a pull request is opened. Fix code first, then tests; do not suppress lint rules without approval.

### 12.10 Implementation Workflow

1. Restate the requirement and ask clarifying questions if needed.
2. Identify affected modules/files; propose a concise design plan.
3. Implement in small, logical steps (do not commit unless asked).
4. Add/update tests per §12.8.
5. Run static checks (§12.9) and fix until green.
6. Provide a final summary using the output format below.

### 12.11 Output Format (every implementation task)

```
## Requirement restatement
## Clarifying questions  (or "Proceeding with implementation")
## Design / approach     (bullets)
## Files changed         (bullets)
## Summary of behaviour changes
## Tests added / updated
## Commands run + results
## Notes / risks / tradeoffs
```

---

## 13. Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| 1 | ~~What database will back `IUserRepository`?~~ | — | **Resolved: PostgreSQL** |
| 2 | ~~What should the Cognito Refresh Token expiry be?~~ | — | **Resolved: 30 days** |
| 3 | ~~Should Cognito account lockout use CAPTCHA, email notification, or time-based unlock?~~ | — | **Resolved: email notification** |
| 4 | Should `USER_PASSWORD_AUTH` (plaintext password to NestJS) be replaced with `USER_SRP_AUTH` (password never leaves browser)? | Security / Architecture | Open |
| 5 | Is a local PostgreSQL user profile row created at login (first-seen upsert) or at registration? | Backend team | Open |
