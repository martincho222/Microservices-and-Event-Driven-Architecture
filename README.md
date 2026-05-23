# The Complete Microservices & Event-Driven Architecture — Study Notes

Personal study notes for the Udemy course **"The Complete Microservices & Event-Driven Architecture"** by **Michael Pogrebinsky** (ex-Google Engineer, iSAQB Certified Software Architect). Course rating: 4.8/5.

> All notes are bilingual (Spanish / English) and follow a consistent format: definitions, ASCII diagrams, ShopFast running example, pros/cons tables, and key conclusions labeled "Regla de Pogrebinsky".

---

## Running Example — ShopFast

All diagrams and examples throughout these notes use **ShopFast**, a fictional e-commerce platform, as the running example. ShopFast is composed of the following squads/services:

`Orders` · `Catalog` · `Payments` · `Users` · `Search` · `Shipping` · `Notifications` · `Reviews` · `ML/Personalization` · `Inventory` · `Analytics` · `Loyalty` · `Fraud Detection` · `Warehouse`

---

## Table of Contents

### 1. Introduction & Foundations
> Root-level files covering the core concepts of microservices architecture.

| File | Topics |
|---|---|
| [introduction.md](introduction.md) | What are microservices, monolith vs microservices, Conway's Law |
| [benefits.md](benefits.md) | Benefits of microservices architecture |
| [challenges.md](challenges.md) | Challenges and trade-offs of microservices |
| [migration.md](migration.md) | High-level migration overview |

---

### 2. Migration to Microservices Architecture
> Strategies and patterns for decomposing a monolith into microservices.

| File | Topics |
|---|---|
| [migration.md](Migration%20to%20Microservices%20architecture/migration.md) | Migration strategies: Strangler Fig, Big Bang, parallel run |
| [migrationPart2.md](Migration%20to%20Microservices%20architecture/migrationPart2.md) | Decomposition patterns: by business capability, by subdomain (DDD) |
| [migrationPart3.md](Migration%20to%20Microservices%20architecture/migrationPart3.md) | Data migration, shared databases, anti-corruption layer |

---

### 3. Microservices Best Practices and Principles
> Design principles, team organization, and architectural best practices.

| File | Topics |
|---|---|
| [DevelopmentTeam.md](Microservices%20Best%20Practices%20and%20Principles/DevelopmentTeam.md) | Team size (Two-Pizza Rule), Conway's Law, squad model |
| [DatabaseInMicroservices.md](Microservices%20Best%20Practices%20and%20Principles/DatabaseInMicroservices.md) | Database per service, polyglot persistence, shared DB anti-pattern |
| [APIManagement.md](Microservices%20Best%20Practices%20and%20Principles/APIManagement.md) | API Gateway, BFF pattern, rate limiting, versioning |
| [DRYPrinciple.md](Microservices%20Best%20Practices%20and%20Principles/DRYPrinciple.md) | DRY vs shared libraries, code duplication trade-offs in microservices |
| [MicroFrontEndsPatterns.md](Microservices%20Best%20Practices%20and%20Principles/MicroFrontEndsPatterns.md) | Micro-frontends: composition patterns, routing, shared state |

---

### 4. Event-Driven Architecture
> Core concepts, patterns, and message delivery guarantees for EDA.

| File | Topics |
|---|---|
| [introduction.md](Event-Drive%20Architecture/introduction.md) | What is EDA, synchronous vs asynchronous communication |
| [benefits.md](Event-Drive%20Architecture/benefits.md) | Benefits of event-driven architecture |
| [challenges.md](Event-Drive%20Architecture/challenges.md) | EDA challenges: eventual consistency, ordering, idempotency |
| [EventDrivenArchitecture.md](Event-Drive%20Architecture/EventDrivenArchitecture.md) | Event broker, topics, partitions, consumers, producers |
| [MessageDeliveryED.md](Event-Drive%20Architecture/MessageDeliveryED.md) | At-most-once, at-least-once, exactly-once delivery semantics |
| [CasesAndPatternsED.md](Event-Drive%20Architecture/CasesAndPatternsED.md) | Event notification, event-carried state transfer, event sourcing |

---

