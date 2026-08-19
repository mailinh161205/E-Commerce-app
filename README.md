# E-Commerce Microservices Platform
 
A backend microservices system for an e-commerce platform, built with Spring Boot and Spring Cloud. Each domain (product, customer) is an independently deployable service with its own database, registered through Eureka Discovery and configured centrally through Spring Cloud Config Server. Product creation events are published to Kafka for downstream consumers.
 
## Tech Stack
 
**Core**
- Java 17, Spring Boot 3.3.5, Maven
**Microservices Infrastructure**
- Spring Cloud Config Server — centralized configuration, served from a native (classpath) repository
- Netflix Eureka — service discovery and registration
**Data Layer**
- PostgreSQL + Spring Data JPA + Flyway Migrations (Product Service)
- MongoDB + Spring Data MongoDB (Customer Service)
**Messaging**
- Apache Kafka + Zookeeper — asynchronous, event-driven communication (Product Service publishes `ProductCreatedEvent`)
**Observability**
- Spring Boot Actuator + Micrometer Tracing (Brave) + Zipkin — distributed tracing
**Infra / DevOps**
- Docker Compose — spins up PostgreSQL, pgAdmin, MongoDB, Mongo Express, Zookeeper, Kafka, Zipkin, and MailDev for local development
## Architecture
 
```
                         ┌───────────────────┐
                         │   Config Server    │  :8888
                         │ (Spring Cloud Cfg)  │
                         └─────────┬──────────┘
                                   │ config
                 ┌─────────────────┼─────────────────┐
                 │                                    │
        ┌────────▼────────┐                 ┌─────────▼────────┐
        │  Product Service  │                 │ Customer Service  │
        │      :8050        │                 │      :8090        │
        │  PostgreSQL/JPA    │                 │  MongoDB           │
        └────────┬──────────┘                 └─────────┬─────────┘
                 │ registers                             │ registers
                 └───────────────┬────────────────────────┘
                                  │
                         ┌────────▼─────────┐
                         │     Discovery      │  :8761
                         │  (Eureka Server)    │
                         └─────────────────────┘
 
Product Service ──publishes──▶  Kafka (product-created-topic)  ──consumed by──▶ downstream listeners
```
 
## Project Structure
 
```
e-commerce-app/
├── docker-compose.yml          # Postgres, pgAdmin, MongoDB, Kafka, Zookeeper, Zipkin, MailDev
└── services/
    ├── config-server/          # Spring Cloud Config Server (serves per-service YAML)
    │   └── src/main/resources/configurations/
    ├── discovery/               # Eureka Server
    ├── product/
    │   └── src/main/java/org/alibou/ecommerce/
    │       ├── product/         # Product entity, controller, service, repository, DTOs
    │       ├── category/        # Category entity, controller, service, repository, DTOs
    │       ├── kafka/           # ProductCreatedEvent, producer, consumer
    │       ├── exception/       # Domain exceptions
    │       └── handler/         # Global exception handling
    └── customer/
        └── src/main/java/org/alibou/ecommerce/
            ├── customer/        # Customer entity, controller, service, repository, DTOs
            ├── exception/
            └── handler/
```
 
## API Overview
 
| Resource | Base Path | Key Endpoints |
|---|---|---|
| Product | `/api/v1/product` | `POST /`, `PUT /`, `GET /`, `GET /{id}`, `GET /exits/{id}`, `DELETE /{id}` |
| Category | `/api/v1/category` | `POST /`, `GET /` |
| Customer | `/api/v1/customer` | `POST /`, `PUT /`, `GET /`, `GET /{id}`, `GET /exits/{id}`, `DELETE /{id}` |
 
### Event-Driven Flow (Kafka)
 
1. `POST /api/v1/product` validates the request, resolves the target `Category`, and persists the product (PostgreSQL, via Flyway-managed schema).
2. On success, Product Service publishes a `ProductCreatedEvent` (product id, name, price, quantity, category id) to the `product-created-topic` Kafka topic.
3. A listener consumes the event and logs it — proving the producer → Kafka → consumer pipeline works end-to-end. In a fuller system, this consumer would live in a separate downstream service (e.g. inventory sync, search indexing, or notifications).
## Getting Started
 
**Prerequisites**: Java 17+, Maven, Docker & Docker Compose
 
**1. Start infrastructure**
 
```bash
cd e-commerce-app
docker compose up -d
```
 
This brings up PostgreSQL (`5432`), pgAdmin (`5050`), MongoDB (`27017`), Mongo Express (`8081`), Zookeeper (`22181`), Kafka (`9092`), Zipkin (`9411`), and MailDev (`1080`).
 
**2. Start services in order** (each in its own terminal)
 
```bash
cd services/config-server && mvn spring-boot:run   # :8888 — must be first
cd services/discovery     && mvn spring-boot:run   # :8761 — Eureka dashboard
cd services/product       && mvn spring-boot:run   # :8050
cd services/customer      && mvn spring-boot:run   # :8090
```
 
Eureka dashboard: `http://localhost:8761`
 
**3. Try the API**
 
```bash
# Create a category
curl -X POST http://localhost:8050/api/v1/category \
  -H "Content-Type: application/json" \
  -d '{"name":"Electronics","description":"Electronic devices"}'
 
# Create a product (use the category id returned above)
curl -X POST http://localhost:8050/api/v1/product \
  -H "Content-Type: application/json" \
  -d '{"name":"Wireless Mouse","description":"2.4GHz","availableQuantity":50,"price":19.99,"categoryId":1}'
 
# Watch the Product Service console — the Kafka consumer logs the ProductCreatedEvent
 
# Create a customer
curl -X POST http://localhost:8090/api/v1/customer \
  -H "Content-Type: application/json" \
  -d '{"firstname":"Linh","lastname":"Do","email":"linh@example.com"}'
```
 
> Note: `spring.datasource.password` for Product Service defaults to `quanla001` for local dev but can be overridden with the `DB_PASSWORD` environment variable — avoid committing real credentials.
 
## Roadmap
 
- [ ] Dedicated downstream consumer for `ProductCreatedEvent` (e.g. inventory or search-indexing service)
- [ ] Extend distributed tracing (Zipkin/Micrometer) to Customer Service
- [ ] Authentication/authorization (JWT) across services via an API Gateway
- [ ] Order Service to complete the purchase flow (cart → order → payment)
