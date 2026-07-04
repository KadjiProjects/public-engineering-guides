# Module 06 — Relational Data & Caching (Azure SQL + Redis)

> **Week 6.** Your SQL Server instincts transfer — but purchasing models, connection behavior over networks, and horizontal-scale caching are where cloud diverges from on-prem. This module makes EF Core + Azure SQL production-grade and adds the cache layer that lets compute stay stateless.

---

## 1. The Azure SQL family — choose deliberately

| Offering | What it is | Choose when |
|---|---|---|
| **Azure SQL Database** | Single managed database (logical server is just a connection/auth namespace) | Default for new apps |
| **Elastic pool** | Shared resource pool across many DBs | Many small DBs with staggered peaks (SaaS-per-tenant) |
| **SQL Managed Instance** | Near-full SQL Server instance surface (Agent, cross-DB, CLR, Service Broker) | Lift & shift of a *real* instance |
| **SQL Server on VM** | You run it | Last resort: OS access, unsupported features |

**Purchasing models for SQL Database:**
- **vCore** (default choice): explicit CPU/memory; **General Purpose** (remote storage, cheapest prod), **Business Critical** (local SSD, in-memory OLTP, readable replica, lowest latency), **Hyperscale** (100 TB+, fast scale, snapshot backups, ≥0 readable replicas).
- **Serverless** (a GP compute option): auto-scales vCores within min/max, **auto-pauses** after idle (you pay storage only) — perfect for dev/test and spiky small apps; first connection after pause takes seconds and *will* fail without retry logic (see §3 — this is a feature of the lab, not a bug).
- **DTU** (legacy blended metric): still around; migrate thinking to vCore.
- **Free/dev offers** exist for the lab budget.

**Platform capabilities you should stop building yourself:** automated backups with PITR (point-in-time restore) + long-term retention; **zone-redundant** configurations; **failover groups** (multi-region, one listener endpoint — your DR story, module 12); auto-tuning (plan-regression detection); TDE at-rest by default; Ledger tables if tamper-evidence is needed.

## 2. Security posture (module 2 applied)

- **Entra-only authentication** on the server (disable SQL auth entirely where possible); admins via an Entra group, apps via **managed identity** contained users (`CREATE USER [app-mi] FROM EXTERNAL PROVIDER`), least-privilege roles.
- **Network:** disable public access → **private endpoint** (module 9). "Allow Azure services" is a lab-only convenience — call it out as such in reviews.
- Row-Level Security, Dynamic Data Masking, Always Encrypted (client-side, for the truly sensitive columns) — know they exist and what problem each solves.
- Auditing → Log Analytics; Microsoft Defender for SQL for vulnerability + threat detection.

## 3. Connections & resilience — where on-prem habits break

Cloud databases are reached over networks that *will* blip, throttle, or fail over. Non-negotiable patterns:

- **Transient-fault retry.** EF Core: `options.UseSqlServer(cs, sql => sql.EnableRetryOnFailure());` (execution strategy retries known-transient error codes; wrap multi-op logic via `CreateExecutionStrategy()` when you also manage explicit transactions). Dapper/raw ADO.NET: use `Microsoft.Data.SqlClient`'s configurable retry logic or Polly.
- **Connection pooling reality:** pools are per (process, connection string). Serverless auto-pause resume, failovers, and SKU scaling all drop connections — retry covers it. Watch pool exhaustion under load (`Timeout expired... max pool size`): fix by not holding contexts open, async all the way, and right-sizing `Max Pool Size`.
- **Timeouts are two different settings:** connect timeout (in the connection string) vs command timeout (`CommandTimeout` / EF `CommandTimeout(...)`). Know which one fired from the exception.
- **DbContext lifetime:** scoped per request (default) is right; for high-throughput use **`AddDbContextPool`**; never share a context across threads.
- **Migrations in the cloud:** don't run `Database.Migrate()` at app startup in multi-instance prod (races, long locks). Apply migrations as a **deployment step** (pipeline job running `dotnet ef database update` or SQL scripts / DbUp) with the expand→migrate→contract discipline for zero-downtime schema change (module 3/12).

## 4. Performance & operations

