# Module 08 — Messaging & Eventing

> **Week 8.** Loose coupling is how systems survive their own success. The expert skill is threefold: pick the right primitive (Service Bus vs Event Grid vs Event Hubs vs Storage Queues), write consumers that survive production (idempotent, DLQ-aware), and name the patterns (outbox, saga, competing consumers) in design reviews.

---

## 1. The taxonomy — one sentence that sorts everything

> **A message carries intent ("do this") to a known handler; an event announces fact ("this happened") to whoever cares.** Commands → queues (Service Bus). Discrete events → pub/sub notification (Event Grid). Event *streams* (telemetry, logs, ordered histories) → Event Hubs.

| | **Service Bus** | **Event Grid** | **Event Hubs** | **Storage Queue** |
|---|---|---|---|---|
| Shape | Enterprise broker: queues + topics/subscriptions | Push-based event router (pub/sub) | Partitioned append log (Kafka-compatible) | Minimal queue |
| Semantics | Peek-lock, at-least-once, per-message settle | At-least-once push with retry + dead-letter to storage | Consumer-managed offsets (checkpointing) | Visibility timeout |
| Killer features | Sessions (FIFO per key), dedup, scheduled delivery, transactions, DLQ, filters/actions on subscriptions | Native Azure-resource events (blob created, resource written), CloudEvents, handlers = Functions/webhooks/queues | Massive throughput, replay, consumer groups, capture-to-storage | Cheap, huge, simple |
| Pick when | Work distribution, ordering, exactly-the-features above | React to things happening (esp. platform events), fan-out notifications | Streaming/analytics/ingestion at scale | Simple decoupling, no broker features needed |

**Tiers:** Service Bus Standard vs **Premium** (dedicated messaging units, predictable latency, private endpoints, larger messages) — Premium for prod. Event Hubs by TU/PU/capacity units; **Kafka endpoint** means Kafka clients work unchanged. Event Grid: system topics (platform events), custom topics, **namespaces** (adds MQTT + pull delivery).

## 2. Service Bus mechanics you must own

