# Overview

[← README](README.md) | [Next: Architecture →](02-architecture.md)

---

## What Is PiggyMetrics?

PiggyMetrics is a **personal finance tracker** built as a microservices demo. It lets users track income, expenses, and savings — with statistics and email notifications.

It's also a reference implementation showing Spring Cloud microservices patterns in action.

## Core Features

| Feature | What it does |
|---------|-------------|
| **Income tracking** | Log income items with amount, currency, period (daily/weekly/monthly/yearly) |
| **Expense tracking** | Log expense items the same way |
| **Savings** | Track savings balance, interest rate, deposit schedule |
| **Statistics** | Time-series charts of your financial data over time |
| **Notifications** | Scheduled email reminders and account backup digests |
| **Demo mode** | Browse a pre-seeded `demo` account without registering |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Java 8 |
| Framework | Spring Boot 2.0.3.RELEASE |
| Cloud | Spring Cloud Finchley.RELEASE |
| Service Discovery | Netflix Eureka |
| API Gateway | Netflix Zuul |
| Circuit Breaker | Netflix Hystrix |
| Service-to-Service | OpenFeign |
| Load Balancing | Ribbon |
| Database | MongoDB (one instance per service) |
| Message Broker | RabbitMQ |
| Config | Spring Cloud Config (native filesystem) |
| Tracing | Spring Cloud Sleuth |
| Frontend | jQuery SPA served by Gateway |
| Containerization | Docker + Docker Compose |

## What This Is NOT

- Not production-ready (no TLS, all passwords are `password`)
- Not a monolith — every service is a separate Spring Boot app
- Not using a shared database — each service owns its own MongoDB

---

[Next: Architecture →](02-architecture.md)