- **Query Store is on** — it's your flight recorder: top resource consumers, plan regressions, forced plans. Learn its portal blades (Query Performance Insight is the simplified view).
- Watch: DTU/vCore %, log IO, worker %, deadlocks, blocked queries (`sys.dm_tran_locks`, `sp_whoisactive` equivalent knowledge transfers directly).
- **Read scale-out:** Business Critical/Hyperscale replicas accept `ApplicationIntent=ReadOnly` connections — free read offloading; ensure your code tolerates slight replica lag.
- Scaling a database tier = online-ish but drops connections at the switch (retry again).
- Cost levers: serverless for spiky, reserved capacity for steady prod, Hyperscale named replicas for heavy read fan-out, elastic pools for fleets.

## 5. Azure Cache for Redis — the statelessness enabler

Why (module 3's ARR-affinity-off promise): session, output caching, distributed locks, rate limiting, hot lookups — all need shared fast state once you scale out.

- **Tiers:** Basic (dev) / Standard (replicated) / **Premium** (VNet/private endpoint, persistence, clustering) / Enterprise (Redis Inc features: RediSearch, JSON, active geo-replication). *Check current portfolio — the managed-Redis lineup has been evolving.*
- **.NET wiring:**

```csharp
builder.Services.AddStackExchangeRedisCache(o =>            // IDistributedCache
    o.Configuration = builder.Configuration["Redis:Conn"]);
builder.Services.AddStackExchangeRedisOutputCache(...);      // ASP.NET Core output caching
// Or raw: ConnectionMultiplexer.Connect(...) as a singleton — always a singleton.
```

- **HybridCache** (modern .NET): L1 in-memory + L2 Redis with stampede protection — prefer it over hand-rolled two-level caches.
- **Patterns & pitfalls:** cache-aside with explicit TTLs (no TTL = slow memory leak); stampede protection (HybridCache or locks); serialize compactly; the multiplexer is a singleton (creating per-request is the classic perf bug); handle `RedisConnectionException` with retry/backoff; eviction policy `allkeys-lru` for pure cache.
- Auth: Entra token-based auth for Redis exists (access policies) — prefer over access keys; private endpoint for prod.

## 6. Decision framework

New app → **SQL Database, vCore GP, zone-redundant, Entra-only, private endpoint, serverless in dev**. Instance-level legacy features → MI (Managed Instance). >4 TB or extreme log throughput → Hyperscale. Fleets of tenant DBs → elastic pools. Read-heavy → BC/Hyperscale replicas + Redis in front. Session/shared state → Redis from day one of scale-out. Postgres/MySQL: the same managed patterns exist (Flexible Server) — choose by team skill and ecosystem, not dogma.

## 7. Lab 06 (~3 h)

> [labs.md → Lab 06](../labs.md)

1. Bicep: logical server (Entra-only admin = you), serverless GP database (auto-pause 60 min), private-endpoint-ready but public for now with your IP firewall rule.
2. EF Core: model + migrations applied via CLI (not startup); `EnableRetryOnFailure`; MI contained user for the App Service app; connect passwordless end-to-end.
3. **Feel auto-pause:** let it pause, hit the API cold, watch the retry succeed on attempt 2 in App Insights dependency telemetry.
4. Load with `hey`, find your top query in Query Store, add the missing index via migration, re-measure.
5. Add Redis (smallest SKU) + HybridCache on the hottest read endpoint; measure P95 before/after; kill Redis and verify graceful degradation (cache-aside falls through to SQL).

## 8. Self-check

1. GP vs BC vs Hyperscale — one differentiator and one ideal workload each.
2. Serverless auto-pause: what breaks naively, and the two-part fix (code + expectation)?
3. Why is `Database.Migrate()` at startup dangerous in prod? What's the alternative pipeline?
4. Which retry mechanism belongs to EF Core, and what extra step when you use explicit transactions?
5. Failover groups vs zone redundancy — which failure does each cover?
6. Why must `ConnectionMultiplexer` be a singleton? What symptom appears when it isn't?
7. What is a cache stampede and which .NET primitive solves it now?

## 9. Explain out loud

- Explain to an on-prem DBA what they *stop* doing on Azure SQL (backups, patching, HA plumbing) and what they *start* watching (Query Store, vCore %, retry telemetry).
- Defend "Entra-only auth + private endpoint" to a dev who says "the firewall rule works fine."

## 10. Verify before you build

- Serverless tier auto-pause specifics & Hyperscale serverless status; free-offer terms.
- Managed Redis product lineup/tiers (actively evolving) & Entra-auth support details.
- Current EF Core version guidance for retry + `HybridCache` GA status.

*Next: [Module 07 — Cosmos DB & storage](07-data-cosmos-storage.md).*
