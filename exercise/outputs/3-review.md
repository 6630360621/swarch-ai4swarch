Agent: AI assistant using Copilot SDK in VS Code; Model: ไม่ระบุ

# Review ADR-0002

## Quality-attribute scenarios

1. **Checkout performance and availability** — Source: customer; stimulus: places orders during the 11.11 peak; environment: 200 orders/s while MailFast has a 50 emails/s limit, p95 1.5 s, p99 8 s, and about 0.5% failed calls; response: checkout accepts orders without waiting for email delivery; measure: `PlaceOrder` p95 ≤ 500 ms and no order fails because MailFast is unavailable. (Scenario: “Quality goals and constraints”; ADR: “Validation”.)
2. **Loyalty correctness and timeliness** — Source: order event or retry; stimulus: the same order is delivered more than once; environment: peak traffic or temporary consumer failure; response: points are credited once and processing recovers; measure: exactly one credit per order within 1 minute. (Scenario: “Quality goals and constraints”; ADR: “Decision” and “Risks and mitigations”.)
3. **Recovery from messaging failure** — Source: customer; stimulus: RabbitMQ or a post-order consumer becomes unavailable during order placement; environment: production traffic; response: checkout still succeeds and accepted work is processed after recovery; measure: no order fails due to email, loyalty, or analytics being down, and no accepted event is lost. (Scenario: “Quality goals and constraints”; ADR: “Decision” and “Open questions”.)

## Findings

| Type | Finding | Why it matters |
|---|---|---|
| **Risk** | The ADR says the queue buffers the gap between 200 orders/s and MailFast’s 50 emails/s, but it does not quantify or bound the resulting backlog. Assuming one confirmation email per order, backlog grows by about 150 emails/s while that peak is sustained. | Buffering does not increase MailFast throughput. A long peak can create significant delivery delays; the acceptable email delay and peak duration are not specified. |
| **Risk** | The ADR correctly raises the order/outbox consistency problem, but leaves its persistence mechanism unresolved. Durable RabbitMQ messages alone do not explain what happens if checkout cannot publish, or if the order succeeds but event publication fails. | Without a reliable handoff, an order may succeed without triggering its email, points, or analytics. The decision depends on resolving this before implementation. |
| **Sensitivity point** | The one-minute loyalty target depends on consumer throughput, queue backlog, retry delays, and recovery time, none of which is sized or bounded. | A durable queue can preserve work without ensuring that loyalty points are awarded within the required minute. |
| **Trade-off point** | Asynchronous processing protects checkout latency and reduces dependence on downstream availability, but delays post-order work and adds RabbitMQ operations and recovery responsibilities. | The ADR notes this trade-off, but email and analytics have no stated delivery-time targets, making the acceptable delay unclear. |
| **Non-risk** | Separate consumers and an `order_id` idempotency key are reasonable proposals for future post-order work and duplicate delivery. | They support modifiability and duplicate-safe loyalty processing, provided points state is updated atomically as the ADR requires. |

## Missing or weak

- The synchronous option overstates the availability downside: the scenario says a failed `emailservice` call is logged as a warning and the order still succeeds. Synchronous calls may still threaten the 500 ms p95 target, but the ADR should distinguish latency risk from checkout failure.
- The RabbitMQ proposal does not specify the publish/acceptance behavior needed to meet “an order must never fail” during a broker outage while also avoiding lost post-order work. The outbox is identified as an unresolved possibility, not a settled design.
- Validation says to confirm the backlog drains, but does not define peak duration, acceptable email delay, or a backlog-drain target. Loyalty consumer capacity and recovery time also need measurable limits against the one-minute requirement.
- The scenario requires recording an analytics event, but its destination and delivery expectations are not established; the ADR appropriately lists these as open questions.

## Verdict

**Accept with changes.** Before implementation:

1. Specify and validate a reliable order-to-event handoff, including behavior when RabbitMQ is unavailable.
2. Define peak duration, acceptable email delay, and backlog capacity/drain targets; verify the one-minute loyalty objective under peak load and recovery.
3. Revise the synchronous-option availability comparison to reflect the scenario’s existing email-failure behavior, while retaining the latency concern.
