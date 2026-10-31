# Architecture

[← Overview](01-overview.md) | [README](README.md) | [Next: Getting Started →](03-getting-started.md)

---

## Service Map

```
Browser
   │
   ▼
┌──────────────────┐
│  Gateway (Zuul)  │  :80 (docker) / :4000 (local)
│  + Static UI     │
└──────┬───────────┘
       │  routes via Eureka discovery
       ├──────────────────────────────────────────────┐
       │                                              │
       ▼                                              ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐
│ Auth Service │    │ Account Svc  │    │  Statistics Service  │
│    :5000     │    │   :6000      │    │       :7000          │
│  (OAuth2 AS) │    │  (core data) │    │  (time-series data)  │
└──────────────┘    └──────┬───────┘    └──────────────────────┘
                           │ Feign
                           ▼
                  ┌──────────────────────┐
                  │ Notification Service │
                  │       :8000          │
                  │  (email reminders)   │
                  └──────────────────────┘

Support Services:
  Config Server  :8888  ← all services fetch config on startup
  Eureka Registry :8761  ← all services register here
  Monitoring (Hystrix Dashboard) :9000
  Turbine Stream  :8989  ← aggregates metrics via RabbitMQ
```

---

## Services Table

| Service | Port | Database | Purpose |
|---------|------|----------|---------|
| **config** | 8888 | — | Centralized config server; serves YAML to all services |
| **registry** | 8761 | — | Eureka service discovery dashboard |
| **auth-service** | 5000 | auth-mongodb | OAuth2 Authorization Server; issues tokens |
| **account-service** | 6000 | account-mongodb | Core domain: accounts, income, expenses, savings |
| **statistics-service** | 7000 | statistics-mongodb | Time-series analytics; fetches exchange rates |
| **notification-service** | 8000 | notification-mongodb | Email notification settings and scheduled sends |
| **gateway** | 4000 / 80 | — | Zuul API gateway + serves static jQuery SPA |
| **monitoring** | 8080 / 9000 | — | Hystrix Dashboard for circuit breaker visibility |
| **turbine-stream** | 8989 | — | Aggregates Hystrix metrics from all services |

---

## Communication Patterns

| Pattern | Used for |
|---------|---------|
| **Zuul proxy** | Browser → backend services via Gateway |
| **Feign client** | Account → Statistics, Notification → Account |
| **Eureka discovery** | All services find each other by name, not IP |
| **Ribbon** | Client-side load balancing on Feign calls |
| **Hystrix** | Circuit breaker on all Feign + Zuul routes |
| **RabbitMQ** | Hystrix metrics stream → Turbine → Monitoring |
| **Spring Cloud Config** | All services pull config from Config Server on boot |
| **Spring Cloud Sleuth** | Adds traceId/spanId to all log lines for tracing |

---

## Infrastructure

| Component | Port | Notes |
|-----------|------|-------|
| RabbitMQ | 15672 (mgmt UI) | AMQP broker for metrics |
| MongoDB (×4) | 27017 | Separate instance per service |

---

See [04-services-deep-dive.md](04-services-deep-dive.md) for per-service detail.

[Next: Getting Started →](03-getting-started.md)
