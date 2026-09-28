
# Razorpay-Inspired Payment Gateway

A backend-focused, multi-tenant payment gateway built with **Java and Spring Boot**, designed to demonstrate real-world payment processing concepts including authentication, idempotency, transaction state management, asynchronous event processing, webhook delivery, reconciliation, and automated settlements.

> **Note:** This is an educational/personal project inspired by payment gateway architectures. It does not process real monetary transactions.

---

## 🚀 Features

### 1. Multi-Tenant Merchant Management

* Merchant onboarding and management
* Merchant-specific API credentials
* Isolated merchant context for API requests
* Merchant-level configuration and rate limiting
* Secure storage of sensitive credentials

### 2. API Key Authentication

* API-key based authentication for merchant APIs
* BCrypt hashing for stored credentials
* Redis caching for frequently accessed API credentials
* API key rotation with a configurable grace period
* Merchant context propagation throughout the request lifecycle

### 3. Payment Processing

Supports multiple payment methods:

* **Card**
* **UPI**
* **Net Banking**

Payment processing follows controlled transaction states to prevent invalid transitions.

Typical lifecycle:

```text
CREATED
   ↓
AUTHORIZED
   ↓
CAPTURED
   ↓
SETTLED
```

Additional flows are supported for:

```text
CREATED → CANCELLED
AUTHORIZED → REFUNDED
CAPTURED → REFUNDED
```

A state-machine-based approach is used to validate allowed payment transitions.

---

## 🔐 Idempotency

Payment APIs support **idempotency keys** to prevent duplicate processing when the same request is retried.

Example:

```http
POST /api/payments
Idempotency-Key: merchant-123-order-456
```

The implementation uses Redis to track requests that are currently being processed.

### Request flow

```text
Client Request
      ↓
Check Idempotency Key
      ↓
 ┌───────────────┐
 │ Key Exists?   │
 └───────┬───────┘
     Yes │       No
         ↓        ↓
   Return cached   Process request
      response          ↓
                       Save result
                          ↓
                  Cache completed response
```

### Handling duplicate requests

* If a request is already **in progress**, duplicate processing is prevented.
* If the request has **completed**, the previously generated response can be returned.
* Failed/incomplete idempotency entries are cleaned up so that legitimate retries can be processed.
* Completed idempotent responses are retained for **24 hours**.

This provides protection against duplicate payments caused by client retries, network failures, or repeated API requests.

---

## 🔄 Payment State Management

Payment transactions are managed using a state-machine approach.

Example:

```text
CREATED
   │
   ├──→ CANCELLED
   │
   ↓
AUTHORIZED
   │
   ├──→ REFUNDED
   │
   ↓
CAPTURED
   │
   ├──→ REFUNDED
   │
   ↓
SETTLED
```

Each operation validates the current payment state before performing the requested transition.

This prevents invalid operations such as:

```text
CANCELLED → CAPTURE
SETTLED   → CAPTURE
CREATED   → REFUND
```

---

## 🔑 API Key Rotation

API credentials can be rotated without immediately invalidating the existing key.

### Rotation flow

```text
Existing Key
     ↓
Generate New Key
     ↓
Activate New Key
     ↓
Grace Period
     ↓
Old Key Expires
```

The grace period allows existing integrations to migrate to the new credentials without immediate service interruption.

---

## ⚡ Redis

Redis is used for several performance and consistency-related operations:

* API-key caching
* Idempotency tracking
* In-progress request tracking
* Completed response caching
* Rate limiting
* Merchant/request context-related temporary data

This reduces unnecessary database lookups for frequently accessed data and provides fast access to short-lived transactional information.

---

## 🚦 Rate Limiting

The gateway applies **merchant-level rate limiting** to prevent excessive API requests from a single merchant.

Example:

```text
Merchant A
   ↓
Request
Request
Request
Request
   ↓
Rate Limit Check
   ↓
Allowed / Rejected
```

Rate limiting is applied using Redis-backed counters.

---

## 📨 Event-Driven Architecture

The application uses **Apache Kafka** for asynchronous event processing.

