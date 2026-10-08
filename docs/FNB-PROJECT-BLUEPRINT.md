# FNB Stock System — Project Blueprint v1.0

## 1. Vision

Build a production-oriented inventory and warehouse management system for F&B operations.

The system focuses on stock accuracy, traceability, batch/expiry management, FEFO issuing, and controlled warehouse operations.

## 2. Problem Statement

F&B operations handle ingredients and goods with different batches, expiry dates, warehouses, receipts, issues, transfers, and adjustments. A simple quantity-based CRUD system is not enough because stock must remain traceable and business rules must be enforced.

FNB Stock System aims to provide a reliable central inventory platform that can support multiple warehouses while preserving stock history and business integrity.

## 3. Goals

- Centralize warehouse and inventory management.
- Track inventory by product, warehouse, and batch/lot where required.
- Support expiry-aware inventory and FEFO issuing.
- Support receiving, issuing, transferring, and adjusting stock.
- Enforce authentication, authorization, and business rules.
- Keep an audit trail for important inventory operations.
- Build a maintainable .NET backend suitable for production deployment.
- Leave clean extension points for future analytics and AI.

## 4. Scope — v1

### In scope

- User authentication and authorization.
- Role/permission management.
- Product and ingredient master data.
- Central warehouse and multiple warehouse locations.
- Batch/lot and expiry tracking.
- Inventory balance and inventory movements.
- Stock receiving.
- Stock issuing.
- FEFO allocation.
- Inter-warehouse transfer.
- Stock adjustment.
- Audit/history for important operations.
- Validation, error handling, logging, and health checks.
- Automated tests for core business rules and workflows.
- Dockerized deployment and CI/CD.
- Production deployment.

### Out of scope — v1

- POS.
- Accounting.
- HR/payroll.
- Customer-facing ordering.
- Mobile application.
- Microservices.
- Kafka/event streaming infrastructure.
- Kubernetes.
- AI/ML features.

These may be considered only after the core system is stable.

## 5. Core Actors

### System Admin

- Manage users, roles, permissions, and system configuration.
- Manage master data with appropriate authority.

### Warehouse Manager

- Manage warehouse operations.
- Review and perform controlled stock operations.
- Review inventory and movement history.

### Warehouse Staff

- Receive stock.
- View inventory.
- Issue stock.
- Perform permitted warehouse operations.

Exact permissions will be refined during authorization design.

## 6. Core Modules

1. Identity & Access
2. Product / Ingredient Management
3. Warehouse Management
4. Batch / Lot Management
5. Receiving
6. Inventory
7. Stock Issue
8. Stock Transfer
9. Stock Adjustment
10. Audit / Inventory History
11. Reporting foundation
12. System Administration

## 7. Core Domain Concepts

Initial domain concepts:

- User
- Role
- Permission
- Product
- Warehouse
- Batch
- Inventory
- Inventory Movement
- Stock Receipt
- Stock Receipt Item
- Stock Issue
- Stock Issue Item
- Stock Transfer
- Stock Transfer Item
- Stock Adjustment
- Audit Entry

The model is intentionally subject to refinement during domain analysis.

## 8. Core Business Rules

### Inventory integrity

- Stock cannot become negative unless an explicit future business rule allows it.
- Every material stock change must have a traceable business operation.
- Inventory updates must be atomic with the operation that caused them.

### Batch / expiry

- Batch-level tracking is required for products where expiry/lot traceability matters.
- Expired stock must not be silently treated as normally available stock.
- Inventory history must retain the affected batch where applicable.

### FEFO

For FEFO-managed products:

1. Select eligible stock.
2. Exclude unavailable/expired stock according to business rules.
3. Order eligible batches by earliest expiry.
4. Consume the earliest-expiring stock first.
5. Continue to the next batch when the requested quantity exceeds the current batch quantity.

### Transfer

A warehouse transfer must preserve consistency between source and destination inventory. A failed transfer must not leave the system with only one side applied.

### Adjustment

Stock adjustments are controlled operations and must record the reason and actor.

## 9. Target Architecture

Initial architecture: modular monolith with clear separation of responsibilities.

