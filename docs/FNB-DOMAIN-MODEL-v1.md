# FNB Stock System — Domain Model v1

**Status:** Discovery / Design baseline  
**Version:** 1.0  
**Scope:** Core inventory and warehouse domain

> This document translates the agreed business discussions into a first domain model. It is framework-agnostic: it is not an EF Core model or database schema yet.

---

## 1. Domain Principles

The FNB system is built around one core rule:

> Every material inventory change must be caused by a traceable business operation.

Inventory must not be edited directly.

~~~
Business Operation
      |
      v
Inventory Movement
      |
      v
Inventory State
~~~

Core operations:

- Receiving
- Issue
- Transfer
- Adjustment
- Waste / Disposal

---

## 2. Core Domain Map

~~~
                         PRODUCT
                            |
                            v
                          BATCH
                            |
                            v
                        INVENTORY
                            |
          +-----------------+-----------------+
          |                 |                 |
          v                 v                 v
       RECEIPT            ISSUE           TRANSFER
          |                 |                 |
          |                 |                 +--> IN_TRANSIT
          |                 |                 |        |
          |                 |                 |        v
          |                 |                 |    RECEIVING
          |                 |                 |        |
          |                 |                 |        v
          |                 |                 | DISCREPANCY
          |                 |                 |
          +-----------------+-----------------+
                            |
                            v
                   INVENTORY MOVEMENT
                            ^
                            |
                    ADJUSTMENT / WASTE
~~~

Supporting concepts:

- User
- Role
- Permission
- Supplier
- Warehouse
- Audit Entry

---

## 3. Entity Catalogue

### 3.1 Product

Represents a stockable item or ingredient.

Responsibilities:

- Define product identity.
- Define whether batch/lot tracking is required.
- Define whether expiry tracking is required.
- Provide master data used by inventory operations.

Key attributes:

| Attribute | Meaning |
|---|---|
| Id | Unique product identifier |
| SKU | Business identifier |
| Name | Product name |
| Unit | Stock unit |
| TrackBatch | Whether inventory must identify a batch/lot |
| TrackExpiry | Whether expiry must be tracked |
| IsActive | Whether the product can be used |

Business rules:

- SKU should be unique.
- Inactive products should not be used for new stock operations.
- TrackExpiry should normally imply batch-level traceability.
- A product that does not require batch tracking does not need a Batch record for normal inventory.

---

### 3.2 Warehouse

Represents a physical stock location.

Types:

- CENTRAL
- BRANCH

The Central Warehouse is not a separate entity. It is a Warehouse with type CENTRAL.

Key attributes:

| Attribute | Meaning |
|---|---|
| Id | Unique warehouse identifier |
| Code | Business identifier |
| Name | Warehouse name |
| Type | CENTRAL or BRANCH |
| IsActive | Whether operations are allowed |

Business rules:

- Warehouse code should be unique.
- A transfer cannot have the same source and destination warehouse.
- Only active warehouses participate in new operations.

---

### 3.3 Batch

Represents a specific lot of a product.

Used when the product requires batch/lot tracking.

Key attributes:

| Attribute | Meaning |
|---|---|
| Id | Unique batch identifier |
| ProductId | Product represented by the batch |
| BatchNumber | Supplier/manufacturer lot number |
| ManufacturedAt | Optional manufacture date |
| ExpiryDate | Optional expiry date |
| Status | ACTIVE or EXPIRED |

Business rules:

- A batch belongs to exactly one Product.
- Batch number must be traceable to receiving.
- A batch becomes EXPIRED when its expiry date has passed.
- Expired stock remains physically recorded.
- Expired stock is not normally available for issue.
- Expired stock cannot be manager-overridden for normal issue in v1.
- Expired stock must be handled through Waste/Disposal.

---

### 3.4 Inventory

Represents stock belonging to a Warehouse for a Product and, when applicable, a Batch.

Conceptually:

~~~
Warehouse + Product + Batch(optional) -> Inventory
~~~

Key attributes:

| Attribute | Meaning |
|---|---|
| Id | Unique inventory record |
| WarehouseId | Owning warehouse |
| ProductId | Stocked product |
| BatchId | Specific batch when required |
| Quantity | Physical quantity currently on hand |

Important concepts:

#### On Hand

Physical stock belonging to the warehouse.

#### Available

Stock eligible for normal issue.

For v1:

~~~
Available <= On Hand
~~~

Expired/unavailable stock remains On Hand but is excluded from Available.

#### In Transit

Stock shipped from a source warehouse but not yet received by the destination.