- **Peek-lock lifecycle:** receive → lock (renewable) → process → `Complete` (success) / `Abandon` (retry now) / `DeadLetter` (park) / lock expiry (retry later). **DeliveryCount** increments each redelivery; exceed **MaxDeliveryCount** → automatic dead-letter.
- **Dead-letter queue (DLQ):** every queue/subscription has one (`/$deadletterqueue`). It is not a trash can — it's a *work queue for humans/repair jobs*: monitor depth, inspect `DeadLetterReason`, build a re-submit path. "What's your DLQ story?" is a favorite senior interview probe.
- **Sessions:** FIFO + exclusive consumption per `SessionId` (e.g. per-order ordering with parallel orders). Required for FIFO; also enables session state.
- **Duplicate detection:** broker drops repeated `MessageId` within a window — *helps producers*; consumers still must be idempotent (redelivery after crash-post-processing isn't a duplicate send).
- **Topics/subscriptions:** pub/sub with per-subscription **filters** (SQL/correlation) — routing without code.
- Also: scheduled messages, deferral, auto-forwarding, batching.

## 3. .NET consumers that survive production

```csharp
// Registration (Microsoft.Extensions.Azure keeps clients singleton + MI-authenticated):
builder.Services.AddAzureClients(b => {
    b.AddServiceBusClientWithNamespace("cloudboard.servicebus.windows.net")
     .WithCredential(new DefaultAzureCredential());
});

// Processor pattern (long-running worker / BackgroundService):
var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions {
    MaxConcurrentCalls = 8,
    PrefetchCount = 16,
    AutoCompleteMessages = false          // settle explicitly, after success
});
processor.ProcessMessageAsync += async args => {
    var order = args.Message.Body.ToObjectFromJson<OrderMessage>();
    if (await _store.AlreadyProcessedAsync(args.Message.MessageId)) {   // idempotency gate
        await args.CompleteMessageAsync(args.Message); return;
    }
    try {
        await _handler.HandleAsync(order, args.CancellationToken);
        await args.CompleteMessageAsync(args.Message);
    }
    catch (TransientDependencyException) {
        await args.AbandonMessageAsync(args.Message);                    // redeliver
    }
    catch (PoisonMessageException ex) {
        await args.DeadLetterMessageAsync(args.Message,
            deadLetterReason: "Unprocessable", deadLetterErrorDescription: ex.Message);
    }
};
```

Functions' Service Bus trigger (module 4) wraps this; know what it does for you (settle-on-return, DLQ on repeated failure) and what it never can: **your idempotency and your poison/transient distinction.**

**Event Hubs consumer:** `EventProcessorClient` + a checkpoint blob container; checkpoint *after* processing a batch; rewind = replay (that's a feature — design consumers to tolerate it). Partition key choice mirrors Cosmos thinking (module 7).

## 4. The patterns to name (and implement)

- **Competing consumers:** N workers, one queue — free horizontal scale; pair with ACA/KEDA queue-length scaling (module 5) for pay-per-backlog.
- **Transactional outbox:** never do `SaveChangesAsync()` then `SendMessageAsync()` naively — a crash between them loses or duplicates. Write the event to an **outbox table in the same DB transaction**, then a relay (background worker / change feed) publishes it. This is *the* reliable-messaging answer.
- **Idempotent consumer:** processed-message-id table (or natural idempotency via upserts/ETags). At-least-once + idempotency ≈ effectively-once.
- **Saga / process manager:** distributed "transaction" as a sequence of local steps with **compensations** (Cancel ≠ Rollback). Durable Functions orchestrations (module 4) are a natural saga host.
- **Claim check:** big payload → Blob; message carries the pointer (respect broker size limits).
- **Retry with backoff + circuit breaker** on all dependencies — in .NET use `Microsoft.Extensions.Resilience` / Polly (Aspire's ServiceDefaults wires much of this, module 5).
- **Choreography vs orchestration:** events + reactions (autonomy, harder to see) vs a coordinator (visibility, coupling). Senior answer: orchestrate workflows with clear ownership; choreograph broad, loosely-interested audiences.

## 5. Event Grid specifics worth having ready

- Platform events without polling: blob created, resource group changed, subscription-level policy events → Functions/webhooks/Service Bus/Hubs as handlers.
- Delivery: at-least-once push with exponential retry up to 24 h, then **dead-letter to a storage blob you configure** (do configure it).
- Webhook handlers must pass the **validation handshake**; prefer Functions/Service Bus destinations to skip that pain.
- CloudEvents 1.0 schema = the interoperable choice for custom topics.

## 6. Decision framework

Ask in order: (1) Fact or intent? (2) Stream or discrete? (3) Which *broker features* does the workload literally require (ordering→sessions; dedup; scheduled; filtered fan-out; replay→Hubs)? (4) Throughput class? (5) Who consumes — code you own (queue/processor) or "anything subscribable" (Event Grid)? Then: Premium for prod Service Bus, private endpoints, MI auth everywhere, DLQ monitoring + runbook, outbox at every DB-write→publish seam.

## 7. Lab 08 (~3 h)

> [labs.md → Lab 08](../labs.md)

1. Bicep: Service Bus (Standard for lab), queue `orders` (dedup on, MaxDeliveryCount 5), topic `board-events` + two filtered subscriptions.
2. Implement the **outbox**: API writes SQL row + outbox row in one transaction; hosted relay publishes to the topic; kill the relay mid-stream and prove nothing is lost.
3. Idempotent consumer with the processed-ids table; send the same message 3× (dedup window off) and prove single effect.
4. Poison drill: message that always throws → watch DeliveryCount → DLQ; build the tiny "DLQ peek + resubmit" console tool; write the runbook lines.
5. Sessions: `SessionId = boardId`, two competing workers, prove per-board ordering with cross-board parallelism.
6. Event Grid: blob-created → Function (from lab 07) — compare its push latency to a polling design.

## 8. Self-check

1. Sort into SB/EG/EH: order-placed command; blob-uploaded notification; clickstream; per-device telemetry needing replay; email-send job.
2. Walk the peek-lock lifecycle including every settle verb and who calls it.
3. Why does duplicate detection at the broker *not* remove the need for consumer idempotency?
4. Sketch the outbox pattern's failure windows and why each is safe.
5. Sessions: what do they cost you in consumer parallelism, exactly?
6. Event Grid delivery guarantees and where undeliverable events end up.
7. Choreography vs orchestration — one system where each is clearly right.

## 9. Explain out loud

- Explain at-least-once delivery to a dev who "just wants exactly-once," ending at outbox + idempotent consumer.
- Present your DLQ runbook (detection → triage → resubmit → prevention) in two minutes.

## 10. Verify before you build

- Service Bus tier limits (message size by tier, partitioning status) and Premium MU guidance.
- Event Grid namespaces (MQTT/pull) capabilities if relevant.
- Event Hubs capacity model naming (TU/PU/CU) and Kafka compatibility surface.

*Next: [Module 09 — Networking](09-networking.md).*
