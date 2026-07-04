# Module 07 — Cosmos DB & Storage

> **Week 7.** Two different "NoSQL" stories: Cosmos DB (a distributed database whose *partitioning model* is the whole game) and Blob Storage (the object store that replaces every `File.WriteAllBytes` in your legacy code). Experts are made or broken on partition keys and RU math — this module makes those instincts real.

---

## Part A — Cosmos DB

## 1. The model that everything hangs on

- Hierarchy: **account → database → container → items**. Throughput and partitioning live at the **container**.
- **Logical partition** = all items sharing a partition-key value (hard cap ~20 GB per logical partition). **Physical partitions** hold many logical ones; **RU/s are divided evenly across physical partitions** — this is why hot partition keys destroy you: a container with 10 physical partitions and 10 000 RU/s gives each partition only 1 000 RU/s, and one hot key can throttle while nine partitions idle.
- **Request Units (RU)** = the currency: 1 RU ≈ a 1 KB point read. Writes cost ~5–10×; queries cost by work done (check `RequestCharge` on every response). Capacity modes: **provisioned** (with **autoscale**: floats between 10%–100% of a max), **serverless** (per-request, small/spiky workloads), and reserved capacity for steady prod.

**Partition key selection — the expert skill:**
1. High cardinality (many distinct values), 2) even distribution of *requests* (not just data), 3) matches your dominant query's filter (point reads and single-partition queries are cheap; **cross-partition fan-out queries** are where money goes to die).
- Patterns: `tenantId` (beware whale tenants), composite/synthetic keys (`tenantId_yyyymm`), **hierarchical partition keys** (up to 3 levels, e.g. `/tenantId/userId`) to break the 20 GB ceiling while keeping locality.
- Model for the queries: denormalize; embed what you read together, reference what changes independently; store multiple entity types in one container discriminated by a `type` property (container ≠ table).

## 2. Consistency — the interview classic, done practically

Five levels: **Strong → Bounded Staleness → Session → Consistent Prefix → Eventual**. Default = **Session** ("read your own writes" within your session token) — right for almost all apps. Strong costs latency/availability (and multi-region write constraints); Eventual buys throughput for feeds/counters. The senior line: name PACELC, then say *which operations* need which level — you can weaken consistency **per request**, not just per account.

## 3. The .NET SDK, production-grade

```csharp
// Singleton — always. Direct mode is default and right.
var client = new CosmosClient(accountEndpoint, tokenCredential /* Entra RBAC data plane! */,
    new CosmosClientOptions {
        ApplicationName = "cloudboard",
        EnableContentResponseOnWrite = false,     // save RUs/bandwidth on writes
    });
var container = client.GetContainer("appdb", "boards");

// Point read — the cheapest operation in Cosmos. Prefer it religiously.
var item = await container.ReadItemAsync<Board>(id, new PartitionKey(tenantId));

// Query — always scope to a partition when you can:
var q = new QueryDefinition("SELECT * FROM c WHERE c.type='card' AND c.boardId=@b")
        .WithParameter("@b", boardId);
using var it = container.GetItemQueryIterator<Card>(q,
        requestOptions: new QueryRequestOptions { PartitionKey = new(tenantId) });
```

- **Handle 429 (throttling):** SDK retries automatically (`MaxRetryAttemptsOnRateLimitedRequests`); sustained 429 = wrong RU budget or hot partition — fix the design, not the retry count.
- **Optimistic concurrency:** ETags + `IfMatchEtag` — the Cosmos version of a rowversion.
- **Transactions:** `TransactionalBatch` — atomic, but only **within one logical partition** (another reason keys matter). Stored procedures likewise.
- **Change feed** = ordered log of changes per partition → the integration goldmine: materialized views, cache invalidation, event sourcing-ish projections. Consume via Functions Cosmos trigger or the change-feed processor library (lease container tracks progress).
- **Multi-region:** add read regions with a click; multi-region *write* brings conflict resolution (LWW or custom) — adopt only with a real requirement.
- **Indexing:** everything is indexed by default; *excluding* paths (`/payload/*`) and adding composite indexes (for ORDER BY pairs) is the tuning dial — check `RequestCharge` before/after.

## Part B — Storage (Blob and friends)

## 4. Blob storage — the workhorse