In-transit stock is represented by the active Stock Transfer rather than pretending it belongs to the destination warehouse.

Reserved inventory is intentionally not part of v1 because there is no order/reservation requirement yet.

---

## 4. Inventory Invariants

### 4.1 No silent inventory edits

Do not allow direct quantity editing as a normal business operation.

Every material change must originate from a business operation and create an Inventory Movement.

### 4.2 No negative stock

Unless a future explicit business rule introduces controlled negative stock:

~~~
Inventory.Quantity >= 0
~~~

### 4.3 Atomic stock changes

The operation and its inventory changes must succeed or fail together.

Examples:

- Receipt completion + inventory increase
- Issue completion + inventory decrease
- Transfer shipment + source decrease/in-transit creation
- Transfer completion + destination increase
- Adjustment approval + inventory change

### 4.4 Traceability

For tracked products, a stock movement must retain the affected Batch.

---

## 5. Inventory Movement

Inventory Movement is the historical ledger of stock changes.

Examples:

~~~
+100  RECEIPT
-50   ISSUE
-20   ISSUE
-30   TRANSFER_OUT
+30   TRANSFER_IN
-3    ADJUSTMENT
-10   WASTE
~~~

A movement should record at least:

| Attribute | Meaning |
|---|---|
| Id | Movement identifier |
| WarehouseId | Warehouse affected |
| ProductId | Product affected |
| BatchId | Batch affected when applicable |
| QuantityChange | Positive or negative change |
| MovementType | RECEIPT / ISSUE / TRANSFER / ADJUSTMENT / WASTE |
| ReferenceId | Source business operation |
| CreatedBy | Actor |
| CreatedAt | Timestamp |

The movement ledger is the basis for auditability, inventory history, reconciliation, and future analytics.

---

## 6. Stock Receipt

Represents receiving stock into a warehouse from a Supplier.

Relationship:

~~~
Supplier
   |
   v
StockReceipt
   |
   +--> StockReceiptItem
             |
             +--> Product
             +--> Batch (when required)
~~~

### Workflow

~~~
DRAFT
  |
  v
PENDING_APPROVAL
  |
  v
APPROVED
  |
  v
COMPLETED
~~~

Inventory changes only when the receiving operation is completed.

### StockReceipt

Key attributes:

| Attribute | Meaning |
|---|---|
| Id | Receipt identifier |
| WarehouseId | Receiving warehouse |
| SupplierId | Supplier |
| Status | Workflow status |
| CreatedBy | Creator |
| ApprovedBy | Approver |
| CreatedAt | Creation timestamp |
| ApprovedAt | Approval timestamp |
| CompletedAt | Completion timestamp |

### StockReceiptItem

| Attribute | Meaning |
|---|---|
| Id | Item identifier |
| ReceiptId | Parent receipt |
| ProductId | Received product |
| BatchId | Batch when required |
| Quantity | Quantity received |

Business rules:

- Draft receipt does not affect inventory.
- Only valid and approved operations can complete.
- Completion creates inventory movement(s).
- Batch and expiry data are required where the Product requires them.

---

## 7. Stock Issue

Represents stock leaving a warehouse for a known business purpose.

### Workflow

~~~
DRAFT
  |
  v
PENDING_APPROVAL
  |
  v
APPROVED
  |
  v
COMPLETED
~~~

Inventory is changed only when the issue is completed.

### StockIssue

Key attributes:

| Attribute | Meaning |
|---|---|
| Id | Issue identifier |
| WarehouseId | Source warehouse |
| Purpose | Why stock is being issued |
| Reason | Human-readable business reason |
| Status | Workflow status |
| CreatedBy | Creator |
| ApprovedBy | Approver |
| CreatedAt | Timestamp |
| ApprovedAt | Timestamp |
| CompletedAt | Timestamp |

### Purpose

Initial controlled values:

- PRODUCTION
- INTERNAL_CONSUMPTION
- WASTE
- EXPIRED
- DAMAGED
- OTHER

Reason is required to explain the concrete business context.

Examples:

~~~
Purpose: PRODUCTION
Reason: Prepare ingredients for evening shift.

Purpose: INTERNAL_CONSUMPTION
Reason: Cleaning the kitchen area.

Purpose: WASTE
Reason: Product damaged during storage.
~~~

### FEFO

For FEFO-managed products:

1. Find eligible stock in the source warehouse.
2. Exclude expired/unavailable batches.
3. Order eligible batches by earliest expiry.
4. Consume earliest-expiring stock first.
5. Continue to the next batch when necessary.
6. Record each affected batch in Inventory Movement.

