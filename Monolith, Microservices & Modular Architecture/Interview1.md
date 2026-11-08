# System Design Interview Question

## Q. Which Module Do You Extract First from a Monolith Without Taking Down Production?

### Answer

I will extract the **Payment module first** because it handles sensitive customer card information. Payment needs stronger security and compliance.

I will move it to a separate secure environment. If the payment gateway fails, it should not affect the entire application.

I will use a **Circuit Breaker** to stop waiting for a slow payment gateway. Users should still be able to browse products even if payment is unavailable.

Before extraction, I will create clear APIs and separate database ownership. I will migrate gradually without stopping the production application.

### Key Concepts to Remember

| Concept | Simple Explanation |
|---|---|
| PCI-DSS | Security standards for protecting card payment information. |
| Blast Radius | Limiting the impact of one service failure on the entire application. |
| Circuit Breaker | Stops sending requests to a slow or failing service temporarily. |
| Clean API | Provides clear communication between services. |
| Database Ownership | Each service manages its own data. |
| Strangler Fig Pattern | Gradually replacing monolith functionality with microservices without stopping production. |

### Interview Tips

- Explain why you selected Payment first.
- Explain how you will isolate payment failures.
- Explain how you will maintain application availability.
- Mention gradual migration using the Strangler Fig Pattern.

**Main takeaway:** Extract the module based on security, compliance, dependencies, and failure isolation — not just traffic.
