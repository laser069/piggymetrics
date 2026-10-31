# API Reference

[← Domain Model](05-domain-model.md) | [README](README.md) | [Next: Auth Flow →](07-auth-flow.md)

All endpoints are accessible through the Gateway at `http://localhost:80`.

---

## Auth Service (`/uaa`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/uaa/oauth/token` | Basic (client creds) | Get OAuth2 access token |
| `GET` | `/uaa/users/current` | Bearer token | Get current authenticated user |
| `POST` | `/uaa/users` | Service (account-service client) | Create new user account |

**Get token example:**
```bash
curl -X POST \
  'http://localhost:80/uaa/oauth/token' \
  -u 'browser:' \
  -d 'grant_type=password&username=YOUR_USER&password=YOUR_PASS&scope=ui'
```

---

## Account Service (`/accounts`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/accounts/` | None | Create new account (also creates auth user) |
| `GET` | `/accounts/current` | Bearer token | Get logged-in user's account |
| `PUT` | `/accounts/current` | Bearer token | Update logged-in user's account (incomes/expenses/savings) |
| `GET` | `/accounts/{name}` | None (demo) / Bearer | Get account by name |

**Source:** `account-service/src/main/java/com/piggymetrics/account/controller/AccountController.java`

---

## Statistics Service (`/statistics`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/statistics/current` | Bearer token | Get time-series data for logged-in user |
| `GET` | `/statistics/{accountName}` | None (demo) / Bearer | Get statistics by account name |
| `PUT` | `/statistics/{accountName}` | Service (account-service) | Save/update statistics snapshot (called by Account Service) |

**Source:** `statistics-service/src/main/java/com/piggymetrics/statistics/controller/StatisticsController.java`

---

## Notification Service (`/notifications`)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `GET` | `/notifications/recipients/current` | Bearer token | Get notification settings for logged-in user |
| `PUT` | `/notifications/recipients/current` | Bearer token | Update notification settings |

**Source:** `notification-service/src/main/java/com/piggymetrics/notification/controller/RecipientController.java`

---

## Notes

- **No auth endpoints** return HTML — this is a JSON API with a separate jQuery SPA frontend.
- **`/accounts/demo`** and **`/statistics/demo`** work without a Bearer token (pre-seeded demo data).
- All Bearer tokens are OAuth2 access tokens from `/uaa/oauth/token`.
- Service-to-service calls use client credentials grant (not user Bearer tokens).

---

[Next: Auth Flow →](07-auth-flow.md)