Expired batches cannot be used for normal Issue in v1.

---

## 8. Stock Transfer

Represents movement of stock between warehouses.

Relationship:

~~~
Source Warehouse
       |
       v
StockTransfer
       |
       +--> StockTransferItem
                 |
                 +--> Product
                 +--> Batch
       |
       v
Destination Warehouse
~~~

### Workflow

~~~
DRAFT
  |
  v
PENDING_APPROVAL
  |
  v
APPROVED
  |
  v
IN_TRANSIT
  |
  v
RECEIVING
  |
  +--------------------+
  |                    |
  v                    v
COMPLETED          DISCREPANCY
                       |
                       v
                 MANAGER REVIEW
                       |
                       v
                    RESOLVED
                       |
                       v
                    COMPLETED
~~~

The business lifecycle above is the baseline for implementation.

### Transfer item

A tracked product must identify the exact batch being transferred.

Example:

~~~
Milk
Batch A -> 50
Batch B -> 30
Total    -> 80
~~~

Do not represent this only as Milk = 80.

### In-transit model

When the source ships stock:

~~~
Source Inventory
       -100

Transfer In Transit
       +100
~~~

The destination does not receive the stock yet.

When the destination receives:

~~~
Transfer In Transit
       -100

Destination Inventory
       +100
~~~

This preserves the fact that stock can physically be between warehouses.

---

## 9. Transfer Discrepancy

A discrepancy occurs when received quantity differs from expected quantity.

Example:

~~~
Expected: 100
Received: 95
Difference: -5
~~~

The system must not silently discard the difference.

A discrepancy records:

| Attribute | Meaning |
|---|---|
| Id | Discrepancy identifier |
| TransferId | Related transfer |
| ExpectedQuantity | Expected amount |
| ReceivedQuantity | Actual amount |
| Difference | Difference |
| Reason | Explanation |
| ReportedBy | Person reporting |
| ResolvedBy | Manager handling it |
| Status | OPEN / RESOLVED |
| CreatedAt | Timestamp |
| ResolvedAt | Timestamp |

Partial receiving is allowed.

The remaining difference requires Manager review before the transfer is finally completed.

---

## 10. Stock Adjustment

Represents a controlled correction between system quantity and physical reality.

Example:

~~~
System: 100
Physical: 97

Adjustment: -3
~~~

### Workflow

~~~
DRAFT
  |
  v
PENDING_APPROVAL
  |
  v
APPROVED
  |
  v
COMPLETED
~~~

Staff can create an adjustment, but Manager approval is required.

Required information:

- Product
- Batch where applicable
- Warehouse
- Quantity change
- Reason
- Actor
- Approval information

Adjustment completion creates an Inventory Movement.

---

## 11. Waste / Disposal

Waste is a controlled stock-decreasing operation.

Typical cases:

- Expired stock
- Damaged stock
- Spoilage
- Operational waste

Example:

~~~
Batch M001
Status: EXPIRED
Quantity: 20

Waste / Disposal
Quantity: 20
Reason: Expired milk
       |
       v
Inventory Movement -20
~~~

The physical stock remains traceable until disposal is recorded.

---

## 12. Supplier

Represents an external source of received stock.

Key attributes:

| Attribute | Meaning |
|---|---|
| Id | Supplier identifier |
| Code | Business identifier |
| Name | Supplier name |
| Contact information | Basic contact data |
| IsActive | Whether supplier can be used |

Supplier management is included in v1 at a basic level.

Full Procurement / Purchase Order management is out of scope for v1.

---

## 13. User / Role / Permission

Core roles:

~~~
SYSTEM_ADMIN
WAREHOUSE_MANAGER
WAREHOUSE_STAFF
~~~

Permissions are separated from roles so authorization can evolve without rewriting business logic.

Examples:

- Manage users
- Manage products
- Manage warehouses
- Create receipt
- Approve receipt
- Create issue
- Approve issue
- Create transfer
- Approve transfer
- Receive transfer
- Create adjustment
- Approve adjustment
- View inventory
- View movement history

The exact permission matrix will be defined during authorization design.

---

## 14. Audit Entry

Audit records answer:

- Who performed an operation?
- What operation was performed?
- When?
- Against which business object?
- What was the result?

Audit is complementary to Inventory Movement.

### Inventory Movement

Answers:

> What happened to stock?

### Audit Entry

Answers:

> Who did what and when?

Both are needed.

---

