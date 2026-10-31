# Services Deep Dive

[← Getting Started](03-getting-started.md) | [README](README.md) | [Next: Domain Model →](05-domain-model.md)

---

## Config Service (port 8888)

**What:** Centralized configuration server using Spring Cloud Config with `native` (filesystem) profile.

**How it works:** On startup, every other service sends a request to `http://config:8888` to fetch its YAML config. Config reads from `config/src/main/resources/shared/`.

**Key files:**
- `config/src/main/resources/shared/application.yml` — shared config (Eureka URL, RabbitMQ, OAuth2 resource server)
- `config/src/main/resources/shared/auth-service.yml` — MongoDB creds for auth
- `config/src/main/resources/shared/notification-service.yml` — email (SMTP) settings

**Must start first** — all other services depend on it.

---

## Registry / Eureka (port 8761)

**What:** Netflix Eureka service discovery server.

**How it works:** Every service registers itself on startup with a heartbeat. When Service A needs to call Service B, it asks Eureka for B's current address instead of hardcoding IPs.

**UI:** `http://localhost:8761` — shows all registered instances and their health.

**Must start second** — after Config, before business services.

---

## Auth Service (port 5000)

**What:** OAuth2 Authorization Server. Issues access tokens for users and service-to-service calls.

**How it works:**
- Stores users in `auth-mongodb` (BCrypt-hashed passwords)
- UI logs in via `browser` OAuth2 client (password grant, no client secret)
- Services call each other using client credentials grant
- Token endpoint: `POST /uaa/oauth/token`
- User creation: `POST /uaa/users`

**Key files:**
- `auth-service/src/main/java/com/piggymetrics/auth/config/OAuth2AuthorizationConfig.java` — client registrations
- `auth-service/src/main/java/com/piggymetrics/auth/service/UserServiceImpl.java` — BCrypt password encoding

See [07-auth-flow.md](07-auth-flow.md) for the full OAuth2 flow.

---

## Account Service (port 6000)

**What:** Core business service. Owns all account data — income items, expense items, savings settings.

**How it works:**
- REST API at `/accounts/**`
- Persists to `account-mongodb`
- When account is updated, notifies Statistics Service via Feign
- Account creation also triggers user creation in Auth Service (via `AuthServiceClient`)

**Key files:**
- `account-service/src/main/java/com/piggymetrics/account/controller/AccountController.java`
- `account-service/src/main/java/com/piggymetrics/account/domain/Account.java`
- `account-service/src/main/java/com/piggymetrics/account/client/StatisticsServiceClient.java`

See [05-domain-model.md](05-domain-model.md) for entity details.

---

## Statistics Service (port 7000)

**What:** Analytics service. Stores time-series snapshots of account data for charting.

**How it works:**
- Receives account updates from Account Service via Feign
- Fetches current exchange rates from external API (to normalize currencies)
- Stores normalized `DataPoint` records per account in `statistics-mongodb`
- REST API at `/statistics/**`

**Key files:**
- `statistics-service/src/main/java/com/piggymetrics/statistics/controller/StatisticsController.java`
- `statistics-service/src/main/java/com/piggymetrics/statistics/domain/` — DataPoint, ItemMetric, etc.

---

## Notification Service (port 8000)

**What:** Manages email notification preferences and sends scheduled emails.

**How it works:**
- Stores per-account notification settings (`Recipient`) in `notification-mongodb`
- Uses Spring `@Scheduled` to send reminders and backups at configured frequencies
- Calls Account Service via Feign to fetch data for backup emails
- REST API at `/notifications/recipients/**`

**Key files:**
- `notification-service/src/main/java/com/piggymetrics/notification/controller/RecipientController.java`
- `notification-service/src/main/java/com/piggymetrics/notification/service/NotificationServiceImpl.java`
- `config/src/main/resources/shared/notification-service.yml` — SMTP settings

---

## Gateway / Zuul (port 80 in Docker, 4000 local)

**What:** Single entry point for the browser. Routes API calls to the right service and serves the jQuery SPA.

**How it works:**
- Zuul reverse-proxies based on routes defined in config
- Static UI files are in `gateway/src/main/resources/static/`
- All browser requests go through here — no direct service access

**Route mapping:**
| Path prefix | Proxied to |
|-------------|-----------|
| `/uaa/**` | auth-service |
| `/accounts/**` | account-service |
| `/statistics/**` | statistics-service |
| `/notifications/**` | notification-service |

**Key files:**
- `gateway/src/main/resources/static/index.html` — SPA entry point
- `gateway/src/main/resources/static/js/main.js` — app logic (User, Savings, Income, Expense)
- `gateway/src/main/resources/static/js/dashboard.js` — visualization

---

## Monitoring / Hystrix Dashboard (port 9000)

**What:** Real-time circuit breaker metrics visualization.

**How it works:**
- Connects to Turbine Stream to get aggregated Hystrix metrics
- Shows circuit state (open/closed), latency, error rates per command

**UI:** `http://localhost:9000/hystrix` → enter `http://turbine-stream:8989` as stream URL.

---

## Turbine Stream Service (port 8989)

**What:** Aggregates Hystrix metrics from all services into a single stream.

**How it works:**
- Listens on RabbitMQ for metrics events published by each service
- Exposes aggregated stream for Hystrix Dashboard to consume
- Uses Eureka to discover which services to pull from

---

[Next: Domain Model →](05-domain-model.md)
