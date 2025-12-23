# Shadow Ledger System - Project Completion Summary


### 1. **Microservices Architecture** 
- [x] Event Service (Port 8081)
- [x] Shadow Ledger Service (Port 8082)
- [x] Drift Correction Service (Port 8083)
- [x] API Gateway (Port 8080)

### 2. **Event Service** 
- [x] POST /events endpoint
- [x] Event validation (amount > 0, type ∈ {debit, credit})
- [x] Unique eventId enforcement (idempotency)
- [x] Kafka producer to `transactions.raw`
- [x] PostgreSQL event logging
- [x] Actuator health & metrics endpoints

### 3. **Shadow Ledger Service** 
- [x] Kafka consumer from `transactions.raw` and `transactions.corrections`
- [x] Event deduplication by eventId
- [x] Deterministic ordering (timestamp, eventId)
- [x] Immutable ledger table in PostgreSQL
- [x] SQL window function for balance calculation
- [x] GET /accounts/{accountId}/shadow-balance endpoint
- [x] Negative balance prevention (business logic)
- [x] Actuator health & metrics endpoints

### 4. **Drift Correction Service** 
- [x] POST /drift-check endpoint
- [x] Shadow balance comparison via REST call
- [x] Mismatch detection (positive/negative drift)
- [x] Correction event generation
- [x] Kafka producer to `transactions.corrections`
- [x] POST /correct/{accountId} manual correction endpoint
- [x] Actuator health & metrics endpoints

### 5. **API Gateway** 
- [x] Spring Cloud Gateway configured
- [x] JWT authentication filter
- [x] RBAC enforcement (USER, AUDITOR, ADMIN roles)
- [x] X-Trace-Id header injection
- [x] POST /auth/token endpoint
- [x] Route forwarding to all services
- [x] 401/403 error responses

### 6. **Kafka Integration** 
- [x] Topic: transactions.raw
- [x] Topic: transactions.corrections
- [x] Event Service produces to raw
- [x] Shadow Ledger consumes from both topics
- [x] Drift Service produces to corrections
- [x] Docker Compose Kafka setup

### 7. **Dockerfiles** 
- [x] event-service/Dockerfile
- [x] shadow-ledger-service/Dockerfile
- [x] drift-correction-service/Dockerfile
- [x] api-gateway/Dockerfile
- [x] Multi-stage builds with Alpine Linux

### 8. **Automated Tests** 
- [x] EventValidationTest (event validation)
- [x] SqlWindowFunctionTest (balance calculation)
- [x] DriftDetectionTest (drift detection)
- [x] CorrectionEventGenerationTest (correction events)

### 9. **Documentation** 
- [x] ordering-rules.md (1 page)
- [x] correction-strategy.md (1 page)
- [x] aws-deployment.md (2 pages)
- [x] api-specs.yaml (OpenAPI 3.0)
- [x] README.md (comprehensive)


### 10. **Infrastructure** 
- [x] docker-compose.yml with Kafka, Zookeeper, PostgreSQL
- [x] PostgreSQL database with ledger_events table
- [x] Kafka topics auto-created

### 11. **Scripts** 
- [x] scripts/run-acceptance.sh (automated acceptance tests)
- [x] Database cleanup scripts
- [x] Service health check scripts

### 12. **Observability** 
- [x] /actuator/health on all services
- [x] /actuator/metrics on all services
- [x] Logging with timestamps
- [x] Service name in logs
- [x] X-Trace-Id propagation

---


### **Project Structure:**
```
shadow-ledger-system/
├── api-gateway/                    ✅ Complete
├── event-service/                  ✅ Complete
├── shadow-ledger-service/          ✅ Complete
├── drift-correction-service/       ✅ Complete
├── scripts/
│   └── run-acceptance.sh           ✅ Complete
├── docker-compose.yml              ✅ Complete
├── api-specs.yaml                  ✅ Complete
├── ordering-rules.md               ✅ Complete
├── correction-strategy.md          ✅ Complete
├── aws-deployment.md               ✅ Complete
├── README.md                       ✅ Complete
```


 **Test end-to-end flow:**
```bash
# Post event
curl -X POST http://localhost:8081/events \
  -H "Content-Type: application/json" \
  -d '{"eventId":"FINAL_TEST","accountId":"A99","type":"credit","amount":1000,"timestamp":1735561800000}'

# Wait 10 seconds
sleep 10

# Check balance
curl http://localhost:8082/accounts/A99/shadow-balance
```

Expected: `{"accountId":"A99","balance":1000.0,"lastEvent":"FINAL_TEST"}`

---

# Shadow Ledger System

A simplified but realistic banking backend system composed of multiple Spring Boot microservices for processing financial events, maintaining a shadow ledger, and detecting/correcting balance discrepancies.

## Architecture Overview

```
                            ┌─────────────────┐
                            │   API Gateway   │
                            │   Port: 8080    │
                            └────────┬────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
         ┌──────────▼────────┐ ┌────▼─────────┐ ┌───▼──────────────┐
         │  Event Service    │ │Shadow Ledger │ │Drift Correction  │
         │   Port: 8081      │ │   Service    │ │    Service       │
         └──────────┬────────┘ │ Port: 8082   │ │  Port: 8083      │
                    │          └──────┬───────┘ └────────┬─────────┘
                    │                 │                   │
                    └────────┬────────┴──────────┬────────┘
                             │                   │
                    ┌────────▼───────────────────▼────────┐
                    │           Kafka Topics              │
                    │  - transactions.raw                 │
                    │  - transactions.corrections         │
                    └────────┬────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   PostgreSQL    │
                    │   Port: 5432    │
                    └─────────────────┘
```

