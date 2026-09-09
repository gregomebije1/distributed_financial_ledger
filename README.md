# Asynchronous Checkout & Ledger System (Dual-Entry Accounting)

An event-driven microservice system designed to handle complex, multi-party checkout transactions (e.g., deducting customer balance, crediting merchants, and holding platform fees). The architecture enforces strict financial ledger principles alongside predictable multi-service rollbacks.

## 🚀 Overview

This engine ensures compliance with dual-entry accounting standards under high concurrency while avoiding heavy distributed transactions (X/Open XA). It relies on workflow orchestration to maintain data consistency across distributed boundaries.

### Tech Stack
*   **Runtime/Framework:** Java, Spring Cloud
*   **Orchestration:** Temporal.io Java SDK
*   **Database:** PostgreSQL (with transactional isolation)
*   **Fault Tolerance:** Resilience4j (Circuit Breakers, Retries)

---

## 🏗️ Architecture & Key Standards

### 1. Orchestrated Saga Pattern
Distributed consistency is maintained asynchronously across microservices using a centralized orchestrator rather than distributed locking.
*   **Coordination:** A central coordinator manages workflow states between the **Order Service**, **Ledger Service**, and **Merchant Settlement Service**.
*   **Compensation:** If a downstream service fails, the orchestrator triggers reverse operations sequentially to undo completed steps.

### 2. Double-Entry Ledger Engine
To ensure compliance and auditability, all internal financial updates are treated as immutable rows following standard ledger constraints:
$$\sum \text{Debits} - \sum \text{Credits} = 0$$

```
   [Checkout Workflow] 
            │
            ▼
   ┌─────────────────────────────────┐
   │ Temporal.io / Saga Orchestrator │
   └────────┬────────────────────────┘
            │
            ├─► [Order Service] ─── (Reserve Inventory / Set PENDING)
            │
            ├─► [Ledger Service] ── (Balanced Multi-Party Journal Entry)
            │
            └─► [Merchant Service]  (Execute Payout / Settlement)

```

## 🛠️ Core Requirements

### Double-Entry Engine
*   Implements an **immutable database schema** for ledger lines. 
*   Every journal entry must strictly balance out to zero across debits and credits before committing.

### Concurrency Protection
*   Protects account balances against race conditions (e.g., concurrent withdrawals causing negative balances).
*   Enforces concurrency guardrails using either **PostgreSQL optimistic locking (`@Version`)** or **Redis distributed lock primitives (Redlock)**.

### Failure Handling & Compensation
*   If the *Merchant Settlement Service* rejects a payout (e.g., due to account restrictions), the Saga orchestrator automatically triggers compensating actions.
*   **Compensating Step:** Credits the funds back to the user's ledger account and transitions the original order state to `FAILED`.

---

## 🧪 Testing Strategy

### 1. Unit Testing
*   **Ledger Validation:** Validates that balance updates with mismatching debits and credits throw an explicit `UnbalancedLedgerException` entirely within memory, preventing any database traffic.

### 2. Saga Failure Mode Simulation
*   Uses Temporal's native testing framework (`TestWorkflowEnvironment`) to inject synthetic network delays or explicitly throw a `RuntimeFailure` within the *Merchant Settlement Service*.
*   **Verification:** Asserts that all reverse compensating steps trigger in strict reverse order, leaving system funds in their exact pre-transaction state.

### 3. High-Concurrency Race Condition Testing
*   **Execution:** Leverages Java's `CountDownLatch` and `ExecutorService` to fire **50 parallel threads** attempting to deduct `$10.00` simultaneously from a single account initialized with a `$100.00` balance.
*   **Success Criteria:** 
    *   Exactly **10 threads** complete successfully (reducing the absolute balance to `$0.00`).
    *   **40 threads** fail safely, throwing either an `InsufficientFundsException` or an optimistic locking collision exception.
    *   No data drift or negative balances occur.

### 4. Database Ledger Integrity Verification
*   Executes systematic validation checks across all rows in the journal table to ensure zero variance across the ecosystem.

```sql
-- Must evaluate to exactly 0 to pass audit verification
SELECT SUM(amount) FROM journal_entries;
```
