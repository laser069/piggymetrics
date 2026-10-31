# Domain Model

[← Services Deep Dive](04-services-deep-dive.md) | [README](README.md) | [Next: API Reference →](06-api-reference.md)

---

## Account (account-service)

Central entity. One `Account` per user.

```
Account
├── name          (String — same as username, unique)
├── lastSeen      (Date)
├── note          (String — free text note)
├── incomes       (List<Item>)
├── expenses      (List<Item>)
└── saving        (Saving)
```

**File:** `account-service/src/main/java/com/piggymetrics/account/domain/Account.java`

---

## Item

Represents a single income or expense entry.

```
Item
├── title         (String — label, e.g. "Salary")
├── amount        (BigDecimal)
├── currency      (Currency — USD, EUR, RUB)
└── period        (TimePeriod — YEAR, QUARTER, MONTH, DAY, HOUR)
```

**File:** `account-service/src/main/java/com/piggymetrics/account/domain/Item.java`

Items are stored as embedded lists inside `Account` (no separate MongoDB collection).

---

## Saving

Embedded inside `Account`. One saving object per account.

```
Saving
├── amount        (BigDecimal)
├── currency      (Currency)
├── interest      (BigDecimal — annual interest rate %)
├── deposit       (Boolean — deposit enabled?)
└── capitalization (Boolean — compound interest?)
```

**File:** `account-service/src/main/java/com/piggymetrics/account/domain/Saving.java`

---

## User (account-service)

Lightweight DTO used only for account creation. Not the auth user.

```
User
├── username      (String)
└── password      (String — raw, sent to auth-service for BCrypt encoding)
```

**File:** `account-service/src/main/java/com/piggymetrics/account/domain/User.java`

---

## User (auth-service)

Spring Security `UserDetails` implementation. Stored in `auth-mongodb`.

```
User
├── username      (String)
└── password      (String — BCrypt hashed)
```

**File:** `auth-service/src/main/java/com/piggymetrics/auth/domain/User.java`

---

## Recipient (notification-service)

Stores notification preferences per account. Stored in `notification-mongodb`.

```
Recipient
├── accountName       (String — links to Account.name)
├── email             (String)
└── scheduledNotifications  (Map<NotificationType, NotificationSettings>)
                              BACKUP → { active, frequency, lastNotified }
                              REMIND → { active, frequency, lastNotified }
```

**Files:**
- `notification-service/src/main/java/com/piggymetrics/notification/domain/Recipient.java`
- `notification-service/src/main/java/com/piggymetrics/notification/domain/NotificationSettings.java`

---

## Statistics Account (statistics-service)

A snapshot copy of account data, normalized to a base currency. Stored as a list of `DataPoint` records.

```
Account (statistics)
└── statisticsPerAccount  (List<DataPoint>)

DataPoint
├── id              (DataPointId → accountName + date)
├── incomes         (Set<ItemMetric>)
├── expenses        (Set<ItemMetric>)
├── statistics      (Map<StatisticMetric, BigDecimal>)
│                    INCOMES_AMOUNT, EXPENSES_AMOUNT, SAVING_AMOUNT
└── rates           (Map<Currency, BigDecimal> — exchange rates at snapshot time)
```

**File:** `statistics-service/src/main/java/com/piggymetrics/statistics/domain/`

---

## Data Flow on Account Save

```
User updates account in UI
    │
    ▼
PUT /accounts/current  (Account Service)
    │  saves to account-mongodb
    │
    ▼
StatisticsServiceClient.updateStatistics()  [Feign]
    │
    ▼
PUT /statistics/{accountName}  (Statistics Service)
    │  normalizes currencies, creates DataPoint
    └─ saves to statistics-mongodb
```

---

See [06-api-reference.md](06-api-reference.md) for the endpoints used in this flow.

[Next: API Reference →](06-api-reference.md)