## Microservices

### 1. API Gateway (Port 8080)
- Single entry point for all client requests
- JWT authentication and RBAC enforcement
- X-Trace-Id header injection for distributed tracing
- Routes requests to backend services

### 2. Event Service (Port 8081)
- Receives financial events (debits/credits)
- Validates event data and enforces idempotency
- Publishes events to Kafka topic `transactions.raw`
- Stores events in PostgreSQL for traceability

### 3. Shadow Ledger Service (Port 8082)
- Consumes events from Kafka topics
- Maintains immutable event ledger
- Calculates running balance using SQL window functions
- Provides shadow balance query endpoint

### 4. Drift Correction Service (Port 8083)
- Compares CBS balances with shadow ledger
- Detects balance mismatches
- Generates correction events
- Publishes corrections to Kafka

## Prerequisites

- **Java 21** or higher
- **Docker** and Docker Compose
- **Maven 3.9+**
- **PostgreSQL** (via Docker)
- **Apache Kafka** (via Docker)

## Quick Start

### 1. Start Infrastructure (Kafka & PostgreSQL)

```bash
docker-compose up -d
```

Wait for services to be healthy (~30 seconds).

### 2. Start Microservices

**Terminal 1: Event Service**
```bash
cd event-service
./mvnw spring-boot:run
```

**Terminal 2: Shadow Ledger Service**
```bash
cd shadow-ledger-service
./mvnw spring-boot:run
```

**Terminal 3: Drift Correction Service**
```bash
cd drift-correction-service
./mvnw spring-boot:run
```

**Terminal 4: API Gateway**
```bash
cd api-gateway
./mvnw spring-boot:run
```

### 3. Verify Services

```bash
curl http://localhost:8081/actuator/health  # Event Service
curl http://localhost:8082/actuator/health  # Shadow Ledger
curl http://localhost:8083/actuator/health  # Drift Correction
curl http://localhost:8080/actuator/health  # API Gateway
```

## API Usage

### Submit Event
```bash
curl -X POST http://localhost:8081/events \
  -H "Content-Type: application/json" \
  -d '{
    "eventId": "E1001",
    "accountId": "A10",
    "type": "credit",
    "amount": 500,
    "timestamp": 1735561800000
  }'
```

### Query Shadow Balance
```bash
# Wait 5-10 seconds for Kafka processing
curl http://localhost:8082/accounts/A10/shadow-balance
```

Response:
```json
{
  "accountId": "A10",
  "balance": 500.0,
  "lastEvent": "E1001"
}
```

### Check Drift
```bash
curl -X POST http://localhost:8083/drift-check \
  -H "Content-Type: application/json" \
  -d '[
    {"accountId": "A10", "reportedBalance": 550}
  ]'
```

### Manual Correction
```bash
curl -X POST "http://localhost:8083/correct/A10?type=credit&amount=50"
```

## Testing

### Run Unit Tests
```bash
# Test all services
./mvnw test

# Test specific service
cd shadow-ledger-service
./mvnw test
```

### Run Acceptance Tests
```bash
./scripts/run-acceptance.sh
```

This runs end-to-end tests covering:
- Event validation
-  SQL window function balance calculation
-  Drift detection
-  Correction event generation

## Database Management

### View Ledger Events
```bash
docker exec -it postgres psql -U postgres -d postgres
```

```sql
SELECT * FROM ledger_events ORDER BY timestamp, event_id;
```

### Clear Database
```bash
docker exec postgres psql -U postgres -d postgres -c \
  "TRUNCATE TABLE ledger_events RESTART IDENTITY CASCADE;"
```

## Docker Deployment

### Build Images
```bash
# Event Service
cd event-service
docker build -t event-service:latest .

# Shadow Ledger Service
cd shadow-ledger-service
docker build -t shadow-ledger-service:latest .

# Drift Correction Service
cd drift-correction-service
docker build -t drift-correction-service:latest .

# API Gateway
cd api-gateway
docker build -t api-gateway:latest .
```

### Run with Docker Compose
```bash
docker-compose up -d
```

## Project Structure

```
shadow-ledger-system/
├── api-gateway/              # API Gateway service
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
├── event-service/            # Event processing service
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
├── shadow-ledger-service/    # Shadow ledger service
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
├── drift-correction-service/ # Drift correction service
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
├── scripts/
│   └── run-acceptance.sh     # Acceptance test suite
├── docker-compose.yml        # Infrastructure setup
├── api-specs.yaml           # OpenAPI specification
├── ordering-rules.md        # Event ordering documentation
├── correction-strategy.md   # Correction strategy documentation
├── aws-deployment.md        # AWS deployment guide
└── README.md               # This file
```

## Documentation

- **[API Specifications](api-specs.yaml)** - OpenAPI 3.0 documentation
- **[Ordering Rules](ordering-rules.md)** - Event ordering and idempotency
- **[Correction Strategy](correction-strategy.md)** - Drift detection and correction
- **[AWS Deployment](aws-deployment.md)** - Cloud deployment guide




## AWS Deployment

The Shadow Ledger Service can be deployed to AWS EC2. See **[aws-deployment.md](aws-deployment.md)** for detailed instructions.

Quick summary:
1. Build Docker image
2. Push to Amazon ECR
3. Launch EC2 instance
4. Run container with environment variables
5. Access via public IP

