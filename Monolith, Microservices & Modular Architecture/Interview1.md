# System Design Interview Question

## Q1. Which Module Do You Extract First from a Monolith Without Taking Down Production?

### Answer

I will extract the **Payment module first** because it handles sensitive customer card information. Payment needs stronger security and compliance.

I will move it to a separate secure environment. If the payment gateway fails, it should not affect the entire application.

I will use a **Circuit Breaker** to stop waiting for a slow payment gateway. Users should still be able to browse products even if payment is unavailable.

Before extraction, I will create clear APIs and separate database ownership. I will migrate gradually without stopping the production application.

### Key Concepts to Remember

| Concept | Simple Explanation |
|---|---|
| PCI-DSS | Security standards for protecting card payment information. |
| Blast Radius | Limiting the impact of one service failure. |
| Circuit Breaker | Stops requests to a slow or failing service temporarily. |
| Clean API | Clear communication between services. |
| Database Ownership | Each service manages its own data. |
| Strangler Fig Pattern | Gradually replacing monolith functionality with microservices. |

---

# Follow-up Questions and Answers

## Q2. Why not extract User or Catalog service first?

**Answer:**

User and Catalog already work efficiently inside the monolith using local calls and caching. Extracting them creates network latency.

Payment has higher security and compliance requirements, so I would extract Payment first.

## Q3. What happens when the payment gateway times out?

**Answer:**

I will mark the payment as PENDING_VERIFICATION instead of immediately marking it failed.

I will use an Idempotency Key to prevent duplicate payments and use webhooks or background reconciliation to check the final payment status.

## Q4. How do you keep card data out of the monolith?

**Answer:**

I will use payment provider hosted fields through a secure iframe.

Card details will go directly to the payment provider. The provider will return a token, and the monolith will store only that token instead of actual card details.

## Q5. Why use HTTP instead of Kafka for checkout payments?

**Answer:**

Checkout needs an immediate response, so I will use synchronous HTTP with a timeout and Circuit Breaker.

I will use Kafka for background tasks like email receipts, ledger updates, and payment settlement.

## Q6. How do you test and monitor the Circuit Breaker?

**Answer:**

I will use Toxiproxy to simulate slow payment gateway responses during testing.

In production, I will use Prometheus to monitor failures, response time, and Circuit Breaker status.

## Q7. How do you handle database joins between Orders and Payments?

**Answer:**

I will avoid direct database joins between microservices.

Instead, I will store the Payment Reference ID in the Orders table. The BFF can call both APIs and combine the data.

I can also use the Transactional Outbox Pattern to publish payment status events reliably.

---

# Important Concepts for Interview Revision

| Concept | Simple Meaning |
|---|---|
| Idempotency | Prevents duplicate payment processing. |
| Webhook | Notification from an external service. |
| Tokenization | Replaces sensitive card data with a token. |
| Kafka | Asynchronous message processing. |
| Toxiproxy | Simulates network failures and delays. |
| Prometheus | Monitors application metrics. |
| BFF | Combines data from multiple backend services. |
| Transactional Outbox | Reliably publishes events after database transactions. |
| Saga Pattern | Manages transactions across multiple services. |
| Zero Downtime | Deploying changes without stopping the application. |

---

## What Interviewers Also Listen For

- **Trade-offs:** Why did you select Payment first?
- **Security:** How will you protect customer data?
- **Fault Tolerance:** Will other services work if Payment fails?
- **Data Consistency:** How will you manage data across services?
- **Observability:** How will you monitor production?
- **Rollback Strategy:** How will you return to the old system if migration fails?

**Main Takeaway:** Extract the module based on security, compliance, dependencies, and failure isolation — not just traffic.