Instead of performing every operation synchronously, important payment events can be published for downstream processing.

Example:

```text
Payment Service
      ↓
Database Transaction
      ↓
Transactional Outbox
      ↓
Kafka
      ↓
Consumers
      ↓
Webhook / Settlement / Reconciliation Processing
```

---

## 📦 Transactional Outbox

The Transactional Outbox pattern is used to reliably publish events without creating inconsistencies between database updates and Kafka messages.

### Problem

Without an outbox:

```text
Update Database
      ↓
Publish Kafka Event
      ↓
Kafka Failure
```

The database transaction may succeed while the event is lost.

### Outbox approach

```text
Database Transaction
      ↓
Payment Update + Outbox Event
      ↓
Commit
      ↓
Outbox Publisher
      ↓
Kafka
```

The payment state change and event creation are persisted within the same database transaction.

A separate process can then publish pending outbox events to Kafka.

---

# 🔔 Webhooks

The gateway provides webhook functionality for notifying merchants about payment events.

Example:

```text
Payment Completed
       ↓
Generate Webhook
       ↓
Sign Payload
       ↓
Send to Merchant
       ↓
Merchant Endpoint
```

### Security

Webhook payloads are protected using **HMAC-based signatures** so that merchants can verify that the notification was generated by the gateway.

---

## 🔁 Webhook Retry Mechanism

Webhook delivery failures are handled using retry logic with exponential backoff.

```text
Attempt 1
   ↓ failure
Wait
   ↓
Attempt 2
   ↓ failure
Wait
   ↓
Attempt 3
   ↓
...
   ↓
Up to 7 attempts
```

If delivery continues to fail after the configured retry attempts, the event is moved to a **Dead Letter Queue (DLQ)**.

---

## ☠️ Dead Letter Queue & Replay

Failed webhook events are retained in the DLQ instead of being permanently discarded.

```text
Webhook Event
      ↓
Delivery Failure
      ↓
Retry
      ↓
Retry Limit Reached
      ↓
DLQ
      ↓
Manual Replay
```

This allows failed events to be investigated and replayed later.

---

# 💳 Payment Method Architecture

Payment methods are handled through an adapter-based design.

```text
              Payment Request
                    ↓
              Payment Router
                    ↓
       ┌────────────┼────────────┐
       ↓            ↓            ↓
     Card          UPI       Net Banking
    Adapter       Adapter      Adapter
```

This separates payment-method-specific processing from the core payment workflow.

New payment methods can therefore be introduced without significantly modifying the core payment-processing logic.

---

# 🔐 Card Vault & Tokenization

The project demonstrates card-vault concepts to avoid repeatedly exposing raw card information throughout the application.

### Flow

```text
Card Details
     ↓
Tokenization
     ↓
Vault
     ↓
Payment Token
     ↓
Future Transactions
```

Sensitive card information is protected using encryption, with **per-card encryption keys** rather than relying on a single shared encryption key.

The application uses AES-based encryption for sensitive stored information.

> This implementation is intended for learning and demonstration and should not be treated as a PCI-DSS-compliant production card vault.

---

# 💰 Settlement

The gateway supports merchant settlement processing.

A simplified flow is:

```text
Captured Payments
       ↓
Settlement Eligibility
       ↓
Settlement Batch
       ↓
Merchant Settlement
       ↓
Settlement Record
```

Settlement processing groups eligible transactions into settlement batches and records the resulting settlement information.

---

# 🔍 Reconciliation

Reconciliation helps identify inconsistencies between payment transaction records and settlement/payment-processing records.

The process can be represented as:

```text
Payment Transactions
        +
Settlement Records
        ↓
   Reconciliation
        ↓
Matched / Mismatched
```

Mismatched transactions can then be investigated instead of silently being treated as successfully settled.

---

# 🛡️ Security

The project incorporates several backend security practices:

* Spring Security
* API-key authentication
* BCrypt password/secret hashing
* JWT-based authentication where applicable
* Merchant-level authorization
* HMAC webhook signatures
* AES-based encryption for sensitive data
* API-key rotation
* Rate limiting
* Merchant context isolation
* Input validation