```
Client
  |
  v
ASP.NET Core API
  |
  +--> Application
  |       |
  |       v
  |     Domain
  |
  +--> Infrastructure
          |
          v
       SQL Server
```

Target project structure:

```
src/
  API/
  Application/
  Domain/
  Infrastructure/

tests/
  UnitTests/
  IntegrationTests/

docs/
```

Microservices are deliberately excluded from v1.

## 10. Technology Direction

Primary stack:

- C#
- .NET / ASP.NET Core Web API
- Entity Framework Core
- SQL Server
- REST / JSON
- JWT-based authentication
- Git / GitHub
- Docker
- GitHub Actions
- Cloud deployment

Specific infrastructure providers will be selected later based on cost, learning value, and deployment requirements.

## 11. Development Strategy

The project will be built using a learn-and-build loop:

```
Learn concept
    |
    v
Apply to a small FNB feature
    |
    v
Run / test
    |
    v
Debug
    |
    v
Understand
    |
    v
Refactor
    |
    v
Commit
```

The project should not be generated wholesale by AI. AI may be used for explanation, review, debugging guidance, and targeted assistance while the developer retains understanding of the implementation.

## 12. Milestones

### M1 — C# Foundation

- C# syntax
- OOP
- Collections
- LINQ
- Exceptions
- async/await
- Git fundamentals

### M2 — Data Foundation

- SQL
- relational modeling
- constraints
- indexes
- transactions
- initial FNB domain model

### M3 — Backend Foundation

- HTTP
- REST
- ASP.NET Core
- dependency injection
- DTOs
- validation
- EF Core
- SQL Server integration

### M4 — FNB MVP

End-to-end flow:

```
Product
  |
Warehouse
  |
Receiving
  |
Batch
  |
Inventory
  |
Issue
```

### M5 — Security

- Authentication
- Authorization
- Roles
- Permissions
- Protected endpoints

### M6 — Inventory Core

- FEFO
- Transfers
- Adjustments
- Transactions
- Concurrency
- Audit trail

### M7 — Quality & Production

- Unit tests
- Integration tests
- API tests where useful
- Logging
- Health checks
- Docker
- Configuration/secrets
- CI/CD

### M8 — Go Live

- Production database
- HTTPS
- Deployment
- Monitoring/logging
- Backup strategy
- Documentation
- Production smoke testing

## 13. Definition of Done

A feature is not considered complete merely because its API works.

A feature should normally satisfy:

- Business rules implemented.
- Validation implemented.
- Authorization checked where required.
- Persistence correct.
- Transaction/concurrency behavior considered.
- Relevant automated tests added.
- Errors handled predictably.
- Logging/audit requirements satisfied.
- Documentation updated where useful.
- Code reviewed/refactored.
- Git commit is clear and focused.

## 14. Go-Live Criteria

Before production:

- Core inventory workflows work end-to-end.
- FEFO behavior is tested.
- Authentication and authorization are enabled.
- Critical business rules have automated tests.
- Database migrations are reproducible.
- Production configuration is separated from source code.
- HTTPS is enabled.
- Application health checks work.
- Logging is available.
- Backup/recovery approach is documented.
- CI/CD pipeline passes.
- Production smoke tests pass.

## 15. Future Extension — AI

AI is intentionally not part of the initial implementation.

Once sufficient historical data exists, possible future capabilities include:

- Demand forecasting.
- Reorder suggestions.
- Waste/spoilage prediction.
- Inventory anomaly detection.

Potential direction:

```
Historical Inventory Data
        |
        v
Analytics / Feature Engineering
        |
        v
AI / ML
        |
        +--> Demand Forecast
        +--> Waste Prediction
        +--> Reorder Suggestion
```

The v1 architecture should preserve clean data and domain boundaries so these capabilities can be added later without forcing AI into the core inventory workflow.

## 16. Scope Control Rule

New ideas must first be classified as:

- Required for v1.
- Useful but post-v1.
- Future experiment.

If a feature does not directly support the core inventory/warehouse problem, it should normally be placed into the post-v1 backlog rather than added immediately.

---

**Status:** Planning / Discovery

**Version:** 1.0

**Next step:** Validate this blueprint against the detailed FNB business requirements before starting implementation.
