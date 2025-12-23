# Shadow Ledger System

A microservices-based shadow ledger system built with Spring Boot and Spring WebFlux for real-time financial transaction tracking, drift detection, and correction mechanisms.

## Architecture Overview

The Shadow Ledger System consists of four core microservices:

- **API Gateway** - Entry point with JWT authentication and request routing
- **Event Service** - Handles financial transaction events
- **Shadow Ledger Service** - Maintains shadow copies of ledger data
- **Drift Correction Service** - Detects and corrects data inconsistencies

## Prerequisites

- Java 11 or higher
- Docker and Docker Compose
- Gradle 7.x or higher

## 🏗️ Architecture
┌─────────────────┐
│   API Gateway   │ ← Single Entry Point
│   (JWT + RBAC)  │
└────────┬────────┘
│
┌──────────────┼──────────────┐
│              │              │
┌───▼───┐    ┌────▼────┐    ┌────▼─────┐
│ Event │    │ Ledger  │    │  Drift   │
│Service│    │ Service │    │ Service  │
└───┬───┘    └────┬────┘    └────┬─────┘
│             │              │
└─────► Kafka ◄──────────────┘
│
PostgreSQL

## Project Structure

shadow-ledger-system/
├── api-gateway/              # Port 8080
│   ├── src/main/java/com/banking/gateway/
│   │   ├── config/           # Security, JWT, Routes
│   │   └── controller/       # Auth endpoints
│   └── Dockerfile
│
├── event-service/            # Port 8081
│   ├── src/main/java/com/banking/event/
│   │   ├── controller/       # POST /events
│   │   ├── service/          # Business logic
│   │   ├── repository/       # JPA repositories
│   │   └── model/            # Entities & DTOs
│   └── Dockerfile
│
├── shadow-ledger-service/    # Port 8082
│   ├── src/main/java/com/banking/ledger/
│   │   ├── controller/       # GET /accounts/{id}/shadow-balance
│   │   ├── service/          # Kafka consumer, balance calc
│   │   └── repository/       # Window function queries
│   └── Dockerfile
│
├── drift-correction-service/ # Port 8083
│   ├── src/main/java/com/banking/drift/
│   │   ├── controller/       # POST /drift-check, /correct
│   │   └── service/          # Drift detection, correction
│   └── Dockerfile
│
├── docker-compose.yml        # Infrastructure setup
├── scripts/                  # Automation scripts
│   └── run-acceptance.sh     # End-to-end tests
├── ordering-rules.md         # Event ordering documentation
├── correction-strategy.md    # Drift correction strategy
├── aws-deployment.md         # AWS deployment guide
└── api-specs.yaml            # OpenAPI 3.0 spec

## Services Description

### 1. API Gateway
- **Port**: 8080 (default)
- **Purpose**: Central entry point for all client requests
- **Features**:
 - JWT-based authentication
 - Role-based access control (USER, AUDITOR, ADMIN)
 - Request routing to downstream services
 - CORS handling

**Security Endpoints**:
- `/auth/**` - Public authentication endpoints
- `/events/**` - Requires USER role
- `/drift-check` - Requires AUDITOR role
- `/correct/**` - Requires ADMIN role

### 2. Event Service
- **Purpose**: Manages financial transaction events
- **Features**:
 - Transaction event ingestion
 - Event validation and processing
 - Real-time event streaming

### 3. Shadow Ledger Service
- **Purpose**: Maintains shadow copies of ledger data
- **Features**:
 - Parallel ledger maintenance
 - Window-based data processing
 - Data consistency checks

### 4. Drift Correction Service
- **Purpose**: Detects and corrects data inconsistencies
- **Features**:
 - Automated drift detection
 - Correction algorithm implementation
 - Audit trail maintenance

## Quick Start

### Using Docker Compose

1. Clone the repository:
```bash
git clone <repository-url>
cd Shadow-Ledger-System

2. Build and start the services:
```bash
docker-compose up --build
```  
3. Verify services are running:
    docker-compose ps

## Manual Setup
1. Build each microservice using Gradle:
```bash
# API Gateway
cd api-Gateway && ./gradlew build

# Event Service
cd ../event-service && ./gradlew build

# Shadow Ledger Service
cd ../shadow-ledger-service && ./gradlew build

# Drift Correction Service
cd ../drift-correction-service && ./gradlew build
``` 

# Start each service in separate terminals
cd api-Gateway && ./gradlew bootRun
cd event-service && ./gradlew bootRun
cd shadow-ledger-service && ./gradlew bootRun
cd drift-correction-service && ./gradlew bootRun

## API Endpoints
Authentication:

POST /auth/login    - User authentication
POST /auth/register - User registration

Events:
GET  /events        - List events (USER role)
POST /events        - Create event (USER role)
GET  /events/{id}   - Get event details (USER role)

Drift Operations:
GET  /drift-check   - Check for drifts (AUDITOR role)
POST /correct/{id}  - Apply corrections (ADMIN role)

## Configuration
Each service has its own application.yml configuration file located in src/main/resources/. Key configuration areas include:
Database connections
Service discovery
Security settings
Logging levels

## Security
The system implements role-based access control with three roles:
USER: Can access event-related endpoints
AUDITOR: Can perform drift checks
ADMIN: Can execute correction operations
JWT tokens are required for all authenticated endpoints.

##  Monitoring
Health check endpoints are available at /actuator/health for each service.