## 15. Workflow Pattern

The system intentionally reuses one common approval pattern.

### Common workflow

~~~
DRAFT
  |
  v
PENDING_APPROVAL
  |
  v
APPROVED
  |
  v
COMPLETED
~~~

Used by:

- Stock Receipt
- Stock Issue
- Stock Adjustment

### Transfer extension

Transfer adds logistics-specific states:

~~~
DRAFT
  |
PENDING_APPROVAL
  |
APPROVED
  |
IN_TRANSIT
  |
RECEIVING
  |
COMPLETED
~~~

If a quantity mismatch occurs:

~~~
RECEIVING
   |
   v
DISCREPANCY
   |
   v
MANAGER REVIEW
   |
   v
RESOLVED
   |
   v
COMPLETED
~~~

---

## 16. FEFO Domain Rule

FEFO is a domain operation, not merely a sorting function.

~~~
Eligible Stock
    |
    +--> Active
    +--> Not expired
    +--> Correct warehouse
    +--> Correct product
    |
    v
Sort by earliest expiry
    |
    v
Allocate requested quantity
    |
    v
Create movements
~~~

For products without expiry tracking, FEFO does not apply.

---

## 17. Concurrency and Integrity

The implementation must protect inventory from concurrent updates.

Example:

~~~
User A sees 50
User B sees 50

A issues 40
B issues 40
~~~

The system must not allow the resulting stock to become negative or otherwise inconsistent.

The application must validate availability against current persisted state as part of an atomic operation.

The exact EF Core / SQL Server concurrency mechanism will be decided during implementation.

---

## 18. Relationships Summary

~~~
Supplier
   |
   +---- StockReceipt
             |
             +---- StockReceiptItem ---- Product
                                      |
                                      +---- Batch

Product
   |
   +---- Batch
   |
   +---- Inventory

Warehouse
   |
   +---- Inventory
   +---- StockReceipt
   +---- StockIssue
   +---- StockTransfer (source)
   +---- StockTransfer (destination)
   +---- StockAdjustment

StockIssue
   |
   +---- StockIssueItem ---- Product
                           |
                           +---- Batch

StockTransfer
   |
   +---- StockTransferItem ---- Product
                              |
                              +---- Batch

All completed stock operations
             |
             v
    Inventory Movement

Important operations
             |
             v
       Audit Entry
~~~

---

## 19. Intentionally Out of Domain v1

- POS
- Customer ordering
- Accounting
- Full Procurement / Purchase Orders
- Recipe / BOM management
- Production Order management
- Inventory reservation
- Mobile application
- Microservices
- Kafka / event streaming
- Kubernetes
- AI / ML

These remain future or post-v1 scope.

---

## 20. Open Decisions Before ERD

The following should be resolved before final database design:

1. Unit-of-measure strategy: single base unit vs conversion.
2. Batch-number uniqueness and supplier relationships.
3. Exact rules for products without batch tracking.
4. Transfer discrepancy resolution and whether unresolved discrepancies can block completion indefinitely.
5. Exact permission matrix.
6. Soft-delete / archive rules for master data.
7. Inventory uniqueness constraints.
8. Exact concurrency strategy.
9. Whether receiving approval is mandatory in every scenario.
10. Whether Waste should remain separate or be represented as a specialized Issue.

These are intentionally open rather than invented.

---

## 21. Agreed Design Decisions

- Central Warehouse is a Warehouse with type CENTRAL.
- Product can opt into batch and expiry tracking.
- Expired stock remains physically recorded but is unavailable for normal issue.
- No manager override for expired stock in v1.
- Stock Issue requires approval.
- Stock Adjustment requires approval.
- Transfer requires approval and has an in-transit phase.
- Transfer tracks exact batches where batch tracking applies.
- Partial transfer receiving is allowed.
- Transfer quantity differences create a discrepancy requiring Manager handling.
- Inventory has On Hand and Available concepts.
- In Transit is represented by the transfer, not destination inventory.
- Reserved inventory is not part of v1.
- FEFO is enforced for expiry-managed products.
- Every material stock change creates an Inventory Movement.
- Important operations have audit information.
- Direct inventory editing is not allowed.
- Atomicity and concurrency are core inventory requirements.

---

## 22. Next Step

Before implementation:

~~~
Domain Model
    |
    v
Conceptual ERD
    |
    v
Validate ERD against workflows
    |
    v
SQL Server schema
    |
    v
EF Core model
~~~

Do not start database implementation until the domain model and ERD have been reviewed against the main business workflows.
