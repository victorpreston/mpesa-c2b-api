# M-Pesa Daraja API

A production-ready Spring Boot backend for Safaricom's M-Pesa Daraja APIs. Covers the full integration surface — STK Push, C2B, B2C, B2B, Transaction Status, Account Balance, Reversals, Dynamic QR, and Bill Manager — with a clean SDK layer, PostgreSQL persistence for C2B callbacks, and Docker support.

[![Java](https://img.shields.io/badge/Java-21-orange)](https://openjdk.org/projects/jdk/21/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-4.0.6-brightgreen)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue)](https://www.postgresql.org/)
[![Tests](https://img.shields.io/badge/Tests-42%20passing-success)]()
[![License](https://img.shields.io/badge/License-MIT-yellow)]()


## Architecture

![Architecture Overview](docs/architecture.png)


## System Flow & Schema

![System Flow and Database Schema](docs/architecture-flow.png)


## API Endpoints

### Outbound — you call these

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/mpesa/stk-push` | Trigger payment prompt on customer phone |
| `POST` | `/api/v1/mpesa/stk-push/query` | Check STK push status |
| `POST` | `/api/v1/mpesa/b2c/payment` | Send money to a customer |
| `POST` | `/api/v1/mpesa/b2b/payment` | Pay another business |
| `POST` | `/api/v1/mpesa/transactions/status` | Query a transaction |
| `POST` | `/api/v1/mpesa/account-balance` | Check business account balance |
| `POST` | `/api/v1/mpesa/reversal` | Reverse a transaction |
| `POST` | `/api/v1/mpesa/dynamic-qr` | Generate a payment QR code |
| `POST` | `/api/v1/mpesa/bill-manager/invoices/single` | Create a single invoice |
| `POST` | `/api/v1/mpesa/bill-manager/invoices/bulk` | Create bulk invoices |
| `POST` | `/api/v1/mpesa/bill-manager/invoices/cancel` | Cancel an invoice |
| `POST` | `/api/v1/mpesa/bill-manager/reconciliation` | Reconcile billing |

### Admin

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/mpesa/admin/register-urls` | Register your callback URLs with Daraja |
| `POST` | `/api/v1/mpesa/admin/simulate` | Simulate a C2B payment (sandbox only) |

### Inbound — Safaricom calls these

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/mpesa/c2b/confirmation` | C2B payment confirmation (persisted) |
| `POST` | `/api/v1/mpesa/c2b/validation` | C2B payment validation |
| `POST` | `/api/v1/mpesa/results/stk` | STK Push async result |
| `POST` | `/api/v1/mpesa/results` | B2C / B2B / Reversal / Status / Balance result |
| `POST` | `/api/v1/mpesa/results/timeout` | Queue timeout notification |

### Transaction Queries

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/v1/mpesa/c2b/all` | All C2B transactions |
| `GET` | `/api/v1/mpesa/c2b/transaction/{transactionId}` | By Safaricom transaction ID |
| `GET` | `/api/v1/mpesa/c2b/msisdn/{msisdn}` | By customer phone number |
| `GET` | `/api/v1/mpesa/c2b/shortcode/{shortcode}` | By business shortcode |
| `GET` | `/api/v1/mpesa/c2b/health` | Health check |


## Getting Started

### Prerequisites

- Java 21+
- Docker & Docker Compose, or PostgreSQL 16+ running locally
- A Safaricom Daraja account — [register here](https://developer.safaricom.co.ke/)
- A publicly accessible HTTPS URL for Daraja callbacks — use [ngrok](https://ngrok.com/) locally

### 1. Clone

```bash
git clone https://github.com/victorpreston/mpesa-api.git
cd mpesa-api
```

### 2. Configure

Create `src/main/resources/application-dev.properties`:

```properties
# Database
spring.datasource.url=jdbc:postgresql://localhost:5432/mpesa_db
spring.datasource.username=postgres
spring.datasource.password=your_password
spring.jpa.hibernate.ddl-auto=update

# Daraja credentials (from developer.safaricom.co.ke)
mpesa.consumer-key=YOUR_CONSUMER_KEY
mpesa.consumer-secret=YOUR_CONSUMER_SECRET

# Business shortcode
mpesa.shortcode=600984
mpesa.lipa-na-mpesa-shortcode=174379
mpesa.passkey=YOUR_LIPA_NA_MPESA_PASSKEY

# Callback URLs (must be publicly accessible HTTPS)
mpesa.confirmation-url=https://your-domain.com/api/v1/mpesa/c2b/confirmation
mpesa.validation-url=https://your-domain.com/api/v1/mpesa/c2b/validation
mpesa.callback-url=https://your-domain.com/api/v1/mpesa/results/stk
mpesa.result-url=https://your-domain.com/api/v1/mpesa/results
mpesa.timeout-url=https://your-domain.com/api/v1/mpesa/results/timeout

# sandbox or production
mpesa.environment=sandbox
```

### 3. Run with Docker

```bash
docker-compose up -d
```

App runs at `http://localhost:8080`. PostgreSQL at `localhost:5432`.

### 4. Run manually

```bash
./mvnw spring-boot:run -Dspring.profiles.active=dev
```


## SDK Layer

The `DarajaSdk` interface is the single entry point for all Daraja operations. Authentication, payload construction, and HTTP are handled internally — inject the interface and call the method.

```
DarajaSdk  ──▶  DarajaSdkService
                    ├── DarajaPayloadFactory   builds Safaricom-compliant request bodies
                    ├── DefaultDarajaClient    executes authenticated HTTPS calls
                    └── MpesaAuthService       fetches and caches the OAuth token
```

**Example:**

```java
@Autowired
DarajaSdk darajaSdk;

DarajaApiResponse response = darajaSdk.stkPush(new StkPushRequest(
    "254712345678",
    new BigDecimal("500"),
    "ORDER-001",
    "Payment for order",
    null, null, null
));
```

## Project Structure

```
src/main/java/com/mpesa_daraja_api/mpesa_daraja_api/
├── config/           DarajaProperties, ApplicationConfig (RestClient, Clock, ObjectMapper)
├── controller/
│   ├── callback/     C2bCallbackController, ResultCallbackController
│   ├── DarajaController.java
│   └── MpesaAdminController.java
├── dto/
│   ├── request/      StkPushRequest, B2cPaymentRequest, B2bPaymentRequest, ...
│   └── response/     DarajaApiResponse, AsyncCallbackResult, MpesaCallbackRequest, ...
├── entity/           MpesaTransaction
├── enums/            CommandId, IdentifierType, TransactionStatus, Environment
├── exception/        DarajaApiException, GlobalExceptionHandler
├── interfaces/       DarajaSdk, DarajaClient
├── repository/       MpesaTransactionRepository
├── sdk/              DefaultDarajaClient
└── service/
    ├── auth/         MpesaAuthService
    ├── c2b/          MpesaTransactionService, MpesaUrlRegistrationService, MpesaSimulationService
    ├── payload/      DarajaPayloadFactory
    └── sdk/          DarajaSdkService
```

## Tests

**42 tests — 100% pass rate**

| Test Class | Tests | What it covers |
|-----------|-------|----------------|
| `MpesaTransactionServiceTest` | 10 | Persistence, deduplication, queries |
| `MpesaSimulationServiceTest` | 9 | Input validation for CommandID, amount, phone |
| `DefaultDarajaClientTest` | 4 | Auth header, URL resolution, error handling |
| `ResultCallbackControllerIntegrationTest` | 4 | STK, async result, timeout endpoints |
| `MpesaAdminControllerIntegrationTest` | 5 | Admin endpoint accessibility |
| `MpesaC2bControllerIntegrationTest` | 6 | C2B callback and query endpoints |
| `MpesaAuthServiceSimpleTest` | 2 | Token fetch and caching |
| `DarajaPayloadFactoryTest` | 1 | STK payload construction |
| `MpesaDarajaApiApplicationTests` | 1 | Spring context loads |

```bash
./mvnw test -Dspring.profiles.active=test
```

## CI/CD

| Workflow | Trigger | Environment |
|----------|---------|-------------|
| `dev.yaml` | Push to `dev` | Development |
| `uat.yaml` | Push to `uat` | UAT |
| `prod.yaml` | Push to `master` | Production |

Each workflow runs the full test suite, builds a Docker image, and pushes to Docker Hub.

## Tech Stack

| | Technology | Version |
|--|-----------|---------|
| Language | Java | 21 |
| Framework | Spring Boot | 4.0.6 |
| ORM | Spring Data JPA / Hibernate | 7.x |
| Database | PostgreSQL | 16 |
| HTTP Client | Spring RestClient | 6.x |
| Validation | Jakarta Validation | 3.x |
| Testing | JUnit 5 + Mockito | Jupiter |
| Build | Maven | 3.8+ |
| Containers | Docker + Compose | — |
| CI | GitHub Actions | — |

## License

MIT
