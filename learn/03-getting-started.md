# Getting Started

[← Architecture](02-architecture.md) | [README](README.md) | [Next: Services Deep Dive →](04-services-deep-dive.md)

---

## Prerequisites

- Docker + Docker Compose installed
- Ports 80, 8761, 8888, 5000, 6000, 7000, 8000, 9000 free

---

## Run the App

```bash
# from repo root
docker-compose up
```

First run pulls all images + builds. Takes ~2-3 min.

Watch for: `Started GatewayApplication` in logs — that means the UI is ready.

---

## Credentials

### App Login

There are **no pre-set user accounts**. Register a new one:

1. Open `http://localhost:80`
2. Click **"Don't have an account? Create one"**
3. Enter username + password
4. You're logged in

### Demo Account (no login required)

The `demo` account is pre-seeded with sample data. Access it via the UI without logging in — it's useful for exploring features.

### Service Credentials (internal)

These are used between services, not for the UI:

| Variable | Value | Used by |
|----------|-------|---------|
| `CONFIG_SERVICE_PASSWORD` | `password` | All services → Config Server |
| `ACCOUNT_SERVICE_PASSWORD` | `password` | Auth Server OAuth2 client |
| `STATISTICS_SERVICE_PASSWORD` | `password` | Auth Server OAuth2 client |
| `NOTIFICATION_SERVICE_PASSWORD` | `password` | Auth Server OAuth2 client |
| `MONGODB_PASSWORD` | `password` | All services → MongoDB |
| MongoDB username | `user` | All services → MongoDB |

Source: `.env` in repo root.

---

## Service URLs

| Service | URL |
|---------|-----|
| Main App (UI) | `http://localhost:80` |
| Eureka Dashboard | `http://localhost:8761` |
| Config Server | `http://localhost:8888` |
| Hystrix Dashboard | `http://localhost:9000/hystrix` |
| RabbitMQ Management | `http://localhost:15672` (user: `guest` / `guest`) |

---

## Common Issues

**Services not finding each other?**
Config Server and Registry must be healthy before other services start. Check: `http://localhost:8761` shows all services registered.

**Auth errors on login?**
Auth Service needs to be `UP` in Eureka. Wait ~30s after all containers start.

**Statistics not loading?**
Statistics Service calls an external exchange rates API. Check container logs for network errors.

---

See [07-auth-flow.md](07-auth-flow.md) to understand how OAuth2 login works under the hood.

[Next: Services Deep Dive →](04-services-deep-dive.md)
