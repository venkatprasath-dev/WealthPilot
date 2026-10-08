# Auth Service

**Language:** Java 21  
**Framework:** Spring Boot 3 · Spring Security 6  
**Phase:** 1  
**Status:** 🔴 Not Started

---

## Responsibility
All authentication and identity. Issues JWT access + refresh tokens. Google OAuth2 social login. User profile management.

## Stack
| Component | Technology |
|-----------|-----------|
| Framework | Spring Boot 3 |
| Security | Spring Security 6 |
| Token | JWT (RS256 — asymmetric) |
| OAuth | Spring Security OAuth2 Client |
| Password | BCrypt (cost factor 12) |
| ORM | Spring Data JPA + Hibernate 6 |
| Primary DB | PostgreSQL 16 |
| Session store | Redis 7 |
| Service discovery | Eureka client |

## Package Structure
```
com.wealthpilot.auth/
├── AuthServiceApplication.java
├── config/
│   ├── SecurityConfig.java          -- SecurityFilterChain + OAuth2 + CORS
│   ├── JwtConfig.java               -- RSA key pair loading from Vault
│   └── RedisConfig.java             -- RedisTemplate + connection factory
├── controller/
│   └── AuthController.java          -- All /auth/** endpoints
├── service/
│   ├── AuthService.java             -- Registration, login, refresh, logout
│   ├── JwtService.java              -- Token generation, validation, JTI extraction
│   ├── GoogleOAuthService.java      -- ID token exchange, user creation
│   └── UserService.java             -- Profile CRUD
├── repository/
│   ├── UserRepository.java
│   └── UserPreferencesRepository.java
├── entity/
│   ├── User.java
│   └── UserPreferences.java
├── dto/
│   ├── RegisterRequest.java
│   ├── LoginRequest.java
│   ├── AuthResponse.java            -- {accessToken, refreshToken, expiresIn}
│   └── UserProfileDTO.java
├── filter/
│   └── JwtAuthenticationFilter.java -- OncePerRequestFilter: validate + set SecurityContext
└── exception/
    ├── InvalidCredentialsException.java
    ├── TokenExpiredException.java
    └── UserAlreadyExistsException.java
```

## Database Tables
```sql
users (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email           VARCHAR(255) UNIQUE NOT NULL,
  name            VARCHAR(255),
  password_hash   VARCHAR(255),          -- NULL for OAuth users
  provider        VARCHAR(20) DEFAULT 'LOCAL', -- LOCAL | GOOGLE
  base_currency   CHAR(3) DEFAULT 'INR',
  timezone        VARCHAR(50) DEFAULT 'Asia/Kolkata',
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

user_preferences (
  user_id           UUID PRIMARY KEY REFERENCES users(id),
  theme             VARCHAR(20) DEFAULT 'dark',
  email_frequency   VARCHAR(20) DEFAULT 'MONTHLY',
  alert_channels    JSONB DEFAULT '{"email": true, "push": false}',
  updated_at        TIMESTAMPTZ DEFAULT NOW()
);
```

## Redis Key Design

| Key | Value | TTL |
|-----|-------|-----|
| `refresh:{userId}` | refresh token (opaque UUID) | 7 days |
| `blacklist:jti:{jti}` | `"1"` | Remaining access token lifetime |

## JWT Design
- Algorithm: RS256 (RSA 2048-bit keypair — private key signs, public key verifies)
- Access token: 15-minute TTL, claims: `sub` (userId), `email`, `jti` (UUID)
- Refresh token: opaque UUID string (not a JWT), 7-day TTL, stored in Redis
- Public key shared with API Gateway for edge validation (no Auth Service call per request)

## API

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|---------|
| `POST` | `/api/v1/auth/register` | None | `{email, name, password}` | `AuthResponse` |
| `POST` | `/api/v1/auth/login` | None | `{email, password}` | `AuthResponse` |
| `POST` | `/api/v1/auth/refresh` | `X-Refresh-Token` header | — | `AuthResponse` |
| `POST` | `/api/v1/auth/logout` | Bearer | — | 204 |
| `GET` | `/api/v1/auth/oauth2/google` | None | — | Redirect to Google |
| `GET` | `/api/v1/auth/oauth2/google/callback` | None | Google code | `AuthResponse` |
| `GET` | `/api/v1/auth/me` | Bearer | — | `UserProfileDTO` |
| `PUT` | `/api/v1/auth/me` | Bearer | `{name, baseCurrency, timezone}` | `UserProfileDTO` |

## Security Edge Cases

| Case | Behavior |
|------|---------|
| Login with wrong password | 401 · No user enumeration (same error for wrong email) |
| Expired access token | 401 · Client should call `/refresh` |
| Expired refresh token | 401 · Client must re-login |
| Refresh token used after logout | 401 · Redis key deleted on logout |
| Replay attack with old access token | 401 · JTI in Redis blacklist |
| Google ID token forged | 401 · Google signature verification fails |
| Account lockout | After 10 failed logins in 5 min: 429 for 15 min (Redis counter per email) |

## Testing Scope

**Unit tests (Mockito):**
- `AuthServiceTest` — token generation, BCrypt verify, refresh rotation, blacklist insertion
- `JwtServiceTest` — encode/decode, expiry detection, JTI extraction

**Integration tests (Testcontainers — PG + Redis):**
- Full register → login → refresh → logout cycle
- Google OAuth with mocked Google endpoint (WireMock)
- Blacklisted token rejected on next request
- Account lockout after 10 failed logins

## Build Checklist
- [ ] `AuthServiceApplication` starts without error
- [ ] POST `/register` returns 201 with token pair
- [ ] POST `/login` with correct creds → tokens; wrong creds → 401
- [ ] GET `/me` with valid token → user profile
- [ ] Google OAuth callback flow → WealthPilot tokens issued
- [ ] Logout → token blacklisted → next request with same token → 401
- [ ] Refresh rotation: old refresh invalid after rotation
- [ ] Jenkins CI pipeline green
- [ ] SonarQube ≥ 80% coverage