- Account (**Standard GPv2** or **Premium block blob** for latency-sensitive) → containers → blobs (block/append/page). **Data Lake Gen2** = blob + hierarchical namespace for analytics estates.
- **Access tiers:** Hot / Cool / Cold / Archive (rehydration hours!) + lifecycle management rules ("move to Cool after 30 days, delete after 365") — cost engineering as config.
- **Redundancy:** LRS → ZRS → GRS/GZRS (+ RA- variants for readable secondaries). Match to RPO/compliance (module 12).
- **Concurrency:** ETags again (`If-Match`) + blob **leases** (the poor man's distributed lock).
- **Data protection:** soft delete, versioning, immutability (WORM) policies, point-in-time restore — turn on what the data's risk profile demands.
- **Events:** Blob-created events → **Event Grid** → Functions: the canonical "process uploaded file" pipeline (no polling; module 8).

## 5. .NET patterns & secure access

```csharp
var blobService = new BlobServiceClient(new Uri("https://acct.blob.core.windows.net"), credential);
var cont = blobService.GetBlobContainerClient("uploads");
await cont.GetBlobClient($"invoices/{id}.pdf")
          .UploadAsync(stream, new BlobUploadOptions {
              HttpHeaders = new BlobHttpHeaders { ContentType = "application/pdf" },
              Conditions = new BlobRequestConditions { IfNoneMatch = ETag.All } // no overwrite
          });
```

- Large files: the SDK chunks automatically (`StorageTransferOptions` to tune parallelism).
- **Browser/mobile upload/download pattern:** never proxy bytes through your API — issue a short-lived **user-delegation SAS** (signed with your Entra credential, not the account key) scoped to one blob + minutes of validity; client talks to storage directly.
- Auth ladder recap (module 2): MI + RBAC (`Storage Blob Data *` roles) → user-delegation SAS for delegation → account keys disabled by policy.
- Queues/Files/Tables in one line each: **Storage Queues** = simple, cheap queueing (vs Service Bus, module 8); **Azure Files** = SMB/NFS share (lift-and-shift file-share code); **Tables** = cheap key-value (Cosmos Table API is its bigger sibling).

## 6. Decision framework

Relational integrity/reporting → SQL (module 6). Global low-latency, flexible schema, known access patterns → Cosmos (design the key first, then say yes). Objects/files/media/backups → Blob + lifecycle. "Just a queue" → Storage Queue; enterprise semantics → Service Bus. Cosmos serverless for small event stores; autoscale for spiky prod; single-partition queries as a design KPI (log cross-partition query counts).

## 7. Lab 07 (~3 h)

> [labs.md → Lab 07](../labs.md)

1. Cosmos (serverless) via Bicep; container `boards` with `/tenantId`. Seed 10k items across 3 synthetic tenants where tenant-A gets 80% of writes.
2. Measure `RequestCharge`: point read vs single-partition query vs cross-partition query — build the table yourself; feel the 10–100× spread.
3. Force 429s: drop autoscale max (or serverless burst) + parallel writers to the hot tenant; watch the SDK retry, then *fix it properly* with a composite key (`tenantId_bucket`) and re-measure.
4. Change feed → Function maintaining a `cardCountByBoard` materialized view; verify it heals after you stop/start the processor.
5. Blob: upload via API with MI; then issue a user-delegation SAS and upload from `curl` directly; add a lifecycle rule (Cool at 30 d) and an Event Grid → Function on blob-created.

## 8. Self-check

1. Why does RU/s divide across physical partitions, and what symptom does a hot key produce even when total RU/s looks sufficient?
2. Three properties of a good partition key; give a synthetic-key example for a multi-tenant SaaS with whale tenants.
3. Session consistency: what does it guarantee, to whom, and via what token?
4. When is `TransactionalBatch` unusable, structurally?
5. What is the change feed *not* (hint: deletes, ordering across partitions), and how do you handle deletes with it?
6. User-delegation SAS vs account-key SAS — two concrete security wins.
7. Archive tier: what's the retrieval story and where does it bite naive lifecycle rules?

## 9. Explain out loud

- Teach partition keys with the "supermarket checkout lanes" analogy — including what a hot key is and why adding lanes (RU/s) doesn't shorten *one* queue.
- Explain to a reviewer why the API issues SAS tokens instead of streaming uploads through itself.

## 10. Verify before you build

- Hierarchical partition keys and per-partition throughput features (evolving).
- Cosmos serverless limits (container size/RU burst) and autoscale ratio.
- Cold tier + Archive rehydration SLAs; current redundancy conversion support.

*Next: [Module 08 — Messaging & eventing](08-messaging.md).*
