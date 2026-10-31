# Auth Flow

[← API Reference](06-api-reference.md) | [README](README.md)

---

## Overview

PiggyMetrics uses **OAuth2 Authorization Server** (Spring Security OAuth2) with two grant types:

| Grant type | Used by | Who calls it |
|-----------|---------|-------------|
| `password` | UI login | Browser (via jQuery) using `browser` client |
| `client_credentials` | Service-to-service | Account/Statistics/Notification services |

---

## User Login Flow (Password Grant)

```
1. User enters username + password in UI

2. Browser POSTs to Gateway:
   POST /uaa/oauth/token
   Authorization: Basic browser:   (client_id=browser, no secret)
   Body: grant_type=password
         username=USER
         password=PASS
         scope=ui

3. Gateway proxies to Auth Service (:5000)

4. Auth Service:
   a. Looks up user in auth-mongodb
   b. Verifies BCrypt password hash
   c. Returns access_token + refresh_token (JSON)

5. Browser stores access_token in memory

6. All subsequent API calls:
   Authorization: Bearer <access_token>
```

---

## Account Registration Flow

Registration creates both an auth user AND an account in one shot:

```
1. User fills sign-up form

2. POST /accounts/
   Body: { "username": "...", "password": "..." }

3. Account Service:
   a. Calls Auth Service via Feign: POST /uaa/users
      (using account-service client credentials)
   b. Auth Service stores BCrypt-hashed user in auth-mongodb
   c. Account Service creates Account doc in account-mongodb

4. UI then logs in (password grant, same as above)
```

**Source:** `account-service/src/main/java/com/piggymetrics/account/client/AuthServiceClient.java`

---

## OAuth2 Clients

Configured in `auth-service/src/main/java/com/piggymetrics/auth/config/OAuth2AuthorizationConfig.java`:

| Client ID | Secret | Grant types | Scope | Used by |
|-----------|--------|------------|-------|---------|
| `browser` | *(none)* | `password`, `refresh_token` | `ui` | jQuery SPA |
| `account-service` | `password` | `client_credentials` | `server` | Account → Auth/Stats |
| `statistics-service` | `password` | `client_credentials` | `server` | Statistics → others |
| `notification-service` | `password` | `client_credentials` | `server` | Notification → Account |

---

## Token Verification (Resource Servers)

Each service is also an OAuth2 Resource Server. When a request arrives with a `Bearer` token:

```
Request with Bearer token
    │
    ▼
Service checks token with Auth Service:
GET http://auth-service:5000/uaa/users/current
    │
    ▼
Auth Service validates token → returns user info
    │
    ▼
Service proceeds (or 401 if invalid)
```

Config: `config/src/main/resources/shared/application.yml`
```yaml
security:
  oauth2:
    resource:
      user-info-uri: http://auth-service:5000/uaa/users/current
```

---

## Password Storage

- Passwords BCrypt-hashed before storage
- **Source:** `auth-service/src/main/java/com/piggymetrics/auth/service/UserServiceImpl.java` — `BCryptPasswordEncoder`
- Raw passwords never stored

---

## Token Endpoints Quick Reference

| Endpoint | Purpose |
|----------|---------|
| `POST /uaa/oauth/token` | Get access + refresh tokens |
| `POST /uaa/oauth/token` with `grant_type=refresh_token` | Refresh expired token |
| `GET /uaa/users/current` | Validate token + get user info |
| `POST /uaa/users` | Create user (service-only) |

---

[← API Reference](06-api-reference.md) | [README](README.md)