### 5. Event-Driven Microservices — Design Patterns
> Advanced patterns for data consistency and complex workflows in EDA.

| File | Topics |
|---|---|
| [SagaPattern.md](Event-Driven%20Microservices%20-%20Design%20Patterns/SagaPattern.md) | Saga pattern: choreography vs orchestration, compensating transactions |
| [CQRSPattern.md](Event-Driven%20Microservices%20-%20Design%20Patterns/CQRSPattern.md) | CQRS: command/query separation, read/write models, projections |
| [EventSourcingPattern.md](Event-Driven%20Microservices%20-%20Design%20Patterns/EventSourcingPattern.md) | Event sourcing: event store, replaying events, snapshots |

---

### 6. Testing Microservices and Event-Driven Architecture
> Testing strategies from unit tests to production testing.

| File | Topics |
|---|---|
| [TestingPyramidForMicroservices.md](Testing%20Microservices%20and%20Event-Driven%20Architecture/TestingPyramidForMicroservices.md) | Testing pyramid adapted for microservices: unit, integration, e2e |
| [ContractTestAndProduction.md](Testing%20Microservices%20and%20Event-Driven%20Architecture/ContractTestAndProduction.md) | Contract tests (Pact, Spring Cloud Contract), Blue/Green, Canary deployments, PactFlow |

---

### 7. Observability in Microservices Architecture
> The three pillars of observability and their implementation.

| File | Topics |
|---|---|
| [TheThreePillarsOfObservability.md](Observability%20in%20Microservices%20Architecture/TheThreePillarsOfObservability.md) | Observability vs monitoring, Logs + Metrics + Traces overview |
| [DistributedLogging.md](Observability%20in%20Microservices%20Architecture/DistributedLogging.md) | Correlation ID, centralized aggregation (ELK), log levels, sampling, retention |
| [Metrics.md](Observability%20in%20Microservices%20Architecture/Metrics.md) | 5 Golden Signals: Latency, Traffic, Errors, Saturation, Utilization; Prometheus + Grafana |
| [DistributedTracing.md](Observability%20in%20Microservices%20Architecture/DistributedTracing.md) | Traces, spans, W3C traceparent, OTel Collector, Jaeger, EDA tracing challenges |

---

### 8. Deployment of Microservices and Event-Driven Architecture in Production
> Deployment models from VMs to containers to serverless.

| File | Topics |
|---|---|
| [MicroservicesDeployment.md](Deployment%20of%20Microservices%20and%20Event-Driven%20Architecture%20in%20Production/MicroservicesDeployment.md) | Multi-tenant VMs, Dedicated Instances, Dedicated Hosts; AWS/GCP/Azure solutions |
| [ServerlessDeployment.md](Deployment%20of%20Microservices%20and%20Event-Driven%20Architecture%20in%20Production/ServerlessDeployment.md) | FaaS motivation, cold start, pay-per-use, Lambda/Cloud Functions/Azure Functions |
| [ContainersForMicroservicesInDev.md](Deployment%20of%20Microservices%20and%20Event-Driven%20Architecture%20in%20Production/ContainersForMicroservicesInDev.md) | Dev/prod parity, VM overhead, container benefits, production challenges |
| [ContainerOrchestrationAndKubernetes.md](Deployment%20of%20Microservices%20and%20Event-Driven%20Architecture%20in%20Production/ContainerOrchestrationAndKubernetes.md) | K8s architecture, Pods, Deployments, Services, HPA, self-healing, EKS/GKE/AKS |

---

## Note Format

Every file follows the same structure:

```
## Section Title

**Español:** Explicación en español...

**English:** Explanation in English...

[ASCII diagram inside fenced code block]

[Markdown table for comparisons]

> **Regla de Pogrebinsky:** Key conclusion or rule of thumb.
```

---

## Course Info

- **Course:** The Complete Microservices & Event-Driven Architecture
- **Instructor:** Michael Pogrebinsky (ex-Google Engineer, iSAQB Certified)
- **Platform:** Udemy
- **Rating:** 4.8 / 5
- **Workbook:** [Course Workbook PDF](The+Complete+Microservices+%26+Event-Driven+Architecture+-+Course+Workbook.pdf)