---

# 🏗️ High-Level Architecture

```text
                    ┌───────────────┐
                    │     Client    │
                    └───────┬───────┘
                            │
                            ↓
                    ┌───────────────┐
                    │   API Layer   │
                    └───────┬───────┘
                            │
                            ↓
                 ┌─────────────────────┐
                 │ Payment Services    │
                 │                     │
                 │ Order               │
                 │ Payment             │
                 │ Refund              │
                 │ Settlement          │
                 └──────────┬──────────┘
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
          PostgreSQL      Redis         Kafka
              │             │             │
              │             │             ↓
              │             │      Async Consumers
              │             │
              ↓             ↓
        Transactional    Idempotency
        Data             & Caching
                           

                    Webhook Service
                            │
                            ↓
                    Merchant Endpoint
```

---

# 🧰 Tech Stack

| Category          | Technology                  |
| ----------------- | --------------------------- |
| Language          | Java                        |
| Framework         | Spring Boot                 |
| Security          | Spring Security             |
| Authentication    | API Keys, JWT               |
| ORM               | Spring Data JPA / Hibernate |
| Database          | PostgreSQL                  |
| Cache             | Redis                       |
| Messaging         | Apache Kafka                |
| Event Reliability | Transactional Outbox        |
| Encryption        | AES                         |
| Hashing           | BCrypt                      |
| APIs              | REST                        |
| Architecture      | Microservices               |
| Build Tool        | Maven                       |
| Version Control   | Git                         |

---

# 📂 Core Concepts Demonstrated

This project focuses on backend engineering concepts commonly used in payment systems:

* Multi-tenancy
* REST API design
* Authentication & authorization
* API-key management
* API-key rotation
* Idempotency
* Distributed caching
* Rate limiting
* State machines
* Event-driven architecture
* Apache Kafka
* Transactional Outbox
* Webhooks
* HMAC signatures
* Retry mechanisms
* Exponential backoff
* Dead Letter Queues
* Event replay
* Card tokenization
* Encryption
* Reconciliation
* Settlement processing
* Database transactions

---

# ⚙️ Getting Started

## Prerequisites

Make sure the following are installed:

* Java
* Maven
* PostgreSQL
* Redis
* Apache Kafka

---

## Clone the Repository

```bash
git clone <your-repository-url>
cd <your-project-directory>
```

---

## Configure the Application

Update the application configuration with your local PostgreSQL, Redis, and Kafka settings.

Example:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/payment_gateway
spring.datasource.username=postgres
spring.datasource.password=your_password

spring.data.redis.host=localhost
spring.data.redis.port=6379

spring.kafka.bootstrap-servers=localhost:9092
```

---

## Build

```bash
mvn clean install
```

---

## Run

```bash
mvn spring-boot:run
```

---

# 🧪 Testing

The project includes automated tests covering important backend components and business logic.

Run tests using:

```bash
mvn test
```

---

# 📌 Example Payment Flow

A simplified payment flow looks like:

```text
Merchant
   ↓
Authenticate using API Key
   ↓
Create Order
   ↓
Create Payment
   ↓
Idempotency Check
   ↓
Validate Payment State
   ↓
Select Payment Method
   ↓
Process Payment
   ↓
Update Transaction State
   ↓
Create Outbox Event
   ↓
Publish Kafka Event
   ↓
Generate Webhook
   ↓
Notify Merchant
   ↓
Settlement
   ↓
Reconciliation
```

---

# ⚠️ Disclaimer

This project is built for **educational and portfolio purposes** and is inspired by concepts used in modern payment gateway systems.

It does not connect to real banking networks, payment processors, or financial institutions and should not be used to process real financial transactions.

---

## 👨‍💻 Author

**Vaibhav Kaushik**

Java Backend Developer | Spring Boot | Microservices

Technologies: **Java · Spring Boot · Spring Security · PostgreSQL · Redis · Kafka · JPA · REST APIs · Microservices**
